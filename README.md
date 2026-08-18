# radps-context

This repository defines the processing context used inside the RADPS pipeline.
`radps-context` maintains pipeline domain state and exposes it through internal
interfaces to workers, heuristics, workflow orchestration, and other pipeline
components.

External interfaces are outside this component's boundary. A separate subsystem
is responsible for user-facing APIs, operator tools, dashboards, notifications,
archive integration, product delivery, and other interactions outside the
pipeline. That subsystem may consume or submit normalized information through
the same internal pipeline interfaces; `radps-context` does not implement the
external interaction.

## RADPS requirements

- [RADPS context use cases](docs/radps_context_use_cases.md) defines the
  internal interactions supported by the context.
- [RADPS context quality requirements](docs/radps_context_quality_requirements.md)
  defines guarantees at those internal interfaces.
- [Context use-case mapping and ownership](docs/requirements_and_ownership.md)
  maps current Pipeline capabilities to the RADPS component boundary.

## Current Pipeline analysis

- [Pipeline context use cases](docs/context_use_cases_current_pipeline.md)
  records responsibilities of the current Pipeline context.
- [Current Pipeline appendix](docs/context_current_pipeline_appendix.md)
  preserves supporting implementation evidence.

These historical documents describe the existing system and do not assign
future RADPS ownership. In particular, current Pipeline reporting, export,
archive, and operator-facing behavior is not automatically in scope for
`radps-context`.

## Reference

- [Glossary](docs/glossary.md) defines terminology shared by the requirements
  and historical analysis.
