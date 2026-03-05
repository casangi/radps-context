# Pipeline context deep dive: current use cases and architecture notes

This note documents how the current Pipeline uses the runtime **Context** today (as implemented in this repository), with a focus on **use cases** and **implications for next-generation context design**.

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
  - ALMA IF/SD: `pipeline/infrastructure/executeppr.py` (used by `pipeline/runpipeline.py`)
  - VLA/VLASS: `pipeline/infrastructure/executevlappr.py` (used by `pipeline/runvlapipeline.py`)
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

## Suggested “context contract” to carry forward

If you’re designing a new context system, the following capabilities appear to be *hard requirements* from current behavior:

- Identify run: `context_id`, recipe/procedure name, inputs, operator/mode
- Path layout: working/report/products + ability to relocate
- Dataset inventory: execution blocks / measurement sets + per-MS metadata
- Stage results timeline: ordered stages, durations, QA outcomes, tracebacks
- Export products: weblog tar, manifest, AQUA report, scripts
- Resume: restart from last known good stage (or after a breakpoint)
