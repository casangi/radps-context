# Glossary

This glossary defines terms used by the current Pipeline and RADPS context requirements documents. It describes problem-domain concepts and externally observable guarantees; implementation-specific terms belong in design documentation.

## Shared Terms

- **Actor**: A person, role, system, or process that directly interacts with the system described by a use case.

- **Artifact**: A durable data product produced or used by pipeline processing, such as a dataset partition, calibration product, image, report, or manifest.

- **Archival product**: A previously generated durable product supplied from outside the active run so processing can begin or resume from an intermediate state.

- **Atomic outcome**: A related set of changes that becomes visible in full or has no visible effect. Consumers do not observe only part of the outcome.

- **Consistency**: The guarantee that processing state obeys its declared rules and that a consumer does not receive mutually incompatible information in one view.

- **Context**: The processing state and associated relationships needed by pipeline work throughout a run. The requirements do not prescribe whether that context is implemented as an object, service, database, or another mechanism.

- **Dependency graph**: A representation of planned work and the dependencies that determine which work may proceed.

- **Deterministic execution / determinism policy**: The expectation that equivalent inputs, software, parameters, and resource conditions produce equivalent results within declared numerical tolerances. Deviations must be explainable from provenance.

- **Execution attempt**: One try to perform a planned unit of work. A retry is a separate attempt and must remain distinguishable from earlier tries.

- **Execution-control directive**: An authorized instruction that changes workflow behavior, such as pausing, skipping, or rerouting work.

- **Idempotency / retry safety**: The guarantee that repeating a request after a timeout or uncertain response does not unintentionally repeat its logical effect.

- **Invariant**: A condition that must remain true while the system is operating.

- **Lineage**: Relationships that explain how an artifact or result was produced from its inputs and upstream products.

- **Manifest**: A machine-readable inventory or summary of processing products and provenance-relevant information.

- **Matching semantics**: The rules used to determine whether metadata elements across datasets correspond. Different consumers may require exact, overlap, partial, or other domain-defined matching behavior.

- **MeasurementSet v4 (MSv4)**: The next-generation MeasurementSet representation targeted for RADPS and exposed through the XRADIO package. RADPS may normalize observation information around this representation even when inputs begin as ASDM or another archive format.

- **Orchestration driver**: A front end that creates or controls pipeline execution. In the current production environment, PLDriver serves this role. Other current driver styles include PPR command lists, XML procedures, and interactive task calls.

- **Postcondition**: A condition that must be true after a use case completes successfully.

- **Precondition**: A condition that must hold before a use case can begin.

- **Processing boundary**: A meaningful point in processing at which the applicable state and completed outputs can be identified consistently, for example when establishing a restart point or producing a report.

- **Provenance**: Information needed to explain or reproduce processing, including inputs, parameters, software, execution environment, hardware and resource conditions, control decisions, and lineage.

- **Quality requirement**: An externally observable guarantee that applies across one or more use cases, such as consistency, durability, portability, or retry safety.

- **Run**: One identifiable instance of pipeline processing from initialization through completion, termination, or archival handoff.

- **Stable identifier**: An identity that remains unambiguous for as long as the identified run, data, work, attempt, artifact, or boundary is referenced.

- **Stakeholder**: A person, team, organization, or external consumer with an interest in system behavior, outputs, or constraints. A stakeholder is not necessarily an actor.

- **Workflow orchestration layer**: The system responsible for planning, scheduling, and coordinating pipeline work, including dependency progression, retries, and enforcement of execution-control decisions.

## Current Pipeline Terms

- **ASDM**: ALMA Science Data Model; an archive and distribution format typically converted into a MeasurementSet for processing.

- **AQUA**: ALMA QA reporting output used by export and reporting code to package quality-assessment summaries.

- **Calibration table / caltable**: A CASA product that stores calibration solutions.

- **CASA**: Common Astronomy Software Applications; the environment and toolset used by the current Pipeline.

- **Dask**: A Python parallel and distributed execution framework used by some current Pipeline task queues.

- **Event bus**: The current Pipeline's in-process publication mechanism for lifecycle markers. It is not the primary state-mutation mechanism.

- **Executor**: The current Pipeline component that runs task jobs and may accept returned results into the shared context.

- **FITS**: Flexible Image Transport System; a standard astronomy format used for exported image products.

- **MeasurementSet (MS)**: A radio astronomy dataset format used by CASA and the current Pipeline.

- **MPI**: Message Passing Interface; a mechanism used by the current Pipeline to distribute work to worker processes.

- **NRO**: Nobeyama Radio Observatory; in these documents, a single-dish processing and export variant.

- **OUS**: Observing Unit Set; an ALMA processing and packaging identity.

- **PPR**: Pipeline Processing Request; an XML bundle containing inputs, metadata, and an ordered command list for automated processing.

- **Results**: A current Pipeline per-task output object. Accepted results update shared context state.

- **ResultsProxy**: A current Pipeline on-disk proxy that allows a task result to be loaded when required.

- **SD (single-dish)**: Single-dish processing, including ALMA TP and NRO paths.

- **SPW (spectral window)**: A spectral subdivision of radio astronomy data.

- **Stage**: A sequential top-level step in a current Pipeline run, generally associated with one task execution.

- **Task**: A registered unit of pipeline work that typically produces a Results object.

- **Weblog**: The human-readable HTML report generated from current Pipeline context and results.
