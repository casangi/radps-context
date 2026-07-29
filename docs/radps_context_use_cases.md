# RADPS Context Use Cases

## Purpose

This document describes what the RADPS context must enable for its users and collaborating systems. It intentionally avoids specifying storage models, service topology, interface transports, record layouts, event structures, or other implementation choices. Those decisions belong in conceptual and detailed design documentation.

The use cases are derived from the current Pipeline context use cases and the requirement gaps identified in [requirements_and_ownership.md](requirements_and_ownership.md). Cross-references preserve that traceability. System-wide behavioral guarantees are defined separately in [radps_context_quality_requirements.md](radps_context_quality_requirements.md).

## Definitions

### Stakeholders

- **Operations**: People responsible for run setup, monitoring, intervention, reruns, restart decisions, storage control, and delivery.
- **Pipeline developers**: People who design, implement, debug, validate, or maintain pipeline and context-facing behavior.
- **QA reviewers**: People or groups responsible for assessing processing quality and release readiness.
- **Reporting consumers**: People who use manifests, reports, dashboards, or rendered summaries to understand processing.
- **Archive consumers**: People or systems responsible for packaging, ingesting, delivering, or retrieving archived products.
- **Audit and reproducibility consumers**: People or systems concerned with provenance, traceability, repeatability, or regression comparison.
- **Science support**: People who apply domain judgment to data-specific decisions and overrides.
- **Domain teams**: Groups responsible for observatory- or domain-specific metadata, policies, and processing state.
- **External system operators**: People responsible for systems that consume RADPS processing state or lifecycle information.

### Actors

- **Operator**: A person or automation acting on behalf of operations.
- **Workflow orchestration layer**: The system responsible for planning, scheduling, and coordinating pipeline work.
- **Processing worker**: A process that performs a unit of pipeline work and reports its outcome.
- **Ingest service**: A system that supplies new or incremental input data.
- **Archive import service**: A system that supplies previously generated products for continued processing.
- **Reporting or QA service**: A system that evaluates processing quality or produces reports and manifests.
- **Lifecycle manager**: A person or system applying retention and cleanup decisions.
- **External consumer**: A system or tool that reads processing state or lifecycle information.

## Use Cases

### RADPS-UC1 - Initialize or Resume a Run Context

| Field | Description |
|---|---|
| **Current Pipeline cross-references** | UC-03, UC-11, UC-12, UC-19 |
| **Stakeholders** | Operations, Pipeline developers, QA reviewers |
| **Actors** | Operator, Workflow orchestration layer |
| **Goal** | Establish the processing context for a new run, or recover an existing context so processing can continue. The result must be equivalent regardless of the orchestration driver used. |
| **Preconditions** | Required inputs are identifiable and the actor is permitted to create or access the run. |
| **Postconditions** | The run has a stable identity, its initial metadata and inputs are available, and authorized processing can begin or resume from the intended point. |

### RADPS-UC2 - Preserve Planned Work for Traceability

| Field | Description |
|---|---|
| **Current Pipeline cross-references** | UC-07, UC-08, UC-12 |
| **Stakeholders** | Pipeline developers, Operations, Audit and reproducibility consumers |
| **Actors** | Workflow orchestration layer |
| **Goal** | Preserve the planned units of work and their dependencies so execution outcomes can be interpreted in relation to the plan that produced them. |
| **Preconditions** | A run exists and planned work has been defined. |
| **Postconditions** | The applicable plan and the identity of its units of work can be determined throughout execution, reporting, and later review. |

### RADPS-UC3 - Track Work Execution and History

| Field | Description |
|---|---|
| **Current Pipeline cross-references** | UC-07, UC-08, UC-15, UC-17, UC-19 |
| **Stakeholders** | Operations, Pipeline developers, QA reviewers, Reporting consumers, Audit and reproducibility consumers |
| **Actors** | Workflow orchestration layer, Processing worker |
| **Goal** | Track the progress and outcome of each execution attempt, including retries and failures, and provide a coherent execution history. |
| **Preconditions** | The run and relevant planned work exist, and the actor is permitted to report execution state. |
| **Postconditions** | Current progress and prior attempts can be identified; failures retain sufficient diagnostic information; retries remain distinguishable from earlier attempts. |

### RADPS-UC4 - Register and Discover Produced Artifacts

| Field | Description |
|---|---|
| **Current Pipeline cross-references** | UC-06, UC-19, GAP-02 |
| **Stakeholders** | Operations, QA reviewers, Reporting consumers, Archive consumers, Audit and reproducibility consumers |
| **Actors** | Processing worker, Reporting or QA service |
| **Goal** | Make a produced artifact discoverable together with its type, producer, inputs, lineage, and accessible locations. |
| **Preconditions** | The artifact has been produced and can be identified. |
| **Postconditions** | Authorized consumers can find the artifact, determine how it relates to the run, and locate an accessible copy without relying on a producer's process-local path. |

### RADPS-UC5 - Establish a Safe Restart Boundary

| Field | Description |
|---|---|
| **Current Pipeline cross-references** | UC-12, GAP-04 |
| **Stakeholders** | Operations, Pipeline developers |
| **Actors** | Workflow orchestration layer, Operator |
| **Goal** | Identify a completed and internally consistent processing boundary from which work can safely resume. |
| **Preconditions** | The work and outputs required for the boundary have completed successfully. |
| **Postconditions** | The boundary can be identified later, and the state and artifacts required to resume from it are known to be complete. |

### RADPS-UC6 - Resume or Partially Re-run Processing

| Field | Description |
|---|---|
| **Current Pipeline cross-references** | UC-12, GAP-04, GAP-06 |
| **Stakeholders** | Operations, Pipeline developers, QA reviewers |
| **Actors** | Operator, Workflow orchestration layer |
| **Goal** | Resume a run or repeat a selected portion with changed inputs or parameters while preserving unaffected valid work. |
| **Preconditions** | A valid restart boundary or rerun scope can be identified, and the actor is permitted to request the operation. |
| **Postconditions** | Selected work can proceed again; dependent prior outputs are not treated as current when no longer valid; unaffected work remains available; the reason and scope are traceable. |

### RADPS-UC7 - Record Operator Annotations and Overrides

| Field | Description |
|---|---|
| **Current Pipeline cross-references** | UC-17, GAP-07, GAP-08 |
| **Stakeholders** | Operations, QA reviewers, Science support, Audit and reproducibility consumers |
| **Actors** | Operator |
| **Goal** | Associate notes, approvals, rationale, and authorized overrides with the affected run, data, work, or artifact. |
| **Preconditions** | The target exists and the operator is permitted to annotate or override it. |
| **Postconditions** | The decision and its rationale remain available to subsequent processing, reporting, and audit consumers. |

### RADPS-UC8 - Produce a Provenance Manifest or Report

| Field | Description |
|---|---|
| **Current Pipeline cross-references** | UC-15, UC-19, GAP-03, GAP-05 |
| **Stakeholders** | Operations, QA reviewers, Reporting consumers, Archive consumers, Audit and reproducibility consumers |
| **Actors** | Reporting or QA service |
| **Goal** | Produce a machine-readable manifest or human-readable report from durable processing state and known artifacts. |
| **Preconditions** | The information required by the selected output is available and the actor is permitted to access it. |
| **Postconditions** | The output identifies the run and source information from which it was produced and is available to its intended consumers. |

### RADPS-UC9 - Apply Artifact Retention and Cleanup Decisions

| Field | Description |
|---|---|
| **Current Pipeline cross-references** | UC-12, UC-19 |
| **Stakeholders** | Operations, Audit and reproducibility consumers |
| **Actors** | Lifecycle manager, Operator |
| **Goal** | Retain or remove artifacts according to policy without silently breaking required provenance, delivery, or resume workflows. |
| **Preconditions** | The relevant policy and artifact status are known, and the actor is permitted to apply the decision. |
| **Postconditions** | Consumers can determine whether an artifact remains available and why it was retained or removed; required processing and audit relationships remain valid. |

### RADPS-UC10 - Query Observation Metadata

| Field | Description |
|---|---|
| **Current Pipeline cross-references** | UC-01, UC-02, GAP-08 |
| **Stakeholders** | Pipeline developers, QA reviewers, Reporting consumers, Science support |
| **Actors** | Processing worker, Reporting or QA service, External consumer |
| **Goal** | Obtain the observation metadata needed for a specified dataset or data selection, including fields, spectral windows, scans, data types, and known cross-dataset relationships. |
| **Preconditions** | The requested data has been made available to the run and the actor is permitted to inspect it. |
| **Postconditions** | The actor receives a coherent view of the requested metadata or an unambiguous explanation that the selection cannot be resolved. |

### RADPS-UC11 - Update Calibration State

| Field | Description |
|---|---|
| **Current Pipeline cross-references** | UC-04 |
| **Stakeholders** | Pipeline developers, Operations, QA reviewers |
| **Actors** | Processing worker |
| **Goal** | Record calibration changes produced by a unit of work, including applicability, source lineage, and whether the calibration is pending, applied, superseded, or withdrawn. |
| **Preconditions** | The producing work and referenced calibration products can be identified. |
| **Postconditions** | Downstream consumers see one complete calibration-state outcome and can determine its applicability and provenance. |

### RADPS-UC12 - Update Imaging State

| Field | Description |
|---|---|
| **Current Pipeline cross-references** | UC-05 |
| **Stakeholders** | Pipeline developers, Operations, Reporting consumers |
| **Actors** | Processing worker |
| **Goal** | Record imaging targets, parameters, masks, thresholds, sensitivities, and other imaging state so later work can inspect or refine the applicable state. |
| **Preconditions** | The run and the data selection affected by the update can be identified. |
| **Postconditions** | Downstream consumers can determine the current imaging state for the intended selection and distinguish it from superseded state. |

### RADPS-UC13 - Provide Consistent State for QA, Reporting, and Debugging

| Field | Description |
|---|---|
| **Current Pipeline cross-references** | UC-15, UC-16, UC-17, UC-19 |
| **Stakeholders** | QA reviewers, Reporting consumers, Pipeline developers, Operations |
| **Actors** | Reporting or QA service, Operator, External consumer |
| **Goal** | Provide a coherent read-only view of processing state and artifacts at a meaningful processing boundary. |
| **Preconditions** | The run and requested boundary can be identified, and the actor is permitted to inspect them. |
| **Postconditions** | The actor can evaluate, report, compare, or diagnose the run without depending on worker-local memory or observing partial updates. |

### RADPS-UC14 - Discover Upstream Outputs

| Field | Description |
|---|---|
| **Current Pipeline cross-references** | UC-09 |
| **Stakeholders** | Pipeline developers, Operations |
| **Actors** | Processing worker, External consumer |
| **Goal** | Find the outputs that satisfy a downstream input requirement by their meaning and scope. |
| **Preconditions** | Upstream outputs have been identified and made available to the run. |
| **Postconditions** | The consumer can determine which output satisfies the requested input, or receives an unambiguous missing or ambiguous dependency result. |

### RADPS-UC15 - Audit Run Evolution

| Field | Description |
|---|---|
| **Current Pipeline cross-references** | UC-08, UC-17, GAP-05 |
| **Stakeholders** | Operations, Pipeline developers, QA reviewers, External system operators, Audit and reproducibility consumers |
| **Actors** | Operator, Workflow orchestration layer, Processing worker, External consumer |
| **Goal** | Determine how significant run state and lifecycle decisions changed over time, including who or what initiated each change. |
| **Preconditions** | A run exists and the actor is permitted to inspect its history. |
| **Postconditions** | Significant changes can be ordered, attributed, and related to the affected processing work and artifacts. |

### RADPS-UC16 - Manage Domain-Specific State

| Field | Description |
|---|---|
| **Current Pipeline cross-references** | UC-18 |
| **Stakeholders** | Domain teams, Pipeline developers, Operations |
| **Actors** | Processing worker, Workflow orchestration layer |
| **Goal** | Make observatory-, telescope-, or array-specific processing state available where required without requiring unrelated consumers to understand it. |
| **Preconditions** | The run's domain and the applicable state are identifiable. |
| **Postconditions** | Authorized domain consumers can update and obtain the state they require, while consumers outside that domain remain unaffected. |

### RADPS-UC17 - Support Distributed Concurrent Processing

| Field | Description |
|---|---|
| **Current Pipeline cross-references** | UC-09, UC-13, UC-14, GAP-01, GAP-02 |
| **Stakeholders** | Operations, Pipeline developers |
| **Actors** | Workflow orchestration layer, Processing worker |
| **Goal** | Allow independent work to execute concurrently across workers that may not share process memory or a filesystem, while preserving coherent processing state. |
| **Preconditions** | Each worker's required inputs and scope can be identified and accessed. |
| **Postconditions** | Each worker operates on a consistent view; incomplete results are not exposed as complete; conflicting outcomes are detected; accepted outcomes become visible before dependent work proceeds. |

### RADPS-UC18 - Provide Run State to External Systems

| Field | Description |
|---|---|
| **Current Pipeline cross-references** | UC-07, UC-15, UC-19, GAP-05 |
| **Stakeholders** | Operations, QA reviewers, Archive consumers, External system operators |
| **Actors** | External consumer |
| **Goal** | Obtain current processing state, lifecycle changes, and summary information in time to support monitoring and integration workflows. |
| **Preconditions** | The run exists and the external consumer is permitted to access the requested information. |
| **Postconditions** | The consumer receives current information or can determine that requested information is unavailable; temporary delivery failures do not silently discard required updates. |

### RADPS-UC19 - Capture Reproducibility Provenance

| Field | Description |
|---|---|
| **Current Pipeline cross-references** | UC-08, UC-17, UC-19, GAP-03 |
| **Stakeholders** | Operations, QA reviewers, Audit and reproducibility consumers |
| **Actors** | Processing worker, Reporting or QA service |
| **Goal** | Capture the inputs, parameters, software, execution environment, hardware and resource information, control decisions, and lineage needed to explain or reproduce processing outcomes. |
| **Preconditions** | The relevant work or exported product can be identified and provenance inputs are available. |
| **Postconditions** | Provenance remains associated with the applicable work and products; missing provenance is explicit rather than silently omitted. |

### RADPS-UC20 - Register Incremental Dataset Updates

| Field | Description |
|---|---|
| **Current Pipeline cross-references** | UC-01, UC-12, GAP-04 |
| **Stakeholders** | Operations, Pipeline developers, External system operators |
| **Actors** | Ingest service, Workflow orchestration layer |
| **Goal** | Add new or revised data to an active run and identify the processing affected by that change without overwriting prior valid results. |
| **Preconditions** | The run permits incremental input and the incoming data and its scope can be identified. |
| **Postconditions** | The new data is distinguishable from earlier versions; affected work can proceed; prior and newly produced results remain distinguishable and traceable. |

### RADPS-UC21 - Resolve Heterogeneous Cross-Dataset Matches

| Field | Description |
|---|---|
| **Current Pipeline cross-references** | UC-02, UC-18, GAP-08 |
| **Stakeholders** | Pipeline developers, Operations, Science support |
| **Actors** | Processing worker, Operator |
| **Goal** | Resolve corresponding spectral windows, fields, sources, and data columns across heterogeneous datasets using semantics appropriate to the consuming task. |
| **Preconditions** | The relevant datasets and metadata are available. |
| **Postconditions** | The consumer receives a resolved match or an explicit ambiguity; authorized overrides and their rationale remain available to later consumers. |

### RADPS-UC22 - Initialize from Intermediate Archival State

| Field | Description |
|---|---|
| **Current Pipeline cross-references** | UC-12, GAP-06 |
| **Stakeholders** | Operations, Archive consumers, External system operators |
| **Actors** | Archive import service, Operator, Workflow orchestration layer |
| **Goal** | Establish a valid mid-pipeline context from previously generated products so available work does not need to be repeated. |
| **Preconditions** | Source products and their provenance are identifiable and accessible, and the intended processing boundary is known. |
| **Postconditions** | The context contains all state required at the declared boundary, imported information remains linked to its sources, and processing can continue from that boundary. A partial or incompatible import is not exposed as valid. |

### RADPS-UC23 - Record Execution-Control Directives

| Field | Description |
|---|---|
| **Current Pipeline cross-references** | GAP-07 |
| **Stakeholders** | Operations, Pipeline developers, Audit and reproducibility consumers |
| **Actors** | Operator, Processing worker, Workflow orchestration layer |
| **Goal** | Record authorized directives to pause, skip, or reroute work together with their scope and rationale so orchestration decisions remain consistent and auditable. |
| **Preconditions** | The affected run, data, or work scope exists and the actor is permitted to issue the directive. |
| **Postconditions** | The directive remains available throughout its effective lifetime; the workflow can determine the required behavior; conflicts and overrides are explicit. |
