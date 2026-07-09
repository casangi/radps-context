# RADPS context: mapping current Pipeline context use cases to next-gen requirements

This note maps the **current Pipeline context** use cases documented in [docs/context_use_cases_current_pipeline.md](context_use_cases_current_pipeline.md) to the **RADPS** execution model and derives a concrete context specification suitable for distributed, ACID, restartable processing.

[docs/requirements_and_ownership.md](requirements_and_ownership.md) provides the companion requirement trace, linking the current use cases and identified GAP scenarios to specific RADPS requirements and assigning implementation ownership across `radps-context` and the workflow orchestration layer.

See also:

- [docs/radps_context_design_use_cases.md](radps_context_design_use_cases.md) (draft RADPS context use cases)
- [docs/context_use_cases_current_pipeline.md](context_use_cases_current_pipeline.md) (source Pipeline use cases)
- [docs/glossary.md](glossary.md) (definitions: ACID, dependency graph, idempotency, etc.)

## Assumptions (from RADPS discussions)

- **Domains**: ALMA WSU, ngVLA, VLBI, etc.
- **Data model**: archive inputs (e.g., ASDM) convert to a normalized observation representation.
- **Scale**: multi-TB artifacts, very large spectral cubes (up to ~1.2M channels), significantly increased SPWs. These figures should be treated as **top-end outputs**, not the typical case.
- **Efficiency (common case)**: the context design must scale to larger datasets, but should remain efficient for smaller/more common runs; avoid “worst-case-first” tradeoffs that significantly degrade day-to-day performance.
- **Execution**: planner service generates a **dependency graph at runtime**; execution is distributed, with concurrent workers and overlapping work whenever dependencies allow.
- **State semantics**: shared state must have **ACID semantics** (multi-writer correctness is required).
- **Operations**: intermediates must remain **locally available** (for pause/inspect/manual intervention); resumability, targeted reruns, and incremental updates are required.
- **Interfaces**: workers and external systems need stable, typed query, update, and event interfaces to the current processing state.
- **Platform**: execution must remain portable across current and future infrastructure choices.
- **Provenance**: must emit machine-readable manifests/audit trails; determinism is “within numerical precision” and depends on identical inputs, versions, hardware, and scheduler/resource envelopes.

## Scope note (context design)

In RADPS terms, “context” means the **durable run state + artifact/provenance references** needed to:

- resume/retry safely
- support concurrent writers (ACID)
- allow operator inspection/annotation
- regenerate reports/manifests
- serve stable query/update/event interfaces to workers and external consumers

Out of scope here (though context must *record* their outcomes):

- scheduler specifics, worker placement, autoscaling
- UI details
- algorithm design
- full orchestration implementation (we assume a planner/executor exists)

Within that scope, the planner and executor are **actors** that read/write context, and the context store is the system-of-record.

## Requirement trace and ownership alignment

The requirement-trace note introduces two design constraints that materially affect this mapping:

- Not every requirement-traced use case becomes a `radps-context` use case. Some are workflow-only, some are shared between workflow and context, and some require coordination with external metadata representations without changing the core context specification.

For context design, the ownership split is:

| Ownership area | Requirement-traced items | Context-design implication |
|---|---|---|
| `radps-context` primary owner | UC-01, UC-02, UC-03, UC-04, UC-05, UC-10, UC-15, UC-16, UC-18, UC-19, GAP-08 | These map directly to durable state, artifact/query surfaces, QA/reporting views, catalog and matching behavior, and domain-specific records. |
| Workflow orchestration primary owner | UC-07, UC-08, GAP-01 | These should influence the context specification, but they are not themselves modeled as standalone context use cases. |
| Shared between workflow and `radps-context` | UC-06, UC-09, UC-11, UC-12, UC-17, GAP-03, GAP-04, GAP-05, GAP-06, GAP-07 | These require explicit context behavior plus workflow-side enforcement, replay, or delivery logic. |

Two current-pipeline use cases are explicitly discarded in the requirement trace:

- **UC-13 Provide State to Parallel Workers** is replaced by stateless workers plus asynchronous task-graph execution in GAP-01.
- **UC-14 Aggregate Results from Parallel Workers** is likewise replaced by asynchronous task graphs and direct artifact/state registration in GAP-01.

## Pipeline responsibilities to RADPS context subsystem mapping

The Pipeline analysis collapses to 15 broad responsibilities. The table below maps each responsibility to its current Pipeline use cases and RADPS context counterparts for cross-reference.

| # | Pipeline Responsibility | Current Pipeline UCs / GAPs | RADPS UCs | Notes |
|---|---|---|---|---|
| 1 | Static Observation & Project Data | UC-01, UC-03 | RADPS-UC1, RADPS-UC10 | Stable identifiers and cross-dataset identity records replace MS-name lookups and single-master assumptions. |
| 2 | Mutable Observation State | UC-01, UC-02 | RADPS-UC10, RADPS-UC20, RADPS-UC21 | Derived datasets are registered with lineage and version history instead of mutating the original inventory in place. |
| 3 | Path Management | UC-12, UC-19, GAP-02 | RADPS-UC1, RADPS-UC4 | Paths become artifact locations or access policies, not embedded process-local strings. |
| 4 | Imaging State Management | UC-05 | RADPS-UC12 | Typed, versioned, partition-scoped. |
| 5 | Calibration State Management | UC-04 | RADPS-UC11 | Transactional, multi-entry, versioned, removable. |
| 6 | Image Library Management | UC-06 | RADPS-UC4, RADPS-UC8 | Separate views (science/cal/RMS/sub-product) can be layered over the registry if needed. |
| 7 | Session Persistence | UC-11, UC-12 | RADPS-UC1, RADPS-UC5, RADPS-UC6, RADPS-UC22 | Explicit persisted state replaces implementation-bound session snapshots and supports portable restore semantics. |
| 8 | Parallel Distribution | UC-13, UC-14, GAP-01, GAP-02 | RADPS-UC17 | Consistent snapshot reads and ACID write-back support overlapping execution. |
| 9 | Inter-Task Data Passing | UC-09 | RADPS-UC14, RADPS-UC17 | Replaces implicit state merging plus results-list walking with named outputs and transactional updates. |
| 10 | Stage Tracking & Result Accumulation | UC-07, UC-08 | RADPS-UC2, RADPS-UC3 | Current node state plus ordered attempt history replaces current-pipeline stage numbering as the main execution record. |
| 11 | Reporting & Export Support | UC-15, UC-19 | RADPS-UC8, RADPS-UC13, RADPS-UC18 | Stable interfaces support weblogs, manifests, scripts, dashboards, and product packaging. |
| 12 | QA Score Storage | UC-16 | RADPS-UC3, RADPS-UC13 | Read-only context snapshots for QA handlers retain per-selection detail. |
| 13 | Debuggability / Inspectability | UC-17 | RADPS-UC13, RADPS-UC15, RADPS-UC19 | Queryable inspection surfaces replace opaque serialized-state inspection as the primary debugging interface. |
| 14 | Telescope- and Array-Specific State | UC-18 | RADPS-UC16 | Typed, versioned, per-dataset scoped. |
| 15 | Lifecycle Notifications | UC-07, UC-08, UC-15 | RADPS-UC15, RADPS-UC18 | Append-only audit trail plus external event delivery. |

## What changes materially in RADPS (vs current Pipeline)

### Orchestration: from linear “commands” to planned dependency graphs

- The operational contract becomes: **inputs + policies → plan (dependency graph) → execution**.
- Planning is explicit and versioned; execution is a separate concern.
- “Mini-graphs” per (field, spw, scan) imply:
  - Many nodes, high fan-out
  - Strong need for **granular checkpointing** and **artifact reference hygiene**

### State: from single-process mutable object to shared ACID ledger

The biggest semantic change is multi-writer concurrency:

- The “context” is no longer an in-process mutable object that tasks modify directly.
- Instead, tasks emit **transactional updates** (events or patches) to a run-scoped store.
- Consumers (planner, UI, report generators, operators, external dashboards) read from the same consistent store.

### Interfaces: from in-process access to stable APIs and event feeds

- The current pipeline assumes in-process attribute access; RADPS must expose the same state through stable, typed APIs.
- External consumers (dashboards, schedulers, archive systems) must be able to query current state or subscribe to lifecycle events without scraping product files or sharing a filesystem.
- API/schema versioning becomes part of the context specification rather than an afterthought.

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

## Proposed RADPS Context Specification (minimum viable)

A workable next-gen context needs a small set of concepts with stable identifiers.

### Identifiers

- Run identifier: globally unique identifier for a pipeline run (immutable)
- Dataset identifier: stable identifier for an imported or derived dataset version within a run
- Plan identifier: identifier for the planned dependency-graph version used by the run
- Node identifier: stable identifier for a dependency-graph node (task instance)
- Attempt identifier: unique identifier per node execution attempt (retries, reschedules)
- Artifact identifier: stable identifier for a produced artifact

### Run ledger (ACID store)

Minimum records/relations (conceptually):

- **Run metadata**: domain (WSU/ngVLA), operator/mode, timestamps, priority, tenant/project, policy bundle, orchestration driver identity
- **Inputs**: datasets + versions, conversion outputs, parameterization
- **Dataset/Observation Catalog**: normalized observation inventory, per-partition metadata (field/spw/scan), data-type metadata, and cross-dataset identity/matching records needed by tasks, QA, and rendering
- **Calibration State**: versioned calibration application state
- **Imaging State**: schema’d imaging configuration/scratch-pad state (partition-scoped where possible)
- **Plan**: dependency-graph structure, node definitions, resource intents (CPU/mem/IO locality), partitioning keys
- **Node execution state**:
  - status values covering pending, in-progress, successful, failed, skipped, or canceled execution
  - timing, resources used, worker identity, error summaries
  - immutable provenance envelope (input hashes, software, environment, hardware, and execution metadata)
  - explicit checkpoints / “safe restart points”
- **External integration state**: subscription definitions, delivery history, materialized summary views, and supported API/schema versions
- **Events** (optional but strongly useful): append-only timeline for audit, debugging, and integration feeds

Implementation constraint: updates must be **transactional** and safe under concurrent writers.

### Artifact registry

For each artifact record:

- Type (dataset partition, calibration table, image cube, plot bundle, weblog, manifest, etc.)
- Producer reference (the originating node and attempt)
- Lineage (input artifact references)
- Locations (storage-agnostic references and access policy references)
- Version / supersedes links when reruns or incremental updates produce new results
- Lifecycle state (retention policy, garbage-collect eligibility)

### Checkpoint/resume semantics

- Checkpoints are explicit objects in the ledger, not implicit serialized process snapshots.
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
- Re-plan or re-run a subset with modified parameters (tracked as a new plan revision or policy revision)

## Additional RADPS context use cases (context-scoped)

Concrete RADPS context use cases are drafted in [docs/radps_context_design_use_cases.md](radps_context_design_use_cases.md). These cover:

- run + plan lifecycle (RADPS-UC1-RADPS-UC3)
- artifact + checkpoint lifecycle (RADPS-UC4-RADPS-UC9)
- “internal pipeline interactions” equivalents: catalog queries, calibration/imaging state, snapshots, named outputs, event log, domain extensions, and worker snapshot/write-back semantics (RADPS-UC10-RADPS-UC17)
- external integrations, reproducibility envelopes, incremental processing, initialization from intermediate state, execution-control tags, and heterogeneous matching semantics (RADPS-UC18-RADPS-UC23), with stable typed-interface constraints distributed across the concrete query, mutation, and subscription use cases

## “Missing today” capabilities (GAPs) and RADPS context implications

Using the requirement-derived GAP set from [docs/requirements_and_ownership.md](requirements_and_ownership.md), the current design gaps are:

- **GAP-01 Asynchronous execution of independent work**: requires snapshot isolation, transactional merges, partition-scoped writes, and conflict detection so concurrent task results integrate without corruption. This is primarily a workflow concern, but its context specification is handled mainly by RADPS-UC3 and RADPS-UC17.
- **GAP-02 Distributed execution without a shared filesystem**: requires artifact references decoupled from POSIX paths and a context that can serve as the system-of-record for artifact locations and access across nodes. This is handled primarily by RADPS-UC4 and RADPS-UC17.
- **GAP-03 Provenance and reproducibility**: requires immutable per-attempt records, input hashing, and lineage capture so past runs can be precisely reproduced or audited. This is handled primarily by RADPS-UC3, RADPS-UC8, and RADPS-UC19.
- **GAP-04 Partial re-execution / targeted stage re-run**: requires explicit dependency tracking and invalidation semantics at the context level so selective re-runs can invalidate or preserve downstream state correctly. This is handled primarily by RADPS-UC6.
- **GAP-05 External system integration**: requires stable identifiers, event subscriptions, and exportable summaries/manifests so external dashboards and ingest systems can track state without waiting for offline products. This is handled primarily by RADPS-UC15 and RADPS-UC18.
- **GAP-06 Initialization from intermediate state**: requires the context to ingest archival products and materialize a valid mid-pipeline state that the workflow layer can resume from without replaying earlier stages. This is handled primarily by RADPS-UC22.
- **GAP-07 Explicit tag-based execution control**: requires persisted execution-control tags on datasets or stages so the workflow layer can pause, skip, or reroute work based on durable context state. This is handled primarily by RADPS-UC23.
- **GAP-08 Heterogeneous dataset coordination and flexible matching semantics**: requires flexible SPW/field/source/column matching semantics plus override hooks across heterogeneous collections rather than assuming a single-master-MS model. This is handled primarily by RADPS-UC10 and RADPS-UC21.

## Compatibility and determinism policy (pragmatic)

- **Backward compatibility**: support resuming runs within a defined release window (schema + node contract). Outside the window: re-plan and re-run may be required.
- **Determinism**: guarantee “same inputs + versions + resources ⇒ same results within numerical precision”; record the environment details needed to explain deviations (hardware, execution environment, key library versions) and treat any change as a new provenance envelope rather than mutating past records.

## Open questions (to finalize the contract)

- What is the authoritative ACID store choice, and what are the required transactional boundaries?
- Do we want **event-sourcing** (append-only events + derived views) or **state tables** (current-state rows), or a hybrid?
- What artifact location model is required in early phases, and what retention policy applies to intermediates?
- How is multi-tenancy enforced (namespacing, quotas, access control) and where do authz decisions live?
- What transport and schema strategy will back the stable context interface, and what compatibility window is required?
- How should external subscriptions be authenticated, retried, and rate-limited?
- What is the minimum manifest schema required at the end of a run (and who consumes it)?
