# RADPS context: mapping current Pipeline context use cases to next-gen requirements

This note maps the **current Pipeline context** use cases documented in [docs/context_use_cases_current_pipeline.md](context_use_cases_current_pipeline.md) to the **RADPS** (WSU + ngVLA) execution model and derives a concrete “context contract” suitable for distributed, ACID, restartable processing.

See also:

- [docs/radps_context_design_use_cases.md](radps_context_design_use_cases.md) (draft RADPS context use cases)
- [docs/context_use_cases_current_pipeline.md](context_use_cases_current_pipeline.md) (source Pipeline use cases)
- [docs/context_gap_use_cases.md](context_gap_use_cases.md) (gap scenarios that expand the RADPS context contract)
- [docs/glossary.md](glossary.md) (definitions: ACID, DAG, idempotency, etc.)

## Assumptions (from RADPS discussions)

- **Domains**: ALMA WSU and ngVLA.
- **Data model**: archive inputs (e.g., ASDM) convert to **MeasurementSet v4**; Python ecosystem emphasis remains important, but non-Python clients must also be supported.
- **Scale**: multi-TB artifacts, very large spectral cubes (up to ~1.2M channels), significantly increased SPWs. These figures should be treated as **top-end outputs**, not the typical case.
- **Efficiency (common case)**: the context design must scale to larger datasets, but should remain efficient for smaller/more common runs; avoid “worst-case-first” tradeoffs that significantly degrade day-to-day performance.
- **Execution**: planner service generates a **DAG at runtime**; execution is **distributed (Dask)** with concurrent workers and overlapping work whenever dependencies allow.
- **State semantics**: shared state must have **ACID semantics** (multi-writer correctness is required).
- **Operations**: intermediates must remain **locally available** (for pause/inspect/manual intervention); resumability, targeted reruns, and incremental updates are required.
- **Interfaces**: external systems and non-Python clients need stable, language-neutral query and event interfaces to the current processing state.
- **Platform**: **Slurm now**, **Kubernetes later**.
- **Provenance**: must emit machine-readable manifests/audit trails; determinism is “within numerical precision” and depends on identical inputs, versions, hardware, and scheduler/resource envelopes.

## Scope note (context design)

In RADPS terms, “context” means the **durable run state + artifact/provenance references** needed to:

- resume/retry safely
- support concurrent writers (ACID)
- allow operator inspection/annotation
- regenerate reports/manifests
- serve stable query/update/event interfaces to workers and external consumers

Out of scope here (though context must *record* their outcomes):

- scheduler specifics (Slurm/K8s), worker placement, autoscaling
- UI details
- algorithm design
- full orchestration implementation (we assume a planner/executor exists)

Within that scope, the planner and executor are **actors** that read/write context, and the context store is the system-of-record.

## Use-case relevance mapping (Pipeline UC-01 through UC-19)

The Pipeline use cases are organized by the 15 context responsibilities identified in [context_use_cases_current_pipeline.md](context_use_cases_current_pipeline.md). The 19 UCs are not 1:1 with those responsibilities: several responsibilities are now teased apart into multiple explicit UCs (for example, UC-01/UC-02 for catalog behavior and UC-15/UC-19 for reporting/export). Each Pipeline UC maps one or more responsibilities to a concrete need that the pipeline’s central state management must satisfy.

Legend:
- **MUST**: RADPS must directly support this capability
- **SHOULD**: valuable; can be deferred, narrowed, or implemented differently
- **WON’T (as-is)**: the current mechanism doesn’t carry forward; replace with an alternative

| Pipeline UC | Summary (today) | RADPS relevance | RADPS interpretation / notes |
|---|---|---:|---|
| UC-01 | Populate, Access, and Provide Observation Metadata | **MUST** | Provide a high-performance **Dataset/Observation Catalog** that can register imported and derived datasets, preserve lineage, and serve cached metadata for the run lifetime. Replaces `context.observing_run` as the system-of-record. |
| UC-02 | Cross-MS Metadata Matching and Lookup | **MUST** | Extend the catalog with unified identifiers, cross-dataset identity records, and data-type-aware lookup. Preserve today’s baseline virtual-ID lookup while allowing richer matching semantics and override records for heterogeneous collections. |
| UC-03 | Store and Provide Project-Level Metadata | **MUST** | Keep run/project metadata as **immutable-after-init** (or versioned) ledger sub-records. Includes project summary, structure, performance parameters, processing intents, and driver-injected metadata. |
| UC-04 | Register, Query, and Update Calibration State | **MUST** | Represent calibration application state as a **typed, versioned sub-record** with **transactional multi-entry updates**, coverage/source tracking, and rollback/removal support. Replaces `context.callibrary`. |
| UC-05 | Manage Imaging State | **MUST** | Replace ad-hoc imaging attributes with a **schema’d imaging state document** (versioned) that supports partition-scoped updates and links to masks, thresholds, sensitivities, beam summaries, and related artifacts. |
| UC-06 | Register and Query Produced Image Products | **MUST** | Represent produced image products as typed artifact registry entries (science, calibrator, RMS, sub-products). Provide add/query semantics and lineage links so downstream tasks, reporting, and export can discover them by type and provenance. |
| UC-07 | Track Current Execution Progress | **MUST** | Replace stage counters with run-ledger execution status: current node/attempt state plus a stable ordered completion timeline across saves, resumes, and replans. |
| UC-08 | Preserve Per-Stage Execution Record | **MUST** | Store immutable per-attempt records with timing, invocation arguments, traceback/error summaries, outcomes, and resource/provenance metadata so reporting, auditing, and resume do not depend on worker memory. |
| UC-09 | Propagate Task Outputs to Downstream Tasks | **MUST** | Replace ad hoc `merge_with_context()` / results walking with **transactional state propagation** and **named outputs** typed by scope, so downstream tasks consume shared state rather than recipe-order accidents. |
| UC-10 | Provide a Transient Intra-Stage Workspace | **SHOULD** | Support child-task isolation through ephemeral workspaces, draft transactions, or other scoped mutation mechanisms so tentative edits are discardable unless explicitly accepted. |
| UC-11 | Support Multiple Orchestration Drivers | **MUST** | Keep the context contract driver-agnostic across batch, recipe, and interactive entry points while preserving driver metadata, breakpoint/resume controls, and equivalent run state. |
| UC-12 | Save and Restore a Processing Session | **MUST** | Replace pickle-based persistence with a **DB-backed run ledger + artifact registry + explicit checkpoints**. Restore must adapt artifact locations to a new filesystem or storage environment. |
| UC-13 | Provide State to Parallel Workers | **MUST** | Formalize **context snapshot semantics**: workers read a consistent, read-only snapshot token or boundary instead of receiving forked pickles as the system-of-record. |
| UC-14 | Aggregate Results from Parallel Workers | **MUST** | Formalize **transactional write-back and aggregation** with conflict detection, partition-scoped merges, and completion guarantees before dependent work begins. |
| UC-15 | Provide Read-Only State for Reporting | **MUST** | Provide **consistent read-only snapshot views** for weblogs, QA reports, reproducibility manifests, scripts, and packaging workflows via a stable query API. |
| UC-16 | Support QA Evaluation and Store Quality Assessments | **MUST** | Guarantee **read-only snapshot views** for QA handlers and store QA outcomes as typed records or artifacts linked to the relevant attempts and selections. |
| UC-17 | Support Inspection and Debugging | **MUST** | Preserve inspectable run state, machine-checkable failure signals, per-node logs, and failure snapshots behind a stable API rather than pickle inspection. |
| UC-18 | Manage Telescope- and Array-Specific State | **MUST** | Replace untyped sidecars with a **typed, versioned extension mechanism** scoped by run, dataset, and/or partition so shared pipeline code remains decoupled from domain-specific state. |
| UC-19 | Provide State for Product Export | **MUST** | Export reads from the artifact registry, run ledger, and reporting/QA records to assemble deliverables. Export manifests and packages are themselves registered as artifacts. |

## Pipeline responsibilities to RADPS context subsystem mapping

The Pipeline analysis still collapses to 15 broad responsibilities. The table below maps each responsibility to its RADPS context counterpart and notes where the newly separated UCs fit.

| # | Pipeline Responsibility | RADPS Context Subsystem | Notes |
|---|---|---|---|
| 1 | Static Observation & Project Data | Run Ledger (immutable-after-init metadata) + Dataset/Observation Catalog | MSv4-centric; stable IDs and cross-dataset identity records replace MS-name lookups and single-master assumptions. |
| 2 | Mutable Observation State | Dataset/Observation Catalog (versioned records) | Replaces ad-hoc MS mutation; derived datasets are registered with lineage and version history. |
| 3 | Path Management | Artifact Registry (storage-agnostic location references) | Paths become artifact locations or access policies, not embedded process-local strings. |
| 4 | Imaging State Management | Schema’d Imaging State sub-record (RADPS UC12) | Typed, versioned, partition-scoped. |
| 5 | Calibration State Management | Calibration State sub-record (RADPS UC11) | Transactional, multi-entry, versioned, removable. |
| 6 | Image Library Management | Artifact Registry (image-typed entries) | Separate views (science/cal/RMS/sub-product) if needed. |
| 7 | Session Persistence | Run Ledger + Artifact Registry + Checkpoints (RADPS UC5/UC6) | DB-backed; no pickle; restore across filesystems/storage backends. |
| 8 | MPI / Parallel Distribution | Worker Snapshot + Transactional Write-Back (RADPS UC17) | Consistent snapshot reads; ACID write-back; supports overlapping execution. |
| 9 | Inter-Task Data Passing | Transactional Updates + Named Outputs (RADPS UC14) | Replaces `merge_with_context` + results-list walking. |
| 10 | Stage Tracking & Result Accumulation | Run Ledger: Node Attempt Lifecycle (RADPS UC3) | Current node state plus ordered attempt history. |
| 11 | Reporting & Export Support | Read-Only Snapshots (RADPS UC8/UC13/UC18) + Artifact Registry | Stable API supports weblogs, manifests, scripts, dashboards, and product packaging. |
| 12 | QA Score Storage | QA records in Run Ledger or typed artifacts | Read-only context snapshots for QA handlers; retains per-selection detail. |
| 13 | Debuggability / Inspectability | Read-Only Snapshots (RADPS UC13) + Event Log (RADPS UC15) | Queryable via stable API, not pickle inspection. |
| 14 | Telescope- and Array-Specific State | Domain Extensions (RADPS UC16) | Typed, versioned, per-dataset scoped. |
| 15 | Lifecycle Notifications | Event Log / Patch Log + External Subscribers (RADPS UC15/UC18) | Append-only audit trail plus external event delivery. |

## What changes materially in RADPS (vs current Pipeline)

### Orchestration: from linear “commands” to planned DAGs

- The operational contract becomes: **inputs + policies → plan (DAG) → execution**.
- Planning is explicit and versioned; execution is a separate concern.
- “Mini-graphs” per (field, spw, scan) imply:
  - Many nodes, high fan-out
  - Strong need for **granular checkpointing** and **artifact reference hygiene**

### State: from single-process mutable object to shared ACID ledger

The biggest semantic change is multi-writer concurrency:

- The “context” is no longer a Python object that tasks mutate directly.
- Instead, tasks emit **transactional updates** (events or patches) to a run-scoped store.
- Consumers (planner, UI, report generators, operators, external dashboards) read from the same consistent store.

### Interfaces: from in-process Python access to stable APIs and event feeds

- The legacy pipeline assumes in-process Python attribute access; RADPS must expose the same state through typed, language-neutral APIs.
- External consumers (dashboards, schedulers, archive systems) must be able to query current state or subscribe to lifecycle events without scraping product files or sharing a filesystem.
- API/schema versioning becomes part of the context contract rather than an afterthought.

### Dataset coordination: from single-master-MS assumptions to explicit matching semantics

- The catalog must carry unified IDs, cross-dataset identity records, data-type metadata, and user/heuristic override history.
- Calibration-style consumers need exact matching semantics; imaging-style consumers may need overlap/partial matching across SPWs, fields, sources, and column layouts.
- Matching policy and override rationale become durable context state, not transient heuristics-only state.

### Artifacts: from “fields on context” to explicit artifact registry

- Large products (MS fragments, calibration tables, images, cubes, QA metrics, plots, manifests) are **artifacts** with:
  - stable IDs
  - content hashes (when feasible)
  - lineage links (which node/attempt produced them)
  - one or more storage-agnostic locations (local path, shared FS path, object store URI, access policy)

## Proposed RADPS “context contract” (minimum viable)

A workable next-gen context needs a small set of concepts with stable identifiers.

### Identifiers

- `run_id`: globally unique ID for a pipeline run (immutable)
- `dataset_id`: stable ID for an imported or derived dataset version within a run
- `plan_id`: ID for the planned DAG/graph version used by the run
- `node_id`: stable ID for a DAG node (task instance)
- `attempt_id`: unique ID per node execution attempt (retries, reschedules)
- `artifact_id`: stable ID for a produced artifact

### Run ledger (ACID store)

Minimum records/relations (conceptually):

- **Run metadata**: domain (WSU/ngVLA), operator/mode, timestamps, priority, tenant/project, policy bundle, orchestration driver identity
- **Inputs**: datasets + versions, conversion outputs (e.g., MSv4 import products), parameterization
- **Dataset/Observation Catalog**: MSv4 inventory, per-partition metadata (field/spw/scan), data-type metadata, and cross-dataset identity/matching records needed by tasks/QA/rendering
- **Calibration State**: versioned calibration application state (the callibrary analogue)
- **Imaging State**: schema’d imaging configuration/scratch-pad state (partition-scoped where possible)
- **Plan**: DAG structure, node definitions, resource intents (CPU/mem/IO locality), partitioning keys
- **Node execution state**:
  - status: `PENDING | RUNNING | SUCCEEDED | FAILED | SKIPPED | CANCELED`
  - timing, resources used, worker identity, error summaries
  - immutable provenance envelope (input hashes, software/env/hardware/scheduler metadata)
  - explicit checkpoints / “safe restart points”
- **External integration state**: subscription/webhook definitions, delivery history, materialized summary views, and supported API/schema versions
- **Events** (optional but strongly useful): append-only timeline for audit, debugging, and integration feeds

Implementation constraint: updates must be **transactional** and safe under concurrent writers.

### Artifact registry

For each `artifact_id`:

- `type` (MS partition, cal table, image cube, plot bundle, weblog, manifest, etc.)
- `producer` (`node_id`, `attempt_id`)
- `lineage` (input artifact IDs)
- `locations[]` (storage-agnostic references: local path(s), shared FS path(s), object store URI(s), access policy references)
- `version / supersedes` links when reruns or incremental updates produce new results
- `lifecycle` (retention policy, garbage-collect eligibility)

### Checkpoint/resume semantics

- Checkpoints are explicit objects in the ledger (not implicit “whatever was pickled”).
- Resume must support:
  - re-run from last successful checkpoint
  - targeted re-runs of subgraphs (e.g., one field/spw)
  - safe invalidation of downstream artifacts when upstream changes
  - incremental dataset updates that trigger new versions rather than overwriting old results

### Manual intervention (operator workflow)

A minimal supported set:

- Pause a run (stop scheduling new nodes)
- Inspect run state + artifacts (without reading worker-local memory)
- Add annotations, approvals, and explicit matching overrides with rationale
- Re-plan or re-run a subset with modified parameters (tracked as a new `plan_id` or policy revision)

## Additional RADPS context use cases (context-scoped)

Concrete RADPS context use cases are drafted in [docs/radps_context_design_use_cases.md](radps_context_design_use_cases.md). These cover:

- run + plan lifecycle (RADPS UC1-RADPS UC3)
- artifact + checkpoint lifecycle (RADPS UC4-RADPS UC9)
- “internal pipeline interactions” equivalents: catalog queries, calibration/imaging state, snapshots, named outputs, event log, domain extensions, and worker snapshot/write-back semantics (RADPS UC10-RADPS UC17)
- external integrations, reproducibility envelopes, language-neutral APIs, incremental processing, and heterogeneous matching semantics (RADPS UC18-RADPS UC22)

## “Missing today” capabilities (GAPs) and RADPS context implications

The Pipeline note also enumerates capabilities the current design cannot handle (GAP-01 through GAP-08: concurrency, cloud without shared FS, provenance/reproducibility, targeted reruns, external integrations, multi-language access, streaming/incremental, cross-MS coordination). Not all of these are strictly “context” features, but **each implies context changes**.

- **GAP-01 Asynchronous / overlapping execution**: requires snapshot isolation, transactional merges, partition-scoped writes, and conflict detection so concurrent task results integrate without corruption. This is handled primarily by RADPS UC3 and RADPS UC17.
- **GAP-02 Distributed execution without shared filesystem**: requires artifact references decoupled from POSIX paths and a context that can serve as the system-of-record for artifact locations and access. This is handled primarily by RADPS UC4 and RADPS UC17.
- **GAP-03 Provenance and reproducibility**: requires immutable per-attempt records, input hashing, and lineage capture so past runs can be precisely reproduced or audited. This is handled primarily by RADPS UC3, RADPS UC8, and RADPS UC19.
- **GAP-04 Partial re-execution / targeted rerun**: requires explicit dependency tracking and invalidation semantics at the context level so selective re-runs can invalidate or preserve downstream state correctly. This is handled primarily by RADPS UC6 and RADPS UC21.
- **GAP-05 External system integration**: requires stable identifiers, event subscriptions/webhooks, and exportable summaries/manifests so external dashboards and ingestion systems can track state without offline products. This is handled primarily by RADPS UC15 and RADPS UC18.
- **GAP-06 Programming-language / client-framework access**: requires a language-neutral contract and a stable middleware/API layer (typed schema + transport) so clients in any language can access context state without coupling to storage representation. This is handled primarily by RADPS UC20.
- **GAP-07 Streaming / incremental processing**: requires versioned dataset registration and versioned results/artifacts so new data can be incorporated incrementally and re-runs produce new versions rather than overwriting. This is handled primarily by RADPS UC21.
- **GAP-08 Cross-MS matching and heterogeneous dataset coordination**: requires flexible SPW matching semantics (exact and partial/overlap), data-type/column tracking, and override hooks across heterogeneous collections rather than assuming a single-master-MS model. This is handled primarily by RADPS UC10 and RADPS UC22.

## Compatibility and determinism policy (pragmatic)

- **Backward compatibility**: support resuming runs within a defined release window (schema + node contract). Outside the window: re-plan and re-run may be required.
- **Determinism**: guarantee “same inputs + versions + resources ⇒ same results within numerical precision”; record the environment details needed to explain deviations (hardware, scheduler/MPI, key library versions) and treat any change as a new provenance envelope rather than mutating past records.

## Open questions (to finalize the contract)

- What is the authoritative ACID store choice (SQLite for dev vs Postgres for ops), and what are the required transactional boundaries?
- Do we want **event-sourcing** (append-only events + derived views) or **state tables** (current-state rows), or a hybrid?
- What artifact location model is required in early phases (local-only, shared FS, object store), and what retention policy applies to intermediates?
- How is multi-tenancy enforced (namespacing, quotas, access control) and where do authz decisions live?
- What transport and schema strategy will back the language-neutral API (REST, gRPC, or another contract), and what compatibility window is required?
- How should external subscriptions/webhooks be authenticated, retried, and rate-limited?
- What is the minimum manifest schema required at the end of a run (and who consumes it)?
