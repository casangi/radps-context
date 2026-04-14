# Pipeline Context: Supplementary Analysis

This document contains implementation details and reference material that supplement the use cases in [context_use_cases_current_pipeline.md](context_use_cases_current_pipeline.md). These sections were separated to keep the use-case document focused on requirements.

---

## Implementation Notes by Use Case

The following implementation notes describe how selected use cases are realized in the current pipeline codebase. They were separated from the use-case definitions to keep the requirements document focused on requirements; use cases not listed here do not currently have appendix-level implementation notes in this document.

### UC-01 — Load, Update, and Provide Access to Observation Metadata

**Implementation notes** — `context.observing_run` holds the observation metadata and is the most frequently queried attribute of the context:

- `context.observing_run.get_ms(name=vis)` — resolve an MS by filename
- `context.observing_run.measurement_sets` — all registered MS objects
- `context.observing_run.get_measurement_sets_of_type(dtypes)` — filter by data type (RAW, REGCAL_CONTLINE_ALL, BASELINED, etc.)
- `context.observing_run.virtual2real_spw_id(vspw, ms)` / `real2virtual_spw_id(...)` — translate between abstract pipeline SPW IDs and CASA-native IDs
- `context.observing_run.virtual_science_spw_ids` — virtual SPW catalog
- `context.observing_run.ms_reduction_group` — per-group reduction metadata (single-dish)
- Provenance attributes: `start_datetime`, `end_datetime`, `project_ids`, `schedblock_ids`, `execblock_ids`, `observers`

The MS objects stored by `context.observing_run` carry information about scans, fields, SPWs, antennas, reference antenna ordering, etc. Tasks read per-MS state like `ms.reference_antenna`, `ms.session`, `ms.start_time`, `ms.origin_ms`.

For the single-dish pipeline, this use case also includes per-MS `DataTable` products referenced through `context.observing_run.ms_datatable_name`. These are not just raw imported metadata tables: they persist row-level metadata and derived quantities used by downstream SD tasks. During SD import, the reader populates `DataTable` columns such as `RA`, `DEC`, `AZ`, `EL`, `SHIFT_RA`, `SHIFT_DEC`, `OFS_RA`, and `OFS_DEC`, including coordinate conversions into the pipeline's chosen celestial frame (for example ICRS) so later imaging, gridding, plotting, and QA code can reuse those values efficiently.

---

### UC-02 — Store and Provide Project-Level Metadata

**Addendum**: conceptually separate *project metadata* (properties of the observation program such as PI, targeted sensitivities, beam requirements) from *workflow/recipe metadata* (processing recipes, execution instructions, heuristics parameters). Project metadata describes the scientific intent and constraints, while workflow metadata captures how the data should be processed; the two interact but have different origins and lifecycles. Recording this distinction helps ensure that processing recipes remain reusable across projects and that project-level constraints are treated as inputs to heuristics rather than being conflated with the workflow definition.

**Implementation notes** — project metadata is set during initialization or import, is not modified after import, and is read many times:

- `context.project_summary = project.ProjectSummary(...)` — set by `executeppr()` / `executevlappr()`
- `context.project_structure = project.ProjectStructure(...)` — set by PPR executors
- `context.project_performance_parameters` — performance parameters from the PPR
- `context.set_state('ProjectStructure', 'recipe_name', value)` — used by `recipereducer.reduce()` and SD heuristics
- `context.processing_intents` — set by `Pipeline` during initialization

---

### UC-03 — Register, Query, and Update Calibration State

**Implementation notes** — `context.callibrary` is the primary cross-stage communication channel for calibration workflows:

- **Write:** `context.callibrary.add(calto, calfrom)` — register a calibration application (cal table + target selection); `context.callibrary.unregister_calibrations(matcher)` — remove by predicate
- **Read:** `context.callibrary.active.get_caltable(caltypes=...)` — list active cal tables; `context.callibrary.get_calstate(calto)` — get full application state for a target selection
- Backed by `CalApplication` → `CalTo` / `CalFrom` objects with interval trees for efficient matching.
 - The callibrary also supports de-registration of trial or reverted calibrations via predicate-based removal; implementations should ensure such removals are atomic and leave an audit entry so provenance is preserved when rollbacks or experiments occur.

---

### UC-04 — Manage Imaging State

Note: imaging workflows commonly separate a lightweight planning phase (for example, `makeimlist`, `editimlist`) that assembles imaging instructions, target lists, and imaging-mode heuristics from a computationally intensive execution phase (for example, `makeimages`) that performs the heavy imaging work. These phases may require different resource allocation and scheduling policies; acknowledging this planning/execution separation helps explain the series of tightly coupled imaging stages and the significant inter-stage interactions observed in practice.

**Implementation notes** — imaging state is stored as ad-hoc attributes on the context object with no formal schema. Defensive `hasattr()` checks appear throughout the code to guard against attributes that may not yet exist:

| Attribute | Written by | Read by |
|---|---|---|
| `clean_list_pending` | `editimlist`, `makeimlist`, `findcont`, `makeimages` | `findcont`, `transformimagedata`, `makeimages`, `vlassmasking` |
| `clean_list_info` | `makeimlist`, `makeimages` | `makeimages` |
| `imaging_mode` | `editimlist` | `makermsimages`, `makecutoutimages`, `makeimages`, VLASS export/display code |
| `imaging_parameters` | `imageprecheck` | `tclean`, `checkproductsize`, `makeimlist`, heuristics |
| `synthesized_beams` | `imageprecheck`, `tclean`, `checkproductsize`, `makeimlist`, `makeimages` | `imageprecheck`, `editimlist`, `tclean`, `uvcontsub`, `checkproductsize`, heuristics |
| `size_mitigation_parameters` | `checkproductsize` | downstream stages |
| `selfcal_targets` | `selfcal` | `makeimlist` |
| `selfcal_resources` | `selfcal` | `exportdata` |

---

### UC-05 — Register and Query Produced Image Products

**Implementation notes** — image libraries provide typed registries:

- `context.sciimlist` — science images
- `context.calimlist` — calibrator images
- `context.rmsimlist` — RMS images
- `context.subimlist` — sub-product images (cutouts, cubes)

---

### UC-06 — Track Current Execution Progress

**Implementation notes:**

- `context.stage`, `context.task_counter`, `context.subtask_counter` track progress

---

### UC-07 — Preserve Per-Stage Execution Record

**Implementation notes:**

- `context.results` holds an ordered list of `ResultsProxy` objects which are proxied to disk to bound memory
- Timetracker integration provides per-stage timing data
- Results proxies store basenames for portability

---

### UC-08 — Propagate Task Outputs to Downstream Tasks

**Implementation notes** — the intended primary mechanism in the current pipeline is immediate propagation through context state updated during result acceptance. Over time, some workflows also came to inspect recorded results directly. Both patterns exist in the codebase, but the second should be understood as an accreted pattern rather than the original design intent.

This use case is also a concrete example of context creep caused by weakly enforced contracts: the intended contract was that downstream tasks would consume explicitly merged shared state, but later code sometimes reached into `context.results` directly when that contract was not maintained consistently.

1. **Immediate state propagation** — `Results.merge_with_context(context)` updates calibration library, image libraries, and dedicated context attributes such as `clean_list_pending`, `clean_list_info`, `synthesized_beams`, `size_mitigation_parameters`, `selfcal_targets`, and `selfcal_resources` so later tasks can access the current processing state directly without parsing another task's results object.
2. **Recorded-result inspection** — some tasks read `context.results` to find outputs from earlier stages when those outputs are needed from the recorded results rather than from merged shared state. This pattern introduces coupling to recipe order or to another task's result class structure. For example:
   - VLA tasks compute `stage_number` from `context.results[-1].read().stage_number + 1`
   - `vlassmasking` iterates `context.results[::-1]` to find the latest `MakeImagesResult`
   - Export/AQUA code reads `context.results[0]` and `context.results[-1]` for timestamps

---

### UC-09 — Provide a Transient Intra-Stage Workspace

**Implementation notes** — the current framework implements this behavior in `pipeline/infrastructure/basetask.py`:

- `StandardTaskTemplate.execute()` replaces `self.inputs` with a pickled copy of the original inputs, including the context, before task logic runs, and restores the original inputs in `finally`
- Child tasks therefore execute against a duplicated context that may be mutated freely during `prepare()` / `analyse()`
- `Executor.execute(job, merge=True)` commits a child result by calling `result.accept(self._context)`; with `merge=False`, the child task may still be run and inspected without committing its state
- This makes it possible for aggregate tasks to try tentative calibration paths or other destructive edits inside a stage and keep only the results they explicitly accept
- The rollback mechanism is in-memory copy/restore of task inputs and context; it is distinct from explicit session save/resume workflows

---

### UC-10 — Support Multiple Orchestration Drivers

**Implementation notes** — multiple entry points converge on the same task execution path:

- **Task-driven**: direct task calls via CLI wrappers in `pipeline/h/cli/`
- **Command-list-driven**: PPR and XML procedure commands via `executeppr.py` / `executevlappr.py` and `recipereducer.py`

They differ in how inputs are specified, how session paths are selected, and how resume is initiated, but the persisted context is the same.

---

### UC-11 — Save and Restore a Processing Session

**Implementation notes:**

- `h_save()` pickles the whole context to `<context.name>.context`
- `h_resume(filename)` loads a `.context` file, defaulting to the most recent context file available if `filename` is `None` or `last` is used.
- Per-stage results are proxied to disk (`saved_state/result-stageX.pickle`) to keep the in-memory context smaller
- Used by driver-managed breakpoint/resume (`executeppr(..., bpaction='resume')`) and developer debugging workflows

Note: a robust "true resume" often requires the ability to recover or reproduce the on-disk data-file state that existed when the context was saved (for example via filesystem snapshots, immutable/versioned data copies, or non-destructive editing). Without an explicit mechanism to capture or restore data-file state, resume guarantees are conditional — the restored context may only be operationally equivalent if the underlying data files are unchanged or otherwise recoverable to the snapshot point. Implementing filesystem-level snapshotting or versioning is an orthogonal concern to the context model and may be required for strong resume guarantees.

---

### UC-12 — Provide State to Parallel Workers

**Implementation notes** — `pipeline/infrastructure/mpihelpers.py`, class `Tier0PipelineTask`:

1. The MPI client saves the context to disk as a pickle: `context.save(path)`.
2. Task arguments are also pickled to disk alongside the context.
3. On the server, `get_executable()` loads the context, modifies `context.logs['casa_commands']` to a server-local temp path, creates the task's `Inputs(context, **task_args)`, then executes the task.
4. For `Tier0JobRequest` (lower-level distribution), the executor is shallow-copied *excluding* the context reference to stay within the pipeline-enforced MPI buffer limit (100 MiB). Comments in the code note CASA's higher native limit (~150 MiB; see PIPE-1337 / CAS-13656).

Note: preventing worker-side direct modifications is the behaviour and safety model used in the existing codebase. Allowing workers to write directly to shared processing state is a viable alternative design but requires explicit concurrency control, transactional commits, and conflict-resolution policies (for example optimistic/pessimistic locking, versioned updates, or a transaction log). Treating worker-side writes as a supported pattern is therefore a design decision with operational and testing implications and is more appropriately evaluated in the GAP analysis and architecture phases rather than assumed by the current context contract.

---

### UC-14 — Provide Read-Only State for Reporting

**Implementation notes** — `WebLogGenerator.render(context)` in `pipeline/infrastructure/renderer/htmlrenderer.py`:

- `WebLogGenerator.render(context)` explicitly does `context.results = [proxy.read() for proxy in context.results]` once before the renderer loop, so individual renderers iterate fully unpickled result objects rather than calling `read()` themselves
- Reads `context.report_dir`, `context.output_dir` — filesystem layout
- Reads `context.observing_run.*` — MS metadata, scheduling blocks, execution blocks, observers, project IDs, start/end times
- Reads `context.project_summary.telescope` — to determine telescope-specific page layouts (ALMA vs VLA vs NRO)
- Reads `context.project_structure.*` — OUS IDs, PPR file, recipe name
- The larger renderer stack, including the Mako templates under `pipeline/infrastructure/renderer/templates/`, reads `context.logs['casa_commands']` and related log references when generating weblog links

---

### UC-15 — Support QA Evaluation and Store Quality Assessments

**Implementation notes** — after `merge_with_context()`, `accept()` triggers `pipelineqa.qa_registry.do_qa(context, result)`:

- QA handlers implement `QAPlugin.handle(context, result)`
- The context provides inputs to QA evaluation:
  - Most handlers call `context.observing_run.get_ms(vis)` to look up metadata for scoring (antenna count, channel count, SPW properties, field intents)
  - Some handlers check `context.imaging_mode` to branch on VLASS-specific scoring
  - Others check things in `context.observing_run`, `context.project_structure`, or the callibrary (`context.callibrary`)
- Scores are appended to `result.qa.pool`, so the scores are stored on the results rather than directly on the context. This also keeps detailed QA collections scoped to the stage result that produced them; in current code, a `QAScorePool` can hold many `QAScore` objects, and each score may carry fine-grained `applies_to` selections (e.g. vis, field, SPW, antenna, polarization), so the per-result pool can become fairly large for detailed assessments.

QA handlers write scores to `result.qa.pool` and do not modify the shared context directly.

---

### UC-17 — Manage Telescope- and Array-Specific State

**Implementation notes** — the current codebase shows at least two different forms of telescope-/array-specific state.

One is a VLA-specific sub-context (`context.evla`) which is created during `hifv_importdata` and is updated by several subsequent tasks. Functionally, it provides a way to store observation metadata and pass state between tasks under `context.evla` rather than using the top-level context directly or other context objects (e.g. the domain objects). `context.evla` is an untyped, dictionary-of-dictionaries sidecar dynamically attached to the top-level context with no schema, no type annotations, and no declaration in `Context.__init__`.

`context.evla` is a `collections.defaultdict(dict)`, keyed as `context.evla['msinfo'][ms_name].<property>`:

- **Written by:** `hifv_importdata` (creates + initializes), `testBPdcals` (gain intervals, ignorerefant), `fluxscale/solint`, `fluxboot`
- **Read by:** Most VLA calibration tasks and heuristics
- Accessed fields include: `gain_solint1`, `gain_solint2`, `setjy_results`, `ignorerefant`, various `*_field_select_string` / `*_scan_select_string` values, `fluxscale_sources`, `spindex_results`, and many more

Another is ALMA TP / single-dish state, which is array-specific rather than telescope-wide and is carried mainly through SD-specific structures under `context.observing_run`, such as `ms_datatable_name`, `ms_reduction_group`, and `org_directions`, plus the per-MS `DataTable` products referenced from that state. This is a useful reminder that array-specific extensions do not always appear as a single sidecar object like `context.evla`; they may instead live in domain-model extensions and array-specific cached metadata products.

---

## Key Implementation References

- `Context` / `Pipeline`: `pipeline/infrastructure/launcher.py`
- CLI lifecycle tasks: `pipeline/h/cli/h_init.py`, `pipeline/h/cli/h_save.py`, `pipeline/h/cli/h_resume.py`
- Task dispatch & result acceptance: `pipeline/h/cli/utils.py`, `pipeline/infrastructure/basetask.py`
- PPR-driven execution loops:
  - ALMA: `pipeline/infrastructure/executeppr.py` (used by `pipeline/runpipeline.py`)
  - VLA: `pipeline/infrastructure/executevlappr.py` (used by `pipeline/runvlapipeline.py`)
- Direct XML procedure execution: `pipeline/recipereducer.py`
- MPI distribution: `pipeline/infrastructure/mpihelpers.py`
- QA framework: `pipeline/infrastructure/pipelineqa.py`, `pipeline/qa/`
- Weblog renderer: `pipeline/infrastructure/renderer/htmlrenderer.py`

---

## Context Lifecycle

The canonical flow through the context is:

1. **Create session** — `h_init()` constructs a `launcher.Pipeline(...)` and returns a new `Context`. In PPR-driven execution, `executeppr()` or `executevlappr()` also populates project metadata at this point.
2. **Load data** — Import tasks (`h*_importdata`) attach datasets to the context's domain model (`context.observing_run`, measurement sets, scans, SPWs, etc.).
3. **Execute tasks** — Tasks execute against the in-memory context and return a `Results` object. After each task, `Results.accept(context)` records the outcome and mutates shared state.
4. **Accept results** — Inside `accept()`, results are merged via `Results.merge_with_context(context)`. A `ResultsProxy` is pickled to disk per-stage to keep the in-memory context bounded. The weblog is typically rendered after each top-level stage.
5. **Save / resume** — `h_save()` pickles the context; `h_resume(filename='last')` restores it. Driver-managed breakpoints and developer debugging workflows rely on this cycle.
