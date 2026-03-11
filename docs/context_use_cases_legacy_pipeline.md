# Pipeline Context Use Cases

## Overview

The pipeline `Context` class (`pipeline.infrastructure.launcher.Context`) is a single mutable state object used for an entire pipeline execution. It is simultaneously a session state container, a domain metadata container, a cross-stage communication channel, and a persistence unit.

This document catalogues the current use cases of the pipeline Context as determined by examination of the codebase. The goal is to support design and development of a prototype of a system which serves a similar role to that of the context for RADPS.

### Key implementation references

- `Context` / `Pipeline`: `pipeline/infrastructure/launcher.py`
- CLI lifecycle tasks: `pipeline/h/cli/h_init.py`, `pipeline/h/cli/h_save.py`, `pipeline/h/cli/h_resume.py`
- Task dispatch & result acceptance: `pipeline/h/cli/utils.py`, `pipeline/infrastructure/basetask.py`
- PPR-driven execution loops:
  - ALMA: `pipeline/infrastructure/executeppr.py` (used by `pipeline/runpipeline.py`)
  - VLA: `pipeline/infrastructure/executevlappr.py` (used by `pipeline/runvlapipeline.py`)
- XML procedure execution: `pipeline/recipereducer.py`
- MPI distribution: `pipeline/infrastructure/mpihelpers.py`
- QA framework: `pipeline/qa/`
- Weblog renderer: `pipeline/infrastructure/renderer/htmlrenderer.py`

---

## 1. Context Lifecycle

The canonical flow through the context is:

1. **Create session** — `h_init()` constructs a `launcher.Pipeline(...)` and returns a new `Context`. In PPR-driven execution, `executeppr()` or `executevlappr()` also populates project metadata at this point.
2. **Load data** — Import tasks (`h*_importdata`) attach datasets to the context's domain model (`context.observing_run`, measurement sets, scans, SPWs, etc.).
3. **Execute tasks** — Tasks execute against the in-memory context and return a `Results` object. After each task, `Results.accept(context)` records the outcome and mutates shared state.
4. **Accept results** — Inside `accept()`, results are merged via `Results.merge_with_context(context)`. A `ResultsProxy` is pickled to disk per-stage to keep the in-memory context bounded. The weblog is typically rendered after each top-level stage.
5. **Save / resume** — `h_save()` pickles the context; `h_resume(filename='last')` restores it. Driver-managed breakpoints and developer debugging workflows rely on this cycle.

---

## 2. Context Responsibility Overview

| # | Responsibility | Description | Examples / References |
|---|---|---|---|
| 1 | **Static Observation & Project Data** | Load, store, and provide access to static observation and project data and metadata in memory | `context.observing_run`, `context.project_summary`, `context.project_structure` |
| 2 | **Mutable Observation State** | Load, store, provide in-memory access, and update mutable dynamic observation data or metadata | MS registration, virtual SPW mappings, reference antenna ordering |
| 3 | **Path Management** | Specify and store output paths as part of configuration setup | `output_dir`, `products_dir`, `report_dir`, log paths |
| 4 | **Imaging State Management** | Manage imaging state across pipeline stages | `clean_list_pending`, `imaging_parameters`, masks, thresholds, `synthesized_beams` |
| 5 | **Calibration State Management** | Register, query, and update calibration state | Calibration library (`callibrary`), active/applied cal tables, interval trees |
| 6 | **Image Library Management** | Register and query image products across pipeline stages | `sciimlist`, `calimlist`, `rmsimlist`, `subimlist` |
| 7 | **Session Persistence** | Save and restore the full pipeline session | Pickle serialization, `h_save()`, `h_resume()`, `ResultsProxy` |
| 8 | **MPI / Parallel Distribution** | Pass context to parallel workers and merge results back | Context pickle broadcast to MPI servers; results merged on client |
| 9 | **Inter-Task Data Passing** | Accept task results and merge state back into the context | `merge_with_context()` pattern |
| 10 | **Stage Tracking & Result Accumulation** | Track execution progress, stage numbering, accumulated results | `context.results`, `stage_number`, `task_counter`, result proxies |
| 11 | **Reporting & Export Support** | Provide context data for weblog, QA reports, AQUA XML, and product packaging | `context.observing_run` for weblog, `context.project_structure` for archive labels |
| 12 | **QA Score Storage** | Store and provide access to QA scores | QA score objects appended to `result.qa.pool` |
| 13 | **Debuggability / Inspectability** | Context state must be human-readable and inspectable for post-mortem analysis | Per-stage tracebacks, timings, timetracker integration |
| 14 | **Telescope-Specific State** | Sub-context used only by telescope-specific code | `context.evla` (VLA), conditionally created |
| 15 | **Lifecycle Notifications** | Emit events at key lifecycle points | Event bus: `ResultAcceptingEvent`, `ContextCreated`, `TaskStarted`, `TaskComplete` |

---

## 3. Use Cases

Each use case describes a need that the pipeline's central state management must satisfy. They are written to be implementation-neutral in their core description, with implementation notes appended where the codebase provides important detail.

---

### UC-01 — Load and Provide Access to Observation Metadata
*Responsibilities: 1, 2*

| Field | Content |
|-------|---------|
| **Actor(s)** | Data import task, any downstream task, heuristics, renderers, QA handlers |
| **Summary** | The system must load observation metadata (datasets, spectral windows, fields, antennas, scans, time ranges) and make it queryable by all subsequent processing steps. It must also provide a unified identifier scheme when multiple datasets use different native numbering. |
| **Postconditions** | All registered datasets are queryable by name, type, or internal identifier without re-reading raw data from disk. |

**Implementation notes** — `context.observing_run` is the single most heavily queried context facet:

- `context.observing_run.get_ms(name=vis)` — resolve an MS by filename
- `context.observing_run.measurement_sets` — iterate all registered MS objects
- `context.observing_run.get_measurement_sets_of_type(dtypes)` — filter by data type (RAW, REGCAL_CONTLINE_ALL, BASELINED, etc.)
- `context.observing_run.virtual2real_spw_id(vspw, ms)` / `real2virtual_spw_id(...)` — translate between abstract pipeline SPW IDs and CASA-native IDs
- `context.observing_run.virtual_science_spw_ids` — virtual SPW catalog
- `context.observing_run.ms_reduction_group` — per-group reduction metadata (single-dish)
- Provenance fields: `.start_datetime`, `.end_datetime`, `.project_ids`, `.schedblock_ids`, `.execblock_ids`, `.observers`

MS objects are rich domain objects carrying scans, fields, SPWs, antennas, reference antenna ordering, etc. Tasks read per-MS state like `ms.reference_antenna`, `ms.session`, `ms.start_time`, `ms.origin_ms`.

---

### UC-02 — Store and Provide Project-Level Metadata
*Responsibilities: 1*

| Field | Content |
|-------|---------|
| **Actor(s)** | Initialization, any task, report generators |
| **Summary** | The system must store project-level metadata (proposal code, PI, telescope, desired sensitivities, processing recipe, OUS identifiers) and make it available to tasks for decision-making and to report generators for labelling outputs. |
| **Postconditions** | Project metadata is available for the lifetime of the processing session. |

**Implementation notes** — project metadata is typically set once at session start and read many times:

- `context.project_summary = project.ProjectSummary(...)` — set by `executeppr()` / `executevlappr()`
- `context.project_structure = project.ProjectStructure(...)` — set by PPR executors
- `context.project_performance_parameters` — performance parameters from the PPR
- `context.set_state('ProjectStructure', 'recipe_name', value)` — used by `recipereducer.reduce()` and SD heuristics
- `context.processing_intents` — set by `Pipeline` during initialization

This is a strong candidate for a separate, immutable-after-init sub-record in any future context schema.

---

### UC-03 — Manage Execution Paths and Output Locations
*Responsibilities: 3*

| Field | Content |
|-------|---------|
| **Actor(s)** | Initialization, any task, report generators, export code |
| **Summary** | The system must centrally define and provide working directories, report directories, product directories, and logical filenames for logs, scripts, and reports. Tasks resolve file paths through these centrally managed locations. On session restore, paths must be overridable to adapt to a new environment. |
| **Postconditions** | All tasks share a consistent set of paths for inputs and outputs. |

**Implementation notes:**

- Path roots: `output_dir`, `report_dir`, `products_dir`
- Context name drives deterministic, named run directories
- Relocation semantics are supported for results proxies (basenames stored) and common output layout
- PPR-driven execution may derive paths from environment variables (e.g., `SCIPIPE_ROOTDIR`)

---

### UC-04 — Register and Query Calibration State
*Responsibilities: 5*

| Field | Content |
|-------|---------|
| **Actor(s)** | Calibration tasks (bandpass, gaincal, applycal, polcal, selfcal, restoredata), heuristics, importdata, mstransform, uvcontsub |
| **Summary** | The system must allow calibration tasks to register solutions (indexed by data selection: field, spectral window, antenna, intent, time interval), and allow downstream tasks to query for all calibrations applicable to a given data selection. It must distinguish between calibrations pending application and those already applied. Registration must support transactional multi-entry updates — tasks often register multiple calibrations atomically within a single result acceptance. |
| **Postconditions** | Calibration state is queryable and correctly scoped to data selections. |

**Implementation notes** — `context.callibrary` is the primary cross-stage communication channel for calibration workflows:

- **Write:** `context.callibrary.add(calto, calfrom)` — register a calibration application (cal table + target selection); `context.callibrary.unregister_calibrations(matcher)` — remove by predicate
- **Read:** `context.callibrary.active.get_caltable(caltypes=...)` — list active cal tables; `context.callibrary.get_calstate(calto)` — get full application state for a target selection
- Backed by `CalApplication` → `CalTo` / `CalFrom` objects with interval trees for efficient matching; append-mostly, ordered by registration time

---

### UC-05 — Accumulate Imaging State Across Multiple Steps
*Responsibilities: 4, 6*

| Field | Content |
|-------|---------|
| **Actor(s)** | Imaging-related tasks (planning, production, quality checking, self-calibration, export) |
| **Summary** | The system must allow imaging state — target lists, imaging parameters, masks, thresholds, sensitivity estimates, and produced image references — to be computed by one step and read or refined by later steps. Multiple steps may contribute to a progressively refined imaging configuration. The system must also maintain typed registries of produced images (science, calibrator, RMS, sub-product) with add/query semantics. |
| **Postconditions** | The accumulated imaging state reflects contributions from all completed imaging-related steps; all produced images are registered and queryable. |

**Implementation notes** — this is the most fragile part of the current context design. Attributes are added ad-hoc, there is no schema, and defensive `hasattr()` checks appear in the code:

| Attribute | Written by | Read by |
|---|---|---|
| `clean_list_pending` | `editimlist`, `makeimlist`, `findcont`, `makeimages` | `findcont`, `tclean`, `transformimagedata`, `uvcontsub`, `checkproductsize` |
| `clean_list_info` | `makeimlist`, `makeimages` | display/renderer code |
| `imaging_mode` | `editimlist` | `makermsimages`, `makecutoutimages`, `makeimages` |
| `imaging_parameters` | PPR / `editimlist` | `tclean`, `checkproductsize`, heuristics |
| `synthesized_beams` | `imageprecheck`, `tclean`, `checkproductsize`, `makeimlist`, `makeimages` | `checkproductsize`, heuristics |
| `size_mitigation_parameters` | `checkproductsize` | downstream stages |
| `selfcal_targets`, `selfcal_resources` | `selfcal` | `exportdata` |

Image libraries provide typed registries:

- `context.sciimlist.add_item(imageitem)` / `.get_imlist()` — science images
- `context.calimlist` — calibrator images
- `context.rmsimlist` — RMS images
- `context.subimlist` — sub-product images (cutouts, cubes)

A future design should formalize imaging state as a typed state machine or versioned configuration sub-document, and consider separating image *metadata* (tracked in context) from image *data* (stored in artifact store).

---

### UC-06 — Track Execution Progress and Stage History
*Responsibilities: 9, 10*

| Field | Content |
|-------|---------|
| **Actor(s)** | Workflow engine, any task, report generators, operators |
| **Summary** | The system must track which processing step is currently executing, assign a unique sequential identifier to each step, and maintain an ordered history of all completed steps and their outcomes. This history must be available for reporting, script generation, and resumption after interruption. Per-stage tracebacks and timings must be preserved. Stage numbering must remain coherent across resumes. |
| **Postconditions** | The full execution history is retrievable in order; the current step is identifiable. |

**Implementation notes:**

- `context.results` holds an ordered list of `ResultsProxy` objects (proxied to disk to bound memory)
- `context.stage_number` and `context.task_counter` track progress
- Timetracker integration provides per-stage timing data
- Results proxies store basenames for portability

---

### UC-07 — Propagate Task Outputs to Downstream Tasks
*Responsibilities: 9, 10*

| Field | Content |
|-------|---------|
| **Actor(s)** | Any task producing output that subsequent tasks depend on |
| **Summary** | When a task produces outputs that change the processing state (e.g., new calibrations, updated flag summaries, image products, revised parameters), the system must provide a mechanism for those outputs to become visible to all subsequent tasks that need them. The system must also record the output for later retrieval by reports and exports. |
| **Postconditions** | Downstream tasks see an updated view of the processing state; the output is recorded in the execution history. |

**Implementation notes** — there are two propagation mechanisms:

1. **Structured state merge** — `Results.merge_with_context(context)` updates calibration library, image libraries, and other typed state.
2. **Results-list walking** — tasks read `context.results` to find outputs from earlier stages. For example:
   - VLA tasks compute `stage_number` from `context.results[-1].read().stage_number + 1`
   - `vlassmasking` iterates `context.results[::-1]` to find the latest `MakeImagesResult`
   - Export/AQUA code reads `context.results[0]` and `context.results[-1]` for timestamps

The results-list walking pattern is fragile (indices shift if stages are inserted/skipped), slow (requires unpickling), and implicit (no declared dependency). A future design should provide explicit stage-to-stage data dependencies.

---

### UC-08 — Support Multiple Orchestration Drivers
*Responsibilities: 9, 10*

| Field | Content |
|-------|---------|
| **Actor(s)** | Operations / automated processing (PPR-driven batch), pipeline developer / power user (interactive), recipe executor |
| **Summary** | The context is created and consumed by multiple front-ends: PPR command lists, XML procedures, or interactive task calls. The system must remain the stable state contract across these drivers. It must be createable and resumable from non-interactive and interactive drivers, support driver-injected run metadata, tolerate partial execution controls (`startstage`, `exitstage`) and breakpoint-driven stop/resume semantics, and provide machine-detectable success/failure signals. |
| **Postconditions** | The same context state is usable regardless of which orchestration driver created or resumed it. |

**Implementation notes** — two orchestration planes converge on the same task implementations:

- **Task-driven**: direct task calls via CLI wrappers in `pipeline/h/cli/`
- **Command-list-driven**: PPR and XML procedure commands via `executeppr.py` / `executevlappr.py` and `recipereducer.py`

They differ in how inputs are marshalled, how session paths are selected, and how resume is initiated, but the persisted context is the same.

---

### UC-09 — Save and Restore a Processing Session
*Responsibilities: 7*

| Field | Content |
|-------|---------|
| **Actor(s)** | Pipeline operator, workflow engine, developers |
| **Summary** | The system must be able to serialize the complete processing state to disk (all observation data, calibration state, execution history, imaging state, project metadata) and later restore it so that processing can resume from the saved point. The serialization must preserve enough state to resume within a compatible version window. On restore, paths must be adaptable to a new filesystem environment. |
| **Postconditions** | After restore, the system is in the same state as when saved; processing can continue. |

**Implementation notes:**

- `h_save()` pickles the whole context to `<context.name>.context`
- `h_resume(filename='last')` loads the most recent `.context` file
- Per-stage results are proxied to disk (`saved_state/result-stageX.pickle`) to keep the in-memory context smaller
- Used by driver-managed breakpoint/resume (`executeppr(..., bpaction='resume')`) and developer debugging workflows

---

### UC-10 — Provide State to Parallel Workers
*Responsibilities: 8*

| Field | Content |
|-------|---------|
| **Actor(s)** | Workflow engine, MPI worker processes |
| **Summary** | When work is distributed across parallel workers, each worker needs read-only access to the current processing state (observation metadata, calibration state, etc.). The system must provide a mechanism for workers to obtain a consistent snapshot of the state. Workers must not be able to modify the authoritative state directly. The snapshot must be small enough to broadcast efficiently. |
| **Postconditions** | Each worker has a consistent, read-only view of the processing state for the duration of its work. |

**Implementation notes** — `pipeline/infrastructure/mpihelpers.py`, class `Tier0PipelineTask`:

1. The MPI client saves the context to disk as a pickle: `context.save(path)`.
2. Task arguments are also pickled to disk alongside the context.
3. On the server, `get_executable()` loads the context, modifies `context.logs['casa_commands']` to a server-local temp path, creates the task's `Inputs(context, **task_args)`, then executes the task.
4. For `Tier0JobRequest` (lower-level distribution), the executor is shallow-copied *excluding* the context reference to stay within the MPI buffer limit (~150 MiB, per PIPE-1337).

---

### UC-11 — Aggregate Results from Parallel Workers
*Responsibilities: 8, 9*

| Field | Content |
|-------|---------|
| **Actor(s)** | Workflow engine |
| **Summary** | After parallel workers complete, the system must collect their individual results and incorporate them into the authoritative processing state. The aggregation must be safe (no conflicting concurrent writes) and complete before the next sequential step begins. |
| **Postconditions** | The processing state reflects the combined outcomes of all parallel workers. |

---

### UC-12 — Provide Data for Report Generation
*Responsibilities: 11, 12*

| Field | Content |
|-------|---------|
| **Actor(s)** | Report generators (weblog, quality reports, reproducibility scripts, AQUA reports, pipeline statistics) |
| **Summary** | The system must provide report generators with read-only access to: observation metadata, project metadata, execution history (including per-step outcomes and QA scores), log references, and path information. Reports include human-readable web pages, machine-readable quality reports, and reproducible processing scripts. |
| **Postconditions** | Reports accurately reflect the processing state at the time of generation. |

**Implementation notes** — `WebLogGenerator.render(context)` in `pipeline/infrastructure/renderer/htmlrenderer.py`:

- Reads `context.results` — unpickled from `ResultsProxy` objects, iterated for every renderer
- Reads `context.report_dir`, `context.output_dir` — filesystem layout
- Reads `context.observing_run.*` — MS metadata, scheduling blocks, execution blocks, observers, project IDs, start/end times
- Reads `context.project_summary.telescope` — to determine telescope-specific page layouts (ALMA vs VLA vs NRO)
- Reads `context.project_structure.*` — OUS IDs, PPR file, recipe name
- Reads `context.logs['casa_commands']` — CASA command history

The renderer iterates `context.results` multiple times (assigning to topics, extracting flags, building timelines). The current approach requires unpickling *every* result into memory, then re-proxying when done. A lazy or streaming model would reduce peak memory.

---

### UC-13 — Compute and Store Quality Assessments
*Responsibilities: 12*

| Field | Content |
|-------|---------|
| **Actor(s)** | QA scoring framework, report generators, downstream decision-making |
| **Summary** | After each processing step completes, the system must support evaluating the outcome against quality thresholds (which may depend on telescope, project parameters, or observation properties) and recording normalized quality scores. These scores must be retrievable for reporting and for downstream decision-making. |
| **Postconditions** | Quality scores are associated with the relevant processing step and accessible to reports and downstream logic. |

**Implementation notes** — after `merge_with_context()`, `accept()` triggers `pipelineqa.qa_registry.do_qa(context, result)`:

- QA handlers implement `QAPlugin.handle(context, result)`
- They typically call `context.observing_run.get_ms(vis)` to look up metadata for scoring (antenna count, channel count, SPW properties, field intents)
- Some handlers check `context.imaging_mode` to branch on VLASS-specific scoring
- Scores are appended to `result.qa.pool` — they don't mutate the context directly

QA handlers are *read-only* with respect to context and could operate on a frozen snapshot, making them a good candidate for parallelization.

---

### UC-14 — Support Interactive Inspection and Debugging
*Responsibilities: 13*

| Field | Content |
|-------|---------|
| **Actor(s)** | Pipeline developer, pipeline operator, CI harnesses |
| **Summary** | The system must allow an operator to inspect the current processing state: which datasets are registered, what calibrations exist, how many steps have completed, what their outcomes were. On failure, a snapshot of the state should be available for post-mortem analysis. The system must provide deterministic paths/outputs that a test harness can validate, and must surface failures beyond raw task exceptions (e.g., weblog rendering failures captured via timetracker). |
| **Postconditions** | The operator can understand the current state of processing and diagnose problems. |

---

### UC-15 — Isolate Telescope-Specific State
*Responsibilities: 14*

| Field | Content |
|-------|---------|
| **Actor(s)** | Telescope-specific tasks and heuristics |
| **Summary** | The system must support storing instrument-specific state (e.g., VLA-specific solution intervals or gain metadata) in a way that is accessible to telescope-specific tasks but does not pollute the state used by generic or other-telescope tasks. This state is created conditionally based on the instrument. |
| **Postconditions** | Telescope-specific state is available to the tasks that need it; absent when the instrument does not require it. |

**Implementation notes** — `context.evla` is a `collections.defaultdict(dict)`, keyed as `context.evla['msinfo'][ms_name].<property>`:

- **Written by:** `hifv_importdata` (creates + initializes), `testBPdcals` (gain intervals, ignorerefant), `fluxscale/solint`, `fluxboot`
- **Read by:** nearly every VLA calibration task and heuristic
- Accessed fields include: `gain_solint1`, `gain_solint2`, `setjy_results`, `ignorerefant`, various `*_field_select_string` / `*_scan_select_string` values, `fluxscale_sources`, `spindex_results`, and many more

This is a completely untyped, dictionary-of-dictionaries sidecar. A future design should define a typed state object, provide accessor methods rather than raw dict lookups, and separate telescope-specific concerns from the generic context via composition (e.g., `context.get_extension('evla')`).

---

### UC-16 — Package and Export Pipeline Products
*Responsibilities: 3, 11*

| Field | Content |
|-------|---------|
| **Actor(s)** | Export task, archive system |
| **Summary** | The system must provide an export mechanism that reads datasets, calibration state, image products, reports, scripts, and project identifiers from the processing state and assembles them into a deliverable product package. The package must be structured for downstream archive ingestion. |
| **Postconditions** | A self-contained product package exists on disk. |

---

### UC-17 — Emit Lifecycle Notifications
*Responsibilities: 15*

| Field | Content |
|-------|---------|
| **Actor(s)** | Workflow engine, event subscribers (loggers, progress monitors) |
| **Summary** | The system must emit notifications at key lifecycle points (session start, session restore, step start, step completion, result acceptance) so that external observers (logging, progress reporting, live dashboards) can track execution without polling. |
| **Postconditions** | Subscribers are notified of lifecycle transitions as they occur. |

**Implementation notes** — `pipeline.infrastructure.eventbus.send_message(event)`:

- Event types: `ResultAcceptingEvent`, `ContextCreated`, `TaskStarted`, `TaskComplete`
- The event bus exists and fires events, but is lightly used — `merge_with_context` remains the primary data flow mechanism
- A future design could elevate the event bus to the primary state mutation channel (event-sourcing pattern), enabling audit trails, undo, and distributed observation

---

## 4. Responsibility-to-Use-Case Traceability

| # | Responsibility | Use Cases |
|---|---|---|
| 1 | Static Observation & Project Data | UC-01, UC-02 |
| 2 | Mutable Observation State | UC-01 |
| 3 | Path Management | UC-03, UC-16 |
| 4 | Imaging State Management | UC-05 |
| 5 | Calibration State Management | UC-04 |
| 6 | Image Library Management | UC-05 |
| 7 | Session Persistence | UC-09 |
| 8 | MPI / Parallel Distribution | UC-10, UC-11 |
| 9 | Inter-Task Data Passing | UC-06, UC-07, UC-11 |
| 10 | Stage Tracking & Result Accumulation | UC-06, UC-07, UC-08 |
| 11 | Reporting & Export Support | UC-12, UC-16 |
| 12 | QA Score Storage | UC-13 |
| 13 | Debuggability / Inspectability | UC-14 |
| 14 | Telescope-Specific State | UC-15 |
| 15 | Lifecycle Notifications | UC-17 |

---

## 5. Use Cases the Current Design Cannot Handle

The following describe scenarios that the current context design *does not support* but that could be valuable in a future architecture.

### FUC-01 — Concurrent / Overlapping Task Execution

Today all task execution is strictly serial. The context is a mutable, shared-everything singleton with no locking or isolation between stages. Many calibration stages are independent per-MS or per-SPW and could benefit from parallelization.

**What would be needed:**

- A context that supports isolated read snapshots (like database transactions or copy-on-write)
- A merge/reconciliation step when concurrent results are accepted
- Explicit declaration of which context fields each task reads and writes

### FUC-02 — Cloud / Distributed Execution Without Shared Filesystem

The current context is a pickle file on a local/shared filesystem. MPI distribution requires all nodes to see the same filesystem.

**What would be needed:**

- A context store backed by a database or object store (S3, GCS)
- Artifact references rather than filesystem paths for cal tables and images
- Tasks that can operate on remote datasets without requiring local copies

### FUC-03 — Multi-Language / Multi-Framework Access to Context

The context is a Python object graph, tightly coupled to CASA's Python runtime. Non-Python clients (C++, Julia, JavaScript dashboards) cannot access it.

**What would be needed:**

- A language-neutral serialization format (Protocol Buffers, JSON-Schema, Arrow)
- A query API (REST, gRPC, or GraphQL)
- Type definitions shared across languages

### FUC-04 — Streaming / Incremental Processing

The current session model assumes all data is available at session start and cannot process data as it arrives from the correlator or archive.

**What would be needed:**

- A context that supports incremental dataset registration (add new scans/EBs to a live session)
- Tasks that can detect "new data available" and re-process incrementally
- A results model that supports versioning (re-run produces a new version rather than overwriting)

### FUC-05 — Provenance and Reproducibility Guarantees

There is no formal record of which context state a task observed when it ran. Re-running from a saved context yields the state *after* the last save, not the state live at stage N.

**What would be needed:**

- Immutable snapshots of context state per-stage (event sourcing)
- Hashing of all task inputs (context fields + parameters) for cache invalidation / reproducibility tokens
- Ability to replay a run from the event log

### FUC-06 — Fine-Grained Access Control / Multi-Tenant Context

The context is an all-or-nothing object with no concept of access control, multi-user sessions, or data isolation between projects.

**What would be needed:**

- Per-project context namespacing
- Role-based access to context fields
- Audit logging of all context mutations

### FUC-07 — Partial Re-Execution / Targeted Stage Re-Run

Today "resume" means "restart from the last completed stage". There is no way to selectively re-run a single mid-pipeline stage with different parameters while keeping earlier and later stages intact.

**What would be needed:**

- Explicit dependency tracking between stages (which context fields does stage N consume?)
- Ability to invalidate downstream stages when a mid-pipeline stage is re-run
- Versioned results per stage (keep old version alongside new version)

### FUC-08 — External System Integration (Archive, Scheduling, QA Dashboards)

Today, context is a local, in-process object. Other systems (archive ingest, scheduling database, QA dashboards) interact only via offline products (weblog, manifest, AQUA XML).

**What would be needed:**

- A stable, queryable context API (REST/gRPC) that external systems can poll or subscribe to
- Webhook / event notification support for state transitions
- A standard schema for context summaries consumable by external systems

---

## 6. Architectural Observations

### The context is a "big ball of state", by design

The current approach is extremely flexible for a long-running, stateful CASA session, but there is no explicit schema boundary between persisted state, ephemeral caches, runtime-only services, and large artifacts. Tasks can (and do) add new fields in an ad-hoc way over time.

### Persistence is pickle-based

Pickle works for short-lived resume/debug use cases, but it is fragile across version changes, risky as a long-term archive format, and not friendly to multi-writer or multi-process updates. The codebase mitigates size by proxying stage results to disk, but the context itself remains a potentially large and unstable object graph.

### Two orchestration planes converge on the same context

Task-driven (interactive CLI) and command-list-driven (PPR / XML procedures) execution both produce and consume the same context. They differ in how inputs are marshalled, how paths are selected, and how resume is initiated, but the persisted context is the same object.

---

## 7. Improvement Suggestions for Next-Generation Design

These are phrased as requirements and design directions, not as a call to rewrite everything immediately.

### 1) Split "context data" from "context services"

Define a minimal, explicit **ContextData** model that is typed, schema-versioned, and serializable in a stable format (JSON/MsgPack/Arrow). Attach runtime-only services (CASA tool handles, caches, heuristics engines) around it rather than mixing them into the same object.

### 2) Introduce a ContextStore interface

Replace "pickle a Python object graph" with a storage abstraction (`get`, `put`, `list_runs`). Backends can start simple (SQLite) and grow (Postgres/object store) without changing task logic.

### 3) Make state transitions explicit (event-sourced or patch-based)

The existing event bus (`pipeline.infrastructure.eventbus`) could be elevated to record task lifecycle events and key state changes, yielding reproducibility, easier partial rebuilds, and better distributed orchestration.

### 4) Treat large artifacts as references, not context fields

Store large arrays/images/tables in an artifact store and carry only references in context data. This avoids "accidentally pickle a GiB array" and makes distribution/cloud execution more realistic.

### 5) Remove reliance on global interactive stacks for non-interactive execution

Make tasks accept an explicit context handle. Keep interactive convenience wrappers but do not make them the core contract.

### 6) Represent the execution plan as context data

Record the effective execution plan (linear or DAG) alongside run state to support provenance, partial execution, and targeted re-runs.

### 7) Adopt a versioned compatibility policy

Define whether operational contexts must be resumable within a supported release window (with schema versioning + migrations) versus best-effort for development contexts.

---

## 8. Context Contract Summary

The following capabilities appear to be **hard requirements** for any replacement system, derived from current behavior and internal usage patterns:

**System-level requirements:**

- Run identity: `context_id`, recipe/procedure name, inputs, operator/mode
- Path layout: working/report/products directories with ability to relocate
- Dataset inventory: execution blocks / measurement sets with per-MS metadata
- Stage results timeline: ordered stages, durations, QA outcomes, tracebacks
- Export products: weblog tar, manifest, AQUA report, scripts
- Resume: restart from last known good stage (or after a breakpoint)

**Internal usage requirements:**

- Fast MS lookup: random-access by name, filtering by data type, virtual↔real SPW translation
- Calibration library: append-oriented, ordered, with transactional multi-entry updates and predicate-based queries
- Image library: four typed registries (science, calibrator, RMS, sub-product) with add/query semantics
- Imaging state: typed, versioned configuration for the imaging sub-pipeline
- QA scoring: read-only context snapshot for parallel-safe QA handler execution
- Weblog rendering: read-only traversal of full results timeline + MS metadata + project metadata
- MPI/distributed: efficient context snapshot broadcast + results write-back
- Cross-stage data flow: explicit named outputs rather than results-list walking
- Project metadata: immutable-after-init sub-record
- Telescope-specific state: typed, composable extension rather than untyped dict

---

## 9. Open Questions

- Are there additional use cases not captured by either review? Reviewer input may surface new cases.
- Should the future use cases (Section 5) be prioritized? If so, which are most impactful for RADPS?
- What compatibility guarantees should the next-generation context provide across pipeline releases?

---

## 10. Contributors

- **Kristin Berry** — Worked on this draft
- **Shawn Booth** — Worked on this draft
