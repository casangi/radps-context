# Glossary

This glossary defines terms used by the RADPS context use cases, quality requirements, and current Pipeline analysis, with a section at the end for future Workflow terms.

## RADPS shared terms

- **Atomic outcome**: A related set of changes that becomes visible in full or has no visible effect.
- **Checkpoint Record**: A record managed by the Workflow Framework that associates a processing boundary with an identifiable context-state version and the processing outputs required for rollback recovery or failure restart.
- **Context**: A record of accepted domain processing outcomes, relevant QA and heuristic results, and state needed by downstream work. It provides domain state and restart traceability through internal Workflow interfaces but does not own node-task execution or external-system interactions.
- **Context-model version**: The version identifier for context information and the rules used to interpret it.
- **Data chunk**: An identifiable portion of a dataset assigned to one or more node tasks for data-parallel processing.
- **Dask**: A parallel and distributed execution framework used by some current Pipeline task queues in development-only configurations and identified as a task-scheduler technology for the Workflow Framework. Logical roles and `radps-context` interfaces remain independent of Dask-specific mechanisms.
- **Domain state**: Processing information whose meaning belongs to science processing, including observation metadata, calibration state, imaging state, quality assessments, domain decisions, and processing-output lineage.
- **Execution-control directive**: An instruction that affects Workflow execution, such as pausing, skipping, or rerouting work. `radps-context` may store the directive, while the Workflow Framework interprets and enforces it.
- **Final data product**: A processing output designated for archive, distribution, or another external consumer.
- **Intermediate artifact**: A processing output retained for later work, checkpointing, recovery, or other internal use rather than designated as a final data product.
- **Internal consumer**: A worker, heuristic, Workflow Framework component, or node task that reads context information.
- **Internal producer**: A worker, heuristic, Workflow Framework component, or node task that submits a context update.
- **Lineage**: Relationships explaining how a processing output or accepted domain state was derived from inputs and upstream outputs.
- **Matching semantics**: Rules used to determine whether metadata elements across datasets correspond, such as exact, overlap, or partial matching.
- **Node task**: A unit of work that can be assigned to a node. It invokes processing functions and consumes and produces dataset partitions or intermediate artifacts.
- **Output reference**: A location-portable reference that allows a Workflow component to locate a processing output.
- **Processing boundary**: A consistent, identifiable point in processing that can be used for an internal read or associated with a Checkpoint Record for rollback, restart, resume, or rerun.
- **Processing output**: A generic term for data or a supporting record produced during a Workflow run when its classification as an intermediate artifact or final data product is not relevant or has not yet been established.
- **Provenance**: Information needed to explain processing. `radps-context` owns domain provenance; the Workflow Framework owns non-domain execution history.
- **Run**: One identifiable instance of Workflow processing from internal initialization through completion or termination.
- **Stable identifier**: An identity that remains unambiguous for as long as an internal Workflow component references the identified entity.
- **Workflow**: An end-to-end data processing procedure for a defined science or operational purpose.
- **Workflow Framework**: The logical component responsible for orchestrating a Workflow, including work decomposition, dependency progression, node-task scheduling, retries, checkpoint management, restart, and enforcement of control decisions, without owning the domain-specific processing invoked by node tasks.

## Current Pipeline terms

- **ASDM**: ALMA Science Data Model; an archive and distribution format typically converted into a MeasurementSet for processing.
- **AQUA**: ALMA QA reporting output used by current export and reporting code.
- **Calibration table / caltable**: A CASA product storing calibration solutions.
- **CASA**: Common Astronomy Software Applications; the environment and toolset used by the current Pipeline.
- **Execution driver**: The generic role that initiates and coordinates current Pipeline processing. `pipelinedriver` is one concrete software implementation of this role.
- **Executor**: The current Pipeline component that runs task jobs and may accept returned results into the shared context.
- **FITS**: Flexible Image Transport System; a format used for exported image products.
- **MeasurementSet (MS)**: A radio astronomy dataset format used by CASA and the current Pipeline.
- **MPI**: Message Passing Interface; a mechanism used by the current Pipeline to distribute work.
- **PPR**: Pipeline Processing Request; an XML bundle containing inputs, metadata, and an ordered command list for automated processing.
- **Results**: A current Pipeline per-task output object whose acceptance may update shared context state.
- **ResultsProxy**: A current Pipeline on-disk proxy used to load a task result when required.
- **SPW (spectral window)**: A spectral subdivision of radio astronomy data.
- **Stage**: A sequential top-level step in a current Pipeline run.
- **Task**: A registered unit of pipeline work that typically produces a Results object.
- **Weblog**: The human-readable HTML report generated from current Pipeline context and results.

## Future Workflow terms

- **External-interface subsystem**: A component outside `radps-context` that handles interactions between the Workflow and external systems, including user-facing APIs, operator tools, dashboards, notifications, archive protocols, and final-data-product delivery. It exchanges normalized requests and responses with internal Workflow interfaces.
