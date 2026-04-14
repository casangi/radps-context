# Missing capabilities (GAPs) and implications for RADPS context design

This document records capabilities the current pipeline context design cannot yet support. Not every item below is strictly a "context" feature, but each implies changes to context responsibilities, schema, or interfaces. The gaps are enumerated as GAP-01 through GAP-08. A separate, more exhaustive gap analysis mapping these use cases to RADPS requirements is recommended.

## High-level gap list

- GAP-01: Asynchronous / overlapping execution — requires snapshot isolation, transactional merges, partition-scoped writes, and conflict detection.
- GAP-02: Distributed execution without a shared filesystem — requires artifact references decoupled from POSIX paths and a context that serves as the system-of-record.
- GAP-03: Provenance and reproducibility — requires immutable per-attempt records, input hashing, and lineage capture.
- GAP-04: Partial re-execution / targeted rerun — requires explicit dependency tracking and invalidation semantics at the context level.
- GAP-05: External system integration — requires stable identifiers, event subscriptions/webhooks, and exportable summaries/manifests.
- GAP-06: Programming-language / client-framework access — requires a language-neutral contract and stable middleware/API layer with extensible value types so clients in any language can access context state without coupling to the storage representation.
- GAP-07: Streaming / incremental processing — requires versioned dataset registration and versioned results/artifacts.
- GAP-08: Cross-MS matching and heterogeneous dataset coordination — requires flexible SPW matching semantics (exact and partial/overlap) and data-type tracking across measurement sets, rather than the current single-master-MS assumption.

## Detailed use cases

### GAP-01 — Asynchronous Execution of Independent Work

| | |
|-------|---------|
| **Actor(s)** | Workflow orchestration layer, parallel task scheduler |
| **Summary** | The context must support asynchronous execution at multiple granularities (stage-level and within-stage parallelism) while preventing inconsistent processing state. This emphasizes that tasks may proceed independently without waiting for others to complete when task dependencies allow. This differs from the current parallel-worker pattern, which waits for all work to finish before proceeding. |
| **Invariant** | Independent tasks may run asynchronously but must not produce conflicting state. |
| **Postconditions** | Results from asynchronously executed tasks are fully and consistently incorporated into processing state before any dependent work begins. |
| **RADPS requirements** | CSS9017, CSS9063 |

### GAP-02 — Distributed execution without a shared filesystem

| | |
|-------|---------|
| **Actor(s)** | Workflow orchestration layer, distributed workers |
| **Summary** | Execution must be possible across nodes that do not share a filesystem. Artifacts, datasets, and processing state must be addressable and accessible without relying on local POSIX paths. |
| **Postconditions** | Processing completes across distributed nodes with context-hosted references providing the necessary artifact access. |
| **RADPS requirements** | CSS9002, CSS9030 |

### GAP-03 — Provenance and reproducibility

| | |
|-------|---------|
| **Actor(s)** | Pipeline operator, auditor, reproducibility tooling |
| **Summary** | The context must record sufficient provenance — software versions, exact input identities/hashes, task parameters, and per-stage state — to enable precise reproduction and audit of past runs. |
| **Postconditions** | Any past processing step can be reproduced or audited using the recorded provenance chain. |
| **RADPS requirements** | ALMA-TR103, ALMA-TR104, ALMA-TR105 |

Addendum: provenance should also capture hardware and execution-environment details (CPU architecture, node/cluster specification, kernel and MPI versions, and workload-manager/scheduler configuration and relevant scheduler limits), since non-deterministic behaviour has been observed even when software versions and inputs are identical across different hardware or scheduler setups.

### GAP-04 — Partial re-execution / targeted stage re-run

| | |
|-------|---------|
| **Actor(s)** | Pipeline operator, developer, workflow engine |
| **Summary** | The context must support selectively re-running one or more mid-pipeline stages with new parameters while preserving unaffected stages. Downstream stages that depend on changed outputs must be invalidated or recomputed. |
| **Postconditions** | Processing state reflects the re-run outcomes; affected downstream stages are invalidated or updated; unaffected stages remain intact. |
| **RADPS requirements** | CSS9038 |

### GAP-05 — External system integration (archive, scheduling, QA dashboards)

| | |
|-------|---------|
| **Actor(s)** | QA dashboards, monitoring tools, archive ingest systems, schedulers |
| **Summary** | External systems need timely access to processing state (current stage, processing time, QA results, lifecycle events) without waiting for offline product files. The context should expose the necessary state via queryable interfaces or event feeds. |
| **Invariant** | External consumers can access the processing state they require while it remains current. |
| **Postconditions** | External systems can track processing progress and lifecycle transitions in near real time. |
| **RADPS requirements** | CSS9046, CSS9047, CSS9048, CSS9049, CSS9050, CSS9056 |

### GAP-06 — Programming-Language / Client-Framework Access to Context

| | |
|-------|---------|
| **Actor(s)** | Non-Python clients (C++, Julia, JavaScript dashboards), external tools |
| **Summary** | Expose context state through a programming-language-neutral contract and a stable middleware/API layer (e.g., REST/gRPC over a typed schema such as Protocol Buffers, JSON-Schema, or Arrow) so local language bindings do not need to know how the context is physically stored. The contract should provide a small set of standard extensible value types (scalars, lists, maps/dictionaries, records/structs) alongside typed core records so developers can add new items quickly without redefining the storage model for each client. |
| **Postcondition** | Clients in any supported programming language can query context state and artifacts through a stable typed API without coupling themselves to the underlying storage representation. |
| **RADPS requirements** |  |

Addendum: the reference API implementation should prioritise mission-critical data-processing needs (metadata management, heuristics, transactional/production requirements) so that language-agnostic access covers the most important use cases first. Requirements for auxiliary functionality (QA, statistics, weblog rendering, dashboards) may be looser and might be better served by separate services or higher-level APIs. If the project elects to separate a small, well-defined `CoreContext` (processing-critical) from broader, softer features, the term "context" may be reserved for the core contract and the auxiliary pieces treated as complementary services.

### GAP-07 — Streaming / incremental processing

| | |
|-------|---------|
| **Actor(s)** | Data ingest systems, workflow engine, incremental processing tasks |
| **Summary** | Support incremental dataset registration (adding new scans or execution blocks to a live session), incremental detection and processing of new data, and versioned results so re-runs produce new versions rather than overwriting. |
| **Postconditions** | New data may be incorporated into an active session and processed incrementally without restarting the pipeline from scratch. |
| **RADPS requirements** |  |

### GAP-08 — Cross-MS matching and heterogeneous dataset coordination

| | |
|-------|---------|
| **Actor(s)** | Data import tasks, calibration tasks, imaging tasks, heuristics |
| **Summary** | The current design provides limited support for heterogeneous multi-MS datasets through virtual SPW translation and per-MS data-column tracking, but many workflows still rely on a single reference-MS or master-MS model and do not expose general cross-MS matching semantics. The context must instead support heterogeneous multi-MS scenarios by providing: (1) cross-MS SPW matching with distinct semantics for exact matching (required by calibration tasks) and partial/overlap matching (required for imaging tasks that can combine overlapping spectral windows); and (2) data-type and column tracking across multiple MSes without assuming a shared layout. Because the current virtual-SPW translation mechanism is tightly coupled to the single-master-MS assumption, a fresh design is preferable to extending it. |
| **Invariant** | SPW identity and data-column state are queryable across all registered datasets, regardless of whether those datasets share native numbering or column layout. |
| **Postconditions** | Calibration and imaging tasks can look up applicable SPWs and data columns across an arbitrary collection of heterogeneous MSes using the appropriate matching semantics for their use. |
| **RADPS requirements** |  |

Addendum: real-world heterogeneous datasets frequently lack fully structured or consistent metadata linking related SPWs across different MOUS/EBs. The system should therefore support a flexible, high-level metadata management layer (for example, permissive flat labels and augmentations rather than rigid relational mapping tables) and explicit hooks for heuristic or user-supplied mapping rules (frequency/channel overlap heuristics, manual SPW mappings, etc.). When heuristics or user interventions are required, the context should record the override, the rationales or decision parameters used, and any associated model/version identifiers (including ML model versions) so decisions remain auditable and reproducible. Storing overrides in a structured form (for example, annotated decision trees or named mapping artifacts) will make it possible to query which metadata gaps commonly trigger manual fixes and to retrain or improve heuristics accordingly.
