# Glossary

This glossary defines terms used by the RADPS context requirements and the
current Pipeline analysis. Future-facing definitions describe contracts at
internal pipeline interfaces. Terms under **Current Pipeline terms** document
the existing system and do not assign RADPS ownership.

## RADPS shared terms

- **Artifact**: A durable data product produced or consumed by pipeline work,
  such as a dataset partition, calibration product, image, or derived metadata
  product.
- **Artifact reference**: An internal, location-portable reference that allows
  an authorized pipeline component to resolve an artifact through the
  appropriate data or artifact service.
- **Atomic outcome**: A related set of changes that becomes visible in full or
  has no visible effect.
- **Context**: The domain state and relationships needed by pipeline work during
  a run. It is consumed through internal pipeline interfaces and does not itself
  provide external-system integration.
- **Context-model version**: The version identifier for context information and
  the rules used to interpret it.
- **Domain state**: Processing information whose meaning belongs to the science
  pipeline, including observation metadata, calibration state, imaging state,
  quality assessments, domain decisions, and artifact lineage.
- **External-interface subsystem**: A component outside `radps-context` that
  handles user-facing APIs, operator tools, dashboards, notifications, archive
  protocols, product delivery, or other interactions beyond the pipeline. It
  exchanges normalized requests and responses with internal pipeline
  interfaces.
- **Internal adapter**: A pipeline component that translates between an
  external-interface subsystem and an internal context operation. The adapter,
  not `radps-context`, owns protocol translation, authentication, delivery, and
  external error handling.
- **Internal consumer**: A worker, heuristic, workflow component, pipeline task,
  or internal adapter that reads context information.
- **Internal producer**: A worker, heuristic, workflow component, pipeline task,
  or internal adapter that submits a context update.
- **Lineage**: Relationships explaining how an artifact or accepted domain
  state was derived from inputs and upstream products.
- **Matching semantics**: Rules used to determine whether metadata elements
  across datasets correspond, such as exact, overlap, or partial matching.
- **Processing boundary**: A consistent, identifiable point in processing that
  can be used for an internal read, checkpoint, resume, or rerun decision.
- **Provenance**: Information needed to explain processing. `radps-context` owns
  domain provenance; workflow orchestration owns non-domain execution history.
- **Run**: One identifiable instance of pipeline processing from internal
  initialization through completion or termination.
- **Stable identifier**: An identity that remains unambiguous for as long as an
  internal pipeline component references the identified entity.
- **Workflow orchestration layer**: The system responsible for planning,
  scheduling, and coordinating pipeline work, including dependency progression,
  retries, resume, and enforcement of control decisions.

## Current Pipeline terms

- **ASDM**: ALMA Science Data Model; an archive and distribution format
  typically converted into a MeasurementSet for processing.
- **AQUA**: ALMA QA reporting output used by current export and reporting code.
- **Calibration table / caltable**: A CASA product storing calibration
  solutions.
- **CASA**: Common Astronomy Software Applications; the environment and toolset
  used by the current Pipeline.
- **Dask**: A parallel and distributed execution framework used by some current
  Pipeline task queues.
- **Event bus**: The current Pipeline's in-process publication mechanism for
  lifecycle markers.
- **Executor**: The current Pipeline component that runs task jobs and may
  accept returned results into the shared context.
- **FITS**: Flexible Image Transport System; a format used for exported image
  products.
- **MeasurementSet (MS)**: A radio astronomy dataset format used by CASA and the
  current Pipeline.
- **MPI**: Message Passing Interface; a mechanism used by the current Pipeline
  to distribute work.
- **PPR**: Pipeline Processing Request; an XML bundle containing inputs,
  metadata, and an ordered command list for automated processing.
- **Results**: A current Pipeline per-task output object whose acceptance may
  update shared context state.
- **ResultsProxy**: A current Pipeline on-disk proxy used to load a task result
  when required.
- **SPW (spectral window)**: A spectral subdivision of radio astronomy data.
- **Stage**: A sequential top-level step in a current Pipeline run.
- **Task**: A registered unit of pipeline work that typically produces a Results
  object.
- **Weblog**: The human-readable HTML report generated from current Pipeline
  context and results.
