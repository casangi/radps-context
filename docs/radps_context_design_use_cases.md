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
- [docs/context_use_cases_current_pipeline.md](context_use_cases_current_pipeline.md) (source Pipeline UCs)
- [docs/context_gap_use_cases.md](context_gap_use_cases.md) (gap scenarios that drive additional RADPS context use cases)
- [docs/glossary.md](glossary.md) (definitions: ACID, DAG, idempotency, etc.)

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
        Create a new run record with stable identifiers, initial metadata, and run-level location configuration, or load an existing run for resume. The context must be driver-agnostic: any orchestration front-end (automated batch, interactive session, recipe evaluator) must produce an equivalent run record, and the context contract must remain stable across drivers (Pipeline UC-11). Run identity, driver metadata, and artifact location/layout policy must be first-class context data so save/restore and export workflows remain portable (Pipeline UC-12, UC-19).
    Preconditions:
        Inputs are identified (dataset IDs/paths); caller is authorized to create or access the run.
    Postconditions / Outputs:
        A `run_id` exists with initial run metadata; a “current plan” slot is empty or points to an existing `plan_id`. Run-level location configuration (working/products/report roots) is recorded, either as explicit paths or as artifact-registry location policies.
    Context Data / Artifacts:
        Writes run metadata (domain, policy bundle, versions, timestamps, orchestration driver identity); records run-level location configuration (output/report/products roots or artifact-location policies). May accept driver-injected metadata (recipe/procedure name, project IDs, performance parameters).
    Transaction / Idempotency Notes:
        Create-run must be atomic; repeated create requests must not create duplicate runs (idempotent by client token or external ID).
    Observability / Audit:
        Record a RunCreated event (who/when/what inputs/policy/driver).
    Basic Flow:
        1. Actor submits create/load request with minimal metadata (including driver identity and location configuration).
        2. Context Store validates authorization and required fields.
        3. Context Store creates the run record (or returns the existing run) and returns `run_id`.
    Alternative Flows (optional):
        - Load fails because the run/version is unsupported; system returns a structured incompatibility error.
        - Driver submits additional metadata (project IDs, performance parameters) as part of creation; Context Store records these as immutable-after-init run metadata.

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

Use Case UC3: Record a Node Attempt Lifecycle and Maintain Execution History (Start/Finish/Retry)

    Relevant Stakeholders
        Operations (recoverability), QA/review, developers (debugging), reporting, regression harnesses.
    Frequency:
        Very high (per executed node attempt).
    Importance:
        High.
    Actors:
        Executor/worker, Context Store.
    Goals:
        Track node execution state under retries and failures, with consistent status and timing. The aggregate of all attempt records must form a queryable, ordered execution history suitable for progress tracking, reporting, QA, debugging, and export (Pipeline UC-07, UC-08, UC-15, UC-17, UC-19). Node ordering within the DAG replaces legacy stage numbering and must remain coherent across resumes.
    Preconditions:
        `run_id` and `plan_id` exist; `node_id` exists in the plan; worker is authorized to update run state.
    Postconditions / Outputs:
        Attempt record(s) exist with `attempt_id`; node status transitions are recorded; failures have structured error summaries. The run ledger exposes an ordered execution timeline (by node/attempt completion) that consumers can traverse for reporting and debugging.
    Context Data / Artifacts:
        Writes attempt state, timestamps, worker identity, tracebacks, QA outcome summaries, and optional resource summaries.
    Transaction / Idempotency Notes:
        State transitions must be ACID and monotonic; attempt start/finish must be idempotent (safe to retry on network failure). The ordered execution history must be consistent under concurrent attempt completions.
    Observability / Audit:
        Emit NodeAttemptStarted/Finished events; attach tracebacks and error codes where applicable. The execution history itself serves as the primary progress-tracking and debugging interface.
    Basic Flow:
        1. Worker requests to start an attempt for a node.
        2. Context Store creates `attempt_id` and marks attempt RUNNING.
        3. Worker completes work and submits completion status and summary.
        4. Context Store marks attempt SUCCEEDED/FAILED, updates node-level derived status, and appends to the ordered execution history.
    Alternative Flows (optional):
        - Worker crashes mid-attempt; Context Store detects lost heartbeats and marks attempt LOST/FAILED for retry.
        - Duplicate completion arrives; Context Store ignores it (idempotent) or returns existing completion.
        - Regression harness queries the execution history to validate deterministic outputs, durations, and failure signals across runs.

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
        Make artifacts discoverable and traceable: what was produced, by whom, from what inputs, and where it lives across local, shared-filesystem, or object-store backends.
    Preconditions:
        Artifact data has been written to durable storage (local/shared/object store) and is readable.
    Postconditions / Outputs:
        `artifact_id` exists with type, lineage, and one or more locations; artifact is linked to producing `attempt_id`.
    Context Data / Artifacts:
        Writes artifact metadata, optional hashes/checksums, retention hints, and one or more storage-agnostic location references/access policies.
    Transaction / Idempotency Notes:
        Registration must be idempotent for the same logical artifact/content and allow multiple durable locations to be attached without creating duplicate logical artifacts.
    Observability / Audit:
        Emit ArtifactRegistered event.
    Basic Flow:
        1. Worker writes artifact payload.
        2. Worker submits artifact metadata (type, inputs, locations) to Context Store.
        3. Context Store registers artifact and links it to the producing attempt.
    Alternative Flows (optional):
        - Artifact is first written to worker-local scratch; registration is deferred or recorded as non-exportable until durable storage is confirmed.
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
        Resume a run safely from a checkpoint or re-run a subgraph/partition while maintaining provenance and explicit dependency/invalidation semantics.
    Preconditions:
        `run_id` exists; a checkpoint exists or a rerun scope is defined; caller is authorized.
    Postconditions / Outputs:
        Selected nodes are marked runnable; downstream nodes/artifacts are marked stale/tombstoned as appropriate; new attempts are tracked.
    Context Data / Artifacts:
        Writes invalidation markers, dependency/version edges, rerun reason, and links to checkpoint/plan revision.
    Transaction / Idempotency Notes:
        Invalidation must be ACID to avoid mixed “old/new” downstream states.
    Observability / Audit:
        Emit RunResumed/RerunRequested events with reason and scope.
    Basic Flow:
        1. Actor requests resume/rerun for a scope.
        2. Context Store marks downstream state stale according to the dependency graph and records the rerun intent.
        3. Executor observes runnable nodes and proceeds (out of scope here).
    Alternative Flows (optional):
        - Resume fails due to schema/version incompatibility; system returns a structured incompatibility error.
        - Requested rerun scope overlaps active work; system rejects the rerun or requires cancellation/serialization before proceeding.

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
        Provide fast, consistent access to observation metadata (MSv4 inventory, fields, SPWs, scans, data-type metadata, and cross-dataset identity/matching records) required by tasks and reporting. This is the read-only catalog surface underlying more specialized matching workflows such as UC22.
    Preconditions:
        `run_id` exists; dataset inventory has been registered in context.
    Postconditions / Outputs:
        No new durable state is required; consumers obtain a consistent view of metadata.
    Context Data / Artifacts:
        Reads Dataset/Observation Catalog records, data-type metadata, and cross-dataset identity/matching tables.
    Transaction / Idempotency Notes:
        Reads should be served from a consistent snapshot (e.g., transaction-level or checkpoint-level read).
    Observability / Audit:
        Optional: record query provenance for expensive reports (not required for all task-level reads).
    Basic Flow:
        1. Consumer requests catalog data for a scope (by dataset, field/spw/scan partition, logical selection, or data type).
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

Use Case UC13: Provide Read-Only Snapshot for QA/Reporting/Rendering/Debugging

    Relevant Stakeholders
        QA, weblog/report generation, developers, CI/regression harnesses, operations.
    Frequency:
        High.
    Importance:
        High.
    Actors:
        QA/reporting service, debugging/inspection tools, CI harness, Context Store.
    Goals:
        Provide a consistent read view of run state and artifact registry for rendering, QA, export, and inspection without depending on worker memory. Must also support debugging use cases: diagnosing failures (what ran, what data was loaded, what state was produced), validating deterministic outputs, and surfacing failures beyond raw task exceptions (Pipeline UC-15, UC-16, UC-17, UC-19).
    Preconditions:
        A checkpoint exists or a consistent snapshot boundary is defined (e.g., “as of attempt X completion”).
    Postconditions / Outputs:
        Snapshot view is consumable; optional derived products (reports) can be produced and registered. Debugging tools can traverse the snapshot without requiring access to the worker runtime.
    Context Data / Artifacts:
        Reads ledger + registry + annotations + execution history (UC3).
    Transaction / Idempotency Notes:
        Snapshot reads must be consistent; re-rendering should be deterministic within declared policy.
    Observability / Audit:
        Record snapshot boundary identifiers used by the report or inspection session.
    Basic Flow:
        1. Service requests a snapshot for `run_id` at a boundary.
        2. Context Store serves a consistent view for queries.
    Alternative Flows (optional):
        - Boundary not available (no checkpoint); service may request “latest committed” with caveats recorded.
        - Debugging/CI tool queries snapshot to compare outputs across runs or validate expected artifacts and QA outcomes.

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
        Allow workers to read a consistent snapshot of required context state while ensuring all writes return through ACID transactions (no “forked pickles” as system-of-record), including asynchronous/overlapping execution of independent work across partitions.
    Preconditions:
        `run_id` exists; a snapshot boundary exists (checkpoint, plan revision boundary, or latest-committed); worker is authorized.
    Postconditions / Outputs:
        Worker obtains a snapshot token/identifier; all updates are committed as transactional patches linked to `attempt_id`.
    Context Data / Artifacts:
        Reads snapshot views of ledger/catalog/state; writes attempt state, artifacts, and patches.
    Transaction / Idempotency Notes:
        Snapshot reads must be consistent; write-back must be ACID with conflict detection and partition-scoped merges where possible. All submissions must be idempotent under retry.
    Observability / Audit:
        Record snapshot boundary and patch application events; link patches to the worker identity and attempt.
    Basic Flow:
        1. Worker requests a snapshot token for its node/partition scope.
        2. Worker performs computation using snapshot reads.
        3. Worker registers artifacts and submits context patches transactionally.
        4. Context Store commits patches and updates derived state.
    Alternative Flows (optional):
        - Write conflict detected; worker must refresh snapshot and retry with a new base version.

Use Case UC18: Publish Run State to External Systems

    Relevant Stakeholders
        QA dashboards, monitoring tools, archive ingest systems, schedulers, operators.
    Frequency:
        High.
    Importance:
        High.
    Actors:
        External consumer/service, Context Store, notification dispatcher.
    Goals:
        Provide timely, stable access to current processing state, lifecycle events, and summary views without requiring consumers to scrape product files or worker-local storage. The design must support both pull-style queries and push-style subscriptions/webhooks for selected lifecycle events.
    Preconditions:
        `run_id` exists; consumer or subscription is authorized; the requested summary/event schema version is supported.
    Postconditions / Outputs:
        External consumers can query current state or receive subscribed notifications; delivery attempts and subscription state are recorded durably.
    Context Data / Artifacts:
        Reads run ledger, artifact registry, QA records, and summary views; writes subscription definitions, delivery records, and exported summary/manifest artifacts when required.
    Transaction / Idempotency Notes:
        Subscription changes and delivery-state updates must be atomic; notifications must be idempotent and retryable.
    Observability / Audit:
        Emit SubscriptionCreated/Updated, NotificationDispatched, and NotificationFailed events with consumer identity and schema version.
    Basic Flow:
        1. Consumer registers or uses an existing subscription/query contract for selected run events or summaries.
        2. Context Store serves query results or dispatches notifications when the selected events occur.
        3. Delivery attempts and any exported summary artifacts are recorded for audit.
    Alternative Flows (optional):
        - Consumer requests an unsupported schema version; request is rejected with negotiation details.
        - Delivery endpoint is unavailable; dispatcher retries per policy and records failure state without losing the event.

Use Case UC19: Capture Reproducibility Envelope and Immutable Attempt Provenance

    Relevant Stakeholders
        Pipeline operators, auditors, regression harnesses, reproducibility tooling.
    Frequency:
        High.
    Importance:
        High.
    Actors:
        Worker, reporting service, Context Store.
    Goals:
        Capture the immutable provenance required to reproduce or audit a run: exact input identities/hashes, parameters, software versions, execution environment, hardware, scheduler/workload-manager details, and lineage links for each attempt and exported product.
    Preconditions:
        An attempt or export operation exists; input identifiers are available for hashing/fingerprinting; environment metadata is available.
    Postconditions / Outputs:
        An immutable provenance record exists and is linked to the relevant `attempt_id`, `artifact_id`, or exported manifest.
    Context Data / Artifacts:
        Writes provenance envelopes, input hashes/fingerprints, environment records, and deterministic-execution annotations.
    Transaction / Idempotency Notes:
        Provenance capture must commit atomically with attempt completion or artifact registration where required by policy; repeated submissions from a retried client must not fork multiple provenance records for the same logical event.
    Observability / Audit:
        Emit ProvenanceCaptured events and surface missing or partial provenance fields explicitly.
    Basic Flow:
        1. Worker or reporting service collects input hashes/fingerprints and environment metadata.
        2. Context Store validates required fields and stores the immutable provenance envelope linked to the attempt or artifact.
        3. Downstream reporting/export queries these records directly.
    Alternative Flows (optional):
        - Some hashes are unavailable at completion time; system records partial provenance with explicit missing-field markers and may block checkpoint/export per policy.
        - Environment details change mid-run; new attempts record new environment versions rather than mutating prior provenance.

Use Case UC20: Serve a Language-Neutral Context API

    Relevant Stakeholders
        Non-Python clients (C++, Julia, JavaScript dashboards), external tools, pipeline services.
    Frequency:
        High.
    Importance:
        High.
    Actors:
        Client application, Context API service, Context Store.
    Goals:
        Provide a stable, typed, language-neutral API for querying and updating context state without coupling clients to storage layout or Python object models. Mission-critical metadata, heuristics inputs, transactional workflow operations, and artifact lookup must be covered first; higher-level QA/reporting endpoints may be layered separately.
    Preconditions:
        An API schema/version is published; client is authorized; requested operation is allowed for the client role.
    Postconditions / Outputs:
        Client completes a query or mutation through a stable contract; response metadata indicates the schema/contract version used.
    Context Data / Artifacts:
        Reads/writes the same ledger, catalog, state, and artifact records used by in-process services; publishes schema descriptors and typed error codes.
    Transaction / Idempotency Notes:
        Mutating API calls must preserve the same ACID/idempotency guarantees as internal callers; schema evolution must be versioned and backward-compatible within the supported window.
    Observability / Audit:
        Record API access, schema version, caller identity, and mutation idempotency keys where relevant.
    Basic Flow:
        1. Client queries service metadata or binds to a published schema version.
        2. Client submits a typed query or mutation request.
        3. Context service validates authorization/schema compatibility, executes the operation, and returns typed records or error codes.
    Alternative Flows (optional):
        - Requested API version is unsupported; service returns compatible versions or upgrade guidance.
        - Client requests an operation not exposed by the public contract; service rejects it with a typed capability error.

Use Case UC21: Register Incremental Dataset Updates and Versioned Results

    Relevant Stakeholders
        Data ingest systems, workflow engine, incremental processing tasks, operators.
    Frequency:
        Medium to high.
    Importance:
        High.
    Actors:
        Ingest service, planner/executor, Context Store.
    Goals:
        Allow new data to be registered into an active run/session, trigger incremental processing, and ensure new outputs are versioned rather than overwriting prior results.
    Preconditions:
        `run_id` exists; incremental-ingest policy allows new data; incoming data is identifiable and scoped to an existing or new session partition.
    Postconditions / Outputs:
        Dataset catalog contains a new dataset/version record; affected plan nodes or partitions are marked runnable; newly produced artifacts/results receive new version identifiers.
    Context Data / Artifacts:
        Writes dataset version records, ingest lineage, runnable-node markers, invalidation/update edges, and result/artifact version metadata.
    Transaction / Idempotency Notes:
        Dataset registration must be atomic and idempotent for the same ingest event; version assignment and downstream invalidation must commit together to avoid mixed old/new state.
    Observability / Audit:
        Emit DatasetVersionRegistered and IncrementalProcessingRequested events with ingest source and scope.
    Basic Flow:
        1. Ingest system submits new dataset material or a new version reference for an active run.
        2. Context Store registers the dataset version and determines the affected scopes/nodes.
        3. Context Store marks the relevant work runnable and ensures subsequent outputs are versioned.
    Alternative Flows (optional):
        - Incoming data conflicts with an existing immutable dataset version; system rejects it or records it as a separate branch/version per policy.
        - Incremental registration arrives while dependent work is running; system serializes, branches, or defers the update according to policy.

Use Case UC22: Resolve Heterogeneous Cross-Dataset Matches and Override Rules

    Relevant Stakeholders
        Calibration tasks, imaging tasks, heuristics, pipeline operators.
    Frequency:
        High.
    Importance:
        High.
    Actors:
        Worker, heuristic service, operator, Context Store.
    Goals:
        Resolve shared identity across heterogeneous datasets that do not share native SPW numbering, field numbering, source labels, or data-column layouts. Matching must support exact semantics for calibration-style consumers, overlap/partial semantics for imaging-style consumers, and explicit override rules with recorded rationale when defaults are ambiguous or incorrect.
    Preconditions:
        Relevant datasets are registered; matching schema/version is known; caller is authorized to read or set overrides.
    Postconditions / Outputs:
        Consumer receives a resolved match set and, when overrides are supplied, the override records are durably stored with rationale and scope.
    Context Data / Artifacts:
        Reads cross-dataset identity records, field/source/SPW/column metadata, and matching policies; writes override records and rationale metadata when explicit mappings are applied.
    Transaction / Idempotency Notes:
        Match-resolution reads must come from a consistent snapshot; override writes must be versioned, scoped, and idempotent for the same logical mapping request.
    Observability / Audit:
        Record MatchResolved and MatchOverrideApplied events including matching mode, scope, and actor identity.
    Basic Flow:
        1. Consumer requests a match set for a scope and matching mode.
        2. Context Store evaluates identity records, matching policy, and any existing overrides.
        3. Context Store returns the resolved match set and records any newly supplied override.
    Alternative Flows (optional):
        - Multiple candidate matches remain after policy evaluation; service returns an ambiguity error or candidate set requiring heuristic/user choice.
        - An override conflicts with an existing locked mapping; service rejects it unless an authorized replacement workflow is used.
