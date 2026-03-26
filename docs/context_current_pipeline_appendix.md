# Pipeline Context: Supplementary Analysis

This document contains and reference material that supplement the use cases in [context_use_cases_current_pipeline.md](context_use_cases_current_pipeline.md). These sections were separated to keep the use-case document focused on requirements.

---

## Implementation Notes by Use Case

The following implementation notes describe how each use case is realized in the current pipeline codebase. They were separated from the use-case definitions to keep the requirements document focused on requirements.

### UC-01 — Load, Update, and Provide Access to Observation Metadata

**Implementation notes** — `context.observing_run` is the most heavily queried attribute of the context:

- `context.observing_run.get_ms(name=vis)` — resolve an MS by filename
- `context.observing_run.measurement_sets` — iterate all registered MS objects
- `context.observing_run.get_measurement_sets_of_type(dtypes)` — filter by data type (RAW, REGCAL_CONTLINE_ALL, BASELINED, etc.)
- `context.observing_run.virtual2real_spw_id(vspw, ms)` / `real2virtual_spw_id(...)` — translate between abstract pipeline SPW IDs and CASA-native IDs
- `context.observing_run.virtual_science_spw_ids` — virtual SPW catalog
- `context.observing_run.ms_reduction_group` — per-group reduction metadata (single-dish)
- Provenance attributes: `start_datetime`, `end_datetime`, `project_ids`, `schedblock_ids`, `execblock_ids`, `observers`

MS objects are rich domain objects carrying scans, fields, SPWs, antennas, reference antenna ordering, etc. Tasks read per-MS state like `ms.reference_antenna`, `ms.session`, `ms.start_time`, `ms.origin_ms`.

---

### UC-02 — Store and Provide Project-Level Metadata

**Implementation notes** — project metadata is set once at session start, is not modified after initialization, and is read many times:

- `context.project_summary = project.ProjectSummary(...)` — set by `executeppr()` / `executevlappr()`
- `context.project_structure = project.ProjectStructure(...)` — set by PPR executors
- `context.project_performance_parameters` — performance parameters from the PPR
- `context.set_state('ProjectStructure', 'recipe_name', value)` — used by `recipereducer.reduce()` and SD heuristics
- `context.processing_intents` — set by `Pipeline` during initialization

Execution paths and output locations are also managed as part of project-level metadata:

- Path roots: `output_dir`, `report_dir`, `products_dir`
- Context name drives deterministic, named run directories
- Relocation semantics are supported for results proxies (basenames stored) and common output layout
- PPR-driven execution may derive paths from environment variables (e.g., `SCIPIPE_ROOTDIR`)

---

### UC-03 — Register, Query, and Update Calibration State

**Implementation notes** — `context.callibrary` is the primary cross-stage communication channel for calibration workflows:

- **Write:** `context.callibrary.add(calto, calfrom)` — register a calibration application (cal table + target selection); `context.callibrary.unregister_calibrations(matcher)` — remove by predicate
- **Read:** `context.callibrary.active.get_caltable(caltypes=...)` — list active cal tables; `context.callibrary.get_calstate(calto)` — get full application state for a target selection
- Backed by `CalApplication` → `CalTo` / `CalFrom` objects with interval trees for efficient matching; append-mostly, ordered by registration time

---

### UC-04 — Manage Imaging State

**Implementation notes** — imaging state is stored as ad-hoc attributes on the context object with no formal schema. Defensive `hasattr()` checks appear throughout the code to guard against attributes that may not yet exist:

| Attribute | Written by | Read by |
|---|---|---|
| `clean_list_pending` | `editimlist`, `makeimlist`, `findcont`, `makeimages` | `findcont`, `tclean`, `transformimagedata`, `uvcontsub`, `checkproductsize` |
| `clean_list_info` | `makeimlist`, `makeimages` | display/renderer code |
| `imaging_mode` | `editimlist` | `makermsimages`, `makecutoutimages`, `makeimages` |
| `imaging_parameters` | PPR / `editimlist` | `tclean`, `checkproductsize`, heuristics |
| `synthesized_beams` | `imageprecheck`, `tclean`, `checkproductsize`, `makeimlist`, `makeimages` | `checkproductsize`, heuristics |
| `size_mitigation_parameters` | `checkproductsize` | downstream stages |
| `selfcal_targets`, `selfcal_resources` | `selfcal` | `exportdata` |

---

### UC-05 — Register and Query Produced Image Products

**Implementation notes** — image libraries provide typed registries:

- `context.sciimlist.add_item(imageitem)` / `.get_imlist()` — science images
- `context.calimlist` — calibrator images
- `context.rmsimlist` — RMS images
- `context.subimlist` — sub-product images (cutouts, cubes)

---

### UC-06 — Track Current Execution Progress

**Implementation notes:**

- `context.stage_number` and `context.task_counter` track progress

---

### UC-07 — Preserve Per-Stage Execution Record

**Implementation notes:**

- `context.results` holds an ordered list of `ResultsProxy` objects (proxied to disk to bound memory)
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

### UC-13 — Provide Read-Only State for Reporting

**Implementation notes** — `WebLogGenerator.render(context)` in `pipeline/infrastructure/renderer/htmlrenderer.py`:

- Reads `context.results` — unpickled from `ResultsProxy` objects, iterated for every renderer
- Reads `context.report_dir`, `context.output_dir` — filesystem layout
- Reads `context.observing_run.*` — MS metadata, scheduling blocks, execution blocks, observers, project IDs, start/end times
- Reads `context.project_summary.telescope` — to determine telescope-specific page layouts (ALMA vs VLA vs NRO)
- Reads `context.project_structure.*` — OUS IDs, PPR file, recipe name
- Reads `context.logs['casa_commands']` — CASA command history

---

### UC-14 — Support QA Evaluation and Store Quality Assessments

**Implementation notes** — after `merge_with_context()`, `accept()` triggers `pipelineqa.qa_registry.do_qa(context, result)`:

- QA handlers implement `QAPlugin.handle(context, result)`
- They typically call `context.observing_run.get_ms(vis)` to look up metadata for scoring (antenna count, channel count, SPW properties, field intents)
- Some handlers check `context.imaging_mode` to branch on VLASS-specific scoring
- Scores are appended to `result.qa.pool` — the context provides inputs to QA evaluation, but the scores are stored on the result rather than as direct context mutations

QA handlers are read-only with respect to the context. 

---

### UC-16 — Manage Telescope-Specific Context Extensions

**Implementation notes** — `context.evla` is a `collections.defaultdict(dict)`, keyed as `context.evla['msinfo'][ms_name].<property>`:

- **Written by:** `hifv_importdata` (creates + initializes), `testBPdcals` (gain intervals, ignorerefant), `fluxscale/solint`, `fluxboot`
- **Read by:** nearly every VLA calibration task and heuristic
- Accessed fields include: `gain_solint1`, `gain_solint2`, `setjy_results`, `ignorerefant`, various `*_field_select_string` / `*_scan_select_string` values, `fluxscale_sources`, `spindex_results`, and many more

This is an untyped, dictionary-of-dictionaries sidecar attached to the top-level context
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
