# Pipeline Context Use Cases

## Overview
The pipeline context is the central state object used for a pipeline execution. It carries observation metadata, calibration state, imaging state, execution history and state, project metadata, and serves as the primary communication channel between pipeline stages.

This document catalogues the use cases of the current pipeline context as determined by examination of the codebase. The goal is to inform the design of a system serving a similar role to the current pipeline context for RADPS.

For additional details about the current implementation and reference material, see [Supplementary Analysis](context_current_pipeline_appendix.md).

---

## Use Cases

Each use case describes the required capabilities of the context system and the interactions through which those capabilities are exercised. They are written to be implementation-neutral — the goal is to capture what the context must do, not how the current pipeline implementation achieves it. For pipeline-specific implementation details by use case, see [Implementation Notes by Use Case](context_current_pipeline_appendix.md#implementation-notes-by-use-case) in the appendix.

The following fields are used in each use case:

- **Actor(s):** The human or system role that directly creates, updates, consumes, or inspects the context state described by the use case. Actors are role categories, not specific task names or current implementations.
- **Summary:** What the context must do to satisfy the use case.
- **Invariant:** A condition that must always be true while the system is operating. Present only where a meaningful invariant exists.
- **Postcondition:** A condition that must be true after a specific operation completes. Present only where a meaningful postcondition exists.

---

### UC-01 — Populate, Access, and Provide Observation Metadata

| | |
|-------|---------|
| **Actor(s)** | Data import task, any downstream task, heuristics, renderers, QA handlers |
| **Summary** | The context must populate (read) observation metadata (datasets, spectral windows, fields, antennas, scans, time ranges), make it queryable by all subsequent processing steps, and allow downstream tasks to access that metadata as processing progresses. When processing produces derived or transformed datasets (for example, calibrated, averaged, or subsetted outputs), the context should register those derived datasets and record lineage rather than mutating the original observation metadata. It must also be able to hold derived or cached metadata products created during import so later stages can reuse them efficiently. |
| **Invariant** | All populated datasets and derived dataset records remain queryable for the lifetime of the session without repeating the import process. |

---

### UC-02 — Cross-MS Metadata Matching and Lookup
| | |
|-------|---------|
| **Actor(s)** | Calibration tasks, imaging tasks, heuristics
| **Summary** | When multiple MSes are registered in the context, downstream tasks must be able to look up and match metadata elements across them even when the MSes use different native numbering. The context must provide a unified identifier scheme (currently for spectral windows) that allows these elements to be referenced consistently across datasets, and must support data-type-aware lookup of MSes and their associated data columns.
| **Postcondition** | Downstream tasks can resolve applicable metadata across registered MSes using a unified identifier scheme, and can look up MSes and data columns by data type.

---

### UC-03 — Store and Provide Project-Level Metadata

| | |
|-------|---------|
| **Actor(s)** | Initialization, any task, report generators |
| **Summary** | The context must store project-level metadata (proposal code, PI, telescope, desired sensitivities, etc.) and make it available to all components, e.g., for decision-making in heuristics and to label outputs in reports. |
| **Invariant** | Project metadata is available for the lifetime of the processing session. |

---

### UC-04 — Register, Query, and Update Calibration State

| | |
|-------|---------|
| **Actor(s)** | Calibration tasks, export tasks, restore tasks, report generators |
| **Summary** | The context must allow calibration tasks to register and update solutions and to record both the coverage where a solution applies and the source(s) from which it was derived. Downstream tasks must be able to query for all calibrations applicable to a given data selection. The context must distinguish between calibrations pending application and those already applied. Registration must support registering multiple calibrations atomically as part of a single operation, and it must also support de-registration/removal of trial or reverted calibrations so alternative solutions can be tested or failed attempts rolled back. |
| **Invariant** | Calibration state is queryable and correctly scoped to data selections, and can be updated as processing progresses. |

---

### UC-05 — Manage Imaging State

| | |
|-------|---------|
| **Actor(s)** | Imaging tasks, downstream heuristics, and export tasks |
| **Summary** | The context must allow imaging state — target lists, imaging parameters, masks, thresholds, and sensitivity estimates — to be computed by one step and read or refined by later steps. Multiple steps may contribute to a progressively refined imaging configuration. |
| **Invariant** | Imaging state reflects contributions from all completed imaging-related stages, and is available for reading or refinement by subsequent stages. |

---

### UC-06 — Register and Query Produced Image Products

| | |
|-------|---------|
| **Actor(s)** | Imaging tasks, export tasks, report generators |
| **Summary** | The context must maintain typed registries of produced image products with add/query semantics. Later tasks must be able to discover previously produced science, calibrator, RMS, and sub-product images through these registries. |
| **Invariant** | Produced image products are registered by type and remain queryable for downstream processing, reporting, and export. |

---

### UC-07 — Track Current Execution Progress

| | |
|-------|---------|
| **Actor(s)** | Workflow orchestration layer, tasks, pipeline operators |
| **Summary** | The context must track which processing stage is currently executing and maintain a stable, ordered record of completed stages. Stage identity and ordering must remain coherent across session saves and resumes. |
| **Invariant** | The currently executing stage is identifiable and completed stages are recorded in stable order. |

___

### UC-08 — Preserve Per-Stage Execution Record

| | |
|-------|---------|
| **Actor(s)** | Report generators, pipeline operators, workflow orchestration layer |
| **Summary** | The context must preserve a complete execution record for each completed stage, including timing, traceback information, outcomes, and the arguments used to invoke it. This record must support reporting, post-mortem diagnosis of failures, and resumption after interruption. |
| **Invariant** | Each completed stage retains its full execution record, including identity, outcome, timing, traceback, and invocation arguments, for the lifetime of the session. |

---

### UC-09 — Propagate Task Outputs to Downstream Tasks

| | |
|-------|---------|
| **Actor(s)** | Any task producing output, downstream tasks |
| **Summary** | When a task produces outputs that change the processing state (e.g., new calibrations, updated flag summaries, image products, revised parameters), the context must provide a mechanism for those outputs to become available to subsequent processing steps before they execute. UC-04, UC-05, UC-06, and UC-16 are specific instances of this pattern. |
| **Postconditions** | Downstream tasks can access the propagated processing state they need. |

---

### UC-10 — Provide a Transient Intra-Stage Workspace

| | |
|-------|---------|
| **Actor(s)** | Aggregate tasks, child tasks, task execution framework |
| **Summary** | Within a stage, the context must be usable as a temporary working space for child-task execution. Child tasks must be able to modify context state destructively while they run, including adding, removing, or replacing tentative calibration and processing state, without requiring explicit cleanup logic. Only outputs that are explicitly accepted into the enclosing task's context should survive stage execution. |
| **Invariant** | State changes made while executing against a temporary child-task context do not escape that workspace unless they are explicitly accepted and merged. |
| **Postcondition** | When a child task finishes, the enclosing task retains only the accepted state changes; unaccepted mutations to the temporary workspace are discarded. |

---
### UC-11 — Support Multiple Orchestration Drivers

| | |
|-------|---------|
| **Actor(s)** | Operations / automated processing (PPR-driven batch), pipeline developer / power user (interactive), recipe executor |
| **Summary** | The state stored by the context must remain consistent and usable regardless of which orchestration driver created or resumed it. It must be creatable and resumable from non-interactive and interactive drivers (ex: PPR command lists, XML procedures, interactive task calls), support driver-injected run metadata, and tolerate partial execution controls and breakpoint-driven stop/resume semantics. |
| **Invariant** | Processing state is consistent and usable regardless of which orchestration driver created or resumed it, and success/failure signals are produced when appropriate. |

---

### UC-12 — Save and Restore a Processing Session

| | |
|-------|---------|
| **Actor(s)** | Pipeline operator, workflow orchestration layer, pipeline developer |
| **Summary** | The context must be able to serialize the complete processing state to disk (all observation data, calibration state, execution history, imaging state, project metadata, etc.) and later restore it so that processing can resume from the saved point, provided the data file state is consistent with the context snapshot. The serialization must preserve enough state to resume; backward compatibility across pipeline releases is not guaranteed. On restore, paths must be adaptable to a new filesystem environment. |
| **Postcondition** | After restore, the processing state is operationally equivalent to the saved state for supported resume workflows, and processing can continue from the specified point, assuming the data files are in a state consistent with the snapshot used to create the saved context. |

---

### UC-13 — Provide State to Parallel Workers

| | |
|-------|---------|
| **Actor(s)** | Workflow orchestration layer, parallel worker processes |
| **Summary** | When work is distributed across parallel workers, each worker needs access to the current processing state (observation metadata, calibration state). The context must provide a mechanism for workers to obtain a consistent snapshot of that state, and the snapshot mechanism must support efficient distribution to workers. |
| **Postcondition** | After distribution, each worker has a consistent view of the processing state for the duration of its work. |

---

### UC-14 — Aggregate Results from Parallel Workers

| | |
|-------|---------|
| **Actor(s)** | Workflow orchestration layer |
| **Summary** | After parallel workers complete, the context must collect their individual results and incorporate them into the shared processing state. The aggregation must be safe (no conflicting concurrent writes) and complete before the next sequential step begins. |
| **Postconditions** | The processing state reflects the combined outcomes of all parallel workers. |

---

### UC-15 — Provide Read-Only State for Reporting

| | |
|-------|---------|
| **Actor(s)** | Report generators (weblog, quality reports, reproducibility scripts, pipeline statistics) |
| **Summary** | The context must provide read-only access to the observation metadata, project metadata, execution history (including per-stage domain-specific outputs such as flag summaries and plot references), QA outcomes, log references, and path information needed to generate reporting products such as weblogs, quality reports, reproducibility scripts, and pipeline statistics. |
| **Postconditions** | Reports accurately reflect the processing state at the time of generation. |

---

### UC-16 — Support QA Evaluation and Store Quality Assessments

| | |
|-------|---------|
| **Actor(s)** | QA scoring framework, report generators |
| **Summary** | After each processing step completes, the context must make the relevant processing state available to QA handlers so they can evaluate the outcome against quality thresholds, which may depend on e.g. telescope, project parameters, or observation properties. The resulting quality scores must be recorded and remain retrievable for reporting. |
| **Postconditions** | Quality scores are associated with the relevant processing step and accessible to reports. |

---

### UC-17 — Support Inspection and Debugging

| | |
|-------|---------|
| **Actor(s)** | Pipeline developer, pipeline operator, CI harnesses |
| **Summary** | The context must allow an operator to inspect the current processing state, for example: which datasets are registered, what calibrations exist, how many steps have completed, and what their outcomes were. On failure, a snapshot of the state must be available for post-mortem analysis. |
| **Invariant** | The current processing state is inspectable at any point during execution, and sufficient information is retained to diagnose failures after the fact. |

---

### UC-18 — Manage Telescope- and Array-Specific State

| | |
|-------|---------|
| **Actor(s)** | Telescope-specific tasks and heuristics, array-specific tasks and heuristics |
| **Summary** | The context must support conditional telescope-specific and array-specific extensions to the processing state. These extensions must be available to the tasks and heuristics that need them, including cases where one array mode within a telescope family has materially different state requirements from another. Generic pipeline components must not depend on or require knowledge of those telescope- or array-specific extensions. |
| **Invariant** | Telescope- and array-specific extensions are present only for runs that require them, available to the tasks that need them, and are never assumed by shared pipeline code. |

---

### UC-19 — Provide State for Product Export

| | |
|-------|---------|
| **Actor(s)** | Export tasks |
| **Summary** | The context must make the datasets, calibration state, image products, reports, scripts, path information, project identifiers, and any other information needed for export available through the processing state so export tasks can assemble them into a deliverable product package. |
| **Invariant** | The information needed to assemble a deliverable product package is accessible through the processing state. |
