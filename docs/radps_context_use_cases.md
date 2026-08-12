# RADPS Use Cases

## Use Case Template

Adapted from “Use Case Modeling” by Kurt Bittner and Ian Spence.

This version is tuned for **Pipeline Context** behavior. Relevant requirements include:

- what durable state changes
- what artifact information must be available
- what consistency and retry guarantees consumers can observe
- what information must remain available for audit and provenance

See also:

- [docs/requirements_and_ownership.md](requirements_and_ownership.md) (Pipeline UC → RADPS requirements and ownership mapping)
- [docs/context_use_cases_current_pipeline.md](context_use_cases_current_pipeline.md) (source Pipeline UCs)
- [docs/radps_context_quality_requirements.md](radps_context_quality_requirements.md) (cross-cutting behavioral guarantees)
- [docs/glossary.md](glossary.md) (shared terminology)

RADPS-UC<number>: <title>

    Current Pipeline Cross-References:
        Related current Pipeline use cases and GAPs that this RADPS use case refines or replaces.

    Relevant Stakeholders
        The people, teams, or consuming groups this use case matters to. Stakeholders are not necessarily the same as actors.
    Actors:
        Actors are logical roles that interact directly with the system.
    Goals:
        Outcomes of actor interactions.
    Preconditions:
        Conditions that must be true before the use case can be executed.
    Postconditions / Outputs:
        Durable state changes, artifacts, and information that must be available after completion, including information needed for audit where relevant.
    Required Information / Artifacts:
        Information the actors need or provide, and artifacts that must be available or related to the outcome.
    Basic Flow:
        Main success scenario.
    Alternative Flows (optional):
        Deviations such as errors, retries, partial completion, and conflicting requests.

---

## Draft RADPS Use Cases

These are first-draft entries focused on **context** (run state, artifact information, and provenance), not the entire RADPS workflow.

Notes:

- These use `UC*` numbering for easy cross-reference.
- They are **RADPS context use cases**; when citing them next to Pipeline UCs, refer to them as “RADPS-UC#”.
- The requirement trace in [docs/requirements_and_ownership.md](requirements_and_ownership.md) distinguishes workflow-only responsibilities from shared workflow/context responsibilities. This document only expands the use cases that require `radps-context` behavior; workflow-only items such as current UC-07, UC-08, and GAP-01 are intentionally referenced but not modeled here as standalone context use cases.
- For consistency, the `Relevant Stakeholders` field uses a controlled vocabulary. Logical participants belong in `Actors`, not `Relevant Stakeholders`.
- Stakeholder definitions:
    - **Operations**: People responsible for run setup, monitoring, intervention, reruns, restart decisions, storage control, and operational delivery.
    - **Pipeline developers**: People who design, implement, debug, validate, or maintain pipeline logic, planning logic, and context-facing behavior.
    - **QA reviewers**: People or review groups responsible for assessing processing quality, QA outcomes, and release readiness.
    - **Reporting consumers**: People who rely on manifests, reports, dashboards, or rendered summaries to understand run state or outputs.
    - **Archive consumers**: People or systems responsible for packaging, ingesting, delivering, or retrieving archived products and manifests.
    - **Audit and reproducibility consumers**: People or systems concerned with provenance, traceability, compliance, repeatability, or regression comparison.
    - **Science support**: People who interpret processing results, apply domain judgment, or provide expert guidance on data-specific issues and overrides.
    - **Domain teams**: Groups responsible for domain- or observatory-specific extensions, policies, metadata, or state models.
    - **External system operators**: People responsible for integrating, operating, or supporting external systems that consume run state, processing changes, or exported summaries.
- Actor definitions:
    - **Operator**: A human or automation acting on behalf of operations to create, inspect, annotate, pause, resume, or rerun processing.
    - **Workflow orchestration layer**: The logical workflow participant that creates or revises dependency graphs, determines and schedules work, dispatches workers, and coordinates retries or resume behavior.
    - **Worker**: A task execution process that reads context state, writes artifacts, and submits state/provenance updates for a node attempt.
    - **Incremental data provider**: A logical participant that supplies new or incremental input data for an active run.
    - **Archive importer**: A logical participant that supplies pre-existing archival products and related information to initialize a run from an intermediate state.
    - **Specialized consumers and producers**: Logical participants that obtain context information, submit domain-specific updates, identify artifacts, or receive relevant processing changes for reporting, QA, heuristics, retention, or external integration.

RADPS-UC1: Initialize or Load a Run Context

    Current Pipeline Cross-References:
        UC-03, UC-11, UC-12, UC-19.

    Relevant Stakeholders
        Operations, Pipeline developers, QA reviewers.
    Actors:
        Operator (human or automation), Workflow orchestration layer.
    Goals:
        Establish a run with stable identity, initial information, and artifact-location information, or recover an existing run for resume. Equivalent run information must be available regardless of the orchestration front-end (automated batch, interactive session, or recipe evaluator) (Pipeline UC-11). Run identity, orchestration origin, and artifact locations must remain available so resume and export workflows are portable (Pipeline UC-12, UC-19).
    Preconditions:
        Inputs are identified (dataset IDs/paths).
    Postconditions / Outputs:
        The run has a stable identity and initial information, including any associated planned work and the locations used for working data, products, and reports. The identity of the actor that created the run, the creation time, inputs, applicable policy, and orchestration origin remain available.
    Required Information / Artifacts:
        Run domain, applicable policies and versions, timestamps, orchestration origin, artifact-location information, and any supplied recipe, procedure, project, or performance information.
    Basic Flow:
        1. Actor submits create/load request with minimal metadata (including driver identity and location configuration).
        2. Context system validates the required fields.
        3. Context system establishes the run (or identifies the existing run) and returns its identity.
    Alternative Flows (optional):
        - The existing run information is incompatible with the requesting consumer; the request fails with a clear explanation.
        - The existing run was created under an unsupported or incompatible context-model version; loading is rejected with the detected version and migration or compatibility guidance.
        - The orchestration actor supplies additional project or performance information during initialization; Context system preserves it as part of the initial run information.

RADPS-UC2: Record Planned Work

    Current Pipeline Cross-References:
        UC-07, UC-08, UC-12.

    Relevant Stakeholders
        Pipeline developers, Operations, Audit and reproducibility consumers.
    Actors:
        Workflow orchestration layer.
    Goals:
        Record the planned computation structure so execution and reporting can be tied back to an explicit plan.
    Preconditions:
        A run exists; the workflow orchestration layer has produced a dependency graph and the provenance of the planning decision.
    Postconditions / Outputs:
        The planned work is associated with the run, and its nodes have stable identities. A later plan change is distinguishable and preserves the previously accepted plan and its provenance.
    Required Information / Artifacts:
        Planned dependencies, partitioning scopes, planning origin and version, applicable policy identity, and user-supplied parameters.
    Basic Flow:
        1. Workflow orchestration layer submits plan definition to Context system.
        2. Context system validates the plan information and links the plan to the run.
        3. Context system returns the plan identity and stable node identities.
    Alternative Flows (optional):
        - The submitted plan cannot be interpreted or validated; it is rejected with a clear explanation.

RADPS-UC3: Record a Node Attempt Lifecycle and Maintain Execution History (Start/Finish/Retry)

    Current Pipeline Cross-References:
        UC-07, UC-08, UC-15, UC-17, UC-19.

    Relevant Stakeholders
        Operations, Pipeline developers, QA reviewers, Reporting consumers, Audit and reproducibility consumers.
    Actors:
        Workflow orchestration layer, Worker.
    Goals:
        Track node execution under retries and failures, with consistent status and timing. The combined attempt information must provide an ordered execution history suitable for progress tracking, reporting, QA, debugging, and export (Pipeline UC-07, UC-08, UC-15, UC-17, UC-19). Node ordering within the dependency graph must remain coherent across resumes.
    Preconditions:
        A run and plan exist; the node exists in the plan.
    Postconditions / Outputs:
        Each attempt has a stable identity; node status transitions do not regress to an earlier lifecycle state; failures include structured error summaries, tracebacks, and error codes where applicable. Consumers can traverse an ordered execution timeline that remains coherent when attempts complete concurrently.
    Required Information / Artifacts:
        Attempt identity and status, timestamps, worker identity, diagnostics, QA outcomes, and optional resource-use summaries.
    Basic Flow:
        1. Worker requests to start an attempt for a node.
        2. Context system accepts the attempt start and makes its in-progress status available.
        3. Worker completes work and submits completion status and summary.
        4. Context system accepts the completion, makes the attempt and node outcomes available, and preserves their order in the execution history.
    Alternative Flows (optional):
        - Worker stops before reporting completion; the workflow orchestration layer reports that the attempt is no longer active, and Context system records the outcome so the work can be retried.
        - Duplicate completion arrives; Context system returns the existing completion without changing the attempt again.
        - A regression consumer uses the execution history to validate deterministic outputs, durations, and failure signals across runs.

RADPS-UC4: Register Produced Artifacts with Lineage

    Current Pipeline Cross-References:
        UC-06, UC-19, GAP-02.

    Relevant Stakeholders
        Operations, QA reviewers, Reporting consumers, Audit and reproducibility consumers.
    Actors:
        Worker.
    Goals:
        Make artifacts discoverable and traceable: what was produced, by whom, from what inputs, and where it is available.
    Preconditions:
        Artifact data has been written to durable storage and is readable.
    Postconditions / Outputs:
        The artifact is discoverable by type, lineage, and one or more locations and is linked to the producing attempt. The time it became available and the responsible actor remain identifiable.
    Required Information / Artifacts:
        Artifact identity, type, producing attempt, input lineage, verifiable content identity when needed, retention information, and one or more usable locations.
    Basic Flow:
        1. Worker writes artifact payload.
        2. Worker submits artifact metadata (type, inputs, locations) to Context system.
        3. Context system registers artifact and links it to the producing attempt.
    Alternative Flows (optional):
        - Artifact is not yet durably available; it is not made available for export until a durable location is confirmed.
        - Location becomes unavailable after write; artifact registration fails and the node attempt is marked failed.
        - Equivalent artifact metadata is submitted again; Context system returns the existing logical artifact and attaches any additional durable location without creating a duplicate artifact.

RADPS-UC5: Create and Validate an Explicit Checkpoint

    Current Pipeline Cross-References:
        UC-12, GAP-04.

    Relevant Stakeholders
        Operations, Pipeline developers.
    Actors:
        Workflow orchestration layer, Operator.
    Goals:
        Define a durable “safe restart point” that references a closed set of artifacts/state.
    Preconditions:
        Required upstream nodes have completed successfully; required artifacts are registered.
    Postconditions / Outputs:
        A checkpoint exists for the run and identifies the required node states and artifacts. Its creation time, responsible actor, and covered scope remain available.
    Required Information / Artifacts:
        Checkpoint scope, required node outcomes and artifacts, and the applicable plan revision.
    Basic Flow:
        1. Actor requests checkpoint creation for a defined scope (e.g., stage boundary or partition set).
        2. Context system verifies prerequisites and creates checkpoint.
    Alternative Flows (optional):
        - Prerequisites missing; checkpoint is rejected with a list of missing nodes/artifacts.

RADPS-UC6: Resume or Partial Re-run with Downstream Invalidation

    Current Pipeline Cross-References:
        UC-12, GAP-04, GAP-06.

    Relevant Stakeholders
        Operations, Pipeline developers, QA reviewers.
    Actors:
        Operator (human or automation).
    Goals:
        Resume a run safely from a checkpoint or re-run a subgraph/partition while maintaining provenance and explicit dependency/invalidation semantics.
    Preconditions:
        A run exists; a checkpoint exists or a rerun scope is defined.
    Postconditions / Outputs:
        The selected rerun scope is recorded; downstream nodes and artifacts are identified as stale as appropriate; new attempts are tracked. Consumers do not observe a mixture of downstream state from before and after the invalidation. The requesting actor, time, reason, and affected scope remain available.
    Required Information / Artifacts:
        Rerun scope and reason, affected dependencies and prior outcomes, and the applicable checkpoint or plan revision.
    Basic Flow:
        1. Actor requests resume/rerun for a scope.
        2. Context system marks downstream state stale according to the dependency graph and records the rerun intent.
        3. Workflow orchestration layer uses the updated state to determine and schedule the required work.
    Alternative Flows (optional):
        - Existing run information cannot be interpreted sufficiently to resume safely; the request fails with a clear explanation.
        - The requested checkpoint or resume boundary references an unavailable or unverifiable artifact; resume is rejected with the missing artifact identified, and the boundary is recorded as unavailable.
        - Requested rerun scope overlaps active work; the conflicting request is not accepted, and the actor is informed which work prevents the rerun.

RADPS-UC7: Operator Annotation and Controlled Overrides

    Current Pipeline Cross-References:
        UC-17, GAP-07, GAP-08.

    Relevant Stakeholders
        Operations, QA reviewers, Science support, Audit and reproducibility consumers.
    Actors:
        Operator, QA reviewer.
    Goals:
        Record human decisions (notes, approvals, override parameters) as durable, auditable context state.
    Preconditions:
        A run exists; the target run, node, or artifact exists.
    Postconditions / Outputs:
        Annotations and overrides are linked to their targets and included in subsequent reporting. Updating a decision preserves its prior value, actor, action, target, and time.
    Required Information / Artifacts:
        Annotation or override content, target, actor, time, rationale, approval status, and applicable policy.
    Basic Flow:
        1. Operator submits an annotation or override request.
        2. Context system validates the target and requested change and records it.
    Alternative Flows (optional):
        - The target does not exist or the requested change is invalid; the request is rejected with a clear explanation.

RADPS-UC8: Export Provenance Manifest / Report as an Artifact

    Current Pipeline Cross-References:
        UC-15, UC-19, GAP-03, GAP-05.

    Relevant Stakeholders
        Operations, QA reviewers, Reporting consumers, Archive consumers, Audit and reproducibility consumers.
    Actors:
        Report producer.
    Goals:
        Produce machine-readable manifests and human-readable reports from durable run, artifact, and provenance information.
    Preconditions:
        Sufficient run, artifact, and provenance information is available.
    Postconditions / Outputs:
        A manifest/report artifact is available with lineage back to the run. Repeating an export from the same context boundary produces a semantically equivalent manifest under the declared determinism policy. The source context boundary and tool versions remain available.
    Required Information / Artifacts:
        Run identity and history, attempts, artifacts, annotations, provenance, and the resulting manifest or report artifact.
    Basic Flow:
        1. Report producer obtains the run summary, attempts, artifacts, annotations, and provenance.
        2. Report producer creates the manifest or report.
        3. Report producer makes the output available as an artifact associated with the run.
    Alternative Flows (optional):
        - Required information is missing; reporting fails with a clear list of missing context elements.

RADPS-UC9: Apply Artifact Retention and Cleanup

    Current Pipeline Cross-References:
        UC-12, UC-19.

    Relevant Stakeholders
        Operations, Audit and reproducibility consumers.
    Actors:
        Retention actor, Operator.
    Goals:
        Apply retention/cleanup without breaking resumability or provenance.
    Preconditions:
        Retention policy exists; artifacts have known locations.
    Postconditions / Outputs:
        Each affected artifact remains available or is identified as no longer available, and any deletion remains traceable. The time, actor, scope, rationale, and cleanup outcome remain available.
    Required Information / Artifacts:
        Retention policy and decision, affected artifact and locations, hold information, rationale, and cleanup outcome.
    Basic Flow:
        1. Retention actor identifies candidates under the applicable policy.
        2. Context system preserves each retention decision and the resulting artifact availability.
        3. Storage cleanup occurs (out of scope here), and completion is recorded.
    Alternative Flows (optional):
        - Artifact is on hold due to an active investigation; cleanup is skipped and hold reason recorded.

RADPS-UC10: Query Dataset / Observation Catalog (Read-Only View)

    Current Pipeline Cross-References:
        UC-01, UC-02, GAP-08.

    Relevant Stakeholders
        Pipeline developers, QA reviewers, Reporting consumers.
    Actors:
        Worker, Heuristic, QA or reporting consumer, External consumer.
    Goals:
        Provide timely, consistent, and unambiguous observation information, including catalog inventory, fields, SPWs, scans, data types, and cross-dataset identities and matches, for tasks, heuristics, external consumers, and reporting. This information supports more specialized matching workflows such as UC21.
    Preconditions:
        A run exists; its dataset inventory is available.
    Postconditions / Outputs:
        No new durable state is required; consumers obtain a consistent, unambiguous view of the requested metadata.
    Required Information / Artifacts:
        Dataset inventory; field, SPW, scan, and data-type information; cross-dataset identities and matches; and the requested selection scope.
    Basic Flow:
        1. Consumer requests catalog data for a scope (by dataset, field/spw/scan partition, logical selection, or data type).
        2. Context system provides the requested information in an unambiguous form.
    Alternative Flows (optional):
        - Requested scope not found; return a structured “unknown dataset/partition” error.
        - The request cannot be interpreted unambiguously; it fails with a clear explanation.

RADPS-UC11: Apply Calibration State Update

    Current Pipeline Cross-References:
        UC-04.

    Relevant Stakeholders
        Pipeline developers, Operations, QA reviewers.
    Actors:
        Worker.
    Goals:
        Record a set of calibration state changes produced by a task as one complete outcome.
    Preconditions:
        Producing attempt exists; calibration artifacts (e.g., tables) are durably available or are submitted as part of the calibration update.
    Postconditions / Outputs:
        Calibration state version advances; downstream consumers see either the old or new version, never a mix. The update remains linked to the producing attempt.
    Required Information / Artifacts:
        Calibration changes, affected scope, producing attempt, and related calibration artifacts.
    Basic Flow:
        1. Worker submits a calibration-state update containing multiple related changes and references to produced artifacts.
        2. Context system validates and accepts the changes as one complete calibration-state update.
    Alternative Flows (optional):
        - A concurrent incompatible update has been accepted; the submitted update is rejected, and the worker must retry using the current calibration state.

RADPS-UC12: Update Imaging State

    Current Pipeline Cross-References:
        UC-05.

    Relevant Stakeholders
        Pipeline developers, Operations, Reporting consumers.
    Actors:
        Worker.
    Goals:
        Preserve imaging state changes for a defined processing scope so downstream work, resume, and rerun use the appropriate state.
    Preconditions:
        A run exists; the plan node identifies which imaging scope is being updated (field, SPW, scan, or equivalent).
    Postconditions / Outputs:
        Imaging state is updated for the intended scope; downstream readers can resolve the appropriate state. The actor, producing attempt, and resulting state identity remain available.
    Required Information / Artifacts:
        Imaging state changes, affected scope, producing attempt, and related artifacts such as masks, thresholds, sensitivities, or beam models.
    Basic Flow:
        1. Worker submits imaging state update scoped to a partition.
        2. Context system validates and accepts the update.
    Alternative Flows (optional):
        - The submitted imaging information cannot be interpreted or validated; the update is rejected with a clear explanation.

RADPS-UC13: Provide a Consistent Run View for QA, Reporting, Rendering, and Debugging

    Current Pipeline Cross-References:
        UC-15, UC-16, UC-17, UC-19.

    Relevant Stakeholders
        QA reviewers, Reporting consumers, Pipeline developers, Operations.
    Actors:
        QA or reporting consumer, Debugging or inspection consumer, Regression consumer, External consumer.
    Goals:
        Provide a consistent view of run state and artifact information for rendering, QA, export, inspection, and debugging. Consumers can determine what ran, what data was used, what state was produced, and what failures occurred without access to the worker runtime (Pipeline UC-15, UC-16, UC-17, UC-19).
    Preconditions:
        A checkpoint exists or a consistent snapshot boundary is defined (e.g., “as of attempt X completion”).
    Postconditions / Outputs:
        Consumers obtain a coherent view of the selected run boundary; optional derived products can be produced and associated with the run. Re-rendering from the same view produces a semantically equivalent result under the declared determinism policy. The state boundary used by a report or inspection remains identifiable.
    Required Information / Artifacts:
        Run state, artifact information, annotations, and execution history (UC3) at the selected boundary.
    Basic Flow:
        1. Consumer requests run information at a defined boundary.
        2. Context system provides a consistent view for that boundary.
    Alternative Flows (optional):
        - The requested boundary is not available; the consumer may request the latest accepted state with the limitation made explicit.
        - A debugging or regression consumer compares outputs across runs or validates expected artifacts and QA outcomes.

RADPS-UC14: Resolve Upstream Outputs by Stable Identity

    Current Pipeline Cross-References:
        UC-09.

    Relevant Stakeholders
        Pipeline developers, Operations.
    Actors:
        Worker, Downstream consumer.
    Goals:
        Allow downstream tasks and consumers to discover upstream outputs by stable names, types, and scopes.
    Preconditions:
        Upstream outputs are available with stable names, types, and scopes.
    Postconditions / Outputs:
        Downstream consumers can bind required inputs deterministically, and the artifacts that satisfied those inputs remain traceable.
    Required Information / Artifacts:
        Logical output name, type, scope, version when relevant, artifact identity, descriptive information, and usable location.
    Basic Flow:
        1. Consumer requests the latest output of a given type and scope or requests a specific version.
        2. Context system provides the matching artifact identities and information.
    Alternative Flows (optional):
        - Output not found; consumer fails fast with a structured missing-dependency error.

RADPS-UC15: Preserve Run Change History for Audit and Reconstruction

    Current Pipeline Cross-References:
        UC-08, UC-17, GAP-05.

    Relevant Stakeholders
        Operations, Pipeline developers, QA reviewers, External system operators, Audit and reproducibility consumers.
    Actors:
        Workflow orchestration layer, Worker, Reporting or monitoring consumer.
    Goals:
        Preserve an ordered history of significant lifecycle outcomes and state changes so that run evolution is auditable and, where feasible, reconstructable without altering prior history.
    Preconditions:
        A run exists.
    Postconditions / Outputs:
        Consumers can inspect an ordered history of significant changes and their relationships to the relevant plan, node, attempt, or artifact. Repeated reporting of the same logical change does not create duplicate history.
    Required Information / Artifacts:
        Significant state changes, their order and time, responsible actors, and relationships to planned work, attempts, and artifacts.
    Basic Flow:
        1. Producer reports a significant lifecycle outcome or state change with its relevant context.
        2. Context system validates and incorporates the change without altering prior history.
        3. Consumers inspect the resulting history by run, node, or time range.
    Alternative Flows (optional):
        - The same logical change is reported again; Context system preserves the existing history without adding a duplicate outcome.

RADPS-UC16: Maintain Domain-Specific Context Information (ngVLA/WSU)

    Current Pipeline Cross-References:
        UC-18.

    Relevant Stakeholders
        Domain teams, Pipeline developers, Operations.
    Actors:
        Worker, Workflow orchestration layer.
    Goals:
        Make domain-specific state available for a run and its relevant dataset or partition scopes while preserving stable shared context behavior.
    Preconditions:
        The extension type is recognized for the run.
    Postconditions / Outputs:
        Extension state is available for the run and, when relevant, its dataset or partition scope. Changes remain distinguishable and attributable to the responsible actor.
    Required Information / Artifacts:
        Extension type, declared validation contract or structure, scope, state, responsible actor, time, and related artifacts.
    Basic Flow:
        1. Workflow orchestration layer or operator enables an extension type for a run.
        2. Workers submit scoped extension updates during execution.
        3. Consumers obtain the extension state for the relevant scope.
    Alternative Flows (optional):
        - The extension type or submitted information is not recognized or is inconsistent with the declared type, scope, or schema; the update is rejected with a clear explanation.

RADPS-UC17: Worker Consistent Read and State Update (Distributed Execution)

    Current Pipeline Cross-References:
        UC-09, UC-13, UC-14, GAP-01, GAP-02.

    Relevant Stakeholders
        Operations, Pipeline developers.
    Actors:
        Workflow orchestration layer, Worker.
    Goals:
        Allow workers to obtain a coherent view of required context state and submit complete outcomes during distributed execution. Independent work may proceed asynchronously across partitions, while incompatible updates are detected and rejected.
    Preconditions:
        A run exists; a consistent read boundary exists (checkpoint, plan revision boundary, or latest accepted state); the requested operation is supported.
    Postconditions / Outputs:
        Worker obtains a reference to the state boundary used for computation; accepted updates become visible as complete outcomes linked to the producing attempt. The boundary, worker, and producing attempt remain identifiable.
    Required Information / Artifacts:
        Selected state boundary, required run and observation information, attempt state, produced artifacts, and resulting context changes.
    Basic Flow:
        1. Worker requests a coherent view for its node/partition scope.
        2. Worker performs computation using that view.
        3. Worker registers artifacts and submits context updates.
        4. Context system accepts the updates as a complete outcome and updates derived state.
    Alternative Flows (optional):
        - An update conflicts with a concurrently accepted incompatible update; worker must request a current view and retry.
        - Consumer requests an unsupported operation; the request is rejected with a clear explanation.

RADPS-UC18: Provide Run State to External Consumers

    Current Pipeline Cross-References:
        UC-07, UC-15, UC-19, GAP-05.

    Relevant Stakeholders
        Operations, QA reviewers, Archive consumers, External system operators.
    Actors:
        External consumer.
    Goals:
        Ensure external consumers can obtain timely, stable processing state, lifecycle changes, and summary information from durable context information that remains available independently of worker execution.
    Preconditions:
        A run exists; the requested information is available.
    Postconditions / Outputs:
        External consumers receive or can obtain the requested state or summary information. Attempts to provide changes and their outcomes remain traceable with the consumer identity and affected scope.
    Required Information / Artifacts:
        Requested run state, artifact information, QA outcomes, processing changes, summary information, and any relevant exported summary or manifest.
    Basic Flow:
        1. External consumer identifies the run state, lifecycle changes, or summary information it needs.
        2. Context system makes the requested information available.
        3. The external consumer receives or retrieves the information, and the outcome is retained for audit.
    Alternative Flows (optional):
        - The request cannot be interpreted or supported; it fails with a clear explanation.
        - The external consumer is unavailable; the failed attempt remains traceable, and the information is not lost.

RADPS-UC19: Capture Reproducibility Information and Immutable Attempt Provenance

    Current Pipeline Cross-References:
        UC-08, UC-17, UC-19, GAP-03.

    Relevant Stakeholders
        Operations, QA reviewers, Audit and reproducibility consumers.
    Actors:
        Worker, Report producer.
    Goals:
        Capture the immutable provenance required to reproduce or audit a run: verifiable input identities, parameters, software versions, execution environment, hardware, execution-control details, and lineage for each attempt and exported product.
    Preconditions:
        An attempt or export operation exists; input identities and execution-environment information are available.
    Postconditions / Outputs:
        Immutable provenance information exists and is linked to the relevant attempt, artifact, or exported manifest. Missing or partial information is explicitly identifiable.
    Required Information / Artifacts:
        Verifiable input identities, parameters, software versions, execution environment, hardware and resource information, execution-control details, lineage, and any declared limits on reproducibility.
    Basic Flow:
        1. Worker or report producer supplies the required input, execution, and environment information.
        2. Context system validates and preserves the provenance linked to the attempt or artifact. Where required by policy, attempt completion or artifact availability is accepted only when this linked provenance is also accepted.
        3. Downstream reporting and export consumers obtain the provenance associated with the relevant attempt or artifact.
    Alternative Flows (optional):
        - Some required input or environment information is unavailable at completion time; Context system identifies the missing information and may prevent checkpoint or export under the applicable policy.
        - Environment details change mid-run; new attempts preserve their own environment information without altering prior provenance.

RADPS-UC20: Register Incremental Dataset Updates and Versioned Results

    Current Pipeline Cross-References:
        UC-01, UC-12, GAP-04.

    Relevant Stakeholders
        Operations, Pipeline developers, External system operators.
    Actors:
        Incremental data provider, Workflow orchestration layer.
    Goals:
        Add identifiable data to an active run or session, initiate the required incremental processing, and preserve both prior and newly produced results with distinguishable identities.
    Preconditions:
        A run exists; the applicable policy allows incremental data; incoming data is identifiable and scoped to an existing or new session partition.
    Postconditions / Outputs:
        The new dataset or version is available in the dataset catalog; the affected processing scope is identifiable; newly produced artifacts and results are distinguishable from prior outputs. The new data identity and related downstream invalidation become visible as one complete update. The source, actor, and time remain available.
    Required Information / Artifacts:
        New data identity and source, relationship to prior data, affected processing scope and dependencies, applicable policy, and identities of resulting artifacts and results.
    Basic Flow:
        1. Incremental data provider supplies new dataset material or a new data identity for an active run.
        2. Context system accepts the new data identity and identifies the affected processing scope.
        3. Workflow orchestration layer uses the affected scope to determine and schedule any additional processing.
        4. Context system associates subsequent outputs with the new data identity.
    Alternative Flows (optional):
        - Incoming data conflicts with an existing immutable data identity; the update is rejected unless it can be accepted as a distinct version under the applicable policy.
        - Incremental registration arrives while dependent work is running; the conflicting update is not accepted until the applicable processing policy determines how to proceed.

RADPS-UC21: Resolve Heterogeneous Cross-Dataset Matches and Override Rules

    Current Pipeline Cross-References:
        UC-02, UC-18, GAP-08.

    Relevant Stakeholders
        Pipeline developers, Operations, Science support.
    Actors:
        Worker, Heuristic, Operator.
    Goals:
        Resolve shared identity across heterogeneous datasets that do not share native SPW numbering, field numbering, source labels, or data-column layouts. Matching must support exact semantics for calibration-style consumers, overlap/partial semantics for imaging-style consumers, and explicit override rules with recorded rationale when defaults are ambiguous or incorrect.
    Preconditions:
        Relevant datasets and their identifying information are available; the applicable matching rules are known.
    Postconditions / Outputs:
        Consumer receives a resolved match set and, when overrides are supplied, they remain available with their rationale, scope, actor, and time. Replacing an override preserves its prior value, rationale, and scope.
    Required Information / Artifacts:
        Dataset identities; field, source, SPW, and column information; matching mode and policy; existing overrides; and the rationale for any new override.
    Basic Flow:
        1. Consumer requests a match set for a scope and matching mode.
        2. Context system evaluates dataset identities, the matching policy, and any existing overrides.
        3. Context system provides the resolved match set and preserves any newly supplied override.
    Alternative Flows (optional):
        - Multiple candidate matches remain after policy evaluation; Context system provides the candidates and indicates that heuristic or operator input is required.
        - An override conflicts with an existing locked mapping; Context system rejects it unless an explicit replacement is permitted by the applicable policy.

RADPS-UC22: Initialize Context from Intermediate Archival State

    Current Pipeline Cross-References:
        UC-12, GAP-06.

    Relevant Stakeholders
        Operations, Archive consumers, External system operators.
    Actors:
        Archive importer, Operator.
    Goals:
        Establish a valid intermediate processing boundary from pre-existing archival products or other durable artifacts so earlier work can be treated as complete and later work can proceed.
    Preconditions:
        Archival products are identifiable and accessible; the target run exists or is being created; the import policy identifies which prior stages are considered satisfied.
    Postconditions / Outputs:
        The dataset, artifact, calibration, imaging, and provenance information required for the imported boundary is available and linked to its archival sources; the boundary is valid for resume. The boundary is not available for downstream use unless all required information has been accepted. The importing actor and any information inferred rather than directly imported remain identifiable.
    Required Information / Artifacts:
        Archival source identities and products, intended processing boundary, required dataset and processing state, related artifacts, provenance, and any inferred information.
    Basic Flow:
        1. Archive importer supplies archival products and identifies the intended processing boundary.
        2. Context system validates the source products, required information, and compatibility of the imported state.
        3. Context system makes the imported artifacts and corresponding processing state available.
        4. Context system identifies the resulting state as a valid resume boundary for downstream workflow logic.
    Alternative Flows (optional):
        - Imported products are insufficient to construct a valid boundary; initialization is rejected with a list of missing state elements.
        - Imported information conflicts with immutable run state already present; Context system rejects the import and explains the conflict.
        - Imported state uses an unsupported or incompatible context-model version; Context system rejects the import with migration or compatibility guidance.

RADPS-UC23: Maintain Execution-Control Tags for Workflow Decisions

    Current Pipeline Cross-References:
        GAP-07.

    Relevant Stakeholders
        Operations, Pipeline developers, Audit and reproducibility consumers.
    Actors:
        Operator, Heuristic, Workflow orchestration layer.
    Goals:
        Keep execution-control tags, such as pause, skip, or reroute directives, available for runs, datasets, or processing scopes so the workflow orchestration layer can apply them throughout the lifetime of the run.
    Preconditions:
        Target run/dataset/stage scope exists.
    Postconditions / Outputs:
        Execution-control tags remain available with their scope, value, rationale, author, and lifecycle information. Updating a tag preserves its prior state and attribution.
    Required Information / Artifacts:
        Tag scope, value, rationale, author, effective time, lifecycle state, and any existing conflicting tag.
    Basic Flow:
        1. Actor submits an execution-control tag for a run, dataset, or stage scope.
        2. Context system validates the scope and any existing conflicting tag state.
        3. Context system accepts the tag and provides its effective state.
        4. Workflow orchestration layer obtains the tag state before scheduling or continuing affected work.
    Alternative Flows (optional):
        - Submitted tag conflicts with a locked or already-effective control decision; request is rejected unless an explicit override is permitted by the applicable policy.
        - Workflow orchestration layer requests tags for an unknown scope; Context system returns a structured not-found error.
