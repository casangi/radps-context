# RADPS context: mapping current Pipeline context use cases to next-gen requirements

This note maps the **current Pipeline context** use cases documented in [docs/current_pipeline_use_cases.md](current_pipeline_use_cases.md) to the **RADPS** (WSU + ngVLA) execution model and derives a concrete “context contract” suitable for distributed, ACID, restartable processing.

See also:

- [docs/context_design_use_cases.md](context_design_use_cases.md) (draft RADPS context use cases)
- [docs/current_pipeline_use_cases.md](current_pipeline_use_cases.md) (source Pipeline use cases)
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

## Use-case relevance mapping (UC1–UC18)

The Pipeline use cases naturally fall into two groups:

- **UC1–UC7**: how the pipeline is launched/operated (entry points, products, save/resume)
- **UC8–UC18**: within-execution context interactions (catalog queries, shared state, reporting traversals, snapshot semantics)

Legend:
- **MUST**: RADPS must directly support this capability
- **SHOULD**: valuable; can be deferred, narrowed, or implemented differently
- **WON’T (as-is)**: the current mechanism doesn’t carry forward; replace with an alternative

### Entry-point use cases (UC1–UC7)

| Current UC | Summary (today) | RADPS relevance | RADPS interpretation / notes |
|---|---|---:|---|
| UC1 | Operate via PPR (ALMA) | **MUST** | Replace PPR XML “command lists” with a **planner-produced DAG/plan**. Preserve operational “run from recipe + inputs” semantics, breakpoint/resume behavior, and standardized outputs (weblog/manifest). |
| UC2 | Operate via PPR (VLA) | **WON’T (as-is)** | RADPS does not target VLA processing. If a legacy ingest format is needed for ngVLA, treat it as an **input adapter** compiled into a planner-produced DAG/plan (not a VLA-specific execution path). |
| UC3 | Interactive task-by-task execution | **SHOULD** | Preserve as a **supported operator/dev mode**: inspect state, run a subset of graph nodes, re-run with modified parameters. Must be safe under ACID semantics (no “hidden globals”). |
| UC4 | Developer XML “procedure” execution | **WON’T (as-is)** | Keep the intent (“execute a recipe/procedure”), but converge on the planner’s plan representation. XML may remain as a legacy authoring format, compiled to the DAG. |
| UC5 | Save/resume/relocate context | **MUST** | Replace pickle with **DB-backed run ledger + artifact registry**. Relocation becomes “artifact locations are references”, not embedded paths. Support restart on a different node/cluster. |
| UC6 | Weblog generation as product | **MUST** | Preserve weblog/QA reporting, but generation becomes a **reporting pipeline** over the run ledger + artifacts. Must be reproducible and re-renderable post-run. |
| UC7 | Testing/regression harness | **MUST** | Preserve deterministic run layouts and machine-checkable failure signals. Extend to distributed runs: per-node logs, structured events, and stable run IDs. |

### Within-execution context interactions (UC8–UC18)

UC8–UC18 capture the “inside execution” context interactions (domain metadata queries, calibration library updates, image registries, rendering traversals, snapshot semantics, etc.). These map cleanly to RADPS **context subsystem responsibilities**.

| Current internal UC | What it means in the current Pipeline | RADPS relevance | RADPS context interpretation / requirements |
|---|---|---:|---|
| UC8 Domain metadata querying | Tasks and renderers query `context.observing_run` for MS/SPW/scan metadata and ID mappings | **MUST** | Provide a high-performance, random-access **Dataset/Observation Catalog** in context (MSv4-centric) including stable IDs and virtual↔real mapping equivalents (or an explicit replacement mapping model). |
| UC9 Calibration library management | Append-mostly, ordered cal-application state (`context.callibrary`) used across stages | **MUST** | Represent calibration application state as a **typed, versioned sub-record** with **transactional multi-entry updates** (single task may register multiple items atomically). |
| UC10 Image library management | `context.*imlist` behaves as an image/artifact registry | **MUST** | Fold into the Artifact Registry as first-class entries with image-specific metadata; keep separate “views” if needed (science/cal/RMS/sub-products). |
| UC11 Imaging scratch-pad state | Ad-hoc imaging attributes used as cross-stage shared state | **MUST** | Replace ad-hoc attributes with a **schema’d imaging state document** (versioned) or a small state machine. Must support concurrent-safe updates (partitioned by field/spw/scan where applicable). |
| UC12 VLA-specific state bag | Untyped `context.evla` dict-of-dicts used by many tasks | **WON’T (as-is)** | VLA is not a RADPS target. If ngVLA needs domain-specific state, use a **typed extension mechanism** (schema’d, versioned, per-dataset scoped), but VLA-specific fields are not requirements. |
| UC13 QA scoring | QA handlers read context + result and add scores | **MUST** | Guarantee **read-only snapshot views** of context for QA/reporting (consistent with a plan/attempt checkpoint). Scores become artifacts or typed records in context. |
| UC14 Weblog rendering | Traverses full results timeline and metadata; read-only | **MUST** | Reports are generated from ledger + registry + annotations using a stable API; support re-rendering without access to worker memory. |
| UC15 MPI distribution snapshot | Broadcast a context snapshot to workers and merge results back | **MUST** | Formalize **context snapshot semantics** vs **results/patch write-back**. In Dask terms: workers read a consistent snapshot; write back via transactional updates. |
| UC16 Cross-stage forward-pass via results list | Tasks “grep” `context.results` to find upstream outputs | **MUST** | Replace list-walking with **named outputs / typed references** (e.g., artifact IDs, logical outputs) that are queryable by key, not by stage index. |
| UC17 Project metadata injection | One-time run metadata written early, read often | **MUST** | Keep run metadata as immutable-after-init (or versioned) sub-records (project summary/structure/performance parameters). |
| UC18 Event bus lifecycle events | Events exist but are not the main state transition mechanism | **SHOULD** | Prefer an event log (append-only) or patch log as part of context (audit + replay). Even if state tables exist, keep an event stream for provenance and debugging. |

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

## “Missing today” capabilities (FUCs) and RADPS context implications

The Pipeline note also enumerates future/missing capabilities (concurrency, cloud without shared FS, multi-language access, streaming/incremental, reproducibility guarantees, access control, targeted reruns, external integrations). Not all of these are “context” features, but **every one requires context changes**.

- **Concurrent / overlapping execution**: requires snapshot isolation + transactional merges; context must support partition-scoped writes and conflict detection.
- **Distributed execution without shared filesystem**: requires artifact references decoupled from POSIX paths; context becomes the system-of-record.
- **Multi-language access**: requires a language-neutral schema + API for context state and artifact queries.
- **Streaming / incremental processing**: requires versioned dataset registration and versioned results/artifacts.
- **Provenance and reproducibility guarantees**: requires immutable per-attempt records and input hashing/lineage.
- **Fine-grained access control**: requires authn/authz metadata and audit logs on all context mutations.
- **Partial re-execution / targeted rerun**: requires explicit dependency tracking and invalidation semantics (context-level).
- **External system integration**: requires stable identifiers, event subscriptions, and exportable summaries/manifests from context.

## Compatibility and determinism policy (pragmatic)

- **Backward compatibility**: support resuming runs within a defined release window (schema + node contract). Outside the window: re-plan and re-run may be required.
- **Determinism**: guarantee “same inputs + versions + resources ⇒ same results within numerical precision”; record enough provenance to explain deviations.

## Open questions (to finalize the contract)

- What is the authoritative ACID store choice (SQLite for dev vs Postgres for ops), and what are the required transactional boundaries?
- Do we want **event-sourcing** (append-only events + derived views) or **state tables** (current-state rows), or a hybrid?
- What artifact location model is required in early phases (local-only, shared FS, object store), and what retention policy applies to intermediates?
- How is multi-tenancy enforced (namespacing, quotas, access control) and where do authz decisions live?
- What is the minimum manifest schema required at the end of a run (and who consumes it)?
