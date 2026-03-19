# Pipeline Context Use Cases

## Overview

The pipeline `Context` class (`pipeline.infrastructure.launcher.Context`) is the central mutable state object used for an entire pipeline execution. It is simultaneously a session state container, a domain metadata container, a cross-stage communication channel, and a persistence unit.

This document catalogues the current use cases of the pipeline Context as determined by examination of the codebase. The goal is to support design and development of a prototype of a system which serves a similar role to that of the context for RADPS. It also includes additional use cases and gap scenarios identified by examining the RADPS documentation and requirements, to ensure future extensibility and alignment with next-generation needs.

See also: [Supplementary analysis and design recommendations](context_current_pipeline_appendix.md)

---

## 1. Use Cases

Each use case describes a need that the pipeline's central state management must satisfy. They are written to be implementation-neutral in their core description. For pipeline-specific implementation details per use case, see [Implementation Notes by Use Case](context_current_pipeline_appendix.md#implementation-notes-by-use-case) in the appendix.

In the tables below, **Actor(s)** identifies the human or system role that directly creates, updates, consumes, or inspects the context state described by the use case. Actors are role categories, not specific task names or implementations.

---

### UC-01 — Load and Provide Access to Observation Metadata

| Field | Content |
|-------|---------|
| **Actor(s)** | Data import task, any downstream task, heuristics, renderers, QA handlers |
| **Summary** | The system must load observation metadata (datasets, spectral windows, fields, antennas, scans, time ranges) and make it queryable by all subsequent processing steps. It must also provide a unified identifier scheme when multiple datasets use different native numbering. |
| **Postconditions** | All registered datasets remain queryable for the lifetime of the session without repeating the import process. |

---

### UC-02 — Store and Provide Project-Level Metadata

| Field | Content |
|-------|---------|
| **Actor(s)** | Initialization, any task, report generators |
| **Summary** | The system must store project-level metadata (proposal code, PI, telescope, desired sensitivities, processing recipe) and make it available to tasks for decision-making and to report generators for labelling outputs. |
| **Postconditions** | Project metadata is available for the lifetime of the processing session. |

---

### UC-03 — Manage Execution Paths and Output Locations

| Field | Content |
|-------|---------|
| **Actor(s)** | Initialization, any task, report generators, export code |
| **Summary** | The system must centrally define and provide working directories, report directories, product directories, and logical filenames for logs, scripts, and reports. Tasks resolve file paths through these centrally managed locations. On session restore, paths must be overridable to adapt to a new environment. |
| **Postconditions** | All tasks share a consistent set of paths for inputs and outputs. |

---

### UC-04 — Register and Query Calibration State

| Field | Content |
|-------|---------|
| **Actor(s)** | Calibration tasks |
| **Summary** | The system must allow calibration tasks to register solutions (indexed by data selection: field, spectral window, antenna, intent, time interval), and allow downstream tasks to query for all calibrations applicable to a given data selection. It must distinguish between calibrations pending application and those already applied. Registration must support transactional multi-entry updates — tasks often register multiple calibrations atomically within a single result acceptance. |
| **Postconditions** | Calibration state is queryable and correctly scoped to data selections. |

---

### UC-05 — Accumulate Imaging State Across Multiple Steps

| Field | Content |
|-------|---------|
| **Actor(s)** | Imaging tasks, downstream heuristics, and export tasks |
| **Summary** | The system must allow imaging state — target lists, imaging parameters, masks, thresholds, and sensitivity estimates — to be computed by one step and read or refined by later steps. Multiple steps may contribute to a progressively refined imaging configuration. |
| **Postconditions** | The accumulated imaging state reflects contributions from all completed imaging-related steps and is available to later imaging steps. |

---

### UC-06 — Register and Query Produced Image Products

| Field | Content |
|-------|---------|
| **Actor(s)** | Imaging tasks, export tasks, report generators |
| **Summary** | The system must maintain typed registries of produced image products with add/query semantics. Later tasks must be able to discover previously produced science, calibrator, RMS, and sub-product images through these registries. |
| **Postconditions** | Produced image products are registered by type and remain queryable for downstream processing, reporting, and export. |

---

### UC-07 — Track Execution Progress and Stage History

| Field | Content |
|-------|---------|
| **Actor(s)** | Workflow orchestration layer, tasks, report generators, human operators |
| **Summary** | The system must track which processing step is currently executing and maintain a stable, ordered history of completed steps and their outcomes. This history must support reporting, script generation, and resumption after interruption. Per-stage tracebacks and timings must be preserved, and stage identity and ordering must remain coherent across resumes. |
| **Postconditions** | The full execution history is retrievable in order; each recorded step retains its stage identity, outcome, timing, traceback information, and the arguments or effective parameters used to invoke it. |

---

### UC-08 — Propagate Task Outputs to Downstream Tasks

| Field | Content |
|-------|---------|
| **Actor(s)** | Any task producing output that subsequent tasks depend on |
| **Summary** | When a task produces outputs that change the processing state (e.g., new calibrations, updated flag summaries, image products, revised parameters), the system must provide a mechanism for those outputs to become available to subsequent processing steps. It must also retain those outputs as part of the execution record for later inspection, reporting, and export. These two needs may be satisfied through different access paths. |
| **Postconditions** | Downstream tasks can access the propagated processing state they need, and the task outputs are retained in the execution history for later retrieval. |

---

### UC-09 — Support Multiple Orchestration Drivers

| Field | Content |
|-------|---------|
| **Actor(s)** | Operations / automated processing (PPR-driven batch), pipeline developer / power user (interactive), recipe executor |
| **Summary** | The context is created and consumed by multiple front-ends: PPR command lists, XML procedures, or interactive task calls. The system must remain the stable state contract across these drivers. It must be creatable and resumable from non-interactive and interactive drivers, support driver-injected run metadata, tolerate partial execution controls (`startstage`, `exitstage`) and breakpoint-driven stop/resume semantics, and provide machine-detectable success/failure signals. |
| **Postconditions** | The same context state is usable regardless of which orchestration driver created or resumed it. |

---

### UC-10 — Save and Restore a Processing Session

| Field | Content |
|-------|---------|
| **Actor(s)** | Pipeline operator, workflow orchestration layer, developers |
| **Summary** | The system must be able to serialize the complete processing state to disk (all observation data, calibration state, execution history, imaging state, project metadata) and later restore it so that processing can resume from the saved point. The serialization must preserve enough state to resume when serialization compatibility is maintained; backward compatibility across pipeline releases is not guaranteed. On restore, paths must be adaptable to a new filesystem environment. |
| **Postconditions** | After restore, the processing state is operationally equivalent to the saved state for supported resume workflows, and processing can continue. |

---

### UC-11 — Provide State to Parallel Workers

| Field | Content |
|-------|---------|
| **Actor(s)** | Workflow orchestration layer, parallel worker processes |
| **Summary** | When work is distributed across parallel workers, each worker needs read-only access to the current processing state (observation metadata, calibration state, etc.). The system must provide a mechanism for workers to obtain a consistent snapshot of that state. Workers must not be able to modify the shared processing state directly. The snapshot mechanism must support efficient distribution to workers. |
| **Postconditions** | Each worker has a consistent, read-only view of the processing state for the duration of its work. |

---

### UC-12 — Aggregate Results from Parallel Workers

| Field | Content |
|-------|---------|
| **Actor(s)** | Workflow orchestration layer |
| **Summary** | After parallel workers complete, the system must collect their individual results and incorporate them into the shared processing state. The aggregation must be safe (no conflicting concurrent writes) and complete before the next sequential step begins. |
| **Postconditions** | The processing state reflects the combined outcomes of all parallel workers. |

---

### UC-13 — Provide Read-Only Context for Reporting Consumers

| Field | Content |
|-------|---------|
| **Actor(s)** | Report generators (weblog, quality reports, reproducibility scripts, AQUA reports, pipeline statistics) |
| **Summary** | The system must provide reporting consumers with read-only access to the observation metadata, project metadata, execution history, QA outcomes, log references, and path information needed to generate reporting products such as weblogs, quality reports, reproducibility scripts, AQUA reports, and pipeline statistics. |
| **Postconditions** | Reports accurately reflect the processing state at the time of generation. |

---

### UC-14 — Support QA Evaluation and Store Quality Assessments

| Field | Content |
|-------|---------|
| **Actor(s)** | QA scoring framework, report generators, later pipeline logic that consults recorded QA outcomes |
| **Summary** | After each processing step completes, the system must make the relevant processing state available to QA handlers so they can evaluate the outcome against quality thresholds, which may depend on telescope, project parameters, or observation properties. The resulting normalized quality scores must be recorded and remain retrievable for reporting and for later pipeline logic that explicitly consults recorded QA outcomes. |
| **Postconditions** | Quality scores are associated with the relevant processing step and accessible to reports and downstream logic. |

---

### UC-15 — Support Interactive Inspection and Debugging

| Field | Content |
|-------|---------|
| **Actor(s)** | Pipeline developer, pipeline operator, CI harnesses |
| **Summary** | The system must allow an operator to inspect the current processing state: which datasets are registered, what calibrations exist, how many steps have completed, and what their outcomes were. On failure, a snapshot of the state should be available for post-mortem analysis. The system must provide deterministic paths and outputs that a test harness can validate, and must surface failures beyond raw task exceptions. |
| **Postconditions** | The operator can understand the current state of processing and diagnose problems. |

---

### UC-16 — Manage Telescope-Specific Context Extensions

| Field | Content |
|-------|---------|
| **Actor(s)** | Telescope-specific tasks and heuristics |
| **Summary** | The system must support conditional telescope-specific extensions to the processing state. These extensions must be available to telescope-specific tasks and heuristics while remaining outside the assumed contract of shared pipeline code. Their presence must depend on the instrument being processed rather than being treated as universally available context state. |
| **Postconditions** | Telescope-specific extensions are present only for runs that require them, available to the tasks that need them, and not assumed by shared pipeline code. |

---

### UC-17 — Provide Context for Product Export

| Field | Content |
|-------|---------|
| **Actor(s)** | Export task, archive system |
| **Summary** | The system must make the datasets, calibration state, image products, reports, scripts, path information, and project identifiers available through the processing state so export tasks can assemble them into a deliverable product package. The package must be structured for downstream archive ingestion. |
| **Postconditions** | The information needed to assemble the product package is accessible through the processing state, and a self-contained product package can be produced. |

---

### UC-18 — Emit Lifecycle Notifications

| Field | Content |
|-------|---------|
| **Actor(s)** | Workflow orchestration layer, event subscribers (loggers, progress monitors) |
| **Summary** | The system must emit notifications at key lifecycle points (session start, session restore, step start, step completion, result acceptance) so that external observers (logging, progress reporting, live dashboards) can track execution without polling. |
| **Postconditions** | Subscribers are notified of lifecycle transitions as they occur. |

---

## 2. Use Cases the Current Design Cannot Handle

The following describe scenarios that the current context design *does not support* but that could be valuable in a future architecture. They are numbered GAP-01 through GAP-07 to indicate gaps in the current design's capabilities.

> **Design note:** The context design should remain compatible with multi-tenant deployments (no global singletons that leak state between runs, no hardcoded single-user paths), but access control, role-based permissions, and audit logging are concerns of the deployment platform rather than the context itself.

### GAP-01 — Concurrent / Overlapping Task Execution

| Field | Content |
|-------|---------|
| **Actor(s)** | Workflow engine, parallel task scheduler |
| **Summary** | The system should support concurrent execution of otherwise independent tasks (e.g., per-MS or per-SPW calibration stages) with isolated read snapshots, a merge/reconciliation step when concurrent results are accepted, and explicit declaration of which state fields each task reads and writes. This gap is about overlapping task execution against shared state, not the existing worker fan-out/fan-in pattern used for some parallel work. |
| **Postconditions** | Independent tasks execute in parallel without corrupting shared state; results are merged safely before the next sequential step. |

---

### GAP-02 — Cloud / Distributed Execution Without Shared Filesystem

| Field | Content |
|-------|---------|
| **Actor(s)** | Workflow engine, cloud orchestrator, distributed workers |
| **Summary** | The system should support execution across nodes that do not share a filesystem. This requires a context store backed by a database or object store, artifact references rather than filesystem paths for calibration tables and images, and tasks that can operate on remote datasets without requiring local copies. |
| **Postconditions** | Processing completes successfully across distributed nodes without reliance on a shared filesystem. |

---

### GAP-03 — Multi-Language / Multi-Framework Access to Context

| Field | Content |
|-------|---------|
| **Actor(s)** | Non-Python clients (C++, Julia, JavaScript dashboards), external tools |
| **Summary** | The system should expose context state through a language-neutral interface, using a portable serialization format (Protocol Buffers, JSON-Schema, Arrow), a query API (REST, gRPC, or GraphQL), and type definitions shared across languages. |
| **Postconditions** | Clients in any supported language can read context state through a stable, typed API. |

---

### GAP-04 — Streaming / Incremental Processing

| Field | Content |
|-------|---------|
| **Actor(s)** | Data ingest system, workflow engine, incremental processing tasks |
| **Summary** | The system should support incremental dataset registration (adding new scans or execution blocks to a live session), tasks that detect new data and re-process incrementally, and a results model that supports versioning so that re-runs produce new versions rather than overwriting previous outputs. |
| **Postconditions** | New data is incorporated into an active session and processed without restarting from scratch. |

---

### GAP-05 — Provenance and Reproducibility Guarantees

| Field | Content |
|-------|---------|
| **Actor(s)** | Pipeline operator, auditor, reproducibility tooling |
| **Summary** | The system should maintain immutable per-stage snapshots of context state (event sourcing), hash all task inputs (context fields and parameters) for cache invalidation and reproducibility tokens, record software provenance (pipeline version, framework version, dependency manifest) and input data identity (checksums or URIs for raw datasets) per run, and support replaying a run from the event log. |
| **Postconditions** | Any past processing step can be precisely reproduced or audited from the recorded provenance chain, including the exact software and data versions that produced it. |

---

### GAP-06 — Partial Re-Execution / Targeted Stage Re-Run

| Field | Content |
|-------|---------|
| **Actor(s)** | Pipeline operator, developer, workflow engine |
| **Summary** | The system should support selectively re-running a single mid-pipeline stage with different parameters while keeping earlier and later stages intact. This requires explicit dependency tracking between stages, the ability to invalidate downstream stages, and versioned per-stage results. |
| **Postconditions** | A targeted stage is re-executed and downstream state is correctly invalidated or updated; unaffected stages are preserved. |

---

### GAP-07 — External System Integration (Archive, Scheduling, QA Dashboards)

| Field | Content |
|-------|---------|
| **Actor(s)** | Archive ingest system, scheduling database, QA dashboards, monitoring tools |
| **Summary** | The system could expose a stable, queryable API (REST/gRPC) that external systems can poll or subscribe to, support webhook/event notifications for state transitions, and publish a standard schema for context summaries consumable by external systems. However, this involves a significant trade-off: the current context has no stable API, which gives development teams full flexibility to evolve internal structures without cross-team coordination. A public API would require a formal stability contract, versioning discipline, and potentially a slow-to-change external interface layered over rapidly evolving internals. Whether this gap is in scope — or whether external integration should remain an offline, product-file-based concern — is an open design question. |
| **Postconditions** | External systems receive timely, structured updates about processing state without relying on offline product files. |
