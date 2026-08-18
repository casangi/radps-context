# RADPS Context Use Cases

## Scope

These use cases define the domain-state operations that `radps-context` exposes
to components inside the RADPS pipeline. Direct actors are limited to workflow
orchestration, workers, heuristics, pipeline tasks, and internal adapters.

The following responsibilities are explicitly outside this boundary:

- public or user-facing APIs and authentication;
- operator interfaces, dashboards, monitoring, and notifications;
- archive discovery, retrieval, ingest protocols, and archival delivery;
- report or manifest formatting, product packaging, and delivery;
- storage-retention decisions and physical cleanup; and
- scheduling, worker dispatch, and general execution-history management.

A subsystem responsible for one of those functions may use an internal adapter
to read context state or submit a normalized update. The use case begins at that
internal interface; it does not include the preceding external interaction or
the subsequent delivery of information outside the pipeline.

## Actors

- **Workflow orchestration layer**: Plans and coordinates pipeline work and
  supplies the context with the identifiers needed to correlate domain state
  with that work.
- **Worker**: Performs pipeline computation, reads context state, writes
  artifacts, and submits complete domain outcomes.
- **Pipeline task**: Performs an internal pipeline operation such as data import,
  calibration, imaging, QA evaluation, or product preparation.
- **Heuristic**: Reads domain state and derives processing decisions or mapping
  proposals under pipeline policy.
- **Internal adapter**: Submits or retrieves normalized information on behalf of
  another RADPS subsystem. External protocols and identities remain opaque to
  the context.

## Use cases

### RADPS-UC1 — Initialize or load a run context

**Current Pipeline cross-references:** UC-03, UC-11, UC-12.

**Actors:** Workflow orchestration layer, internal ingestion adapter.

**Goal:** Establish a run with stable identity and initial domain information,
or load compatible persisted context state for internal pipeline use.

**Preconditions:** The actor supplies an internal run identity, input dataset
identities, applicable policy versions, and any initial project information.

**Outcome:** The run and its initial domain state are available to internal
pipeline components. The creating component, creation time, inputs, and context
model version remain identifiable.

**Alternative flows:** An incompatible context version, duplicate immutable run
identity, or incomplete normalized request is rejected explicitly.

### RADPS-UC2 — Provide observation and project metadata

**Current Pipeline cross-references:** UC-01, UC-03.

**Actors:** Pipeline task, worker, heuristic.

**Goal:** Register and retrieve initial or incremental dataset, observation, and
project information needed by pipeline work, including version identity and
lineage from transformed datasets to their sources.

**Outcome:** Internal consumers can obtain a coherent view of the requested
datasets, fields, spectral windows, scans, antennas, time ranges, data types,
project properties, and derived metadata products. A newly accepted dataset
version remains distinguishable from prior versions so workflow orchestration
can determine any affected work.

**Alternative flows:** Unknown or ambiguous scopes return an explicit error.

### RADPS-UC3 — Resolve heterogeneous dataset matches

**Current Pipeline cross-references:** UC-02, UC-18; GAP-08.

**Actors:** Worker, heuristic, pipeline task, internal adapter.

**Goal:** Resolve corresponding fields, sources, spectral windows, and data
columns across datasets using a declared matching mode. An internal adapter may
submit a normalized override already authorized by another subsystem.

**Outcome:** The consumer receives the resolved match set. Accepted overrides
retain their scope, rationale, source reference, and supersession history.

**Alternative flows:** Ambiguous matches return candidates without selecting an
answer. Conflicting or invalid overrides are rejected.

### RADPS-UC4 — Apply a calibration-state update

**Current Pipeline cross-references:** UC-04.

**Actors:** Worker, calibration task.

**Goal:** Register a complete set of calibration changes, their applicability,
and related artifacts as one domain outcome.

**Outcome:** Internal consumers observe either the preceding calibration-state
version or the new version, never a partial mixture. The update remains linked
to its internal producer, inputs, and produced artifacts.

**Alternative flows:** An incompatible concurrent update is rejected so the
producer can recompute against a current view.

### RADPS-UC5 — Apply an imaging-state update

**Current Pipeline cross-references:** UC-05, UC-06.

**Actors:** Worker, imaging task.

**Goal:** Record imaging state and image-product references for a declared
dataset or processing scope.

**Outcome:** The intended state version and image products are available to
dependent pipeline work and linked to their producer and inputs.

**Alternative flows:** Invalid scope, inconsistent state, or unavailable
required artifacts cause the complete update to be rejected.

### RADPS-UC6 — Register an artifact with domain lineage

**Current Pipeline cross-references:** UC-06, UC-19; GAP-02, GAP-03.

**Actors:** Worker, pipeline task.

**Goal:** Register the identity, type, lineage, and location-portable references
of an artifact produced or adopted by pipeline work.

**Preconditions:** The payload is available through the pipeline's artifact or
data service. The context does not write, transfer, package, or deliver it.

**Outcome:** Internal components can resolve the artifact by stable identity,
type, scope, or lineage and associate it with the producing domain outcome.

**Alternative flows:** Registration fails if required references cannot be
validated. Repeating an equivalent registration returns the existing logical
artifact rather than creating a duplicate.

### RADPS-UC7 — Resolve upstream domain outputs

**Current Pipeline cross-references:** UC-09.

**Actors:** Worker, pipeline task, workflow orchestration layer.

**Goal:** Resolve accepted upstream domain state and artifacts by stable name,
type, scope, and optional version for use by dependent pipeline work.

**Outcome:** The consumer can bind inputs deterministically, and the exact state
and artifact identities used remain traceable.

**Alternative flows:** Missing, stale, or ambiguous dependencies produce a
structured error and are not silently substituted.

### RADPS-UC8 — Read and submit state during distributed execution

**Current Pipeline cross-references:** UC-10, UC-13, UC-14; GAP-01, GAP-02.

**Actors:** Worker, workflow orchestration layer.

**Goal:** Give a worker a coherent state view for its scope and accept its
complete domain outcome while independent work proceeds concurrently.

**Outcome:** The read boundary and producing work identity remain traceable.
Accepted updates become visible atomically; tentative or incomplete work does
not change accepted state.

**Alternative flows:** Conflicting updates are rejected. A retry using the same
logical update identity does not duplicate its effect.

### RADPS-UC9 — Create and restore an internal processing boundary

**Current Pipeline cross-references:** UC-12; GAP-04, GAP-06.

**Actors:** Workflow orchestration layer, internal adapter.

**Goal:** Identify a closed, compatible set of domain state and artifacts that
can be loaded for resume or used as the basis of a targeted rerun.

**Preconditions:** All supplied state is expressed in the internal context model.
If it originated in an archive or other external system, another subsystem has
already retrieved, interpreted, and normalized it.

**Outcome:** The boundary identifies its state versions, required artifacts, and
domain provenance. The workflow layer can separately decide which work to
schedule, skip, or rerun.

**Alternative flows:** A boundary with incompatible state, missing references,
or unverifiable required artifacts is rejected and remains unavailable for
resume.

### RADPS-UC10 — Maintain normalized domain decisions

**Current Pipeline cross-references:** UC-17; GAP-07, GAP-08.

**Actors:** Heuristic, workflow orchestration layer, internal adapter.

**Goal:** Store annotations, matching overrides, or domain-relevant control
directives that have already been produced or authorized by an internal
pipeline component or another subsystem.

**Outcome:** The accepted decision remains available with its scope, rationale,
internal producer, optional opaque source reference, effective state, and
supersession history.

**Boundary:** `radps-context` does not provide the operator interface,
authenticate a person, decide whether a request is authorized, or enforce
workflow control. The workflow layer reads applicable directives and enforces
them.

### RADPS-UC11 — Store and provide domain quality assessments

**Current Pipeline cross-references:** UC-16.

**Actors:** QA pipeline task, worker, heuristic.

**Goal:** Associate a domain quality assessment with the dataset, state version,
artifact, or processing scope that it evaluates.

**Outcome:** Subsequent pipeline work can retrieve the assessment and its inputs,
method or policy version, producing component, and rationale.

**Boundary:** Human review, release approval, dashboards, and presentation of QA
results are external-interface responsibilities.

### RADPS-UC12 — Maintain domain-specific context extensions

**Current Pipeline cross-references:** UC-18.

**Actors:** Workflow orchestration layer, worker, pipeline task.

**Goal:** Store validated telescope-, array-, or domain-specific state without
making shared pipeline consumers depend on that extension.

**Outcome:** Recognized extension state is available only for its declared run,
dataset, or partition scope and remains attributable to its producer.

**Alternative flows:** Unknown extension types or state that violates the
declared contract are rejected.

### RADPS-UC13 — Provide a consistent internal domain-state view

**Current Pipeline cross-references:** UC-15, UC-17, UC-19; GAP-03.

**Actors:** Worker, pipeline task, heuristic, workflow orchestration layer,
internal adapter.

**Goal:** Provide a coherent, read-only view of domain state, artifact
relationships, domain decisions, QA state, and domain provenance at an
identified processing boundary.

**Outcome:** The requesting internal component receives the information and the
boundary used remains identifiable. An internal adapter may transform or
deliver that information elsewhere, but those actions and their outcomes are
not context responsibilities.

**Alternative flows:** If the requested boundary is unavailable, the context
returns an explicit error; it does not silently substitute the latest state.

## Capabilities intentionally not modeled as context use cases

- Planning, scheduling, worker dispatch, retry orchestration, and execution
  history belong to workflow orchestration.
- Public queries, subscriptions, lifecycle-event delivery, monitoring, and
  dashboards belong to the external-interface subsystem.
- Report rendering, provenance-manifest formatting, export packaging, archive
  ingest and delivery, and storage cleanup belong to other subsystems or
  pipeline tasks. Those components use UC6 and UC13 when they need context data
  or need to register a resulting internal artifact.
- Direct operator and reviewer interactions are not context interactions. Only
  their normalized, authorized outcomes may enter through an internal adapter.
