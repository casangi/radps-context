# RADPS Context Quality Requirements

## Purpose and boundary

These requirements define guarantees observable through the internal workflow interfaces of `radps-context`. They state required behavior without selecting an implementation mechanism or defining interfaces to systems outside the workflow.

## Requirements

### RADPS-QR1 — Atomic outcomes

A related set of domain-state changes must become visible as one complete outcome or have no visible effect. Internal consumers must not observe a partially applied calibration update, imaging update, artifact registration, or restart-boundary change.

### RADPS-QR2 — Consistent reads

An internal consumer must receive a coherent view for the run, dataset, partition, or processing boundary it requests. A read must not combine mutually incompatible state from before and after a concurrent update.

### RADPS-QR3 — Concurrent-update safety

Independent workflow tasks may proceed concurrently, but incompatible updates must not silently overwrite or combine with one another. Conflicts must be detected or prevented, and incomplete outcomes must not become eligible inputs to dependent work.

### RADPS-QR4 — Retry safety

Repeating an internal request after a timeout, worker failure, or uncertain response must not unintentionally duplicate its logical effect. This applies to run initialization, state updates, artifact registration, annotations, directives, and checkpoint operations.

### RADPS-QR5 — Durability

Accepted domain state, provenance, annotations, directives, and artifact relationships must survive failures within the guarantees agreed between workflow components.

### RADPS-QR6 — Stable identity and traceability

Runs, datasets, state versions, attempts referenced by domain updates, artifacts, and processing boundaries must remain unambiguously identifiable for as long as workflow components reference them. Relationships among inputs, updates, outputs, and superseded state must remain traceable.

### RADPS-QR7 — Location and execution portability

State and artifact references must remain usable when workflow tasks move between workers or execution environments. An internal consumer must not depend on another process's memory or assume that a producer's local filesystem path is available.

### RADPS-QR8 — Compatibility and explicit failure

Internal components must be able to determine whether the context information and operations they use are compatible. Unsupported versions, operations, or ambiguous requests must fail explicitly rather than returning silently misinterpreted state.

### RADPS-QR9 — Source attribution

For an accepted change whose origin matters to later processing, the context must record the producing workflow component and affected scope.

### RADPS-QR10 — Domain provenance

The context must retain the domain information required to explain how accepted state and artifacts were derived from their inputs. The workflow system owns non-domain execution history.
