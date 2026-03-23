# RADPS context: mapping current Pipeline context use cases to next-gen requirements

This note maps the **current Pipeline context** use cases documented in [docs/context_use_cases_current_pipeline.md](context_use_cases_current_pipeline.md) to the **RADPS** (WSU + ngVLA) execution model and derives a concrete “context contract” suitable for distributed, ACID, restartable processing.

See also:

- [docs/context_design_use_cases.md](context_design_use_cases.md) (draft RADPS context use cases)
- [docs/context_use_cases_current_pipeline.md](context_use_cases_current_pipeline.md) (source Pipeline use cases)
- [docs/glossary.md](glossary.md) (definitions: ACID, DAG, idempotency, etc.)

## Assumptions (from RADPS discussions)

- **Domains**: ALMA WSU and ngVLA.
- **Data model**: archive inputs (e.g., ASDM) convert to **MeasurementSet v4**; Python ecosystem emphasis.
- **Scale**: multi-TB artifacts, very large spectral cubes (up to ~1.2M channels), significantly increased SPWs. These figures should be treated as **top-end outputs**, not the typical case.
- **Efficiency (common case)**: the context design must scale to larger datasets, but should remain efficient for smaller/more common runs; avoid “worst-case-first” tradeoffs that significantly degrade day-to-day performance.
- **Execution**: planner service generates a **DAG at runtime**; execution is **distributed (Dask)** with concurrent workers.
- **State semantics**: shared state must have **ACID semantics** (multi-writer correctness is required).
- **Operations**: intermediates must remain **locally available** (for pause/inspect/manual intervention); resumability and partial reruns are required.
- **Platform**: **Slurm now**, **Kubernetes later**.
- **Provenance**: must emit machine-readable manifests/audit trails; determinism is “within numerical precision” and depends on identical inputs/versions/hardware/resources.

## Scope note (context design)

In RADPS terms, “context” means the **durable run state + artifact/provenance references** needed to:

- resume/retry safely
- support concurrent writers (ACID)
- allow operator inspection/annotation
- regenerate reports/manifests

Out of scope here (though context must *record* their outcomes):

- scheduler specifics (Slurm/K8s), worker placement, autoscaling
- UI details
- algorithm design
- full orchestration implementation (we assume a planner/executor exists)

Within that scope, the planner and executor are **actors** that read/write context, and the context store is the system-of-record.

## Use-case relevance mapping (Pipeline UC-01 through UC-18)

The Pipeline use cases are organized by the 15 context responsibilities identified in [context_use_cases_current_pipeline.md](context_use_cases_current_pipeline.md). Each UC maps one or more responsibilities to a concrete need that the pipeline’s central state management must satisfy.

Legend:
- **MUST**: RADPS must directly support this capability
- **SHOULD**: valuable; can be deferred, narrowed, or implemented differently
- **WON’T (as-is)**: the current mechanism doesn’t carry forward; replace with an alternative

| Pipeline UC | Summary (today) | RADPS relevance | RADPS interpretation / notes |
|---|---|---:|---|
| UC-01 | Load and provide access to observation metadata | **MUST** | Provide a high-performance, random-access **Dataset/Observation Catalog** in context (MSv4-centric) including stable IDs and any necessary mapping equivalents. Replaces `context.observing_run`. |
| UC-02 | Store and provide project-level metadata | **MUST** | Keep run/project metadata as **immutable-after-init** (or versioned) sub-records in the run ledger. Includes project summary, structure, performance parameters, and processing intents. |
| UC-03 | Manage execution paths and output locations | **MUST** | Replace embedded filesystem paths with an **artifact registry** backed by stable IDs and location references. Run identity becomes a first-class `run_id` in the context store. Relocation becomes “artifact locations are references”, not embedded paths. |
| UC-04 | Register and query calibration state | **MUST** | Represent calibration application state as a **typed, versioned sub-record** with **transactional multi-entry updates** (single task may register multiple items atomically). Replaces `context.callibrary`. |
| UC-05 | Accumulate imaging state across multiple steps | **MUST** | Replace ad-hoc imaging attributes with a **schema’d imaging state document** (versioned) or a state machine. Must support concurrent-safe, partition-scoped updates. Fold image libraries into the **artifact registry** as first-class entries with image-specific metadata; keep separate “views” if needed (science/cal/RMS/sub-products). |
| UC-06 | Register and query produced image products | **MUST** | Represent produced image products as typed artifact registry entries (science, calibrator, RMS, sub-products). Provide add/query semantics and lineage links so downstream tasks, reporting, and export can discover images by type and provenance. |
| UC-07 | Track execution progress and stage history | **MUST** | Replace the in-memory `context.results` timeline with a **run ledger**: per-node attempt records with status, timing, QA outcomes, and tracebacks. Stage numbering becomes node execution ordering within the DAG. |
| UC-08 | Propagate task outputs to downstream tasks | **MUST** | Replace `Results.merge_with_context()` with **transactional updates** to a shared ACID store. Replace results-list walking with **named outputs / typed references** (artifact IDs, logical outputs) queryable by key, not by stage index. |
| UC-09 | Support multiple orchestration drivers | **MUST** | Replace PPR/XML/interactive driver-specific entry points with a **planner-produced DAG/plan**. The context must remain the stable state contract regardless of how the run was initiated. Preserve breakpoint/resume semantics. |
| UC-10 | Save and restore a processing session | **MUST** | Replace pickle-based persistence with **DB-backed run ledger + artifact registry**. Checkpoints become explicit context objects rather than whole-object snapshots. Support restart on a different node/cluster. |
| UC-11 | Provide state to parallel workers | **MUST** | Formalize **context snapshot semantics**: workers read a consistent, read-only snapshot of required context state at a defined boundary. Snapshot must be efficiently distributable (no forked pickles as system-of-record). |
| UC-12 | Aggregate results from parallel workers | **MUST** | Formalize **transactional write-back**: worker results are committed via ACID transactions with conflict detection. Aggregation must be safe (no conflicting concurrent writes) and complete before the next sequential step begins. |
| UC-13 | Provide data for report generation | **MUST** | Provide **consistent read-only snapshot views** of context for weblog, QA reports, AQUA XML, and product packaging. Reports are generated from **ledger + registry + annotations** using a stable API; must support re-rendering post-run without access to worker memory. |
| UC-14 | Compute and store quality assessments | **MUST** | Guarantee **read-only snapshot views** of context for QA handlers (consistent with a plan/attempt checkpoint). QA scores become artifacts or typed records in context. QA handlers are parallelization-safe since they are read-only with respect to context. |
| UC-15 | Support interactive inspection and debugging | **MUST** | Preserve deterministic run layouts and machine-checkable failure signals. Extend to distributed runs: per-node logs, structured events, and stable run IDs. Context must be inspectable via a stable API, not by unpickling. |
| UC-16 | Manage telescope-specific state | **WON’T (as-is)** | VLA is not a RADPS target. If ngVLA needs domain-specific state, use a **typed extension mechanism** (schema’d, versioned, per-dataset scoped) via RADPS UC16. VLA-specific fields are not requirements. |
| UC-17 | Package and export pipeline products | **MUST** | Export reads from the artifact registry and run ledger to assemble deliverable packages. Artifact references (not embedded paths) make this possible across filesystems. The export result itself is registered as an artifact. |
| UC-18 | Emit lifecycle notifications | **SHOULD** | Prefer an **event log** (append-only) or patch log as part of context (audit + replay). Even if state tables exist, keep an event stream for provenance and debugging. Could be elevated to primary state mutation channel. |

## Pipeline responsibilities to RADPS context subsystem mapping

The Pipeline analysis identifies 15 context responsibilities. The table below maps each to its RADPS context counterpart.

| # | Pipeline Responsibility | RADPS Context Subsystem | Notes |
|---|---|---|---|
| 1 | Static Observation & Project Data | Run Ledger (immutable-after-init metadata) + Dataset/Observation Catalog | MSv4-centric; stable IDs replace MS-name lookups |
| 2 | Mutable Observation State | Dataset/Observation Catalog (versioned records) | Replaces ad-hoc MS mutation; version-tracked |
| 3 | Path Management | Artifact Registry (location references) | Paths become artifact locations, not embedded strings |
| 4 | Imaging State Management | Schema’d Imaging State sub-record (RADPS UC12) | Typed, versioned, partition-scoped |
| 5 | Calibration State Management | Calibration State sub-record (RADPS UC11) | Transactional, multi-entry, versioned |
| 6 | Image Library Management | Artifact Registry (image-typed entries) | Separate views (science/cal/RMS/sub-product) if needed |
| 7 | Session Persistence | Run Ledger + Artifact Registry + Checkpoints (RADPS UC5/UC6) | DB-backed; no pickle |
| 8 | MPI / Parallel Distribution | Worker Snapshot + Transactional Write-Back (RADPS UC17) | Consistent snapshot reads; ACID write-back |
| 9 | Inter-Task Data Passing | Transactional Updates + Named Outputs (RADPS UC14) | Replaces `merge_with_context` + results-list walking |
| 10 | Stage Tracking & Result Accumulation | Run Ledger: Node Attempt Lifecycle (RADPS UC3) | Per-node attempts with status, timing, QA outcomes |
| 11 | Reporting & Export Support | Read-Only Snapshots (RADPS UC8/UC13) + Artifact Registry | Stable API; no worker memory access required |
| 12 | QA Score Storage | QA records in Run Ledger or typed artifacts | Read-only context snapshot for QA handlers |
| 13 | Debuggability / Inspectability | Read-Only Snapshots (RADPS UC13) + Event Log (RADPS UC15) | Queryable via stable API, not pickle inspection |
| 14 | Telescope-Specific State | Domain Extensions (RADPS UC16) | Typed, versioned, per-dataset scoped |
| 15 | Lifecycle Notifications | Event Log / Patch Log (RADPS UC15) | Append-only; audit + replay |

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
- Consumers (planner, UI, report generators, operators) read from the same consistent store.

### Artifacts: from “fields on context” to explicit artifact registry

- Large products (MS fragments, calibration tables, images, cubes, QA metrics, plots) are **artifacts** with:
  - stable IDs
  - content hashes (when feasible)
  - lineage links (which node produced them)
  - one or more locations (local path, shared FS, object store)

## Proposed RADPS “context contract” (minimum viable)

A workable next-gen context needs a small set of concepts with stable identifiers.

### Identifiers

- `run_id`: globally unique ID for a pipeline run (immutable)
- `plan_id`: ID for the planned DAG/graph version used by the run
- `node_id`: stable ID for a DAG node (task instance)
- `attempt_id`: unique ID per node execution attempt (retries, reschedules)
- `artifact_id`: stable ID for a produced artifact

### Run ledger (ACID store)

Minimum records/relations (conceptually):

- **Run metadata**: domain (WSU/ngVLA), operator/mode, timestamps, priority, tenant/project, policy bundle
- **Inputs**: datasets + versions, conversion outputs (e.g., MSv4 import products), parameterization
- **Dataset/Observation Catalog**: MSv4 inventory + per-partition metadata (field/spw/scan) and ID mapping needed by tasks/QA/rendering
- **Calibration State**: versioned calibration application state (the callibrary analogue)
- **Imaging State**: schema’d imaging configuration/scratch-pad state (partition-scoped where possible)
- **Plan**: DAG structure, node definitions, resource intents (CPU/mem/IO locality), partitioning keys
- **Node execution state**:
  - status: `PENDING | RUNNING | SUCCEEDED | FAILED | SKIPPED | CANCELED`
  - timing, resources used, worker identity, error summaries
  - explicit checkpoints / “safe restart points”
- **Events** (optional but strongly useful): append-only timeline for audit and debugging

Implementation constraint: updates must be **transactional** and safe under concurrent writers.

### Artifact registry

For each `artifact_id`:

- `type` (MS partition, cal table, image cube, plot bundle, weblog, manifest, etc.)
- `producer` (`node_id`, `attempt_id`)
- `lineage` (input artifact IDs)
- `locations[]` (local path(s), shared FS path(s), object store URI(s))
- `lifecycle` (retention policy, garbage-collect eligibility)

### Checkpoint/resume semantics

- Checkpoints are explicit objects in the ledger (not implicit “whatever was pickled”).
- Resume must support:
  - re-run from last successful checkpoint
  - targeted re-runs of subgraphs (e.g., one field/spw)
  - safe invalidation of downstream artifacts when upstream changes

### Manual intervention (operator workflow)

A minimal supported set:

- Pause a run (stop scheduling new nodes)
- Inspect run state + artifacts (without reading worker-local memory)
- Re-plan or re-run a subset with modified parameters (tracked as a new `plan_id` or policy revision)

## Additional RADPS context use cases (context-scoped)

Concrete RADPS context use cases are drafted in [docs/context_design_use_cases.md](context_design_use_cases.md). These cover:

- run + plan lifecycle (RADPS UC1–UC3)
- artifact + checkpoint lifecycle (RADPS UC4–UC9)
- “internal pipeline interactions” equivalents: catalog queries, calibration/imaging state, snapshots, named outputs (RADPS UC10–UC14)
- event/patch logging, domain extensions, and worker snapshot/write-back semantics (RADPS UC15+)

## “Missing today” capabilities (GAPs) and RADPS context implications

The Pipeline note also enumerates capabilities the current design cannot handle (GAP-01 through GAP-07: concurrency, cloud without shared FS, multi-language access, streaming/incremental, reproducibility guarantees, targeted reruns, external integrations). Not all of these are “context” features, but **every one requires context changes**.

- **GAP-01 Concurrent / overlapping execution**: requires snapshot isolation + transactional merges; context must support partition-scoped writes and conflict detection.
- **GAP-02 Distributed execution without shared filesystem**: requires artifact references decoupled from POSIX paths; context becomes the system-of-record.
- **GAP-03 Multi-language access**: requires a language-neutral schema + API for context state and artifact queries.
- **GAP-04 Streaming / incremental processing**: requires versioned dataset registration and versioned results/artifacts.
- **GAP-05 Provenance and reproducibility guarantees**: requires immutable per-attempt records and input hashing/lineage.
- **GAP-06 Partial re-execution / targeted rerun**: requires explicit dependency tracking and invalidation semantics (context-level).
- **GAP-07 External system integration**: requires stable identifiers, event subscriptions, and exportable summaries/manifests from context.

## Compatibility and determinism policy (pragmatic)

- **Backward compatibility**: support resuming runs within a defined release window (schema + node contract). Outside the window: re-plan and re-run may be required.
- **Determinism**: guarantee “same inputs + versions + resources ⇒ same results within numerical precision”; record enough provenance to explain deviations.

## Open questions (to finalize the contract)

- What is the authoritative ACID store choice (SQLite for dev vs Postgres for ops), and what are the required transactional boundaries?
- Do we want **event-sourcing** (append-only events + derived views) or **state tables** (current-state rows), or a hybrid?
- What artifact location model is required in early phases (local-only, shared FS, object store), and what retention policy applies to intermediates?
- How is multi-tenancy enforced (namespacing, quotas, access control) and where do authz decisions live?
- What is the minimum manifest schema required at the end of a run (and who consumes it)?
