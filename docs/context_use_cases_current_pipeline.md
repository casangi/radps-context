# Pipeline Context Use Cases

## Overview
The pipeline `Context` is the central state object used for an entire pipeline execution. 
It carries observation data, calibration state, imaging state, execution history, 
and project metadata, and serves as the primary communication channel between pipeline stages.

This document catalogues the current use cases of the pipeline Context as determined by examination of the codebase. The goal is to inform the design of a system serving a similar role to the current pipeline Context for RADPS. It also identifies an initial set of use cases that the current design does not support but that are required or implied by RADPS requirements documentation.

For additional details about the current implementation, reference material, and exploratory future use cases, see [Supplementary Analysis](context_current_pipeline_appendix.md).

---

## 1. Use Cases

Each use case describes a need that the pipeline `Context` must satisfy. They are written to be implementation-neutral — the goal is to capture what the system must do, not 
how the current pipeline implementation achieves it. For pipeline-specific implementation details by use case, see [Implementation Notes by Use Case](context_current_pipeline_appendix.md#implementation-notes-by-use-case) in the appendix.

The following fields are used in each use case:

- **Actor(s):** The human or system role that directly creates, updates, consumes, or inspects the 
context state described by the use case. Actors are role categories, not specific task names or 
current implementations.
- **Summary:** What the system must do to satisfy the use case.
- **Invariant:** A condition that must always be true while the system is operating. Present only where a meaningful invariant exists.
- **Postcondition:** A condition that must be true after a specific operation completes. Present only where a meaningful postcondition exists.

---

### UC-01 — Load and Provide Access to Observation Metadata

| Field | Content |
|-------|---------|
| **Actor(s)** | Data import task, any downstream task, heuristics, renderers, QA handlers |
| **Summary** | The system must load observation metadata (datasets, spectral windows, fields, antennas, scans, time ranges) and make it queryable by all subsequent processing steps. It must also provide a unified identifier scheme when multiple datasets use different native numbering. |
| **Invariant** | All registered datasets remain queryable for the lifetime of the session without repeating the import process. |

---

### UC-02 — Store and Provide Project-Level Metadata

| Field | Content |
|-------|---------|
| **Actor(s)** | Initialization, any task, report generators |
| **Summary** | The system must store project-level metadata (proposal code, PI, telescope, desired sensitivities, processing recipe) and make it available to tasks for decision-making and to report generators for labelling outputs. |
| **Invariant** | Project metadata is available for the lifetime of the processing session. |

---

### UC-03 — Manage Execution Paths and Output Locations

| Field | Content |
|-------|---------|
| **Actor(s)** | Initialization, any task, report generators, export code |
| **Summary** | The system must centrally define and provide working directories, report directories, product directories, and logical filenames for logs, scripts, and reports. Tasks resolve file paths through these centrally managed locations. On session restore, paths must be overridable to adapt to a new environment. |
| **Invariant** | All tasks share a consistent set of paths for inputs and outputs. |

---

### UC-04 — Register and Query Calibration State

| Field | Content |
|-------|---------|
| **Actor(s)** | Calibration tasks |
| **Summary** | The system must allow calibration tasks to register solutions (indexed by data selection: field, spectral window, antenna, intent, time interval), and allow downstream tasks to query for all calibrations applicable to a given data selection. It must distinguish between calibrations pending application and those already applied. Registration must support transactional multi-entry updates — tasks often register multiple calibrations atomically within a single result acceptance. |
| **Invariant** | Calibration state is queryable and correctly scoped to data selections. |

---

### UC-05 — Accumulate Imaging State Across Multiple Steps

| Field | Content |
|-------|---------|
| **Actor(s)** | Imaging tasks, downstream heuristics, and export tasks |
| **Summary** | The system must allow imaging state — target lists, imaging parameters, masks, thresholds, and sensitivity estimates — to be computed by one step and read or refined by later steps. Multiple steps may contribute to a progressively refined imaging configuration. |
| **Invariant** | The accumulated imaging state reflects contributions from all completed imaging-related steps and is available to later imaging steps. |

---

### UC-06 — Register and Query Produced Image Products

| Field | Content |
|-------|---------|
| **Actor(s)** | Imaging tasks, export tasks, report generators |
| **Summary** | The system must maintain typed registries of produced image products with add/query semantics. Later tasks must be able to discover previously produced science, calibrator, RMS, and sub-product images through these registries. |
| **Invariant** | Produced image products are registered by type and remain queryable for downstream processing, reporting, and export. |

---

### UC-07 — Track Execution Progress and Stage History

| Field | Content |
|-------|---------|
| **Actor(s)** | Workflow orchestration layer, tasks, report generators, human operators |
| **Summary** | The system must track which processing step is currently executing and maintain a stable, ordered history of completed steps and their outcomes. This history must support reporting, script generation, and resumption after interruption. Per-stage tracebacks and timings must be preserved, and stage identity and ordering must remain coherent across resumes. |
| **Invariant** | The full execution history is retrievable in order; each recorded step retains its stage identity, outcome, timing, traceback information, and the arguments or effective parameters used to invoke it. |

---

### UC-08 — Propagate Task Outputs to Downstream Tasks

| Field | Content |
|-------|---------|
| **Actor(s)** | Any task producing output that subsequent tasks depend on |
| **Summary** | When a task produces outputs that change the processing state (e.g., new calibrations, updated flag summaries, image products, revised parameters), the system must provide a mechanism for those outputs to become available to subsequent processing steps. It must also retain those outputs as part of the execution record for later inspection, reporting, and export. These two needs may be satisfied through different access paths. |
| **Postconditions** | Downstream tasks can access the propagated processing state they need, and the task outputs are retained in the execution history for later retrieval. |

---

### UC-09 — Support Multiple Orchestration Drivers

| Field | Content |
|-------|---------|
| **Actor(s)** | Operations / automated processing (PPR-driven batch), pipeline developer / power user (interactive), recipe executor |
| **Summary** | The context is created and consumed by multiple front-ends: PPR command lists, XML procedures, or interactive task calls. Processing state must remain consistent and usable regardless of which driver created or resumed it. It must be creatable and resumable from non-interactive and interactive drivers, support driver-injected run metadata, tolerate partial execution controls (`startstage`, `exitstage`) and breakpoint-driven stop/resume semantics, and provide machine-detectable success/failure signals. |
| **Invariant** | Processing state is consistent and usable regardless of which orchestration driver created or resumed it. |

---

### UC-10 — Save and Restore a Processing Session

| Field | Content |
|-------|---------|
| **Actor(s)** | Pipeline operator, workflow orchestration layer, developers |
| **Summary** | The system must be able to serialize the complete processing state to disk (all observation data, calibration state, execution history, imaging state, project metadata, etc) and later restore it so that processing can resume from the saved point. The serialization must preserve enough state to resume; backward compatibility across pipeline releases is not guaranteed. On restore, paths must be adaptable to a new filesystem environment. |
| **Postconditions** | After restore, the processing state is operationally equivalent to the saved state for supported resume workflows, and processing can continue from the specified point. |

---

### UC-11 — Provide State to Parallel Workers

| Field | Content |
|-------|---------|
| **Actor(s)** | Workflow orchestration layer, parallel worker processes |
| **Summary** | When work is distributed across parallel workers, each worker needs read-only access to the current processing state (observation metadata, calibration state). The system must provide a mechanism for workers to obtain a consistent snapshot of that state. Workers must not be able to modify the shared processing state directly. The snapshot mechanism must support efficient distribution to workers. |
| **Invariant:**| Worker processes cannot modify shared processing state directly. |
| **Postconditions** | After distribution, each worker has a consistent, read-only view of the processing state for the duration of its work. |

---

### UC-12 — Aggregate Results from Parallel Workers

| Field | Content |
|-------|---------|
| **Actor(s)** | Workflow orchestration layer |
| **Summary** | After parallel workers complete, the system must collect their individual results and incorporate them into the shared processing state. The aggregation must be safe (no conflicting concurrent writes) and complete before the next sequential step begins. |
| **Postconditions** | The processing state reflects the combined outcomes of all parallel workers. |

---

### UC-13 — Provide Read-Only State for Reporting

| Field | Content |
|-------|---------|
| **Actor(s)** | Report generators (weblog, quality reports, reproducibility scripts, AQUA reports, pipeline statistics) |
| **Summary** | The system must provide reporting consumers with read-only access to the observation metadata, project metadata, execution history, QA outcomes, log references, and path information needed to generate reporting products such as weblogs, quality reports, reproducibility scripts, AQUA reports, and pipeline statistics. |
| **Postconditions** | Reports accurately reflect the processing state at the time of generation. |

---

### UC-14 — Support QA Evaluation and Store Quality Assessments

| Field | Content |
|-------|---------|
| **Actor(s)** | QA scoring framework, report generators, tasks that consult recorded QA outcomes |
| **Summary** | After each processing step completes, the system must make the relevant processing state available to QA handlers so they can evaluate the outcome against quality thresholds, which may depend on telescope, project parameters, or observation properties. The resulting quality scores must be recorded and remain retrievable for reporting and for later pipeline logic that consults recorded QA outcomes. |
| **Postconditions** | Quality scores are associated with the relevant processing step and accessible to reports and downstream logic. |

---

### UC-15 — Support Interactive Inspection and Debugging

| Field | Content |
|-------|---------|
| **Actor(s)** | Pipeline developer, pipeline operator, CI harnesses |
| **Summary** | The system must allow an operator to inspect the current processing state, for example: which datasets are registered, what calibrations exist, how many steps have completed, and what their outcomes were. On failure, a snapshot of the state must be available for post-mortem analysis. The system must provide deterministic paths and outputs that a test harness can validate, and must surface failures that happen outside of task execution, e.g. weblog rendering. |
| **Invariant** | The current processing state is inspectable at any point during execution, and sufficient information is retained to diagnose failures after the fact. |

---

### UC-16 — Manage Telescope-Specific Context Extensions

| Field | Content |
|-------|---------|
| **Actor(s)** | Telescope-specific tasks and heuristics |
| **Summary** | The system must support conditional telescope-specific extensions to the processing state. These extensions must be available to telescope-specific tasks and heuristics while remaining outside the assumed contract of shared pipeline code. Generic pipeline components must not depend on or require knowledge of telescope-specific extensions. |
| **Invariant** | Telescope-specific extensions are present only for runs that require them, available to the tasks that need them, and are never assumed by shared pipeline code. |

---

### UC-17 — Provide State for Product Export

| Field | Content |
|-------|---------|
| **Actor(s)** | Export task |
| **Summary** | The system must make the datasets, calibration state, image products, reports, scripts, path information, and project identifiers available through the processing state so export tasks can assemble them into a deliverable product package. |
| **Invariant** | The information needed to assemble a deliverable product package is accessible through the processing state. |

---

### UC-18 — Emit Lifecycle Notifications

| Field | Content |
|-------|---------|
| **Actor(s)** | Workflow orchestration layer, event subscribers (loggers, progress monitors) |
| **Summary** | The system must emit notifications at key lifecycle points (session start, session restore, step start, step completion, result acceptance) so that external observers (logging, progress reporting, live dashboards) can track execution without polling. |
| **Invariant** | Subscribers are notified of lifecycle transitions as they occur. |

---

## 2. Use Cases the Current Design Cannot Handle

The following use cases are not supported by the current context design but are required or strongly 
implied by RADPS requirement and design documentation. These use cases were identified through a first pass 
of the RADPS requirements documentation and are not exhaustive. A full gap analysis mapping current context use cases to RADPS requirements is a separate activity which is underway. These are numbered GAP-01 through GAP-04 to indicate gaps in the current design's capabilities.

Reviewer input on missing or incorrectly included items is welcome.

### GAP-01 — Concurrent Execution of Independent Work

| Field | Content |
|-------|---------|
| **Actor(s)** | Workflow orchestration layer, parallel task scheduler |
| **Summary** | The system must support concurrent execution of independent work at multiple granularities — both at the stage level, where independent stages execute simultaneously, and within a stage, where work is parallelized across processing axes such as MS or SPW. In both cases the system must ensure results are correctly incorporated into processing state without inconsistency. This is distinct from the existing parallel worker pattern (UC-11, UC-12), which distributes work within a single stage but requires all work to complete before the next stage can begin.|
| **Invariant** | Independent tasks are executed concurrently without producing inconsistent or incorrect processing state. | 
| **Postconditions** | Results from concurrently executed work are fully incorporated into processing state before any dependent work begins. |
| **RADPS Requirements** | CSS9017, CSS9063 |

---

### GAP-02 — Distributed Execution Without Shared Filesystem

| Field | Content |
|-------|---------|
| **Actor(s)** | Workflow orchestration layer, distributed workers |
| **Summary** | The system must support execution across nodes that do not share a filesystem. Processing state, artifacts, and datasets must be accessible to all participating nodes without relying on a shared local filesystem.|
| **Postconditions** | Processing completes correctly across distributed nodes without reliance on a shared filesystem. |
| **RADPS Requirements** | CSS9002, CSS9030 |

---

### GAP-03 — Provenance and Reproducibility

| Field | Content |
|-------|---------|
| **Actor(s)** | Pipeline operator, auditor, reproducibility tooling |
| **Summary** | The system must record sufficient provenance information — software versions, input data identity, task parameters, and processing state at each stage — to enable a past processing run to be precisely reproduced or audited.|
| **Postconditions** | Any past processing step can be precisely reproduced or audited from the recorded provenance chain, including the exact software and data versions that produced it. |
| **RADPS Requirements** | ALMA-TR103, ALMA-TR104, ALMA-TR105 |

---

### GAP-04 — Partial Re-Execution / Targeted Stage Re-Run

| Field | Content |
|-------|---------|
| **Actor(s)** | Pipeline operator, developer, workflow engine |
| **Summary** | The system must support selectively re-running a single mid-pipeline stage with different parameters while keeping earlier and later stages intact. Stages that depend on the re-executed stage's outputs must be invalidated or updated; stages that do not must be preserved. Note: CSS9038 explicitly requires re-start at discrete stages; dependency-aware invalidation of downstream stages is implied rather than explicitly stated.|
| **Postconditions** | After a targeted re-execution, processing state reflects the new outcome for the re-run stage, affected downstream stages are invalidated or updated, and unaffected stages are preserved. |
| **RADPS Requirements** | CSS9038 |

