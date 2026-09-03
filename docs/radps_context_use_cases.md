# RADPS Context Use Cases

This document defines RADPS context use cases. It focuses on domain state, processing-output information, and provenance rather than the orchestration performed by the Workflow Framework or the behavior of the entire Workflow.

## Use-case template

Adapted from “Use Case Modeling” by Kurt Bittner and Ian Spence.

This template is tuned for `radps-context` behavior. A use case should identify:

- the actor goal and direct interaction with `radps-context`
- the observable domain-state or processing-output outcome
- any conditions required before the interaction
- relevant failure, retry, consistency, or ownership boundaries

See also:

- [Requirements and ownership](requirements_and_ownership.md) for current Pipeline UC traceability and responsibility allocation
- [Current Pipeline context use cases](context_use_cases_current_pipeline.md) for the source use cases
- [RADPS context quality requirements](radps_context_quality_requirements.md) for cross-cutting behavioral guarantees
- [Glossary](glossary.md) for shared terminology

Use the following structure. Preconditions, alternative flows, and boundaries may be omitted when they do not apply.

```markdown
### RADPS-UC<number> — <title>

**Current Pipeline cross-references:** <related current Pipeline use cases and GAPs>

**Actors:** <logical roles that interact directly with `radps-context`>

**Goal:** <outcome sought by the actors>

**Preconditions:** <conditions that must hold before the interaction>

**Outcome:** <observable domain state, processing-output information, or traceability available after completion>

**Alternative flows:** <errors, retries, conflicts, or other deviations>

**Boundary:** <related behavior owned by the Workflow Framework or another component>
```

## Scope

These use cases define the domain-state operations that `radps-context` exposes to components inside the RADPS Workflow. Interfaces from the Workflow to external systems are outside this document's scope.

## Actors

- **Workflow Framework**: Orchestrates the Workflow, supplies identifiers that correlate domain state with work, and manages decomposition, scheduling, retries, and checkpoint use without performing domain-specific processing.
- **Worker**: Executes a node task, reads context state, writes processing outputs, and submits complete domain outcomes.
- **Node task**: Invokes processing functions as a unit of work assignable to a node, consuming and producing dataset partitions or intermediate artifacts. Examples include data import, calibration, imaging, QA evaluation, and output preparation.
- **Heuristic**: Reads domain state and applies configured domain rules or algorithms to derive processing decisions or propose mappings between related metadata elements.

## Use cases

### RADPS-UC1 — Initialize or load a run context

**Current Pipeline cross-references:** UC-03, UC-11, UC-12.

**Actors:** Workflow Framework, data import node task.

**Goal:** Establish a run with stable identity and initial domain information, or load compatible persisted context state for internal Workflow use.

**Preconditions:** The actor supplies an internal run identity, input dataset identities, and applicable policy versions, together with either any initial project information needed to initialize the run or the persisted context state to restore.

**Outcome:** The run and its initial domain state or resumed persisted context are available to internal Workflow components. The creating component, creation time, inputs, and context-model version remain identifiable.

**Alternative flows:** An incompatible context version or missing required initialization information is rejected explicitly. Repeating initialization for the same run with equivalent information returns the existing run context; attempting to reuse the run identity with conflicting information is rejected.

### RADPS-UC2 — Provide observation and project metadata

**Current Pipeline cross-references:** UC-01, UC-03.

**Actors:** Node task, worker, heuristic.

**Goal:** Register and retrieve initial or incremental dataset, observation, and project information needed by node tasks, including version identity and lineage from transformed datasets to their sources.

**Outcome:** Internal consumers can obtain a coherent view of the requested datasets, fields, spectral windows, scans, antennas, time ranges, data types, project properties, and derived metadata outputs. A newly accepted dataset version remains distinguishable from prior versions so the Workflow Framework can determine any affected work.

**Alternative flows:** Unknown or ambiguous scopes return an explicit error.

### RADPS-UC3 — Resolve heterogeneous dataset matches

**Current Pipeline cross-references:** UC-02, UC-18; GAP-08.

**Actors:** Worker, heuristic, node task, Workflow Framework.

**Goal:** Resolve corresponding fields, sources, spectral windows, and data columns across datasets using a declared matching mode, including an explicitly supplied override when automatic matching is insufficient.

**Outcome:** The consumer receives the resolved match set. Accepted overrides retain their scope, rationale, source reference, and supersession history.

**Alternative flows:** Ambiguous matches return candidates without selecting an answer. Conflicting or invalid overrides are rejected.

### RADPS-UC4 — Apply a calibration-state update

**Current Pipeline cross-references:** UC-04.

**Actors:** Worker, calibration task.

**Goal:** Register a complete set of calibration changes, their applicability, and related processing outputs as one domain outcome.

**Outcome:** Internal consumers observe either the preceding calibration-state version or the new version, never a partial mixture. The update remains linked to its internal producer, inputs, and processing outputs.

**Alternative flows:** An incompatible concurrent update is rejected so the producer can recompute against a current view.

### RADPS-UC5 — Apply an imaging-state update

**Current Pipeline cross-references:** UC-05, UC-06.

**Actors:** Worker, imaging task.

**Goal:** Record imaging state and image-output references for a declared dataset or processing scope.

**Outcome:** The intended state version and image outputs are available to dependent node tasks and linked to their producer and inputs.

**Alternative flows:** Invalid scope, inconsistent state, or unavailable required outputs cause the complete update to be rejected.

### RADPS-UC6 — Register a processing output with domain lineage

**Current Pipeline cross-references:** UC-06, UC-19; GAP-02, GAP-03.

**Actors:** Worker, node task.

**Goal:** Register the identity, type, lineage, and location-portable references of a processing output produced or adopted by a node task.

**Preconditions:** The processing output has been produced and its location is known.

**Outcome:** Internal components can resolve the processing output by stable identity, type, scope, or lineage and associate it with the producing domain outcome.

**Alternative flows:** Registration fails if required references cannot be validated. Repeating an equivalent registration returns the existing logical processing output rather than creating a duplicate.

### RADPS-UC7 — Resolve upstream domain outputs

**Current Pipeline cross-references:** UC-09.

**Actors:** Worker, node task, Workflow Framework.

**Goal:** Resolve accepted upstream domain state and processing outputs by stable name, type, scope, and optional version for use by dependent node tasks.

**Outcome:** The consumer can bind inputs deterministically, and the exact state and processing-output identities used remain traceable.

**Alternative flows:** Missing, stale, or ambiguous dependencies produce a structured error and are not silently substituted.

### RADPS-UC8 — Read and submit state during distributed execution

**Current Pipeline cross-references:** UC-10, UC-13, UC-14; GAP-01, GAP-02.

**Actors:** Worker, Workflow Framework.

**Goal:** Give a worker a coherent state view for an identified node task and data chunk and accept its complete domain outcome while independent work proceeds concurrently.

**Outcome:** The read boundary, node-task identity, and data-chunk identity remain traceable. Accepted updates for independently processed chunks become visible atomically and remain distinguishable so downstream work can combine them deterministically; tentative or incomplete work does not change accepted state.

**Alternative flows:** Conflicting updates are rejected. A retry using the same logical update identity does not duplicate its effect.

### RADPS-UC9 — Provide context state for a Checkpoint Record

**Current Pipeline cross-references:** UC-12; GAP-04, GAP-06.

**Actors:** Workflow Framework.

**Goal:** Provide a closed, compatible version of domain state and required processing-output references that the Workflow Framework can associate with a Checkpoint Record for rollback, failure restart, resume, or targeted rerun.

**Preconditions:** The state is expressed in a supported context model and its required processing outputs are identifiable.

**Outcome:** The processing boundary identifies its context-state versions, required processing outputs, and domain provenance. The Workflow Framework can create a Checkpoint Record that refers to that boundary and separately decide which work to schedule, skip, or rerun.

**Alternative flows:** A boundary with incompatible state, missing references, unverifiable required outputs, or incomplete domain outcomes is rejected and cannot be used for a Checkpoint Record.

### RADPS-UC10 — Store domain annotations, matching overrides, and control directives

**Current Pipeline cross-references:** UC-17; GAP-07, GAP-08.

**Actors:** Heuristic, Workflow Framework.

**Goal:** Store annotations, matching overrides, or domain-relevant control directives supplied through an internal Workflow interface.

**Outcome:** The accepted decision remains available with its scope, rationale, producer, effective state, and supersession history.

**Boundary:** The Workflow Framework, rather than `radps-context`, enforces execution-control directives.

### RADPS-UC11 — Store and provide domain quality assessments

**Current Pipeline cross-references:** UC-16.

**Actors:** QA node task, worker, heuristic.

**Goal:** Associate a domain quality assessment with the dataset, state version, processing output, or processing scope that it evaluates.

**Outcome:** Subsequent node tasks can retrieve the assessment and its inputs, method or policy version, producing component, and rationale.

### RADPS-UC12 — Maintain domain-specific context extensions

**Current Pipeline cross-references:** UC-18.

**Actors:** Workflow Framework, worker, node task.

**Goal:** Store validated telescope-, array-, or domain-specific state without making shared Workflow consumers depend on that extension.

**Outcome:** Recognized extension state is available only for its declared run, dataset, data-chunk, or partition scope and remains attributable to its producer.

**Alternative flows:** Unknown extension types or state that violates the declared contract are rejected.

### RADPS-UC13 — Provide a consistent internal domain-state view

**Current Pipeline cross-references:** UC-15, UC-17, UC-19; GAP-03.

**Actors:** Worker, node task, heuristic, Workflow Framework.

**Goal:** Provide a coherent, read-only view of domain state, processing-output relationships, domain decisions, QA state, and domain provenance at an identified processing boundary.

**Outcome:** The requesting Workflow component receives the information and the boundary used remains identifiable.

**Alternative flows:** If the requested boundary is unavailable, the context returns an explicit error; it does not silently substitute the latest state.

## Capabilities out of scope for radps-context

- Planning, scheduling, worker dispatch, retry coordination, checkpoint management, and non-domain execution history belong to the Workflow Framework. See [requirements_and_ownership.md](requirements_and_ownership.md) for more information on functionality ownership.
- Interfaces from the Workflow to external systems are not direct context interactions. GAP-05 in [requirements_and_ownership.md](requirements_and_ownership.md) describes the internal context interface needed to support them.
- Report generation and final-data-product export may read context state or register output references, but the context does not perform rendering, packaging, or delivery.
