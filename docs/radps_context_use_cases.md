# RADPS Context Use Cases

This document defines RADPS context use cases. It focuses on domain state, processing-output information, and provenance rather than the orchestration performed by the Workflow Framework or the behavior of the entire Workflow.

## Use-case template

Adapted from “Use Case Modeling” by Kurt Bittner and Ian Spence.

This template is tuned for `radps-context` behavior. A use case should identify:

- the stakeholder need served by the use case, including needs mediated through another Workflow component
- the direct interaction with `radps-context`
- the observable domain-state or processing-output outcome
- any conditions required before the interaction
- relevant failure, retry, consistency, or ownership boundaries

See also:

- [Requirements and ownership](requirements_and_ownership.md) for current Pipeline UC traceability and responsibility allocation
- [Current Pipeline context use cases](context_use_cases_current_pipeline.md) for the source use cases
- [RADPS context quality requirements](radps_context_quality_requirements.md) for cross-cutting behavioral guarantees
- [Glossary](glossary.md) for shared terminology

Use the following structure. Stakeholders may be omitted when the direct actors are also the immediate beneficiaries. Preconditions, alternative flows, and boundaries may be omitted when they do not apply.

```markdown
### RADPS-UC<number> — <title>

**Current Pipeline cross-references:** <related current Pipeline use cases and GAPs>

**Stakeholders:** <human or system roles whose needs are served>

**Direct actors:** <logical roles that interact directly with `radps-context`>

**Goal:** <outcome sought by the stakeholders or direct actors>

**Preconditions:** <conditions that must hold before the interaction>

**Outcome:** <observable domain state, processing-output information, or traceability available after completion>

**Alternative flows:** <errors, retries, conflicts, or other deviations>

**Boundary:** <related behavior owned by the Workflow Framework or another component>
```

## Scope

These use cases define the domain-state operations that `radps-context` exposes to components inside the RADPS Workflow. They include the needs of users, operators, developers, and external systems even when another Workflow component mediates the interaction. Identifying a stakeholder does not imply that `radps-context` provides a user interface or direct external interface; interfaces from the Workflow to external systems remain outside this document's scope.

## Roles

### Stakeholders

- **Workflow user**: Requests science processing, supplies applicable science or project information, or consumes the resulting information and products.
- **Workflow operator**: Initiates, configures, monitors, recovers, or otherwise controls Workflow execution through Workflow-owned interfaces.
- **Workflow developer**: Develops, tests, or diagnoses Workflow processing and its domain-state interactions.
- **External consumer**: A person or system that obtains Workflow information or products through an external-interface subsystem rather than by calling `radps-context` directly.

### Direct actors

- **Workflow Framework**: Orchestrates the Workflow, supplies identifiers that correlate domain state with work, and manages decomposition, scheduling, retries, and checkpoint use without performing domain-specific processing.
- **Worker**: Executes a node task, reads context state, writes processing outputs, and submits complete domain-state updates.
- **Node task**: Invokes processing functions as a unit of work assignable to a node, consuming and producing data chunks or intermediate artifacts. Examples include data import, calibration, imaging, QA evaluation, and output preparation.
- **Heuristic**: Reads domain state and applies configured domain rules or algorithms to derive processing decisions or propose mappings between related metadata elements.
- **Reporting component**: An internal component that retrieves context information to generate reports for Workflow stakeholders.
- **External-interface subsystem**: A component that retrieves context information through an internal Workflow interface and handles communication with external consumers.
- **Diagnostic client**: An internal Workflow diagnostic or test component used by developers, operators, or CI systems to retrieve context state for inspection and diagnosis.

## Use cases

### RADPS-UC1 — Initialize or load a run context

**Current Pipeline cross-references:** UC-03, UC-11, UC-12.

**Stakeholders:** Workflow operator.

**Direct actors:** Workflow Framework, data import node task.

**Goal:** Establish a run with stable identity and initial domain information, or load compatible persisted context state for internal Workflow use.

**Preconditions:** A direct actor supplies an internal run identity, input dataset identities, and applicable policy versions, together with either any initial project information needed to initialize the run or the persisted context state to restore.

**Outcome:** The run and its initial domain state or resumed persisted context are available to internal Workflow components. The creating component, creation time, inputs, and context-model version remain identifiable.

**Alternative flows:** An incompatible context version or missing required initialization information is rejected explicitly. Repeating initialization for the same run with equivalent information returns the existing run context; attempting to reuse the run identity with conflicting information is rejected.

### RADPS-UC2 — Provide observation metadata

**Current Pipeline cross-references:** UC-01.

**Stakeholders:** Workflow user.

**Direct actors:** Node task, worker, heuristic.

**Goal:** Register and retrieve initial or incremental dataset and observation information needed by node tasks, including a stable identifier for each dataset version and lineage from transformed datasets to their sources.

**Outcome:** Internal consumers can obtain a coherent view of the requested datasets, fields, spectral windows, scans, antennas, time ranges, data types, and derived metadata outputs. A newly accepted dataset version remains distinguishable from prior versions so the Workflow Framework can determine any affected work.

**Alternative flows:** A request that does not identify a known, unique dataset, dataset version, or observation metadata element returns an explicit error.

### RADPS-UC3 — Provide project metadata

**Current Pipeline cross-references:** UC-03.

**Stakeholders:** Workflow user, Workflow operator, external consumer.

**Direct actors:** Workflow Framework, data import node task, node task, worker, heuristic.

**Goal:** Register run-scoped project information during initialization or import and make it available to internal consumers throughout the run.

**Outcome:** Internal consumers can retrieve a consistent view of project properties such as the proposal code, principal investigator, telescope, intended sensitivities, and beam requirements. Once established, the project metadata remains unchanged for the lifetime of the run.

**Alternative flows:** Missing required project information or an attempt to replace established project metadata with conflicting information returns an explicit error. Repeating registration with equivalent information returns the existing project metadata.

### RADPS-UC4 — Resolve heterogeneous dataset matches

**Current Pipeline cross-references:** UC-02, UC-18; GAP-08.

**Stakeholders:** Workflow user, Workflow operator.

**Direct actors:** Worker, heuristic, node task, Workflow Framework.

**Goal:** Resolve corresponding fields, sources, spectral windows, and data columns across datasets using a declared matching mode, including an explicitly supplied override when automatic matching is insufficient.

**Outcome:** The consumer receives the resolved match set. Accepted overrides retain their scope, rationale, source reference, and supersession history.

**Alternative flows:** If the declared matching mode produces multiple valid candidates, `radps-context` returns a structured ambiguity result containing those candidates without accepting a match. A conflicting or invalid override returns a structured error and leaves the accepted state unchanged.

**Boundary:** The Workflow Framework determines how to resolve an ambiguous result, such as applying a configured override, invoking a heuristic, requesting operator input when available, or failing the affected work. `radps-context` does not choose among ambiguous candidates.

### RADPS-UC5 — Apply a calibration-state update

**Current Pipeline cross-references:** UC-04.

**Stakeholders:** Workflow user.

**Direct actors:** Worker, calibration node task.

**Goal:** Atomically register a complete set of calibration changes, their applicability, and related processing outputs.

**Outcome:** Internal consumers observe either the preceding calibration-state version or the new version, never a partial mixture. The update remains linked to its internal producer, inputs, and processing outputs.

**Alternative flows:** An incompatible concurrent update is rejected so the producer can recompute against a current view.

**Boundary:** `radps-context` detects and rejects incompatible concurrent updates but does not merge them or initiate retries. The Workflow Framework decides whether and when to retry or fail the affected work; on retry, the producing node task recomputes its update against a current state view.

### RADPS-UC6 — Apply an imaging-state update

**Current Pipeline cross-references:** UC-05, UC-06.

**Stakeholders:** Workflow user, external consumer.

**Direct actors:** Worker, imaging node task.

**Goal:** Record imaging state and image-output references for a declared dataset or processing scope.

**Outcome:** The accepted imaging-state version and associated image outputs are available to dependent node tasks and linked to their producer and inputs.

**Alternative flows:** Invalid scope, inconsistent state, or unavailable required outputs cause the complete update to be rejected.

### RADPS-UC7 — Register and resolve processing outputs with domain lineage

**Current Pipeline cross-references:** UC-06, UC-19; GAP-02, GAP-03.

**Stakeholders:** Workflow user, external consumer.

**Direct actors:** Worker, node task.

**Goal:** Register the identity, type, lineage, and location-portable references of a processing output produced or adopted by a node task.

**Preconditions:** The processing output has been produced and its location is known.

**Outcome:** Internal components can resolve the processing output by stable identity, type, processing scope, or lineage and trace it to the accepted update that registered it, the node task that produced or adopted it, and its inputs.

**Alternative flows:** Registration fails if required references cannot be validated. If an equivalent registration with the same update identity was already accepted, `radps-context` returns the existing registration and its stable processing-output identity without creating another processing output or duplicating its lineage relationships. Reusing the update identity with different registration information is rejected.

### RADPS-UC8 — Resolve declared upstream domain-state dependencies

**Current Pipeline cross-references:** UC-09.

**Stakeholders:** Workflow user.

**Direct actors:** Worker, node task, Workflow Framework.

**Goal:** Resolve the accepted upstream domain state required by a node task's declared dependencies, using stable name, type, processing scope, and optional version.

**Outcome:** The consumer can bind its required domain-state inputs deterministically, and the exact state-version identities used remain traceable.

**Alternative flows:** Missing, stale, or ambiguous domain-state dependencies produce a structured error and are not silently substituted.

**Boundary:** The Workflow Framework defines node-task dependencies and controls scheduling. `radps-context` resolves declared domain-state dependencies but does not infer the dependency graph. Processing-output registration and lookup are covered by RADPS-UC7.

### RADPS-UC9 — Read and submit state during distributed execution

**Current Pipeline cross-references:** UC-10, UC-13, UC-14; GAP-01, GAP-02.

**Stakeholders:** Workflow user, Workflow operator.

**Direct actors:** Worker, Workflow Framework.

**Goal:** Give a worker a coherent state view for an identified node task and data chunk and accept its complete domain-state update while independent work proceeds concurrently.

**Outcome:** Updates are accepted atomically; tentative or incomplete work does not change accepted state. The processing boundary used as input and the node-task and data-chunk identities remain traceable. Accepted updates for independently processed data chunks remain distinguishable so downstream work can combine them deterministically.

**Alternative flows:** An update that conflicts with accepted state is rejected with a structured conflict result and leaves accepted state unchanged. If an equivalent outcome with the same update identity was previously accepted, `radps-context` returns the existing accepted update without creating another state version or repeating its registrations. Reusing an update identity for a different outcome is rejected.

**Boundary:** `radps-context` detects conflicts and enforces idempotent submission. The worker or node-task execution environment provides any transient workspace for tentative changes; only complete outcomes are submitted to `radps-context`. The Workflow Framework decides whether to fail or retry conflicting work; on retry, the worker recomputes its outcome against a current state view.

### RADPS-UC10 — Support checkpoint-based restoration and rerun

**Current Pipeline cross-references:** UC-12; GAP-04, GAP-06.

**Stakeholders:** Workflow operator.

**Direct actors:** Workflow Framework.

**Goal:** Enable an operator or automated recovery policy to restore accepted domain state from a recorded processing boundary during the current or a later Workflow invocation for rollback, failure restart, resume, or targeted rerun.

**Preconditions:** The Workflow Framework identifies the processing boundary to preserve or restore. The boundary’s state is compatible with a supported context-model version, and all required processing outputs are identifiable and retrievable.

**Outcome:** After restoration, `radps-context` reproduces the accepted domain state recorded at the processing boundary. References to required processing outputs resolve to retrievable outputs, and relationships linking the restored state and outputs to their inputs and producing components remain available.

**Alternative flows:** A boundary with incompatible state, missing references, unverifiable required outputs, or incomplete domain-state updates is rejected and cannot be used to create or restore from a Checkpoint Record.

**Boundary:** The Workflow Framework provides operator-facing controls, manages the Checkpoint Record, selects the recovery boundary, and determines which work to schedule, skip, or rerun. `radps-context` provides and restores the accepted domain state associated with that boundary; it does not provide a user interface.

### RADPS-UC11 — Store domain annotations, matching overrides, and execution-control directives

**Current Pipeline cross-references:** GAP-07, GAP-08.

**Stakeholders:** Workflow user, Workflow operator.

**Direct actors:** Heuristic, Workflow Framework.

**Goal:** Store domain annotations, matching overrides, and execution-control directives originating from users, heuristics, or Workflow policy and submitted through an internal Workflow interface.

**Outcome:** Each accepted annotation, override, or directive remains available with its processing scope, rationale, producer, effective state, and supersession history.

**Boundary:** The Workflow Framework, rather than `radps-context`, enforces execution-control directives.

### RADPS-UC12 — Store and provide domain quality assessments

**Current Pipeline cross-references:** UC-16.

**Stakeholders:** Workflow user, Workflow operator, external consumer.

**Direct actors:** QA node task, worker, heuristic, reporting component.

**Goal:** Associate a domain quality assessment with the dataset, state version, processing output, or processing scope that it evaluates.

**Outcome:** Subsequent node tasks and reporting components can retrieve the assessment together with its inputs, method or policy version, producing component, and rationale.

### RADPS-UC13 — Maintain telescope- and array-specific context extensions

**Current Pipeline cross-references:** UC-18.

**Stakeholders:** Workflow user, Workflow developer.

**Direct actors:** Workflow Framework, worker, node task, heuristic.

**Goal:** Store validated telescope- or array-specific state without making shared Workflow consumers depend on those extensions.

**Outcome:** Recognized extension state is available only for its declared run, dataset, or data-chunk scope and remains attributable to its producer.

**Alternative flows:** An unrecognized extension type, unsupported extension schema version, or extension state that fails the structural, scope, or semantic validation defined for that extension is rejected.

### RADPS-UC14 — Provide a consistent internal domain-state view

**Current Pipeline cross-references:** UC-15, UC-19; GAP-03, GAP-05.

**Stakeholders:** Workflow user, Workflow operator, external consumer.

**Direct actors:** Worker, node task, heuristic, Workflow Framework, reporting component, external-interface subsystem.

**Goal:** Provide a coherent, read-only view of domain state, processing-output relationships, domain decisions, QA state, and domain provenance at an identified processing boundary.

**Outcome:** The requesting Workflow component receives the information and the boundary used remains identifiable.

**Alternative flows:** If the requested boundary is unavailable, the context returns an explicit error; it does not silently substitute the latest state.

**Boundary:** Reporting, export, and external-interface components determine how to present or deliver the retrieved information. `radps-context` provides the internal read interface but does not render reports, package products, or communicate directly with external consumers.

### RADPS-UC15 — Inspect domain state for diagnosis

**Current Pipeline cross-references:** UC-17.

**Stakeholders:** Workflow developer, Workflow operator.

**Direct actors:** Diagnostic client, Workflow Framework.

**Goal:** Inspect accepted domain state and domain-specific processing outputs at the current or an identified historical processing boundary during execution or after a failure.

**Outcome:** The diagnostic client can inspect registered datasets, calibration and imaging state, quality assessments, domain decisions, processing-output relationships, and provenance. The accepted state boundary associated with failed work remains identifiable and available for post-mortem analysis.

**Alternative flows:** If the requested state boundary is unavailable, `radps-context` returns an explicit error rather than silently substituting another boundary.

**Boundary:** `radps-context` provides domain state and domain-specific processing-output information. The Workflow Framework provides node-task execution status, logs, and tracebacks and coordinates pausing or debugging execution.

## Capabilities out of scope for radps-context

- Planning, scheduling, worker dispatch, retry coordination, checkpoint management, and non-domain execution history belong to the Workflow Framework. See [requirements_and_ownership.md](requirements_and_ownership.md) for more information on functionality ownership.
- Interfaces from the Workflow to external systems are not direct context interactions. GAP-05 in [requirements_and_ownership.md](requirements_and_ownership.md) describes the internal context interface needed to support them.
- Report generation and final-data-product export may read context state or register output references, but the context does not perform rendering, packaging, or delivery.
