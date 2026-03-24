# Pipeline Context: Supplementary Analysis

This document contains architectural observations, design recommendations, and reference material that supplement the use cases in [context_use_cases_current_pipeline.md](context_use_cases_current_pipeline.md). These sections were separated to keep the use-case document focused on requirements.

---

## Context Responsibility Overview

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

## Responsibility-to-Use-Case Traceability

| # | Responsibility | Use Cases |
|---|---|---|
| 1 | Static Observation & Project Data | UC-01, UC-02 |
| 2 | Mutable Observation State | UC-01 |
| 3 | Path Management | UC-03, UC-17 |
| 4 | Imaging State Management | UC-05 |
| 5 | Calibration State Management | UC-04 |
| 6 | Image Library Management | UC-06 |
| 7 | Session Persistence | UC-10 |
| 8 | MPI / Parallel Distribution | UC-11, UC-12 |
| 9 | Inter-Task Data Passing | UC-07, UC-08, UC-09, UC-12 |
| 10 | Stage Tracking & Result Accumulation | UC-07, UC-08, UC-09 |
| 11 | Reporting & Export Support | UC-13, UC-17 |
| 12 | QA Score Storage | UC-13, UC-14 |
| 13 | Debuggability / Inspectability | UC-15 |
| 14 | Telescope-Specific State | UC-16 |
| 15 | Lifecycle Notifications | UC-18 |

---

## Implementation Notes by Use Case

The following implementation notes describe how each use case is realized in the current pipeline codebase. They were separated from the use-case definitions to keep the requirements document focused on requirements.

### UC-01 — Load and Provide Access to Observation Metadata

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

**Implementation notes** — project metadata is typically set once at session start and read many times:

- `context.project_summary = project.ProjectSummary(...)` — set by `executeppr()` / `executevlappr()`
- `context.project_structure = project.ProjectStructure(...)` — set by PPR executors
- `context.project_performance_parameters` — performance parameters from the PPR
- `context.set_state('ProjectStructure', 'recipe_name', value)` — used by `recipereducer.reduce()` and SD heuristics
- `context.processing_intents` — set by `Pipeline` during initialization

This is a strong candidate for a separate, immutable-after-init sub-record in any future context schema.

---

### UC-03 — Manage Execution Paths and Output Locations

**Implementation notes:**

- Path roots: `output_dir`, `report_dir`, `products_dir`
- Context name drives deterministic, named run directories
- Relocation semantics are supported for results proxies (basenames stored) and common output layout
- PPR-driven execution may derive paths from environment variables (e.g., `SCIPIPE_ROOTDIR`)

---

### UC-04 — Register and Query Calibration State

**Implementation notes** — `context.callibrary` is the primary cross-stage communication channel for calibration workflows:

- **Write:** `context.callibrary.add(calto, calfrom)` — register a calibration application (cal table + target selection); `context.callibrary.unregister_calibrations(matcher)` — remove by predicate
- **Read:** `context.callibrary.active.get_caltable(caltypes=...)` — list active cal tables; `context.callibrary.get_calstate(calto)` — get full application state for a target selection
- Backed by `CalApplication` → `CalTo` / `CalFrom` objects with interval trees for efficient matching; append-mostly, ordered by registration time

---

### UC-05 — Accumulate Imaging State Across Multiple Steps

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

A future design should formalize imaging state as a typed state machine or versioned configuration sub-document, and consider separating image *metadata* (tracked in context) from image *data* (stored in artifact store).

---

### UC-06 — Register and Query Produced Image Products

**Implementation notes** — image libraries provide typed registries:

- `context.sciimlist.add_item(imageitem)` / `.get_imlist()` — science images
- `context.calimlist` — calibrator images
- `context.rmsimlist` — RMS images
- `context.subimlist` — sub-product images (cutouts, cubes)

---

### UC-07 — Track Execution Progress and Stage History

**Implementation notes:**

- `context.results` holds an ordered list of `ResultsProxy` objects (proxied to disk to bound memory)
- `context.stage_number` and `context.task_counter` track progress
- Timetracker integration provides per-stage timing data
- Results proxies store basenames for portability

---

### UC-08 — Propagate Task Outputs to Downstream Tasks

**Implementation notes** — the current pipeline satisfies these needs through two different propagation paths:

1. **Immediate state propagation** — `Results.merge_with_context(context)` updates calibration library, image libraries, and other typed state so later tasks can access the current processing state directly.
2. **Retained step-result access** — tasks read `context.results` to find outputs from earlier stages when those outputs are needed from the recorded execution history rather than from merged shared state. For example:
   - VLA tasks compute `stage_number` from `context.results[-1].read().stage_number + 1`
   - `vlassmasking` iterates `context.results[::-1]` to find the latest `MakeImagesResult`
   - Export/AQUA code reads `context.results[0]` and `context.results[-1]` for timestamps

The results-list walking pattern is fragile (indices shift if stages are inserted/skipped), slow (requires unpickling), and implicit (no declared dependency). A future design should provide explicit stage-to-stage data dependencies.

---

### UC-09 — Support Multiple Orchestration Drivers

**Implementation notes** — two orchestration planes converge on the same task implementations:

- **Task-driven**: direct task calls via CLI wrappers in `pipeline/h/cli/`
- **Command-list-driven**: PPR and XML procedure commands via `executeppr.py` / `executevlappr.py` and `recipereducer.py`

They differ in how inputs are marshalled, how session paths are selected, and how resume is initiated, but the persisted context is the same.

---

### UC-10 — Save and Restore a Processing Session

**Implementation notes:**

- `h_save()` pickles the whole context to `<context.name>.context`
- `h_resume(filename='last')` loads the most recent `.context` file
- Per-stage results are proxied to disk (`saved_state/result-stageX.pickle`) to keep the in-memory context smaller
- Used by driver-managed breakpoint/resume (`executeppr(..., bpaction='resume')`) and developer debugging workflows

---

### UC-11 — Provide State to Parallel Workers

**Implementation notes** — `pipeline/infrastructure/mpihelpers.py`, class `Tier0PipelineTask`:

1. The MPI client saves the context to disk as a pickle: `context.save(path)`.
2. Task arguments are also pickled to disk alongside the context.
3. On the server, `get_executable()` loads the context, modifies `context.logs['casa_commands']` to a server-local temp path, creates the task's `Inputs(context, **task_args)`, then executes the task.
4. For `Tier0JobRequest` (lower-level distribution), the executor is shallow-copied *excluding* the context reference to stay within the MPI buffer limit (~150 MiB, per PIPE-1337).

---

### UC-13 — Provide Read-Only Context for Reporting Consumers

**Implementation notes** — `WebLogGenerator.render(context)` in `pipeline/infrastructure/renderer/htmlrenderer.py`:

- Reads `context.results` — unpickled from `ResultsProxy` objects, iterated for every renderer
- Reads `context.report_dir`, `context.output_dir` — filesystem layout
- Reads `context.observing_run.*` — MS metadata, scheduling blocks, execution blocks, observers, project IDs, start/end times
- Reads `context.project_summary.telescope` — to determine telescope-specific page layouts (ALMA vs VLA vs NRO)
- Reads `context.project_structure.*` — OUS IDs, PPR file, recipe name
- Reads `context.logs['casa_commands']` — CASA command history

The renderer iterates `context.results` multiple times (assigning to topics, extracting flags, building timelines). The current approach requires unpickling *every* result into memory, then re-proxying when done. A lazy or streaming model would reduce peak memory.

---

### UC-14 — Support QA Evaluation and Store Quality Assessments

**Implementation notes** — after `merge_with_context()`, `accept()` triggers `pipelineqa.qa_registry.do_qa(context, result)`:

- QA handlers implement `QAPlugin.handle(context, result)`
- They typically call `context.observing_run.get_ms(vis)` to look up metadata for scoring (antenna count, channel count, SPW properties, field intents)
- Some handlers check `context.imaging_mode` to branch on VLASS-specific scoring
- Scores are appended to `result.qa.pool` — the context provides inputs to QA evaluation, but the scores are stored on the result rather than as direct context mutations

QA handlers are *read-only* with respect to context and could operate on a frozen snapshot, making them a good candidate for parallelization.

---

### UC-16 — Manage Telescope-Specific Context Extensions

**Implementation notes** — `context.evla` is a `collections.defaultdict(dict)`, keyed as `context.evla['msinfo'][ms_name].<property>`:

- **Written by:** `hifv_importdata` (creates + initializes), `testBPdcals` (gain intervals, ignorerefant), `fluxscale/solint`, `fluxboot`
- **Read by:** nearly every VLA calibration task and heuristic
- Accessed fields include: `gain_solint1`, `gain_solint2`, `setjy_results`, `ignorerefant`, various `*_field_select_string` / `*_scan_select_string` values, `fluxscale_sources`, `spindex_results`, and many more

This is a completely untyped, dictionary-of-dictionaries sidecar attached to the top-level context. A future design should define a typed state object, provide accessor methods rather than raw dict lookups, and separate telescope-specific concerns from the generic context via composition (e.g., `context.get_extension('evla')`).

---

### UC-18 — Emit Lifecycle Notifications

**Implementation notes** — `pipeline.infrastructure.eventbus.send_message(event)`:

- Event types: `ResultAcceptingEvent`, `ContextCreated`, `TaskStarted`, `TaskComplete`
- The event bus exists and fires events, but is lightly used — `merge_with_context` remains the primary data flow mechanism
- A future design could elevate the event bus to the primary state mutation channel (event-sourcing pattern), enabling audit trails, undo, and distributed observation

---

## Key Implementation References

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

## Context Lifecycle

The canonical flow through the context is:

1. **Create session** — `h_init()` constructs a `launcher.Pipeline(...)` and returns a new `Context`. In PPR-driven execution, `executeppr()` or `executevlappr()` also populates project metadata at this point.
2. **Load data** — Import tasks (`h*_importdata`) attach datasets to the context's domain model (`context.observing_run`, measurement sets, scans, SPWs, etc.).
3. **Execute tasks** — Tasks execute against the in-memory context and return a `Results` object. After each task, `Results.accept(context)` records the outcome and mutates shared state.
4. **Accept results** — Inside `accept()`, results are merged via `Results.merge_with_context(context)`. A `ResultsProxy` is pickled to disk per-stage to keep the in-memory context bounded. The weblog is typically rendered after each top-level stage.
5. **Save / resume** — `h_save()` pickles the context; `h_resume(filename='last')` restores it. Driver-managed breakpoints and developer debugging workflows rely on this cycle.

---
## Exploratory Future Use Cases

The following are potential future use cases that do not trace to current RADPS architecture or requirements.
They are recorded here for completeness but should not be treated as requirements. 

### Multi-Language / Multi-Framework Access to Context

| Field | Content |
|-------|---------|
| **Actor(s)** | Non-Python clients (C++, Julia, JavaScript dashboards), external tools |
| **Summary** | The system should expose context state through a language-neutral interface, using a portable serialization format (Protocol Buffers, JSON-Schema, Arrow), a query API (REST, gRPC, or GraphQL), and type definitions shared across languages. |
| **Postconditions** | Clients in any supported language can read context state through a stable, typed API. |

---

### Streaming / Incremental Processing

| Field | Content |
|-------|---------|
| **Actor(s)** | Data ingest system, workflow engine, incremental processing tasks |
| **Summary** | The system should support incremental dataset registration (adding new scans or execution blocks to a live session), tasks that detect new data and re-process incrementally, and a results model that supports versioning so that re-runs produce new versions rather than overwriting previous outputs. |
| **Postconditions** | New data is incorporated into an active session and processed without restarting from scratch. |

---

### External System Integration (Archive, Scheduling, QA Dashboards)

| Field | Content |
|-------|---------|
| **Actor(s)** | Archive ingest system, scheduling database, QA dashboards, monitoring tools |
| **Summary** | The system could expose a stable, queryable API (REST/gRPC) that external systems can poll or subscribe to, support webhook/event notifications for state transitions, and publish a standard schema for context summaries consumable by external systems. However, this involves a significant trade-off: the current context has no stable API, which gives development teams full flexibility to evolve internal structures without cross-team coordination. A public API would require a formal stability contract, versioning discipline, and potentially a slow-to-change external interface layered over rapidly evolving internals. Whether this gap is in scope — or whether external integration should remain an offline, product-file-based concern — is an open design question. |
| **Postconditions** | External systems receive timely, structured updates about processing state without relying on offline product files. |


---

## Architectural Observations

### The context is a "big ball of state", by design

The current approach is extremely flexible for a long-running, stateful CASA session, but there is no explicit schema boundary between persisted state, ephemeral caches, runtime-only services, and large artifacts. Tasks can (and do) add new fields in an ad-hoc way over time.

### Persistence is pickle-based

Pickle works for short-lived resume/debug use cases, but it is fragile across version changes, risky as a long-term archive format, and not friendly to multi-writer or multi-process updates. The codebase mitigates size by proxying stage results to disk, but the context itself remains a potentially large and unstable object graph.

### Two orchestration planes converge on the same context

Task-driven (interactive CLI) and command-list-driven (PPR / XML procedures) execution both produce and consume the same context. They differ in how inputs are marshalled, how paths are selected, and how resume is initiated, but the persisted context is the same object.

---

## Improvement Suggestions for Next-Generation Design

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

## Context Contract Summary

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

## Open Questions

- Are there additional use cases not captured by either review? Reviewer input may surface new cases.
- Should the future use cases (GAP section) be prioritized? If so, which are most impactful for RADPS?
- What compatibility guarantees should the next-generation context provide across pipeline releases?
