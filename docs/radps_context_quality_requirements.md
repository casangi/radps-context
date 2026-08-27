# RADPS Context Quality Requirements

## Purpose and boundary

These requirements define guarantees observable through the internal Workflow interfaces of `radps-context`. They state required behavior without selecting an implementation mechanism or defining interfaces to systems outside the Workflow.

## Requirements

### RADPS-QR1 — Atomic outcomes

A related set of domain-state changes must become visible as one complete outcome or have no visible effect. Internal consumers must not observe a partially applied calibration update, imaging update, processing-output registration, or processing-boundary change.

### RADPS-QR2 — Consistent reads

An internal consumer must receive a coherent view for the run, dataset, data chunk, partition, node task, or processing boundary it requests. A read must not combine mutually incompatible state from before and after a concurrent update.

### RADPS-QR3 — Concurrent-update safety

Independent node tasks, including tasks operating over distinct data chunks, may proceed concurrently, but incompatible updates must not silently overwrite or combine with one another. Conflicts must be detected or prevented, accepted chunk-scoped outcomes must remain distinguishable for deterministic combination, and incomplete outcomes must not become eligible inputs to dependent work.

### RADPS-QR4 — Retry safety

Repeating an internal request after a timeout, worker failure, or uncertain response must not unintentionally duplicate its logical effect. This applies to run initialization, state updates, processing-output registration, annotations, directives, and operations that establish context state for a Checkpoint Record.

### RADPS-QR5 — Durability

Accepted domain state, provenance, annotations, directives, and processing-output relationships must survive failures within the guarantees agreed between Workflow components.

### RADPS-QR6 — Stable identity and traceability

Runs, datasets, data chunks, node tasks, attempts, state versions, processing outputs, and processing boundaries must remain unambiguously identifiable for as long as Workflow components reference them. Relationships among inputs, updates, outputs, and superseded state must remain traceable.

### RADPS-QR7 — Location and execution portability

State and output references must remain usable when node tasks move between workers or execution environments. An internal consumer must not depend on another process's memory or assume that a producer's local filesystem path is available.

### RADPS-QR8 — Compatibility and explicit failure

Internal components must be able to determine whether the context information and operations they use are compatible. Unsupported versions, operations, or ambiguous requests must fail explicitly rather than returning silently misinterpreted state.

### RADPS-QR9 — Source attribution

For an accepted change whose origin matters to later processing, the context must record the producing node task or Workflow component and affected data-chunk or processing scope.

### RADPS-QR10 — Domain provenance

The context must retain the domain information required to explain what processing outcomes have been accepted and how accepted state and processing outputs were derived from their inputs. The Workflow Framework owns non-domain execution history.

### RADPS-QR11 — Checkpoint and restart consistency

Context state referenced by a Checkpoint Record must represent a complete, committed processing boundary and remain loadable under its declared context-model compatibility guarantees. Restoring that state must reproduce the accepted domain state at the referenced boundary; incomplete or incompatible state must be rejected explicitly.
