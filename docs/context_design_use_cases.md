Use Case Template (Context Design)

Adapted from “Use Case Modeling” by Kurt Bittner and Ian Spence.

Aim for less than 3 pages per use case. The goal here is to capture the major features of the individual use case, not every single detail. Additional details will be captured later in the design phase.

This version is tuned for **Pipeline Context** design. In a distributed system, the “hard” requirements are often in:

- what durable state changes
- what artifacts are registered
- what transactional boundaries and idempotency guarantees are required
- what audit/provenance must be emitted

See also:

- [docs/radps_use_case_mapping.md](radps_use_case_mapping.md) (Pipeline UC → RADPS context mapping)
- [docs/current_pipeline_use_cases.md](current_pipeline_use_cases.md) (source Pipeline UCs)

Use Case <number>: <title>

    Relevant Stakeholders
        The relevant stakeholders for this use case.
    Frequency:
        How frequently (e.g., high/medium/low) the stakeholder group anticipates encountering the use case.
    Importance:
        How important (e.g., high/medium/low) the use case is to the stakeholder group for achieving their objectives.
    Actors:
        Actors are entities that interact with the system (user, service, worker).
    Goals:
        Outcomes of actor interactions. Include context as necessary.
    Preconditions:
        Conditions that must be true before the use case can be executed.
    Postconditions / Outputs:
        Durable state changes and/or artifacts that must exist after completion.
    Context Data / Artifacts:
        What context records are read/written, and what artifacts (if any) are referenced/registered.
    Transaction / Idempotency Notes:
        What must be ACID? What can be eventual? What operations must be idempotent?
    Observability / Audit:
        Events, logs, metrics, provenance records required.
    Basic Flow:
        Main success scenario.
    Alternative Flows (optional):
        Deviations: errors, retries, partial completion, permission failures.

---

Draft Use Cases (RADPS Context; numbered UC*)

These are first-draft entries focused on **context** (run ledger + artifact registry + provenance), not the entire RADPS workflow.

Notes on numbering:

- These use `UC*` numbering for easy cross-reference.
- They are **RADPS context use cases**; when citing them next to Pipeline UCs, refer to them as “RADPS UC#”.

Use Case UC1: Initialize or Load a Run Context

    Relevant Stakeholders
        Operations, pipeline developers, QA reviewers, runtime services (planner/executor/reporting).
    Frequency:
        High.
    Importance:
        High.
    Actors:
        Operator (human or automation), Planner service, Context Store.
    Goals:
        Create a new run record with stable identifiers and initial metadata, or load an existing run for resume.
    Preconditions:
        Inputs are identified (dataset IDs/paths); caller is authorized to create or access the run.
    Postconditions / Outputs:
        A `run_id` exists with initial run metadata; a “current plan” slot is empty or points to an existing `plan_id`.
    Context Data / Artifacts:
        Writes run metadata (domain, policy bundle, versions, timestamps); may record initial working/products/report locations.
    Transaction / Idempotency Notes:
        Create-run must be atomic; repeated create requests must not create duplicate runs (idempotent by client token or external ID).
    Observability / Audit:
        Record a RunCreated event (who/when/what inputs/policy).
    Basic Flow:
        1. Actor submits create/load request with minimal metadata.
        2. Context Store validates authorization and required fields.
        3. Context Store creates the run record (or returns the existing run) and returns `run_id`.
    Alternative Flows (optional):
        - Load fails because the run/version is unsupported; system returns a structured incompatibility error.

Use Case UC2: Persist a Plan Representation (Plan Registration)

    Relevant Stakeholders
        Planner developers, operations (reproducibility), reporting/audit consumers.
    Frequency:
        High (at least once per run; may recur on re-plan).
    Importance:
        High.
    Actors:
        Planner service, Context Store.
    Goals:
        Record the planned computation structure so execution and reporting can be tied back to an explicit plan.
    Preconditions:
        `run_id` exists; planner has produced a plan structure (DAG/graph) and policy provenance.
    Postconditions / Outputs:
        A `plan_id` exists and is associated to `run_id`; nodes have stable `node_id` values.
    Context Data / Artifacts:
        Stores plan definition, partitioning keys, planner version, policy bundle hash, and user-supplied parameters.
    Transaction / Idempotency Notes:
        Register-plan must be atomic; plan updates should be append-only (new `plan_id`) rather than in-place mutation.
    Observability / Audit:
        Record a PlanRegistered event with plan provenance.
    Basic Flow:
        1. Planner submits plan definition to Context Store.
        2. Context Store validates schema and links plan to run.
        3. Context Store returns `plan_id` and `node_id` map.
    Alternative Flows (optional):
        - Validation fails (schema/version mismatch); plan is rejected with a clear error.

Use Case UC3: Record a Node Attempt Lifecycle (Start/Finish/Retry)

    Relevant Stakeholders
        Operations (recoverability), QA/review, developers (debugging), reporting.
    Frequency:
        Very high (per executed node attempt).
    Importance:
        High.
    Actors:
        Executor/worker, Context Store.
    Goals:
        Track node execution state under retries and failures, with consistent status and timing.
    Preconditions:
        `run_id` and `plan_id` exist; `node_id` exists in the plan; worker is authorized to update run state.
    Postconditions / Outputs:
        Attempt record(s) exist with `attempt_id`; node status transitions are recorded; failures have structured error summaries.
    Context Data / Artifacts:
        Writes attempt state, timestamps, worker identity, and optional resource summaries.
    Transaction / Idempotency Notes:
        State transitions must be ACID and monotonic; attempt start/finish must be idempotent (safe to retry on network failure).
    Observability / Audit:
        Emit NodeAttemptStarted/Finished events; attach tracebacks and error codes where applicable.
    Basic Flow:
        1. Worker requests to start an attempt for a node.
        2. Context Store creates `attempt_id` and marks attempt RUNNING.
        3. Worker completes work and submits completion status and summary.
        4. Context Store marks attempt SUCCEEDED/FAILED and updates node-level derived status.
    Alternative Flows (optional):
        - Worker crashes mid-attempt; Context Store detects lost heartbeats and marks attempt LOST/FAILED for retry.
        - Duplicate completion arrives; Context Store ignores it (idempotent) or returns existing completion.

Use Case UC4: Register Produced Artifacts with Lineage

    Relevant Stakeholders
        Operations (delivery/retention), QA, reporting, reproducibility.
    Frequency:
        Very high.
    Importance:
        High.
    Actors:
        Worker, Context Store (artifact registry).
    Goals:
        Make artifacts discoverable and traceable: what was produced, by whom, from what inputs, and where it lives.
    Preconditions:
        Artifact data has been written to durable storage (local/shared/object store) and is readable.
    Postconditions / Outputs:
        `artifact_id` exists with type, lineage, and one or more locations; artifact is linked to producing `attempt_id`.
    Context Data / Artifacts:
        Writes artifact metadata, optional hashes/checksums, and retention hints.
    Transaction / Idempotency Notes:
        Registration must be idempotent for the same content/location; avoid duplicate artifacts for retried attempts.
    Observability / Audit:
        Emit ArtifactRegistered event.
    Basic Flow:
        1. Worker writes artifact payload.
        2. Worker submits artifact metadata (type, inputs, locations) to Context Store.
        3. Context Store registers artifact and links it to the producing attempt.
    Alternative Flows (optional):
        - Location becomes unavailable after write; artifact registration fails and the node attempt is marked failed.

Use Case UC5: Create and Validate an Explicit Checkpoint

    Relevant Stakeholders
        Operations, developers (safe restart), cost control.
    Frequency:
        Medium to high (stage boundaries; before expensive fan-out).
    Importance:
        High.
    Actors:
        Executor/controller, Operator, Context Store.
    Goals:
        Define a durable “safe restart point” that references a closed set of artifacts/state.
    Preconditions:
        Required upstream nodes are SUCCEEDED; required artifacts are registered.
    Postconditions / Outputs:
        A checkpoint record exists for `run_id` and references the required node states and artifacts.
    Context Data / Artifacts:
        Writes checkpoint metadata (scope, node set, artifact set, plan revision).
    Transaction / Idempotency Notes:
        Checkpoint creation must be atomic and consistent (no half-created checkpoint).
    Observability / Audit:
        Emit CheckpointCreated event.
    Basic Flow:
        1. Actor requests checkpoint creation for a defined scope (e.g., stage boundary or partition set).
        2. Context Store verifies prerequisites and creates checkpoint.
    Alternative Flows (optional):
        - Prerequisites missing; checkpoint is rejected with a list of missing nodes/artifacts.

Use Case UC6: Resume or Partial Re-run with Downstream Invalidation

    Relevant Stakeholders
        Operations (recovery), QA, developers.
    Frequency:
        Medium.
    Importance:
        High.
    Actors:
        Operator/automation, Context Store.
    Goals:
        Resume a run safely from a checkpoint or re-run a subgraph while maintaining provenance.
    Preconditions:
        `run_id` exists; a checkpoint exists or a rerun scope is defined; caller is authorized.
    Postconditions / Outputs:
        Selected nodes are marked runnable; downstream nodes/artifacts are marked stale/tombstoned as appropriate; new attempts are tracked.
    Context Data / Artifacts:
        Writes invalidation markers, rerun reason, and links to checkpoint/plan revision.
    Transaction / Idempotency Notes:
        Invalidation must be ACID to avoid mixed “old/new” downstream states.
    Observability / Audit:
        Emit RunResumed/RerunRequested events with reason and scope.
    Basic Flow:
        1. Actor requests resume/rerun for a scope.
        2. Context Store marks downstream state stale and records the rerun intent.
        3. Executor observes runnable nodes and proceeds (out of scope here).
    Alternative Flows (optional):
        - Resume fails due to schema/version incompatibility; system returns a structured incompatibility error.

Use Case UC7: Operator Annotation and Controlled Overrides

    Relevant Stakeholders
        Operations, QA, science support, audit/provenance consumers.
    Frequency:
        Medium.
    Importance:
        Medium to high.
    Actors:
        Operator/QA reviewer, Context Store.
    Goals:
        Record human decisions (notes, approvals, override parameters) as durable, auditable context state.
    Preconditions:
        `run_id` exists; actor is authorized; the target (run/node/artifact) exists.
    Postconditions / Outputs:
        Annotation/override records exist and are linked to targets; subsequent reporting includes them.
    Context Data / Artifacts:
        Writes annotations, approval flags, policy override references.
    Transaction / Idempotency Notes:
        Writes must be atomic; edits should be versioned or append-only for audit.
    Observability / Audit:
        Emit AnnotationAdded/OverrideApplied events including actor identity.
    Basic Flow:
        1. Operator submits an annotation or override request.
        2. Context Store validates authorization and writes the record.
    Alternative Flows (optional):
        - Permission denied; request is rejected and logged.

Use Case UC8: Export Provenance Manifest / Report as an Artifact

    Relevant Stakeholders
        Operations (delivery), QA, archive/consumers.
    Frequency:
        High (at least once per run; may be re-generated).
    Importance:
        High.
    Actors:
        Reporting service, Context Store.
    Goals:
        Produce machine-readable manifests and/or human-readable reports based solely on durable context + registered artifacts.
    Preconditions:
        Run ledger and artifact registry contain sufficient records; reporting actor is authorized.
    Postconditions / Outputs:
        A manifest/report artifact exists and is registered with lineage back to the run.
    Context Data / Artifacts:
        Reads run ledger and artifact registry; writes new manifest/report artifact record.
    Transaction / Idempotency Notes:
        Export should be repeatable; the same inputs should produce a semantically equivalent manifest (within defined determinism policy).
    Observability / Audit:
        Emit ReportGenerated event and include tool versions.
    Basic Flow:
        1. Reporting service queries context for run summary, attempts, artifacts, annotations.
        2. Reporting service writes the manifest/report.
        3. Reporting service registers the output as an artifact.
    Alternative Flows (optional):
        - Missing required records; reporting fails with a clear list of missing context elements.

Use Case UC9: Artifact Retention and Tombstoning (Safe Cleanup)

    Relevant Stakeholders
        Operations (storage control), audit/provenance consumers.
    Frequency:
        Medium.
    Importance:
        Medium to high.
    Actors:
        Lifecycle manager, Operator, Context Store.
    Goals:
        Apply retention/cleanup without breaking resumability or provenance.
    Preconditions:
        Retention policy exists; artifacts have known locations; actor is authorized.
    Postconditions / Outputs:
        Artifacts are marked retained or tombstoned; deletions (if any) are recorded as durable state.
    Context Data / Artifacts:
        Writes retention decisions and tombstone records.
    Transaction / Idempotency Notes:
        Cleanup operations must be idempotent; tombstoning must be ACID with respect to resume/invalidations.
    Observability / Audit:
        Emit ArtifactTombstoned/RetentionApplied events.
    Basic Flow:
        1. Lifecycle manager identifies candidates per policy.
        2. Context Store records tombstones/retention decisions.
        3. Storage cleanup occurs (out of scope here), and completion is recorded.
    Alternative Flows (optional):
        - Artifact is on hold due to an active investigation; cleanup is skipped and hold reason recorded.

Use Case UC10: Query Dataset / Observation Catalog (Read-Only View)

    Relevant Stakeholders
        Pipeline algorithms, heuristics, QA, reporting.
    Frequency:
        Very high.
    Importance:
        High.
    Actors:
        Worker, QA/reporting service, Context Store.
    Goals:
        Provide fast, consistent access to observation metadata (MSv4 inventory, fields, SPWs, scans, and mapping metadata) required by tasks and reporting.
    Preconditions:
        `run_id` exists; dataset inventory has been registered in context.
    Postconditions / Outputs:
        No new durable state is required; consumers obtain a consistent view of metadata.
    Context Data / Artifacts:
        Reads Dataset/Observation Catalog records; may read mapping tables (virtual↔real equivalents) if applicable.
    Transaction / Idempotency Notes:
        Reads should be served from a consistent snapshot (e.g., transaction-level or checkpoint-level read).
    Observability / Audit:
        Optional: record query provenance for expensive reports (not required for all task-level reads).
    Basic Flow:
        1. Consumer requests catalog data for a scope (by dataset, field/spw/scan partition, or logical selection).
        2. Context Store returns typed metadata records.
    Alternative Flows (optional):
        - Requested scope not found; return a structured “unknown dataset/partition” error.

Use Case UC11: Apply Transactional Calibration State Update

    Relevant Stakeholders
        Calibration tasks, operations (resume correctness), QA.
    Frequency:
        High.
    Importance:
        High.
    Actors:
        Worker, Context Store.
    Goals:
        Atomically record a set of calibration state changes produced by a task (the callibrary analogue).
    Preconditions:
        Producing attempt exists; calibration artifacts (e.g., tables) are written and registered (or will be registered in the same transaction if supported).
    Postconditions / Outputs:
        Calibration state version advances; downstream consumers see either the old or new version, never a mix.
    Context Data / Artifacts:
        Writes calibration application records and links them to artifact IDs and producing attempts.
    Transaction / Idempotency Notes:
        Must be ACID; multi-entry updates must commit atomically; repeated submissions from a retried client must not duplicate entries.
    Observability / Audit:
        Emit CalibrationStateUpdated event and link to attempt.
    Basic Flow:
        1. Worker submits a calibration-state patch (multiple entries) referencing produced artifacts.
        2. Context Store validates and commits the patch atomically.
    Alternative Flows (optional):
        - Conflict detected with a concurrent incompatible update; patch is rejected and worker must retry with updated base version.

Use Case UC12: Update Imaging State (Schema’d Scratch Pad)

    Relevant Stakeholders
        Imaging tasks/heuristics, operations (reproducibility), reporting.
    Frequency:
        High.
    Importance:
        High.
    Actors:
        Worker, Context Store.
    Goals:
        Replace ad-hoc imaging attributes with a versioned, typed imaging state document that supports partition-scoped updates.
    Preconditions:
        `run_id` exists; plan node scope identifies which imaging partition is being updated (field/spw/scan or equivalent).
    Postconditions / Outputs:
        Imaging state is updated for the intended scope; downstream readers can resolve the correct version.
    Context Data / Artifacts:
        Writes imaging state records; may link to artifact IDs (masks, thresholds, sensitivities, beam models).
    Transaction / Idempotency Notes:
        Must be ACID for a given partition scope; updates should be versioned to support resume and rerun.
    Observability / Audit:
        Emit ImagingStateUpdated event.
    Basic Flow:
        1. Worker submits imaging state update scoped to a partition.
        2. Context Store validates schema/version and commits.
    Alternative Flows (optional):
        - Schema/version mismatch; update rejected with required migration/version info.

Use Case UC13: Provide Read-Only Snapshot for QA/Reporting/Rendering

    Relevant Stakeholders
        QA, weblog/report generation, developers.
    Frequency:
        High.
    Importance:
        High.
    Actors:
        QA/reporting service, Context Store.
    Goals:
        Provide a consistent read view of run state and artifact registry for rendering/scoring without depending on worker memory.
    Preconditions:
        A checkpoint exists or a consistent snapshot boundary is defined (e.g., “as of attempt X completion”).
    Postconditions / Outputs:
        Snapshot view is consumable; optional derived products (reports) can be produced and registered.
    Context Data / Artifacts:
        Reads ledger + registry + annotations.
    Transaction / Idempotency Notes:
        Snapshot reads must be consistent; re-rendering should be deterministic within declared policy.
    Observability / Audit:
        Record snapshot boundary identifiers used by the report.
    Basic Flow:
        1. Service requests a snapshot for `run_id` at a boundary.
        2. Context Store serves a consistent view for queries.
    Alternative Flows (optional):
        - Boundary not available (no checkpoint); service may request “latest committed” with caveats recorded.

Use Case UC14: Resolve Named Outputs Instead of Stage-Index Walking

    Relevant Stakeholders
        Task developers, operations (correct reruns), performance.
    Frequency:
        High.
    Importance:
        High.
    Actors:
        Worker, Context Store.
    Goals:
        Allow downstream tasks to discover upstream outputs by stable keys (names/types/scopes) instead of walking an ordered results list.
    Preconditions:
        Upstream artifacts/records have been registered with names/types/scopes.
    Postconditions / Outputs:
        Downstream consumers can bind required inputs deterministically.
    Context Data / Artifacts:
        Reads from artifact registry and/or typed output tables keyed by logical output name + scope.
    Transaction / Idempotency Notes:
        Output registration must be idempotent; lookups must be consistent under concurrency.
    Observability / Audit:
        Optional: record bindings for provenance (which artifact IDs satisfied which logical inputs).
    Basic Flow:
        1. Consumer requests “latest output of type X for scope Y” (or a specific version).
        2. Context Store returns artifact IDs and metadata.
    Alternative Flows (optional):
        - Output not found; consumer fails fast with a structured missing-dependency error.

Use Case UC15: Append-Only Event Log / Patch Log (Audit + Replay)

    Relevant Stakeholders
        Operations, QA, developers (debugging), external integrations, provenance/audit consumers.
    Frequency:
        Very high.
    Importance:
        High.
    Actors:
        Context Store, workers/executor, reporting/monitoring consumers.
    Goals:
        Maintain an append-only record of significant lifecycle events and/or state patches so that run evolution is auditable and (where feasible) replayable.
    Preconditions:
        `run_id` exists; writer is authorized.
    Postconditions / Outputs:
        Event records exist with stable IDs and ordering; events are linked to `plan_id`/`node_id`/`attempt_id`/`artifact_id` where applicable.
    Context Data / Artifacts:
        Writes event records; may include compact patch payloads or references to patch artifacts.
    Transaction / Idempotency Notes:
        Appends must be ACID; producers must be able to retry safely without duplicating logical events (idempotency keys).
    Observability / Audit:
        Events are the audit trail; provide query-by-run, query-by-node, and time-bounded queries.
    Basic Flow:
        1. Producer submits an event (and optional patch reference) with an idempotency key.
        2. Context Store appends the event and returns an event ID.
        3. Consumers query or subscribe to events for monitoring/reporting.
    Alternative Flows (optional):
        - Duplicate event submission returns the existing event ID.

Use Case UC16: Register and Query Domain-Specific Extensions (ngVLA/WSU)

    Relevant Stakeholders
        Domain teams (ngVLA, WSU), pipeline algorithm developers, operations.
    Frequency:
        Medium.
    Importance:
        Medium to high.
    Actors:
        Worker/planner, Context Store.
    Goals:
        Support domain-specific state without reintroducing untyped “state bags”, while keeping the core context contract stable.
    Preconditions:
        Extension schema/type is registered/known for the run; caller is authorized.
    Postconditions / Outputs:
        Extension state is stored as typed, versioned records scoped to run and (optionally) dataset/partition.
    Context Data / Artifacts:
        Reads/writes extension records; may link extension state to artifacts.
    Transaction / Idempotency Notes:
        Extension updates should be ACID within their scope; versioning is required for reruns/resume.
    Observability / Audit:
        Emit ExtensionRegistered/ExtensionUpdated events.
    Basic Flow:
        1. Planner or operator enables an extension type for a run (schema/version).
        2. Workers write scoped extension updates during execution.
        3. Consumers query extension state via typed APIs.
    Alternative Flows (optional):
        - Unknown extension schema/version; update is rejected with a structured error.

Use Case UC17: Worker Snapshot Read + Transactional Write-Back (Distributed Execution)

    Relevant Stakeholders
        Executor/workers, operations (scalability/reliability), developers.
    Frequency:
        Very high.
    Importance:
        High.
    Actors:
        Executor/worker, Context Store.
    Goals:
        Allow workers to read a consistent snapshot of required context state while ensuring all writes return through ACID transactions (no “forked pickles” as system-of-record).
    Preconditions:
        `run_id` exists; a snapshot boundary exists (checkpoint, plan revision boundary, or latest-committed); worker is authorized.
    Postconditions / Outputs:
        Worker obtains a snapshot token/identifier; all updates are committed as transactional patches linked to `attempt_id`.
    Context Data / Artifacts:
        Reads snapshot views of ledger/catalog/state; writes attempt state, artifacts, and patches.
    Transaction / Idempotency Notes:
        Snapshot reads must be consistent; write-back must be ACID with conflict detection (per-partition where possible). All submissions must be idempotent under retry.
    Observability / Audit:
        Record snapshot boundary and patch application events; link patches to the worker identity and attempt.
    Basic Flow:
        1. Worker requests a snapshot token for its node/partition scope.
        2. Worker performs computation using snapshot reads.
        3. Worker registers artifacts and submits context patches transactionally.
        4. Context Store commits patches and updates derived state.
    Alternative Flows (optional):
        - Write conflict detected; worker must refresh snapshot and retry with a new base version.
