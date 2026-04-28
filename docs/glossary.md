# Glossary (RADPS context docs)

This glossary defines common distributed-systems, data-management, and context-contract terms used across the RADPS context documentation.

- **ACID**: Database transaction properties.
  - **Atomicity**: a multi-step update is “all or nothing”.
  - **Consistency**: committed updates preserve declared invariants (schema constraints, foreign keys, etc.).
  - **Isolation**: concurrent updates don’t observe partial/interleaved state; reads see a well-defined view.
  - **Durability**: once committed, changes persist across crashes/restarts.

- **Artifact**: A durable data product produced by the pipeline (for example MS partitions, calibration tables, images, reports, manifests). Artifacts are tracked by stable IDs, typed metadata, lineage, version/supersedes links, and one or more storage-agnostic locations.

- **Artifact registry**: The subsystem (often a table/service) that records artifact IDs, types, producers, locations, hashes (when feasible), lineage, version relationships, and lifecycle state.

- **Archival product**: A previously generated durable product imported from outside the active run so the context can be initialized or resumed from a valid mid-pipeline state.

- **Attempt**: A single execution try of a planned unit of work (a node). Retries create new attempts.

- **Checkpoint**: A durable “safe restart point” that references a closed, consistent set of state + artifacts.

- **Conflict detection**: The mechanism that identifies incompatible concurrent writes (for example, overlapping partition updates based on the same stale snapshot) and rejects or serializes them.

- **DAG (Directed Acyclic Graph)**: A graph of computation nodes with dependency edges, with no cycles. Used to represent planned work and explicit dependencies.

- **Deterministic execution / determinism policy**: The expectation that the same inputs, software versions, and resource envelope produce the same results within numerical precision. Any deviations must be explainable from recorded provenance.

- **Event log / patch log**: An append-only timeline of significant events and/or state changes. Used for audit, debugging, integration feeds, and (sometimes) replay.

- **Event sourcing**: A design where the event log is the primary source-of-truth, and current state is derived from replaying events (often with materialized views).

- **Event subscription / webhook**: A push-style integration mechanism where external consumers receive selected lifecycle events or summary notifications. Delivery must be retryable and idempotent.

- **Execution-control tag**: Persisted metadata on a dataset, stage, or run that the workflow orchestration layer interprets to alter execution behavior (for example pause, skip, or divert processing).

- **Idempotency**: A property where repeating an operation (due to retries/timeouts) does not create additional side effects beyond the first successful application.

- **Initialization from intermediate state**: Constructing a valid run context from pre-existing archival products or prior checkpoints so earlier stages can be skipped and processing can continue from a mid-pipeline boundary.

- **Invariant**: A condition that must remain true while the system is operating, not just at the end of a single operation.

- **Isolation level / snapshot**: How a system defines what a read can see while writes are happening.
  - **Read-only snapshot**: a consistent view of data at a defined boundary (for example, “as of checkpoint X”).
  - **Snapshot isolation**: a common isolation model where each transaction reads from a snapshot and writes commit if no conflicting writes occurred.

- **Language-neutral API**: A stable, typed contract (for example REST, gRPC, or another service interface) that allows clients in any supported language to query/update context state without depending on storage layout or Python objects.

- **Ledger (run ledger)**: The durable run-scoped record of plans, nodes, attempts, checkpoints, subscriptions, and other state transitions.

- **Lineage**: Links that explain how an artifact/result was produced (inputs consumed, node/attempt that produced it, and upstream artifacts).

- **Matching semantics**: The rules used to decide when metadata elements across datasets correspond to one another. Common modes include **exact** matching and **overlap/partial** matching.

- **MeasurementSet v4 (MSv4)**: The next-generation MeasurementSet representation targeted for RADPS. The RADPS observation catalog is assumed to be MSv4-centric even when inputs begin as ASDM or another archive format.

- **Middleware/API layer**: The service boundary that exposes typed context operations, schema/version negotiation, authorization, and error handling to internal and external clients.

- **Partition / scope**: A key that identifies a subset of work/state (for example by dataset, field, SPW, scan, or data type). Partition-scoped updates reduce contention and enable concurrency.

- **Pipeline responsibility**: One of the broad context-management capabilities extracted from the current-pipeline analysis. Multiple Pipeline use cases may map to the same responsibility.

- **Postcondition**: A condition that must be true after a use case or operation completes successfully.

- **Precondition**: A condition that must hold before a use case or operation can begin.

- **Provenance**: The record of inputs, parameters, software versions, execution environment, hardware/scheduler details, and lineage needed to explain and reproduce results.

- **Schema’d / typed record**: A structured record with explicit fields and versions (as opposed to untyped “bags” like free-form dictionaries).

- **Stable identifier**: A durable ID whose meaning does not depend on process memory or local path layout (for example `run_id`, `dataset_id`, or `artifact_id`).

- **Streaming / incremental processing**: The ability to register new data into an active run and produce versioned downstream state/results without restarting the pipeline from scratch.

- **Transaction boundary**: The scope of data changes that must commit atomically.

- **Tombstone / tombstoning**: Recording that an artifact/state was intentionally removed or invalidated (often without immediately deleting underlying storage), preserving audit and preventing dangling references.

- **Workflow orchestration layer / system**: The component that plans, schedules, and coordinates execution of pipeline tasks. In RADPS it owns task-graph progression, retries, and enforcement of execution-control decisions, while `radps-context` persists the durable state those decisions depend on.

- **xradio**: The xarray-based data access layer expected to expose MSv4-centric observation metadata and storage structures to RADPS. `radps-context` is expected to coordinate with, rather than duplicate, these self-describing dataset representations where possible.

## Legacy Pipeline terms (used in current Pipeline context docs)

- **ASDM**: ALMA Science Data Model; an archive/distribution format that is typically imported/converted into a MeasurementSet for pipeline processing.

- **CASA**: Common Astronomy Software Applications; the legacy pipeline runs “inside CASA” (Python + CASA tools), and many tasks assume CASA data models and IDs.

- **Context (legacy Pipeline)**: A long-lived, in-process Python object that acts as session state + shared mutable state + persistence unit (via pickle). In the legacy design it also carries convenience methods and domain objects (for example `observing_run`).

- **ContextData / Context services**: A separation where durable/serializable state (ContextData) is distinct from runtime-only helpers (services like caches, tool handles, heuristic engines).

- **Event bus (legacy Pipeline)**: An in-process publish/subscribe mechanism for lifecycle markers (task start/complete, result acceptance). It exists today but is not the primary state mutation channel.

- **Imaging “scratch pad” state**: A collection of ad-hoc context attributes used to coordinate the imaging sub-pipeline across multiple stages (for example cleaning lists, beam summaries, thresholds). This is intentionally flexible but fragile without schema/versioning.

- **`observing_run` (legacy Pipeline)**: The main domain-model handle hanging off context that provides MS/scan/field/SPW metadata queries and ID mapping utilities; effectively the “dataset/observation catalog” of the legacy design.

- **`callibrary` (legacy Pipeline)**: The calibration application registry hanging off context. It is append-oriented and ordered, supports predicate-based queries, and acts as the primary cross-stage communication mechanism for calibration workflows.

- **MeasurementSet (MS)**: A radio astronomy dataset format used by CASA and the pipeline. Many legacy context queries are “MS-centric” (look up by MS name, filter by MS type, map SPW IDs).

- **Orchestration driver**: A front-end that creates and drives execution against the pipeline context. The legacy pipeline supports multiple orchestration drivers: PPR command lists (ALMA/VLA automated processing), XML procedures (production recipes), and interactive task calls (developer/operator sessions). The context must remain a stable state contract across all drivers (Pipeline UC-11).

- **PPR**: Pipeline Processing Request; an XML bundle used to drive automated processing (inputs + metadata + an ordered command list).

- **Pickle**: Python’s built-in object serialization format used by the legacy pipeline for persisting context and proxying results. Convenient for short-lived resume/debug, but fragile across version changes and not designed for multi-writer concurrency.

- **Results**: A per-task return object that contains outputs (and often QA, tracebacks, and metadata). In the legacy pipeline, results are “accepted” into context via `Results.accept(context)` / `merge_with_context()`.

- **ResultsProxy**: A lightweight on-disk proxy for a Results object (pickled per stage) used to keep the in-memory context smaller; unpickled lazily during weblog rendering and other traversals.

- **SPW (Spectral Window)**: A spectral subdivision of the data (frequency window). The legacy pipeline commonly uses a “virtual SPW” abstraction and maps virtual↔real (CASA-native) SPW IDs per MS.

- **Virtual SPW mapping**: A legacy abstraction where pipeline-defined “virtual” SPW IDs are mapped to the “real” (CASA-native) SPW IDs for a given MS. This enables stable cross-MS logic, but requires explicit mapping tables/functions.

- **Stage**: A sequential, top-level step in a legacy Pipeline run, usually associated with a task execution and a stage number used for ordering and reporting.

- **Task**: A registered unit of work (often class-based) invoked by the CLI wrappers or recipe execution; typically produces a Results object.

- **Timetracker**: A timing/telemetry record (often stored as a `*.timetracker` SQLite DB) used to detect abnormal exits and summarize stage timings.

- **Weblog**: The human-readable HTML report product generated from context + results (and often re-rendered stage-by-stage during execution).

- **`pipelineqa` / QA plugins**: The legacy QA framework where plugins read context + a task’s results to emit QA scores/diagnostics. QA handlers are typically read-only with respect to context state.
