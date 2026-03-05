# RADPS context: mapping current Pipeline context use cases to next-gen requirements

This note maps the **current Pipeline context** use cases documented in [docs/current_pipeline_use_cases.md](current_pipeline_use_cases.md) to the **RADPS** (WSU + ngVLA) execution model and derives a concrete “context contract” suitable for distributed, ACID, restartable processing.

## Assumptions (from RADPS discussions)

- **Domains**: ALMA WSU and ngVLA.
- **Data model**: archive inputs (e.g., ASDM) convert to **MeasurementSet v4**; Python ecosystem emphasis.
- **Scale**: multi-TB artifacts, very large spectral cubes (up to ~1.2M channels), significantly increased SPWs.
- **Execution**: planner service generates a **DAG at runtime**; execution is **distributed (Dask)** with concurrent workers.
- **State semantics**: shared state must have **ACID semantics** (multi-writer correctness is required).
- **Operations**: intermediates must remain **locally available** (for pause/inspect/manual intervention); resumability and partial reruns are required.
- **Platform**: **Slurm now**, **Kubernetes later**.
- **Provenance**: must emit machine-readable manifests/audit trails; determinism is “within numerical precision” and depends on identical inputs/versions/hardware/resources.

## Use-case relevance mapping (UC1–UC7)

Legend:
- **MUST**: RADPS must directly support this capability
- **SHOULD**: valuable; can be deferred, narrowed, or implemented differently
- **WON’T (as-is)**: the current mechanism doesn’t carry forward; replace with an alternative

| Current UC | Summary (today) | RADPS relevance | RADPS interpretation / notes |
|---|---|---:|---|
| UC1 | Operate via PPR (ALMA) | **MUST** | Replace PPR XML “command lists” with a **planner-produced DAG/plan**. Preserve operational “run from recipe + inputs” semantics, breakpoint/resume behavior, and standardized outputs (weblog/manifest). |
| UC2 | Operate via PPR (VLA) | **MUST** | Same as UC1; domain-specific plan rules remain, but encoded as **planner logic + policy**, not ad-hoc in an execution loop. |
| UC3 | Interactive task-by-task execution | **SHOULD** | Preserve as a **supported operator/dev mode**: inspect state, run a subset of graph nodes, re-run with modified parameters. Must be safe under ACID semantics (no “hidden globals”). |
| UC4 | Developer XML “procedure” execution | **WON’T (as-is)** | Keep the intent (“execute a recipe/procedure”), but converge on the planner’s plan representation. XML may remain as a legacy authoring format, compiled to the DAG. |
| UC5 | Save/resume/relocate context | **MUST** | Replace pickle with **DB-backed run ledger + artifact registry**. Relocation becomes “artifact locations are references”, not embedded paths. Support restart on a different node/cluster. |
| UC6 | Weblog generation as product | **MUST** | Preserve weblog/QA reporting, but generation becomes a **reporting pipeline** over the run ledger + artifacts. Must be reproducible and re-renderable post-run. |
| UC7 | Testing/regression harness | **MUST** | Preserve deterministic run layouts and machine-checkable failure signals. Extend to distributed runs: per-node logs, structured events, and stable run IDs. |

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

## Compatibility and determinism policy (pragmatic)

- **Backward compatibility**: support resuming runs within a defined release window (schema + node contract). Outside the window: re-plan and re-run may be required.
- **Determinism**: guarantee “same inputs + versions + resources ⇒ same results within numerical precision”; record enough provenance to explain deviations.

## Open questions (to finalize the contract)

- What is the authoritative ACID store choice (SQLite for dev vs Postgres for ops), and what are the required transactional boundaries?
- Do we want **event-sourcing** (append-only events + derived views) or **state tables** (current-state rows), or a hybrid?
- What artifact location model is required in early phases (local-only, shared FS, object store), and what retention policy applies to intermediates?
- How is multi-tenancy enforced (namespacing, quotas, access control) and where do authz decisions live?
- What is the minimum manifest schema required at the end of a run (and who consumes it)?
