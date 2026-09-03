# Context Use Cases Mapping and RADPS Module Ownership

The use cases detailed in “Pipeline Context Use Cases” were derived from the current pipeline context. They were compiled to capture existing system needs and serve two primary purposes:

1. **Requirement Evaluation:** To identify context capabilities that could then be evaluated for inclusion in `radps-context` to satisfy RADPS requirements.  
2. **Knowledge Transfer:** To ensure valuable lessons learned from previous pipeline development are carried forward to RADPS when applicable, even if they do not map to a strict requirement. 

In the current pipeline, all of these capabilities are provided by a single context system with no separate Workflow Framework or other components: domain state, execution tracking, and inter-stage communication are all handled by this one system. A central goal of the RADPS redesign is to separate those concerns. This document builds off of the current pipeline use cases document by evaluating each use case on two fronts: whether it satisfies a related RADPS requirement, and if so, which logical component should be responsible for it -- `radps-context`, the Workflow Framework, or another defined subsystem.

An initial investigation of `xradio` was recorded only to clarify ownership and does not prescribe a technology choice. Interactions with a domain data-access library would be internal to `radps-context` and could affect how it obtains information such as observation metadata, but not what information it is responsible for providing.

`radps-context` will be a software component of RADPS responsible for maintaining a record of accepted domain processing outcomes — including observation metadata, calibration state, imaging state, QA and heuristic results, processing-output relationships, and state needed by downstream work — throughout the lifecycle of a Workflow run. The Workflow Framework will orchestrate the end-to-end Workflow and own work decomposition, scheduling, execution lifecycle, retries, checkpoint management, and non-domain execution history.

Based on this evaluation, the use cases are first mapped to RADPS requirements (Section 1). In Section 2, GAP use cases which are required by the RADPS requirements but were not covered by the current context use cases are enumerated. In Section 3, current context use cases not applicable to RADPS that will not be carried forward are documented. Finally, in Section 4, the applicable use cases and gaps are sorted into their designated responsible component. For use cases that were not cleanly separable between `radps-context` and the Workflow Framework, the responsibilities of each component are called out.

## 1. Context UCs and RADPS Requirements

This section lists the RADPS requirements associated with each current pipeline use case. 
Full descriptions of the associated use cases are available in the documents listed at the end of this file.

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
RADPS Requirements: ALMA-TR51.1, ALMA-TR51.2, ALMA-TR65
Related to ALMA-TR66, but this requirement is out of scope. 

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
| **Actor(s)** | Workflow Framework, task scheduler, workers |
| **Summary** | The Workflow must support horizontal and vertical scaling through data parallelism. The Workflow Framework schedules identifiable node tasks over dataset chunks so independent work can execute concurrently, while `radps-context` provides chunk-scoped state and accepts complete domain outcomes without allowing conflicting or partial state. This differs from the current parallel-worker pattern, which waits for all work to finish before proceeding. |
| **Invariant** | Each node task and data chunk remains identifiable; independent work may run asynchronously, but incompatible updates cannot produce conflicting state. |
| **Postconditions** | Accepted chunk-scoped outcomes remain distinguishable and can be combined deterministically before dependent work begins. |
| **RADPS requirements** | CSS9017, CSS9063, CSS9064.2, CSS9600 |
| **Notes** | GAP-01 covers all parallel/asynchronous execution functionality, which includes the current pipeline use cases UC-13 and UC-14. Dask is identified as a task-scheduler technology, but the logical responsibilities and `radps-context` interfaces do not depend on Dask-specific mechanisms. |

### GAP-02 — Distributed Execution Without a Shared Filesystem

| | |
|-------|---------|
| **Actor(s)** | Workflow Framework, distributed workers |
| **Summary** | Execution must be possible across nodes that do not share a filesystem. Processing outputs, datasets, and processing state must be addressable and accessible without relying on local paths. |
| **Postconditions** | Processing completes across distributed nodes with references in the context providing the necessary output access. |
| **RADPS requirements** | CSS9002, CSS9030 |

### GAP-03 — Provenance and Reproducibility

| | |
|-------|---------|
| **Actor(s)** | Workflow operator, auditor, reproducibility tooling |
| **Summary** | The system must record sufficient provenance to enable precise reproduction and audit of past runs. This provenance is of two kinds: domain-specific (which datasets, calibrations, and processing outputs were derived from which inputs), and execution-environment detail (software versions, node-task parameters, execution state, CPU architecture, node/cluster specification, kernel, workload-manager/scheduler configuration, and relevant scheduler limits). |
| **Postconditions** | Any past processing step can be reproduced or audited using the recorded provenance chain. |
| **RADPS requirements** | ALMA-TR103, ALMA-TR104, ALMA-TR105 |

### GAP-04 — Partial Re-execution / Targeted Node Task Re-run

| | |
|-------|---------|
| **Actor(s)** | Workflow operator, developer, Workflow Framework |
| **Summary** | Long-running processing must be decomposed into bounded work units, such as node tasks, with Checkpoint Records at suitable processing boundaries. `radps-context` makes the accepted domain state at a selected boundary available for rollback or restart, and the Workflow Framework uses the associated Checkpoint Record to rerun one or more node tasks with new parameters, invalidate or recompute dependent work, and preserve unaffected outcomes. |
| **Postconditions** | Context state reflects the accepted rerun outcomes; affected downstream work is invalidated or updated, and unaffected state remains intact. |
| **RADPS requirements** | CSS9038 |

### GAP-05 — External System Integration

| | |
|-------|---------|
| **Actor(s)** | External-interface subsystem, Workflow Framework |
| **Summary** | The Workflow must make timely processing information available to external systems without waiting for offline output files. The external-interface subsystem obtains domain state, including QA values and processing-output references, from `radps-context` through its Workflow-internal interface and obtains node-task lifecycle state from the Workflow Framework. |
| **Invariant** | `radps-context` remains responsible for domain state, the Workflow Framework remains responsible for node-task lifecycle state, and external interactions are handled by the external-interface subsystem. |
| **Postconditions** | External systems can obtain current processing information through the external-interface subsystem. |
| **RADPS requirements** | CSS9046, CSS9047, CSS9048, CSS9049, CSS9050, CSS9056 |

### GAP-06 — Initialization from Intermediate State

| | |
|-------|---------|
| **Actor(s)** | Workflow operator, archive ingest systems, Workflow Framework |
| **Summary** | The context must be initializable from pre-existing archival data so that it represents valid intermediate domain state for the Workflow. The Workflow Framework can associate that state with a Checkpoint Record, identify node tasks already covered by the state, and resume without reprocessing from scratch. |
| **Postconditions** | The context reflects valid intermediate domain state constructed from the supplied data. Separately, the Workflow Framework can identify and skip node tasks covered by that state. |
| **RADPS requirements** | CSS9038 |

### GAP-07 — Explicit Tag-Based Execution Control

| | |
|-------|---------|
| **Actor(s)** | Workflow operators, Workflow Framework, heuristics |
| **Summary** | The context must store metadata tags (e.g., `[PAUSE]`, `[SKIP]`) associated with datasets, node tasks, or processing scopes and make them available to the Workflow Framework for execution control. |
| **Invariant** | Tags affecting execution control are durably recorded in the context and remain readable by the Workflow Framework throughout the Workflow run. |
| **Postconditions** | Workflow execution is modified in accordance with persisted tags; any tag-driven halts or diversions are recorded alongside their rationale. |
| **RADPS requirements** | CSS9037 |

### GAP-08 — Heterogeneous Dataset Coordination and Flexible Matching Semantics

| | |
|-------|---------|
| **Actor(s)** | Data import tasks, calibration tasks, imaging tasks, heuristics, Workflow operators |
| **Summary** | Calibration tasks, imaging tasks, and heuristics must be able to match and coordinate data across heterogeneous collections of MSes that may not share native SPW numbering, column layout, or other assumptions. Downstream tasks must be able to select the matching semantics appropriate to their use: calibration tasks require exact SPW matching; imaging tasks require partial/overlap matching (including by frequency or channel range) to combine related spectral windows. Matching must extend beyond SPWs to cover fields, sources, and data column layouts. Where automated matching is ambiguous or fails, heuristics or users must be able to supply explicit mapping rules or override the default matching behavior, with overrides recorded alongside their rationale. |
| **Invariant** | SPW, field, source, and data-column identity are queryable across all registered datasets, regardless of whether those datasets share native numbering or column layout. |
| **Postconditions** | Downstream tasks can look up applicable SPWs, fields, sources, and data columns across an arbitrary collection of heterogeneous MSes using the appropriate matching semantics for their use, and any user or heuristic overrides are recorded alongside their rationale. |
| **RADPS requirements** | ALMA-TR07 |
| **Notes** | UC-02 covers the baseline cross-MS lookup capability currently supported by the context: a unified SPW identifier scheme with a single name-based matching strategy. GAP-08 extends this to multiple selectable matching semantics, additional metadata dimensions (fields, sources, column layouts), and user/heuristic override hooks — none of which are currently supported. |

## 3. Not Applicable to RADPS (Discarded)

These use cases reflect specific architectural choices made in the design of the current pipeline and are not applicable to the future design of RADPS. Similar functionality is now covered by GAP-01.

UC-13 — Provide State to Parallel Workers

UC-14 — Aggregate Results from Parallel Workers

## 4. Context Use Cases by Responsibility Allocation

### `radps-context` component only

These use cases do not have any obvious overlap with Workflow Framework functionality. While they *interact* with the Workflow Framework in some cases, the functionality will need to be implemented by the `radps-context` component.

UC-01 — Populate, Access, and Provide Observation Metadata  
UC-02 — Cross-MS Metadata Matching and Lookup  
UC-03 — Store and Provide Project-Level Metadata  
UC-04 — Register, Query, and Update Calibration State  
UC-05 — Manage Imaging State  
UC-10 — Provide a Transient Intra-Stage Workspace  
UC-16 — Support QA Evaluation and Store Quality Assessments  
UC-18 — Manage Telescope- and Array-Specific State  
UC-19 — Provide State for Product Export  
GAP-08 — Heterogeneous Dataset Coordination and Flexible Matching Semantics

### Workflow Framework only

UC-07 and UC-08 can be fully satisfied by the Workflow Framework and do not need to be implemented in `radps-context`.

UC-07 — Track Current Execution Progress  
Tracking current execution progress is core Workflow Framework functionality.

UC-08 — Preserve Per-Stage Execution Record  
The Workflow Framework maintains detailed node-task execution records as a standard capability.

### Workflow Framework and `radps-context` both

These use cases involve both the Workflow Framework and `radps-context`. Responsibilities of each component are indicated below:

**UC-06 — Register and Query Produced Image Products**

* **`radps-context`:** Handles the registration and querying of image-output references and their domain relationships.
* **Workflow Framework:** Coordinates the availability of those outputs to dependent node tasks.

**UC-09 — Propagate Task Outputs to Downstream Tasks**

* **`radps-context`:** Makes domain states and parameters available to downstream tasks.
* **Workflow Framework:** Makes processing outputs available to the dependent node tasks that require them.

**UC-11 — Support Multiple Orchestration Drivers**

* **`radps-context`:** Needs to be able to be instantiated and remain consistent across multiple different drivers.
* **Workflow Framework:** Selects the applicable orchestration path and initiates the required node tasks.

**UC-12 — Save and Restore a Processing Session**

* **`radps-context`:** Provides an identifiable, compatible version of accepted domain state for association with a Checkpoint Record and restores that state when requested.
* **Workflow Framework:** Manages the Checkpoint Record and determines which node tasks must be restarted when execution resumes or rolls back.

**UC-15 — Provide Read-Only State for Reporting**

* **`radps-context`:** Provides read-only access to accepted domain processing outcomes, including QA and heuristic results and domain provenance.
* **Workflow Framework:** Provides node-task lifecycle and non-domain execution history separately so reporting can combine it with context information.

**UC-17 — Support Inspection and Debugging**

* **`radps-context`:** Exposes the current domain state and domain-specific processing outputs (e.g., registered datasets and calibration tables) for inspection.
* **Workflow Framework:** Exposes node-task logs, tracebacks, and execution status and coordinates the ability to pause or debug a failing node task.

**GAP-01 — Asynchronous Execution of Independent Work**
* **`radps-context`:** Provides state scoped to identified node tasks and data chunks, prevents concurrent work from observing incomplete writes, and keeps accepted chunk-scoped outcomes distinguishable for deterministic combination.

* **Workflow Framework:** Schedules node tasks over data chunks, resolves dependencies, and coordinates independent work across available workers. It ensures dependent work does not begin until all required upstream outcomes have been committed to the context.

**GAP-02 — Distributed Execution Without a Shared Filesystem**

* **`radps-context`:** Stores location-independent references to processing outputs.
* **Workflow Framework:** Routes node tasks to workers and, in coordination with the underlying infrastructure, ensures the execution environment can resolve required output references before work begins.

**GAP-03 — Provenance and Reproducibility**

* **`radps-context`:** Stores domain-specific provenance and lineage persistently alongside processing-output references.
* **Workflow Framework:** Tracks non-domain execution history, including which node-task versions ran, the execution environment used, and the parameters passed at runtime.

**GAP-04 — Partial Re-execution / Targeted Node Task Re-run**

* **`radps-context`:** Provides an identifiable, serializable version of accepted domain state at a processing boundary.
* **Workflow Framework:** Associates that state with a Checkpoint Record, selects the rollback or restart boundary, and reruns only the required node tasks.

**GAP-05 — External System Integration**

* **`radps-context`:** Exposes a Workflow-internal interface through which the external-interface subsystem can read current domain state, including QA values and processing-output references.
* **Workflow Framework:** Owns information about when node tasks start, finish, or transition states and, if external lifecycle notifications are required, supplies that information to the external-interface subsystem.

**GAP-06 — Initialization from Intermediate State**

* **`radps-context`:** Uses pre-processed archival data to instantiate a valid intermediate domain state.
* **Workflow Framework:** Associates that state with a Checkpoint Record and skips node tasks whose accepted outcomes are already represented.

**GAP-07 — Explicit Tag-Based Execution Control**

* **`radps-context`:** Stores execution-control tags (e.g., `[PAUSE]`) so they can be persisted on datasets.  
* **Workflow Framework:** Queries these metadata tags before node-task execution and enforces the logic (e.g., halting the Workflow or altering reporting paths).

## Referenced documents:
The following documents were used to determine the relevant RADPS use cases:

* ALMA Data Processing Technical Requirements  
* CSS Stakeholder Needs – SDA, SDP, SIT, TI  
* Data Processing and Archive Workflow Stakeholder Needs  
* Computing and Software System Design Description: SDP
