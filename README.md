# OpenFastTrace AI Skills

This repository contains a collection of AI skills around using [OpenFastTrace](https://openfasttrace.itsallcode.org).

## MIT License

While OpenFastTrace itself is licensed under the GPL, the skills in this repository are licensed under the [MIT license](LICENSE). The reason for this is to allow developers to use and modify the skills without having to release their modifications.

Skills are often used as templates, and the individual modifications are highly user-specific.

## Skills in This Repository

- **[OpenFastTrace](skills/openfasttrace/SKILL.md)** — Covers OpenFastTrace requirement tracing, including item ID syntax, Markdown and Gherkin coverage markers, and trace verification through the CLI, Maven, or Gradle. Use it when creating, reviewing, or validating traced specifications, code, tests, or documentation.
- **[OFT Spec-Driven Development](skills/openfasttrace-spec-driven-development/SKILL.md)** — Guides a spec-first development workflow that turns issues into changeset task plans and keeps traced requirements, arc42 design documents, implementation, and quality gates aligned before and during development.
- **[OpenFastTrace Reverse Specs](skills/openfasttrace-reverse-specs/SKILL.md)** — Comprehensive, architecture-oriented skill that reverse-engineers system requirements and arc42-style design documents (`arch`, `dsn`) from documentation, user guides, source code, and tests using its own full architectural tracing chain.
- **[OpenFastTrace Reverse Requirements](skills/openfasttrace-reverse-requirements/SKILL.md)** — Lightweight, code-focused skill that reverse-engineers domain requirements directly from existing source code and unit tests, adapts flexibly to the project's own established tracing chains and standards, and links coverage markers at usage points.
