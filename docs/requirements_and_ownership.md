# Context Use cases mapping and RADPS module ownership 

**Current Context Use Cases and RADPS requirements:**

The use cases detailed in “Pipeline Context Use Cases” were derived from the current pipeline context. They were compiled to capture existing system needs and serve two primary purposes:

1. **Requirement Evaluation:** To identify context capabilities that could then be evaluated for inclusion in `radps-context` to satisfy RADPS requirements.  
2. **Knowledge Transfer:** To ensure valuable lessons learned from previous pipeline development are carried forward to RADPS when applicable, even if they do not map to a strict requirement.

This document builds off of the current pipeline use cases document by evaluating each use case on two fronts: if each use case satisfies a related RADPS requirement, and if so, which architectural component should be responsible for its implementation. Currently, we are assessing implementation across `radps-context` and the Workflow Orchestration System (currently Prefect), with other layers potentially added in the future.

Based on this evaluation, the use cases are first mapped to RADPS requirements (Section 1). In Section 2, GAP use cases which are required by the RADPS requirements but were not covered by the current context use cases are enumerated. In Section 3, use cases not applicable to RADPS that will not be carried forward are documented. Finally, in Section 4, the applicable use cases and gaps are sorted into their designated implementation component.

## 1. Context UCs and RADPS Requirements

UC-01 — Populate, Access, and Provide Observation Metadata  
RADPS Requirements: ALMA-TR48, ALMA-TR107, CSS9018

UC-02 — Cross-MS Metadata Matching and Lookup  
RADPS Requirements: ALMA-TR07, ALMA-TR10

UC-03 — Store and Provide Project-Level Metadata  
RADPS Requirements: ALMA-TR48

UC-04 — Register, Query, and Update Calibration State  
RADPS Requirements: ALMA-TR53

UC-05 — Manage Imaging State  
RADPS Requirements: ALMA-TR53

UC-06 — Register and Query Produced Image Products  
RADPS Requirements: ALMA-TR51.1, ALMA-TR51.2, ALMA-TR65, ALMA-TR66

UC-07 — Track Current Execution Progress  
RADPS Requirements: CSS9037, CSS9034, CSS9064.1

UC-08 — Preserve Per-Stage Execution Record  
RADPS Requirements: CSS9051, ALMA-TR105, CSS9010

UC-09 — Propagate Task Outputs to Downstream Tasks  
RADPS Requirements: CSS9063, CSS9063.5

UC-10 — Provide a Transient Intra-Stage Workspace  
RADPS Requirements: ALMA-TR74, ALMA-TR24

UC-11 — Support Multiple Orchestration Drivers  
RADPS Requirements: ALMA-TR47, ALMA-TR31

UC-12 — Save and Restore a Processing Session  
RADPS Requirements: ALMA-TR29, ALMA-TR30, CSS9038, CSS9034

UC-13 — Provide State to Parallel Workers  
RADPS Requirements: CSS9600, CSS9064.2 *(Note: Discarded/Replaced, see Section 3)*

UC-14 — Aggregate Results from Parallel Workers  
RADPS Requirements: CSS9600, CSS9064.2 *(Note: Discarded/Replaced, see Section 3)*

UC-15 — Provide Read-Only State for Reporting  
RADPS Requirements: ALMA-TR50.4, ALMA-TR83

UC-16 — Support QA Evaluation and Store Quality Assessments  
RADPS Requirements: ALMA-TR49, ALMA-TR50

UC-17 — Support Inspection and Debugging  
RADPS Requirements: ALMA-TR27, ALMA-TR28, ALMA-TR112

UC-18 — Manage Telescope- and Array-Specific State  
RADPS Requirements: ALMA-TR07.1, ALMA-TR07.2, ALMA-TR08, ALMA-TR05, ALMA-TR03

UC-19 — Provide State for Product Export  
RADPS Requirements: ALMA-TR51, CSS9066

## 2. GAP Use Cases 

The following gap use cases capture critical system capabilities that are explicitly required or implied by RADPS requirements, but are not currently supported by the existing pipeline context design:

### GAP-01 — Asynchronous Execution of Independent Work

| | |
|-------|---------|
| **Actor(s)** | Workflow orchestration layer, parallel task scheduler |
| **Summary** | The context must support asynchronous execution at multiple granularities (stage-level and within-stage parallelism) while preventing inconsistent processing state. Tasks must be able to proceed independently without waiting for others to complete when task dependencies allow. This differs from the current parallel-worker pattern, which waits for all work to finish before proceeding. |
| **Invariant** | Independent tasks may run asynchronously but must not produce conflicting state. |
| **Postconditions** | Results from asynchronously executed tasks are fully and consistently incorporated into processing state before any dependent work begins. |
| **RADPS requirements** | CSS9017, CSS9063, CSS9064.2, CSS9600 |
| **Notes** | GAP-01 covers all parallel/asynchronous execution functionality, which includes the current pipeline use cases UC-13 and UC-14. |

### GAP-02 — Distributed Execution Without a Shared Filesystem

| | |
|-------|---------|
| **Actor(s)** | Workflow orchestration layer, distributed workers |
| **Summary** | Execution must be possible across nodes that do not share a filesystem. Artifacts, datasets, and processing state must be addressable and accessible without relying on local POSIX paths. |
| **Postconditions** | Processing completes across distributed nodes with context-hosted references providing the necessary artifact access. |
| **RADPS requirements** | CSS9002, CSS9030 |

### GAP-03 — Provenance and Reproducibility

| | |
|-------|---------|
| **Actor(s)** | Pipeline operator, auditor, reproducibility tooling |
| **Summary** | The context must record sufficient provenance – software versions, exact input identities/hashes, task parameters, per-stage state, hardware and execution-environment details (CPU architecture, node/cluster specification, kernel and MPI versions, workload-manager/scheduler configuration, and relevant scheduler limits) — to enable precise reproduction and audit of past runs. |
| **Postconditions** | Any past processing step can be reproduced or audited using the recorded provenance chain. |
| **RADPS requirements** | ALMA-TR103, ALMA-TR104, ALMA-TR105 |

### GAP-04 — Partial Re-execution / Targeted Stage Re-run

| | |
|-------|---------|
| **Actor(s)** | Pipeline operator, developer, workflow engine |
| **Summary** | The context must support selectively re-running one or more mid-pipeline stages with new parameters while preserving unaffected stages. Downstream stages that depend on changed outputs must be invalidated or recomputed. |
| **Postconditions** | Processing state reflects the re-run outcomes; affected downstream stages are invalidated or updated; unaffected stages remain intact. |
| **RADPS requirements** | CSS9038 |

### GAP-05 — External System Integration

| | |
|-------|---------|
| **Actor(s)** | QA dashboards, monitoring tools, archive ingest systems, schedulers |
| **Summary** | External systems need timely access to processing state (current stage, processing time, QA results, lifecycle events) without waiting for offline product files. The context should expose the necessary state via queryable interfaces or event feeds. |
| **Invariant** | External consumers can access the processing state they require while it remains current. |
| **Postconditions** | External systems can track processing progress and lifecycle transitions in near real time. |
| **RADPS requirements** | CSS9046, CSS9047, CSS9048, CSS9049, CSS9050, CSS9056 |

### GAP-06 — Initialization from Intermediate State

| | |
|-------|---------|
| **Actor(s)** | Pipeline operator, archive ingest systems, workflow engine |
| **Summary** | The context must be initializable from pre-existing archival products so that it appears as a valid mid-pipeline state. This allows the pipeline to skip stages whose outputs are already available (e.g., calibration products archived from a prior run) and resume execution from an intermediate point without reprocessing from scratch. |
| **Postconditions** | The context reflects a valid mid-pipeline state constructed from ingested archival products. Separately, the workflow engine can identify and skip stages covered by that state. |
| **RADPS requirements** | CSS9038 |

### GAP-07 — Explicit Tag-Based Execution Control

| | |
|-------|---------|
| **Actor(s)** | Pipeline operators, workflow orchestration layer, heuristics |
| **Summary** | The context must store metadata tags (e.g., `[PAUSE]`, `[SKIP]`) on datasets or pipeline stages that actively influence workflow execution and make the information available to the workflow orchestration system.|
| **Invariant** | Tags affecting execution control are durably recorded in the context and remain readable by the workflow layer throughout the pipeline run. |
| **Postconditions** | Workflow execution is modified in accordance with persisted tags; any tag-driven halts or diversions are recorded alongside their rationale. |
| **RADPS requirements** | CSS9037 |

### GAP-08 — Heterogeneous Dataset Coordination and Flexible Matching Semantics

| | |
|-------|---------|
| **Actor(s)** | Data import tasks, calibration tasks, imaging tasks, heuristics, pipeline operators |
| **Summary** | Calibration tasks, imaging tasks, and heuristics must be able to match and coordinate data across heterogeneous collections of MSes that may not share native SPW numbering, column layout, or other assumptions. Downstream tasks must be able to select the matching semantics appropriate to their use: calibration tasks require exact SPW matching; imaging tasks require partial/overlap matching (including by frequency or channel range) to combine related spectral windows. Matching must extend beyond SPWs to cover fields, sources, and data column layouts. Where automated matching is ambiguous or fails, heuristics or users must be able to supply explicit mapping rules or override the default matching behavior, with overrides recorded alongside their rationale.
| **Invariant** | SPW, field, source, and data-column identity are queryable across all registered datasets, regardless of whether those datasets share native numbering or column layout. |
| **Postconditions** | Downstream tasks can look up applicable SPWs, fields, sources, and data columns across an arbitrary collection of heterogeneous MSes using the appropriate matching semantics for their use, and any user or heuristic overrides are recorded alongside their rationale. |
| **RADPS requirements** | ALMA-TR07 |
| **Notes** | UC-02 covers the baseline cross-MS lookup capability currently supported by the context: a unified SPW identifier scheme with a single name-based matching strategy. GAP-08 extends this to multiple selectable matching semantics, additional metadata dimensions (fields, sources, column layouts), and user/heuristic override hooks — none of which are currently supported. |

## 3. Not Applicable to RADPS (Discarded)

These use cases reflect specific architectural choices made in the design of the current pipeline and are not applicable to the future design of RADPS. 

UC-13 — Provide State to Parallel Workers: This is replaced by stateless workers and asynchronous task graphs in GAP-01.

UC-14 — Aggregate Results from Parallel Workers: This is replaced by asynchronous task graphs and direct, independent artifact registration in GAP-01.

## 4. Context Use Cases by Implementation Area

### `radps-context` package only

These use cases do not have any obvious overlap with workflow orchestration functionality. While they may need to interact with the workflow orchestration in some cases, the functionality will need to be satisfied by the `radps-context` package. 

UC-01 — Populate, Access, and Provide Observation Metadata  
UC-02 — Cross-MS Metadata Matching and Lookup
UC-03 — Store and Provide Project-Level Metadata  
UC-04 — Register, Query, and Update Calibration State  
UC-05 — Manage Imaging State  
UC-10 — Provide a Transient Intra-Stage Workspace  
UC-15 — Provide Read-Only State for Reporting  
UC-16 — Support QA Evaluation and Store Quality Assessments  
UC-18 — Manage Telescope- and Array-Specific State
UC-19 — Provide State for Product Export  
GAP-08 — Heterogeneous Dataset Coordination

### Workflow orchestration layer only 

All of UC-07, UC-08, and GAP-01 can be fully satisfied by a workflow orchestration system such as Prefect and do not need to be implemented in `radps-context`. 

UC-07 — Track Current Execution Progress  
UC-08 — Preserve Per-Stage Execution Record  
GAP-01 — Asynchronous Execution of Independent Work  

### Workflow and `radps-context` both:

These use cases involve both the workflow manager system and `radps-context` components. Responsibilities of each component are indicated below:

**UC-06 — Register and Query Produced Image Products**

* **`radps-context`:** Handles the registration and querying of image products.  
* **Workflow system:** Image products themselves might be tracked as part of the artifact system.

**UC-09 — Propagate Task Outputs to Downstream Tasks**

* **`radps-context`:** Makes domain states and parameters available to downstream tasks.  
* **Workflow system:** Takes care of literally passing task outputs downstream to the relevant tasks that need them (e.g., via task results/futures).

**UC-11 — Support Multiple Orchestration Drivers**

* **`radps-context`:** Needs to be able to be instantiated and remain consistent across multiple different drivers.  
* **Workflow system:** Will likely be what is actually “driving” the various orchestration drivers and executing the pipeline tasks.

**UC-12 — Save and Restore a Processing Session**

* **`radps-context`:** Needs to provide the mechanism to save and restore the domain state.  
* **Workflow system:** Manages the actual resumption of the execution graph, tracking which tasks need to be restarted and picking up the execution flow from the restored point.

**UC-17 — Support Inspection and Debugging**

* **`radps-context`:** Exposes the current processing state and domain-specific artifacts (e.g., registered datasets, calibration tables) for inspection.  
* **Workflow system:** Exposes task logs, tracebacks, and execution status, and orchestrates the ability to pause or debug a failing node.

**GAP-03 — Provenance and Reproducibility**

* **`radps-context`:** Stores the domain-specific provenance lineage data persistently alongside the artifacts.  
* **Workflow system:** Tracks the actual execution history, which versions of tasks ran, the hardware used, and the parameters passed at runtime.

**GAP-04 — Partial Re-execution / Targeted Stage Re-run**

* **`radps-context`:** Creates a discrete, serializable snapshot of the domain state at any point in the pipeline.  
* **Workflow system:** Handles the invalidation of downstream tasks in the task graph and re-triggers only the necessary execution paths. Fetches the context snapshot from just before the stage to be executed.

**GAP-05 — External System Integration**

* **`radps-context`:** Serves as the queryable system holding the current state and QA values for external tools to read.  
* **Workflow system:** Actively pushes updates, fires webhooks, and notifies external dashboards when tasks start, finish, or transition states.

**GAP-06 — Initialization from Intermediate State**

* **`radps-context`:** Uses the pre-processed archival data to instantiate its state so it appears as a valid mid-pipeline state.  
* **Workflow system:** Is able to read this context and intelligently skip the prior stages (e.g., calibration) in the task graph. 

**GAP-07 — Explicit Tag-Based Execution Control**

* **`radps-context`:** Records information about metadata tags (e.g., `[PAUSE]`) so they can be persisted on datasets.  
* **Workflow system:** Queries these tags before task execution and actively enforces the logic (e.g., halting the workflow or altering reporting paths).

**Referenced documents:**   
The following documents were used to determine the RADPS use cases relevant to the pipeline context: 

* ALMA Data Processing Technical Requirements  
* CSS Stakeholder Needs – SDA, SDP, SIT, TI  
* Data Processing and Archive Workflow Stakeholder Needs  
* Computing and Software System Design Description: SDP