# Context Use-Case Mapping and RADPS Ownership

## Purpose

The current Pipeline context combines domain state, execution tracking, inter-stage communication, reporting support, export support, and operator inspection. RADPS separates those responsibilities.

This document evaluates current Pipeline context behavior for knowledge transfer and requirement traceability. A capability's presence in the current context does not place it inside the future `radps-context` boundary.

## Component boundary

### `radps-context`

Maintains pipeline domain state, including observation and project metadata, calibration state, imaging state, domain quality assessments, domain decisions, artifact relationships, and domain provenance. It exposes these capabilities only through internal pipeline interfaces.

### Workflow orchestration layer

Plans, schedules, and coordinates pipeline work. It owns dependency progression, worker dispatch, retries, execution status and history, resume scheduling, and enforcement of execution-control decisions.

### External-interface subsystem

Owns all interaction beyond the internal pipeline boundary, including public or user-facing APIs, authentication and authorization, operator tools, dashboards, notifications, archive protocols, report and product delivery, and external error handling. It may use an internal adapter to exchange normalized requests and responses with `radps-context` or workflow orchestration.

Other data, artifact, reporting, and packaging services may exist in RADPS. The context records domain identities and relationships; it does not transfer payloads, render reports, package products, or administer storage.

Interactions with `xradio` are internal implementation concerns that may affect how observation information is obtained or represented, not the external responsibility boundary.

## 1. Current Pipeline use cases mapped to RADPS

### UC-01 — Populate, access, and provide observation metadata

Requirements: ALMA-TR48, ALMA-TR107, CSS9018. RADPS context use cases: RADPS-UC1, RADPS-UC2.

`radps-context` registers and serves normalized observation information. An ingestion subsystem owns any external source protocol or conversion preceding that registration.

### UC-02 — Cross-MS metadata matching and lookup

Requirements: ALMA-TR07, ALMA-TR10. RADPS context use cases: RADPS-UC3.

Matching used by pipeline tasks is a context responsibility. User-facing override collection and authorization are external-interface responsibilities.

### UC-03 — Store and provide project-level metadata

Requirements: ALMA-TR48. RADPS context use cases: RADPS-UC1, RADPS-UC2.

The context stores normalized project information needed by pipeline work. It does not retrieve that information from an external project or archive system.

### UC-04 — Register, query, and update calibration state

Requirements: ALMA-TR53. RADPS context use cases: RADPS-UC4.

### UC-05 — Manage imaging state

Requirements: ALMA-TR53. RADPS context use cases: RADPS-UC5.

### UC-06 — Register and query produced image products

Requirements: ALMA-TR51.1, ALMA-TR51.2, ALMA-TR65. RADPS context use cases: RADPS-UC5, RADPS-UC6.

The context registers domain metadata and artifact relationships. Artifact storage, rendering, packaging, and delivery are outside its boundary. ALMA-TR66 remains outside this assessment.

### UC-07 — Track current execution progress

Requirements: CSS9037, CSS9034, CSS9064.1. Owner: workflow orchestration layer.

External presentation or notification of progress is owned by the external-interface subsystem, not by context or workflow.

### UC-08 — Preserve per-stage execution record

Requirements: CSS9051, ALMA-TR105, CSS9010. Owner: workflow orchestration layer.

`radps-context` retains only the work or attempt identity needed to attribute a domain update; it does not duplicate general execution history.

### UC-09 — Propagate task outputs to downstream tasks

Requirements: CSS9063, CSS9063.5. RADPS context use cases: RADPS-UC7, RADPS-UC8.

The context makes accepted domain outcomes resolvable. Workflow orchestration manages dependencies and passes non-domain task results or futures.

### UC-10 — Provide a transient intra-stage workspace

Requirements: ALMA-TR74, ALMA-TR24. RADPS context use cases: RADPS-UC8.

The context prevents tentative changes from entering accepted state. The task execution framework owns worker-local scratch objects and their lifecycle.

### UC-11 — Support multiple orchestration drivers

Requirements: ALMA-TR47, ALMA-TR31. RADPS context use cases: RADPS-UC1.

The context accepts a stable internal initialization contract. Driver-specific commands, interactive interfaces, and external protocols terminate in workflow or the external-interface subsystem.

### UC-12 — Save and restore a processing session

Requirements: ALMA-TR29, ALMA-TR30, CSS9038, CSS9034. RADPS context use cases: RADPS-UC1, RADPS-UC9.

The context preserves and validates domain-state boundaries. Workflow restores execution. Artifact or data services make referenced payloads available.

### UC-13 and UC-14 — Parallel worker distribution and aggregation

Requirements: CSS9600, CSS9064.2. RADPS context use cases: RADPS-UC8.

The current Pipeline mechanisms are not carried forward. RADPS requires a coherent internal read and atomic update contract; workflow and execution infrastructure own distribution and aggregation mechanics.

### UC-15 — Provide read-only state for reporting

Requirements: ALMA-TR50.4, ALMA-TR83. RADPS context use cases: RADPS-UC13.

The context provides a consistent internal domain-state view. Report generation, formatting, publication, and delivery are outside the context boundary.

### UC-16 — Support QA evaluation and store quality assessments

Requirements: ALMA-TR49, ALMA-TR50. RADPS context use cases: RADPS-UC11, RADPS-UC13.

Pipeline QA computation and stored domain assessments remain internal. Human review, release decisions, and QA presentation belong to other subsystems.

### UC-17 — Support inspection and debugging

Requirements: ALMA-TR27, ALMA-TR28, ALMA-TR112. RADPS context use cases: RADPS-UC13.

The context supplies internal domain state. Workflow supplies logs, tracebacks, and execution state. An external-interface subsystem owns operator-facing inspection and debugging tools.

### UC-18 — Manage telescope- and array-specific state

Requirements: ALMA-TR07.1, ALMA-TR07.2, ALMA-TR08, ALMA-TR05, ALMA-TR03. RADPS context use cases: RADPS-UC3, RADPS-UC12.

### UC-19 — Provide state for product export

Requirements: ALMA-TR51, CSS9066. RADPS context use cases: RADPS-UC6, RADPS-UC13.

Only the internal state-read and artifact-registration portions belong to `radps-context`. Export selection, manifest formatting, packaging, archive handoff, and delivery belong to other subsystems. These requirements must not be treated as assigning end-to-end export responsibility to the context.

## 2. Gap analysis

### GAP-01 — Asynchronous execution of independent work

- **Context:** Provides consistent reads, atomic outcomes, and conflict detection through RADPS-UC8.
- **Workflow:** Resolves dependencies and schedules independent work.
- **Requirements:** CSS9017, CSS9063, CSS9064.2, CSS9600.

### GAP-02 — Distributed execution without a shared filesystem

- **Context:** Stores location-portable artifact references and never assumes a producer's local path is visible to another worker.
- **Workflow/infrastructure:** Places work and ensures workers can resolve the referenced data through the appropriate service.
- **Requirements:** CSS9002, CSS9030.

### GAP-03 — Provenance and reproducibility

- **Context:** Stores domain lineage and links state or artifacts to the internal work identities that produced them.
- **Workflow:** Stores task versions, parameters, execution attempts, runtime environment, hardware, and scheduler information.
- **External-interface subsystem:** Supports audit queries, presentation, and export of combined provenance.
- **Requirements:** ALMA-TR103, ALMA-TR104, ALMA-TR105.

### GAP-04 — Partial re-execution and targeted rerun

- **Context:** Identifies compatible domain-state boundaries and marks affected state or artifacts stale when an internal rerun decision is accepted.
- **Workflow:** Determines the affected graph and schedules the rerun.
- **External-interface subsystem:** Captures and authorizes any operator request before submitting a normalized internal decision.
- **Requirements:** CSS9038.

### GAP-05 — External system integration

This gap is explicitly outside `radps-context`. The external-interface subsystem owns queries, APIs, event delivery, dashboards, archive interaction, and their availability or retry semantics. It may use RADPS-UC13 through an internal adapter but does not make the external consumer a context actor.

Requirements CSS9046, CSS9047, CSS9048, CSS9049, CSS9050, and CSS9056 must be allocated to that subsystem.

### GAP-06 — Initialization from intermediate state

- **External-interface or ingestion subsystem:** Retrieves and interprets any archival or externally supplied products and converts them into the supported internal model.
- **Context:** Validates and establishes the normalized domain boundary through RADPS-UC9.
- **Workflow:** Determines which work can be skipped or resumed.
- **Requirements:** CSS9038.

### GAP-07 — Explicit tag-based execution control

- **External-interface subsystem:** Captures, authenticates, and authorizes an operator directive.
- **Context:** Stores a normalized domain-relevant directive when pipeline work needs durable access to it, through RADPS-UC10.
- **Workflow:** Reads and enforces the effective directive.
- **Requirements:** CSS9037.

### GAP-08 — Heterogeneous dataset coordination

- **Context:** Provides matching semantics and stores normalized, accepted overrides through RADPS-UC3.
- **Pipeline heuristics/tasks:** Select matching modes and propose automated mappings.
- **External-interface subsystem:** Captures and authorizes human overrides.
- **Requirements:** ALMA-TR07.

## 3. Explicitly excluded context responsibilities

The following are not `radps-context` requirements even when the current Pipeline context happens to support them:

- direct access by people, dashboards, archive systems, or other external consumers;
- public API, query-service, subscription, or event-feed behavior;
- authentication, authorization, user identity, or approval workflow;
- external-delivery tracking, consumer availability, and notification retry;
- archive retrieval, archive-format compatibility, ingest, or handoff;
- report and manifest rendering, product packaging, and delivery;
- storage-retention policy, holds, deletion, and physical cleanup;
- task planning, scheduling, dispatch, execution status, and general execution history; and
- payload transfer or guarantees supplied by artifact and data services.

## Referenced source documents

- ALMA Data Processing Technical Requirements
- CSS Stakeholder Needs – SDA, SDP, SIT, TI
- Data Processing and Archive Workflow Stakeholder Needs
- Computing and Software System Design Description: SDP
