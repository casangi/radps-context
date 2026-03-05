# Pipeline context deep dive: current use cases and architecture notes

This note documents how the current Pipeline uses the runtime **Context** today (as implemented in this repository), with a focus on **use cases** and **implications for next-generation context design**.

See also:

- [docs/radps_use_case_mapping.md](radps_use_case_mapping.md) (which Pipeline UCs carry forward into RADPS)
- [docs/context_design_use_cases.md](context_design_use_cases.md) (draft RADPS context use cases)

## What “context” means in this codebase

In this repository, the Pipeline “context” is primarily the Python object `pipeline.infrastructure.launcher.Context`, created and managed by `pipeline.infrastructure.launcher.Pipeline`.

The context is simultaneously:

- A **session state container** (counters, directories, results list)
- A **domain metadata container** (observing run + measurement set abstractions)
- A **cross-stage communication channel** (tasks read/write shared state)
- A **persistence unit** (pickled to disk via `Context.save()`)

Key implementation references:

- `Context` / `Pipeline`: `pipeline/infrastructure/launcher.py`
- CLI lifecycle tasks: `pipeline/h/cli/h_init.py`, `pipeline/h/cli/h_save.py`, `pipeline/h/cli/h_resume.py`
- Task dispatch & result acceptance: `pipeline/h/cli/utils.py`, `pipeline/infrastructure/basetask.py`
- PPR-driven execution loops:
  - ALMA: `pipeline/infrastructure/executeppr.py` (used by `pipeline/runpipeline.py`)
  - VLA: `pipeline/infrastructure/executevlappr.py` (used by `pipeline/runvlapipeline.py`)
- XML “procedure” execution for devs: `pipeline/recipereducer.py`

## Context lifecycle (the canonical flow)

### 1) Create a new session

- **Interactive** / scripted use: call `h_init()`.
  - `h_init()` constructs a `launcher.Pipeline(...)` and stores it into a global interactive “stack” (`pipeline.h.cli.cli.stack`).
  - It returns the newly created `Context`.

### 2) Load data and metadata

- Import tasks (`h*_importdata`) attach one or more datasets to the context’s domain model (`context.observing_run`, measurement sets, scans, spws, etc.).
- When the session is PPR-driven, `executeppr()` also populates project metadata up-front:
  - `context.project_summary`
  - `context.project_structure`
  - `context.project_performance_parameters`

### 3) Execute tasks (stages)

- A “task” is usually a class-based pipeline stage registered in the task registry (`pipeline.infrastructure.task_registry`).
- CLI functions are thin wrappers; `pipeline/h/cli/utils.py` resolves task name → inputs → task class.
- Each task returns a `Results` object (or list), then `Results.accept(context)` is invoked.

### 4) Accept results and mutate context

Acceptance is the primary *state transition mechanism*.

Inside `pipeline/infrastructure/basetask.py`:

- Results are merged into the context via `Results.merge_with_context(context)`
- A proxy `ResultsProxy` is created and pickled to disk per-stage (`saved_state/result-stageX.pickle`) to keep the in-memory context smaller
- The context’s `results` list is appended with proxies (not full results)
- The weblog is (typically) rendered after each top-level stage
- Debug builds can also snapshot the entire context per-stage

### 5) Save / resume

- `h_save()` pickles the context to `<context.name>.context`.
- `h_resume(filename='last')` loads the most recent `.context` file.

This persistence model is used by:

- Breakpoint/resume in PPR execution (`executeppr(..., bpaction='resume')`)
- Developer workflows (debugging, “pause and inspect”, regression reproduction)

## Current use cases (as implemented)

This list is intended to be directly reusable for next-gen context requirements.

### UC1 — Operate the pipeline via PPR (ALMA)

**Actor**: operations / automated processing

**Entry point**: run inside CASA via `casa --nogui --nologger -c runpipeline.py <PPR.xml>`

**Core flow**:

- `runpipeline.py` → `pipeline.infrastructure.executeppr.executeppr(ppr_file, importonly=...)`
- PPR provides:
  - Relative paths / ASDM list
  - Procedure/recipe name
  - A list of commands with arguments
- `executeppr()` creates a new context or resumes a prior one
- Executes commands sequentially, with special handling for import/restore stages

**Context requirements**:

- Must store project metadata from the PPR
- Must record task results + tracebacks
- Must support “export on exception” (copy error exit + tar weblog)
- Must support breakpoint-driven stop/resume semantics

### UC2 — Operate the pipeline via PPR (VLA)

**Actor**: operations / automated processing

**Entry point**: `casa --nogui --nologger -c runvlapipeline.py <PPR.xml>`

**Distinctive behaviors**:

- `executevlappr.py` has VLA-specific rules (e.g., skip `hifv_hanning` when `SPECTRAL_MODE=True`)
- Project structure is simplified compared to ALMA

**Context requirements**:

- Must handle VLA-specific control flags (“intent-like” values)

### UC3 — Interactive “task-by-task” execution

**Actor**: pipeline developer / power user

**Entry point**: a Python/CASA session, typically:

- `import pipeline; pipeline.initcli()`
- `context = h_init()`
- then call tasks directly: `hifa_importdata(...)`, `hifa_flagdata()`, ...

**Context requirements**:

- A stable, discoverable task interface (`h_*`, `hif_*`, `hifa_*`, `hifv_*`, `hsd_*`, `hsdn_*`)
- Ability to inspect/modify a live in-memory context between stages
- Convenient weblog rendering for stage-by-stage debugging

### UC4 — Developer “procedure” execution from XML recipes

**Actor**: pipeline developer

**Entry point**: `pipeline.recipereducer.reduce(vis=[...], procedure='procedure_*.xml', ...)`

**Core flow**:

- Load an XML procedure from `pipeline/recipes/`
- Translate `<ProcessingCommand>` nodes to CLI task calls
- Run tasks sequentially; save context at the end

**Context requirements**:

- Must allow overriding the context name to route output to a named directory
- Must tolerate partial execution (`startstage`, `exitstage`)

### UC5 — Save/resume/relocate context

**Actor**: developer, operations (breakpoint resume)

**Entry points**:

- `h_save()` / `h_resume()`
- `Pipeline(context='last')`

**Context requirements**:

- Persistence must be resilient enough for “same version” resume
- Relocation must work for results proxies (only basenames are stored)

### UC6 — Weblog generation as a first-class product

**Actor**: operations + QA + developers

**Implementation**:

- Weblog is rendered after each top-level stage (in `Results.accept`)
- Timetracker + eventbus capture stage timing and abnormal exits

**Context requirements**:

- Must supply stable paths: `output_dir`, `report_dir`, `products_dir`
- Must expose `context.results` in a form that renderers can traverse

### UC7 — Testing and regression harness

**Actor**: CI, developers

**Relevant components**:

- `tests/testing_utils.py` (PipelineTester)
- Rendering failure detection reads the `*.timetracker` database

**Context requirements**:

- Deterministic paths and predictable outputs
- Ability to detect failures *outside* of raw task exceptions (e.g., weblog rendering failures)

## Observed architectural properties (what the code implies)

### The pipeline has two orchestration planes

- **Plan A: task-driven**: direct task calls via CLI wrappers (`pipeline/h/cli/utils.py`)
- **Plan B: command-list-driven**: PPR command lists executed by `executeppr.py` / `executevlappr.py`

Both eventually converge on the same task implementations, but they differ in:

- How inputs are marshalled (PPR dict vs direct function call)
- How session paths are selected (SCIPIPE_ROOTDIR vs local dev assumptions)
- How “resume” is modeled (PPR breakpoint logic vs explicit `h_resume`)

### The context is a “big ball of state”, by design

The current approach is extremely flexible for a long-running, stateful CASA session, but it also means:

- No explicit schema boundary between:
  - persisted state
  - ephemeral caches
  - runtime-only services
  - large artifacts
- Tasks can (and do) add new fields in an ad-hoc way over time

### Persistence is pickle-based

Pickle works for short-lived resume/debug use cases, but it is:

- fragile across version changes
- risky if used as a long-term archive format
- not friendly to multi-writer or multi-process updates

The codebase already mitigates size by proxying stage results to disk, but the *context itself* remains a potentially large and unstable object graph.

## Improvement suggestions for a next-generation context design

These suggestions are intentionally phrased as **requirements and design directions**, not “rewrite everything immediately”.

### 1) Split “context data” from “context services”

Today, the same object mixes:

- structured metadata
- caching
- filesystem layout
- libraries (calibration/image libraries)
- methods and convenience accessors

A next-gen design should define a minimal, explicit **ContextData** model that is:

- typed
- schema-versioned
- serializable in a stable format (e.g. JSON/MsgPack/Arrow)

Then attach runtime-only **services** (CASA tool handles, caches, heuristics engines) around it.

### 2) Introduce a ContextStore interface

Replace “pickle a Python object graph” with a storage abstraction:

- `ContextStore.get(context_id)`
- `ContextStore.put(context_id, patch/event)`
- `ContextStore.list_runs(...)`

Backends can start simple (SQLite) and grow (Postgres/object store) without changing task logic.

### 3) Make state transitions explicit (event-sourced or patch-based)

The repository already has an event bus (`pipeline.infrastructure.eventbus`).

A next-gen design can leverage that by recording:

- task started/completed
- result accepted
- key state changes (added MS, updated cal library, exported products)

This yields:

- reproducibility (what changed when)
- easier partial rebuilds
- better distributed orchestration

### 4) Treat large artifacts as artifacts, not context fields

A consistent pattern is needed for large arrays/images/tables:

- store in an artifact store (`products/`, object store, or CAS tables)
- store only *references* in context data

This avoids “accidentally pickle a GiB array” failure modes and makes distribution/cloud execution more realistic.

### 5) Remove reliance on global interactive stacks for non-interactive execution

The global `pipeline.h.cli.cli.stack` is convenient in interactive sessions, but it makes composition and concurrency hard.

Next-gen design direction:

- make tasks accept an explicit context handle
- keep interactive convenience wrappers, but do not make them the core contract

### 6) Unify orchestration

Long-term, it will be simpler if:

- PPR execution and “procedure XML” execution share a single execution engine
- both compile into a common intermediate representation (a DAG or linear plan)

Even if execution stays sequential, having a common plan representation will help:

- retries
- provenance
- partial execution
- parallelization boundaries

### 7) Versioned compatibility policy

The notebook in this repo explicitly calls out the lack of API/backward-compat guarantees. For next-gen work, pick a stance:

- **Operational context**: must be resumable within a supported release window
- **Development context**: best-effort

…and encode that into the serialization/store layer (schema versions + migrations).

## Fine-grained internal use cases (within-execution context interactions)

The 7 use cases above describe *when and how* the pipeline is launched. The use cases below describe *how the context is consumed and mutated inside a running pipeline*, organized by the subsystem that interacts with it. These are the patterns that constrain any replacement context design.

### UC8 — Domain metadata querying (`context.observing_run`)

**Consumers**: nearly every task, heuristic, renderer, and QA handler.

**Access pattern**:

- `context.observing_run.get_ms(name=vis)` — resolve a measurement set (MS) object by filename
- `context.observing_run.measurement_sets` — iterate all registered MS objects
- `context.observing_run.get_measurement_sets_of_type(dtypes)` — filter MS list by data type (e.g., RAW, REGCAL_CONTLINE_ALL, BASELINED)
- `context.observing_run.virtual2real_spw_id(vspw, ms)` / `real2virtual_spw_id(...)` — translate between abstract pipeline SPW IDs and CASA-native IDs
- `context.observing_run.get_real_spwsel(spw_list, vis_list)` — bulk SPW mapping
- `context.observing_run.virtual_science_spw_ids` — virtual SPW catalog
- `context.observing_run.ms_reduction_group` — per-group reduction metadata (SD)
- `context.observing_run.ms_datatable_name` — shared data table (SD flagging)
- `context.observing_run.start_datetime`, `.end_datetime`, `.project_ids`, `.schedblock_ids`, `.execblock_ids`, `.observers` — provenance fields consumed by weblog renderers

MS objects themselves are rich domain objects carrying scans, fields, SPWs, antennas, reference antenna ordering, etc. Tasks read per-MS state like `ms.reference_antenna`, `ms.session`, `ms.start_time`, `ms.origin_ms`.

**Impact**: `observing_run` is the single most heavily queried context facet. Any future design must expose an equivalent high-performance, random-access MS lookup.

### UC9 — Calibration library management (`context.callibrary`)

**Consumers**: calibration tasks (bandpass, gaincal, applycal, polcal, selfcal, restoredata), heuristics (atm, bandpass selection), importdata, mstransform, uvcontsub.

**Write patterns** (from `merge_with_context`):

- `context.callibrary.add(calto, calfrom)` — register a new calibration application (caltable + target selection)
- `context.callibrary.unregister_calibrations(matcher)` — remove calibrations matching a predicate

**Read patterns** (from task `prepare`/`analyse` phases and heuristics):

- `context.callibrary.active.get_caltable(caltypes=...)` — list currently active cal tables, optionally filtered by type
- `context.callibrary.get_calstate(calto)` — get the full cal-application state for a given target selection

The callibrary is an append-mostly, ordered-by-registration-time data structure backed by `CalApplication` → `CalTo` / `CalFrom` objects with interval trees for efficient matching. It is *the* primary cross-stage communication channel for calibration workflows.

**Impact**: if context is split into microservices or a database, the callibrary must support transactional multi-entry updates (tasks often register multiple CalApplications atomically within a single `merge_with_context` call).

### UC10 — Image library management (`context.sciimlist`, `.calimlist`, `.rmsimlist`, `.subimlist`)

**Write patterns** (from `merge_with_context` in imaging result objects):

- `context.sciimlist.add_item(imageitem)` — register science images
- `context.calimlist.add_item(imageitem)` — register calibrator images
- `context.rmsimlist.add_item(imageitem)` — register RMS images
- `context.subimlist.add_item(imageitem)` — register sub-product images (cutouts, cubes)

**Read patterns**:

- `context.sciimlist.get_imlist()` — retrieve science image list (used by pbcor, exportdata, heuristics)
- `context.subimlist.get_imlist()` — retrieve sub-product list (used by analyzestokescubes, exportvlassdata, analyzealpha)

**Impact**: image libraries function as an artifact registry. A next-gen design might separate the image *metadata* (tracked in context) from the image *data* (stored in artifact store), with image items carrying references rather than inline data.

### UC11 — Imaging pipeline state (`context.clean_list_*`, `.imaging_*`, `.synthesized_beams`, etc.)

**What the code does**:

Multiple imaging-related attributes form a "shared scratch pad" that lets tasks collaborate across the imaging sub-pipeline:

| Attribute | Written by | Read by |
|---|---|---|
| `clean_list_pending` | `editimlist`, `makeimlist`, `findcont`, `makeimages` | `findcont`, `tclean`, `transformimagedata`, `uvcontsub`, `checkproductsize` |
| `clean_list_info` | `makeimlist`, `makeimages` | display/renderer code |
| `imaging_mode` | `editimlist` | `makermsimages`, `makecutoutimages`, `makeimages` |
| `imaging_parameters` | PPR / `editimlist` | `tclean`, `checkproductsize`, heuristics |
| `synthesized_beams` | `imageprecheck`, `tclean`, `checkproductsize`, `makeimlist`, `makeimages` | `checkproductsize`, heuristics |
| `per_spw_cont_sensitivities_all_chan` | `imageprecheck`, `tclean`, `makeimages` | heuristics |
| `size_mitigation_parameters` | `checkproductsize` | downstream stages |
| `contfile`, `linesfile` | `makeimlist` | downstream imaging |
| `selfcal_targets`, `selfcal_resources` | `selfcal` | `exportdata` |
| `clean_masks`, `clean_thresholds` | set during imaging | used for masking decisions |

**Impact**: this is the most fragile part of the current context design. Attributes are added ad-hoc, there is no schema, and defensive `hasattr()` checks appear in the code. A future design should formalize this as a typed imaging state machine or at least a versioned configuration sub-document.

### UC12 — VLA-specific state bag (`context.evla`)

**What it is**: a `collections.defaultdict(dict)` attached to the context during VLA import, keyed as `context.evla['msinfo'][ms_name].<property>`.

**Written by**: `hifv_importdata` (creates + initializes), `testBPdcals` (gain intervals, ignorerefant), `fluxscale/solint` (gain_solint2, longsolint, short_solint, new_gain_solint1), `fluxboot` (fluxscale_sources, flux_densities, spws, spindex_results, fbversion).

**Read by**: nearly every VLA calibration task and heuristic — `semiFinalBPdcals`, `finalcals`, `fluxboot`, `fluxbootdisplay`, `bandpass` heuristic, `circfeedpolcal`, `selfcal`.

Accessed fields include:

- `gain_solint1`, `gain_solint2`, `shortsol1`, `shortsol2`, `longsolint`, `short_solint`, `new_gain_solint1`
- `setjy_results`, `ignorerefant`
- `bandpass_field_select_string`, `bandpass_scan_select_string`
- `delay_field_select_string`, `delay_scan_select_string`
- `calibrator_field_select_string`, `calibrator_scan_select_string`
- `flux_field_select_string`, `phase_scan_select_string`
- `testgainscans`
- `fluxscale_sources`, `fluxscale_flux_densities`, `fluxscale_spws`, `fluxscale_result`, `fbversion`, `spindex_results`

**Impact**: `context.evla` is a completely untyped, dictionary-of-dictionaries sidecar. It demonstrates the "big ball of state" anti-pattern most clearly. A future design should:

1. Define a typed `VLACalibrationState` (or per-MS calibration state) with explicit fields.
2. Provide accessor methods rather than raw dict lookups.
3. Separate VLA-specific concerns from the generic context via composition (e.g., `context.get_extension('evla')`).

### UC13 — QA scoring (`pipelineqa` framework)

**Mechanism**: after `merge_with_context()`, `accept()` triggers `pipelineqa.qa_registry.do_qa(context, result)`.

**How QA handlers use context**:

- QA handlers implement `QAPlugin.handle(context, result)`.
- They typically call `context.observing_run.get_ms(vis)` to look up metadata needed for scoring (antenna count, channel count, SPW properties, field intents).
- Some QA handlers also check `context.imaging_mode` to branch on VLASS-specific scoring.
- Scores are appended to `result.qa.pool` — they don't mutate the context directly, but the context *holds* those scored results via `context.results`.

**Impact**: QA handlers are *read-only* with respect to context. They could operate on a frozen snapshot or read-only view, which is a good candidate for parallelization.

### UC14 — Weblog rendering (context traversal)

**Mechanism**: `WebLogGenerator.render(context)` in `pipeline/infrastructure/renderer/htmlrenderer.py`.

**What it reads from context**:

- `context.results` — unpickled from `ResultsProxy` objects, iterated for every renderer
- `context.report_dir`, `context.output_dir` — filesystem layout
- `context.observing_run.*` — MS metadata, scheduling blocks, execution blocks, observers, project IDs, start/end times
- `context.project_summary.telescope` — to determine telescope-specific page layouts (ALMA vs VLA vs NRO)
- `context.project_structure.*` — OUS status entity ID, PPR file, recipe name, OUS entity ID
- `context.name` — for timetracker database path
- `context.logs['casa_commands']` — for including CASA command history

The renderer iterates `context.results` multiple times (assigning to topics, extracting flags, building timelines). It also reads per-result fields like `stage_number`, `timestamps`, `vis`, `qa`, and `task` to populate per-stage pages.

**Impact**: renderers need *read-only* access to context + full results. The current approach requires unpickling *every* result into memory, then re-proxying them when done. A lazy or streaming model would reduce peak memory.

### UC15 — MPI parallel task distribution

**Mechanism**: `pipeline/infrastructure/mpihelpers.py`, class `Tier0PipelineTask`.

**How it works**:

1. The MPI client saves the context to disk as a pickle: `context.save(path)`.
2. Task arguments are also pickled to disk alongside the context.
3. `Tier0PipelineTask` is pushed to each MPI server.
4. On the server, `get_executable()` loads the context from the pickle file, modifies `context.logs['casa_commands']` to a server-local temp path, creates the task's `Inputs(context, **task_args)`, then executes the task.
5. Results are collected back and merged on the client.

For `Tier0JobRequest` (lower-level job distribution), the executor is shallow-copied *excluding* the context reference to stay within the MPI buffer limit (~150 MiB, per PIPE-1337).

**Impact**: MPI distribution fundamentally *forks* the context — each server gets a read-only snapshot. This means:

- Tasks running on MPI servers cannot write back to the main context during execution.
- The context pickle must be small enough to broadcast efficiently.
- Any future distributed execution model must formalize "read-only context snapshot" vs. "results write-back" as first-class concepts.

### UC16 — Cross-stage forward-pass communication via results list

**Pattern**: tasks read from `context.results` to discover outputs of earlier stages.

**Examples**:

- VLA tasks compute `stage_number` from `context.results[-1].read().stage_number + 1`
- VLA heuristics retrieve `setjy_results` from `context.results[0].read().setjy_results`
- `vlassmasking` iterates `context.results[::-1]` to find the latest `MakeImagesResult` for mask names
- `sky.py` display reads `context.results[-1]` for the most recent result
- Export/AQUA code reads `context.results[0]` and `context.results[-1]` for execution timestamps
- `pipeline_statistics` iterates all results for task descriptions and durations

**Impact**: this is essentially "grep through the stage history for a value". It works because execution is serial and deterministic. But it is:

- Fragile: result indices shift if stages are inserted/removed/skipped.
- Slow: requires unpickling results from disk for each query.
- Implicit: there is no declared dependency between stages.

A future design should provide explicit stage-to-stage data dependencies (e.g., named outputs that downstream tasks subscribe to) rather than requiring tasks to walk the results list.

### UC17 — Project metadata and state injection (`set_state` / direct assignment)

**Patterns**:

- `context.set_state('ProjectStructure', 'recipe_name', value)` — used by `recipereducer.reduce()` and SD heuristics to inject project metadata
- `context.project_summary = project.ProjectSummary(...)` — set by `executeppr()` / `executevlappr()`
- `context.project_structure = project.ProjectStructure(...)` — set by PPR executors
- `context.project_performance_parameters = ...` — performance parameters from the PPR
- `context.processing_intents = processing_intents` — set by `Pipeline` during initialization

These are typically one-time writes at session start, read many times by tasks and renderers.

**Impact**: project metadata is a good candidate for a separate, immutable-after-init sub-record in any future context schema.

### UC18 — Event bus for lifecycle events

**Mechanism**: `pipeline.infrastructure.eventbus.send_message(event)`.

**Event types observed**:

- `ResultAcceptingEvent` — sent at the start of `Results.accept()`
- `ContextCreated`, `TaskStarted`, `TaskComplete` — lifecycle bookmarks
- Potentially others registered by plugins

**Current status**: the event bus exists and fires events, but it is lightly used. It does *not* serve as the primary data flow mechanism — `merge_with_context` does that.

**Impact**: the event bus is an underutilized asset. A future design could elevate it to the primary state mutation channel (event-source pattern), enabling audit trails, undo, and distributed observation.

## Use cases the current pipeline cannot handle

The following describe scenarios that the current context design *does not support* but that could be valuable in a future architecture.

### FUC1 — Concurrent / overlapping task execution

**Problem**: today, all task execution is strictly serial. The context is a mutable, shared-everything singleton. No locking, no isolation between stages.

**Why it matters**: many calibration stages are independent per-MS or per-SPW. Parallelizing them could significantly reduce wall-clock time.

**What would be needed**:

- A context that supports isolated read snapshots (like database transactions or COW snapshots)
- A merge reconciliation step when concurrent results are accepted
- Explicit declaration of which context fields each task reads and writes

### FUC2 — Cloud / distributed execution without shared filesystem

**Problem**: the current context is a pickle file on a local/shared filesystem. MPI distribution requires all nodes to see the same filesystem.

**Why it matters**: cloud execution (AWS Batch, Kubernetes Jobs, etc.) typically does not offer shared POSIX filesystems.

**What would be needed**:

- A context store backed by a database or object store (S3, GCS)
- Artifact references rather than filesystem paths for cal tables and images
- Tasks that can operate on remote datasets without requiring local copies

### FUC3 — Multi-language / multi-framework access to context

**Problem**: the context is a Python object graph, tightly coupled to CASA's Python runtime. Non-Python clients (C++, Julia, JavaScript dashboards) cannot access it.

**Why it matters**: next-gen tools (e.g., browser-based monitoring dashboards, C++ processing kernels, or notebook UIs) need to query pipeline state.

**What would be needed**:

- A language-neutral serialization format (Protocol Buffers, JSON-Schema, Arrow)
- A query API (REST, gRPC, or GraphQL)
- Type definitions shared across languages

### FUC4 — Streaming / incremental processing

**Problem**: the pipeline assumes all data is available at session start. It cannot process data as it arrives from the correlator or archive.

**Why it matters**: real-time or near-real-time QA during observations would improve operational efficiency.

**What would be needed**:

- A context that supports incremental dataset registration (add new scans/EBs to a live session)
- Tasks that can detect "new data available" and re-process incrementally
- A results model that supports versioning (re-run of a stage with updated data produces a new version rather than overwriting)

### FUC5 — Provenance and reproducibility guarantees

**Problem**: today, there is no formal record of which context state a task observed when it ran. If you re-run from a saved context, you get the state *after* the last save, not the state that was live at stage N.

**Why it matters**: regulatory / scientific reproducibility requires knowing *exactly* what inputs produced a given result.

**What would be needed**:

- Immutable snapshots of context state per-stage (event sourcing)
- Hashing of all task inputs (context fields + parameters) for cache invalidation / reproducibility tokens
- Ability to "replay" a pipeline run from the event log

### FUC6 — Fine-grained access control / multi-tenant context

**Problem**: the context is an all-or-nothing object. There is no concept of access control, multi-user sessions, or data isolation between projects.

**Why it matters**: shared processing environments (clusters, cloud) may need to isolate project data while sharing infrastructure.

**What would be needed**:

- Per-project context namespacing
- Role-based access to context fields (e.g., "operator can view but not modify calibration library")
- Audit logging of all context mutations

### FUC7 — Partial re-execution / targeted stage re-run

**Problem**: today, "resume" means "restart from the last completed stage". There is no way to selectively re-run stage 5 with different parameters while keeping stages 1-4 and 6-N intact.

**Why it matters**: interactive development and debugging often requires tweaking a single stage and seeing the downstream effect.

**What would be needed**:

- Explicit dependency tracking between stages (which context fields does stage N consume?)
- Ability to invalidate downstream stages when a mid-pipeline stage is re-run
- Versioned results per stage (keep old version alongside new version)

### FUC8 — External system integration (archive, scheduling, QA dashboards)

**Problem**: today, context is a local, in-process object. Other systems (archive ingest, scheduling database, QA dashboards) interact via offline products (weblog, manifest, AQUA XML).

**Why it matters**: tighter integration could enable real-time monitoring, automated re-submission on failure, and direct archive ingest.

**What would be needed**:

- A stable, queryable context API (REST/gRPC) that external systems can poll or subscribe to
- Webhook / event notification support for state transitions (stage complete, QA score below threshold)
- A standard schema for context summaries consumable by external systems

## Suggested "context contract" to carry forward

If you're designing a new context system, the following capabilities appear to be *hard requirements* from current behavior:

- Identify run: `context_id`, recipe/procedure name, inputs, operator/mode
- Path layout: working/report/products + ability to relocate
- Dataset inventory: execution blocks / measurement sets + per-MS metadata
- Stage results timeline: ordered stages, durations, QA outcomes, tracebacks
- Export products: weblog tar, manifest, AQUA report, scripts
- Resume: restart from last known good stage (or after a breakpoint)

And these are *hard requirements from internal usage patterns* (UC8-UC18):

- Fast MS lookup: random-access by name, filtering by data type, virtual↔real SPW translation
- Calibration library: append-oriented, ordered, with transactional multi-entry updates and predicate-based queries
- Image library: four typed registries (science, calibrator, RMS, sub-product) with add/query semantics
- Imaging state scratch pad: typed, versioned configuration for the imaging sub-pipeline
- QA scoring: read-only context snapshot for parallel-safe QA handler execution
- Weblog rendering: read-only traversal of full results timeline + MS metadata + project metadata
- MPI/distributed: efficient context snapshot broadcast + results write-back
- Cross-stage data flow: explicit named outputs rather than results-list walking
- Project metadata: immutable-after-init sub-record
- VLA-specific state: typed, composable extension rather than untyped dict
