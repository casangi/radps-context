# radps-context

This repository defines what the RADPS context must provide through internal workflow interfaces and preserves the analysis of current Pipeline behavior that informed those requirements. Use cases describe required capabilities and observable outcomes; current implementation details are documented separately so they do not prescribe the future RADPS design.

## RADPS Requirements

- [docs/radps_context_use_cases.md](docs/radps_context_use_cases.md): RADPS context use cases describing actor goals and observable outcomes.
- [docs/radps_context_quality_requirements.md](docs/radps_context_quality_requirements.md): Cross-cutting behavioral guarantees that apply to the RADPS context use cases.

## Current Pipeline Analysis

- [docs/context_use_cases_current_pipeline.md](docs/context_use_cases_current_pipeline.md): Implementation-neutral use cases describing current Pipeline context responsibilities.
- [docs/context_current_pipeline_appendix.md](docs/context_current_pipeline_appendix.md): Supplementary implementation notes, code references, and lifecycle analysis for the current Pipeline use cases.

## Traceability and Responsibility

- [docs/requirements_and_ownership.md](docs/requirements_and_ownership.md): Traceability from current Pipeline capabilities and identified gaps to RADPS requirements, together with responsibility allocation across `radps-context` and workflow orchestration.

## Reference

- [docs/glossary.md](docs/glossary.md): Definitions for terminology used throughout the RADPS requirements and current Pipeline analysis.

Conceptual and detailed design decisions are outside the scope of these requirements and analysis documents. This project is distributed under the terms in [LICENSE](LICENSE).
