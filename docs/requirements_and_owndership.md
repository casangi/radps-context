# Context Use cases mapping and RADPS module ownership 

**Current Context Use Cases and RADPS requirements:**

The use cases detailed in “Pipeline Context Use Cases” were derived from the current pipeline context. They were compiled to capture existing system needs and serve two primary purposes:

1. **Requirement Evaluation:** To identify context capabilities that could then be evaluated for inclusion in `radps-context` to satisfy RADPS requirements.  
2. **Knowledge Transfer:** To ensure valuable lessons learned from previous pipeline development are carried forward to RADPS when applicable, even if they do not map to a strict requirement.

This document evaluates each use case on two fronts: if each use case satisfies a related RADPS requirement, and if so, which architectural component should be responsible for its implementation. Currently, we are assessing implementation across `radps-context`, the Workflow Orchestration System (currently Prefect), and the xradio/MSv4 layer (with other layers potentially added in the future).

Based on this evaluation, the use cases are first mapped to RADPS requirements (Section 1). In Section 2, GAP use cases which are required by the RADPS requirements but were not covered by the current context use cases are enumerated. In Section 3, use cases not applicable to RADPS that will not be carried forward are documented. Finally, in Section 4, the applicable use cases and gaps are sorted into their designated implementation component.

## 1\. Context UCs and RADPS Requirements

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
RADPS Requirements: CSS9600, CSS9064.2 *(Note: Discarded/Replaced, see Section 3\)*

UC-14 — Aggregate Results from Parallel Workers  
RADPS Requirements: CSS9600, CSS9064.2 *(Note: Discarded/Replaced, see Section 3\)*

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

## 2\. GAP Use Cases 

These use cases are implied by RADPS Requirements and were not covered by existing use cases: 

* **GAP-01: Asynchronous Execution** — requires snapshot isolation, transactional merges, and conflict detection.  
* **GAP-02: Provenance and Reproducibility** — requires immutable per-attempt records, input hashing, and lineage capture.  
* **GAP-03: Partial Re-execution / Targeted Rerun** — requires explicit dependency tracking and invalidation semantics at the context level.  
* **GAP-04: External System Integration** — requires stable identifiers, event subscriptions/webhooks, and exportable summaries/manifests.  
* **GAP-05: Initialization from Intermediate State ("Warm Start")** — requires the ability to ingest disjointed archival products to construct a valid mid-pipeline state.  
* **GAP-06: Explicit Tag-Based Execution Control** — requires persisting metadata tags that actively inhibit state transitions or pause workflow orchestration.  
* **GAP-07: Compute Resource & Manual Intervention Book-keeping** — requires tracking execution duration, hardware telemetry, and manual workflow interruptions.

## 3\. Not Applicable to RADPS (Discarded)

These use cases reflect specific architectural choices made in the design of the current pipeline and are not applicable to the future design of RADPS. 

UC-13 — Provide State to Parallel Workers: This is placed by stateless workers and asynchronous task graphs in GAP-01.

UC-14 — Aggregate Results from Parallel Workers: This is replaced by asynchronous task graphs and direct, independent artifact registration in GAP-01

## 4\. Context use cases by implementation area

## radps-context package only

These use cases do not have any obvious overlap with workflow orchestration functionality. While they may need to interact with the workflow orchestration in some cases, the functionality will need to be satisfied by the radps-context package. 

UC-03 — Store and Provide Project-Level Metadata  
UC-04 — Register, Query, and Update Calibration State  
UC-05 — Manage Imaging State  
UC-10 — Provide a Transient Intra-Stage Workspace  
UC-15 — Provide Read-Only State for Reporting  
UC-16 — Support QA Evaluation and Store Quality Assessments  
UC-19 — Provide State for Product Export  
UC-18 — Manage Telescope- and Array-Specific State

## Workflow orchestration layer only 

All of UC-07, UC-08, GAP-01, and GAP-07 can be fully satisfied by a workflow orchestration system such as Prefect and do not need to be implemented in radps-context. 

UC-07 — Track Current Execution Progress  
UC-08 — Preserve Per-Stage Execution Record  
GAP-01 — Asynchronous Execution of Independent Work  
GAP-07 – Compute Resource & Manual Intervention Book-keeping

## Workflow and radps-context both:

These use cases involve both the workflow manager system and radps-context components. Responsibilities of each component are indicated below:

**UC-06 — Register and Query Produced Image Products**

* **radps-context:** Handles the registration and querying of image products.  
* **Workflow system:** Image products themselves might be tracked as part of the artifact system.

**UC-09 — Propagate Task Outputs to Downstream Tasks**

* **radps-context:** Makes domain states and parameters available to downstream tasks.  
* **Workflow system:** Takes care of literally passing task outputs downstream to the relevant tasks that need them (e.g., via task results/futures).

**UC-11 — Support Multiple Orchestration Drivers**

* **radps-context:** Needs to be able to be instantiated and remain consistent across multiple different drivers.  
* **Workflow system:** Will likely be what is actually “driving” the various orchestration drivers and executing the pipeline tasks.

**UC-12 — Save and Restore a Processing Session**

* **radps-context:** Needs to provide the mechanism save and restore the domain state.  
* **Workflow system:** Manages the actual resumption of the execution graph, tracking which tasks need to be restarted and picking up the execution flow from the restored point.

**UC-17 — Support Inspection and Debugging**

* **radps-context:** Exposes the current processing state and domain-specific artifacts (e.g., registered datasets, calibration tables) for inspection.  
* **Workflow system:** Exposes task logs, tracebacks, and execution status, and orchestrates the ability to pause or debug a failing node.

**GAP-02 — Provenance and Reproducibility**

* **radps-context:** Stores the domain-specific provenance lineage data persistently alongside the artifacts.  
* **Workflow system:** Tracks the actual execution history, which versions of tasks ran, the hardware used, and the parameters passed at runtime.

**GAP-03 — Partial Re-execution / Targeted Stage Re-run**

* **Radps-context:** Creates a discrete, serializable snapshot of the domain state at any point in the pipeline.  
* **Workflow system:** Handles the invalidation of downstream tasks in the task graph and re-triggers only the necessary execution paths. Fetches the context snapshot from just before the stage to be executed.

**GAP-04 — External System Integration**

* **radps-context:** Serves as the queryable system holding the current state and QA values for external tools to read.  
* **Workflow system:** Actively pushes updates, fires webhooks, and notifies external dashboards when tasks start, finish, or transition states.

**GAP-05: Initialization from Intermediate State ("Warm Start")**

* **radps-context:** Uses the pre-processed archival data to instantiate its state so it appears as a valid mid-pipeline state.  
* **Workflow system:** Is able to read this context and intelligently skip the prior stages (e.g., calibration) in the task graph. 

**GAP-06: Explicit Tag-Based Execution Control**

* **radps-context:** Records information about metadata tags (e.g., `[PAUSE]`) so they can be persisted on datasets.  
* **Workflow system:** Queries these tags before task execution and actively enforces the logic (e.g., halting the workflow or altering reporting paths).

## radps-context and xradio / MSv4: 

These use cases will need to be implemented in the `radps-context` package, but it seemed worth pointing out that due to the design of xradio / MSv4, some of the heavy lifting for metadata access, cross-dataset matching, and memory representation will likely be handled natively by xarray datasets and the MSv4 storage schema. Therefore, `radps-context` may not need to duplicate or manage as much low-level observation metadata as the current pipeline context does; instead, it can act as a lightweight coordinator that directly leverages xradio's built-in, self-describing data structures**.**

UC-01 — Populate, Access, and Provide Observation Metadata  
UC-02 — Cross-MS Metadata Matching and Lookup

**Referenced documents:**   
The following documents were used to determine the RADPS use cases relevant to the pipeline context: 

* ALMA Data Processing Technical Requirements  
* CSS Stakeholder Needs – SDA, SDP, SIT, TI  
* Data Processing and Archive Workflow Stakeholder Needs  
* Computing and Software System Design Description: SDP