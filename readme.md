# Development Open Service Broker

<div align="center">

**Reference implementations, tooling, and documentation for the Open Service Broker API**

[![Status: Active Development](https://img.shields.io/badge/status-active%20development-7c3aed?style=for-the-badge)](#roadmap)
[![Go](https://img.shields.io/badge/Go-reference%20implementation-00ADD8?style=for-the-badge&logo=go&logoColor=white)](./osb-reference-go)
[![Java](https://img.shields.io/badge/Java-Spring%20Boot-6DB33F?style=for-the-badge&logo=springboot&logoColor=white)](./osb-reference-java-spring-boot)
[![Checker](https://img.shields.io/badge/OSB-checker-F59E0B?style=for-the-badge)](./osb-checker)

*One project. Multiple implementations. Reliable broker integrations.*

</div>

---

## Overview

**Development Open Service Broker** is a developer-focused workspace for exploring, implementing, and validating Open Service Broker integrations.

The repository combines two reference implementations with an OSB checker so that API behavior, broker workflows, and conformance can be developed side by side.

```text
┌─────────────────────────────────────────────────────────────┐
│              Development Open Service Broker               │
├──────────────────┬──────────────────────┬───────────────────┤
│   Go Reference   │ Java / Spring Boot   │    OSB Checker    │
│   Implementation │ Reference Service    │ Validation Tool   │
└──────────────────┴──────────────────────┴───────────────────┘
```

## Components

| Component | Description | Technology |
|---|---|---|
| [`osb-reference-go`](./osb-reference-go) | Lightweight reference implementation for Go-based services | Go |
| [`osb-reference-java-spring-boot`](./osb-reference-java-spring-boot) | Enterprise-friendly reference implementation with Spring Boot | Java, Spring Boot |
| [`osb-checker`](./osb-checker) | Validation and conformance tooling for OSB implementations | See component README |
| [`docs`](./docs) | Shared architecture, development, and integration documentation | Markdown |

## Why this project?

- **Learn by example** — compare equivalent broker concepts in Go and Java.
- **Build with confidence** — validate implementations with a dedicated checker.
- **Keep the contract visible** — document API behavior, assumptions, and workflows in one place.
- **Support experimentation** — use the reference services as a foundation for prototypes and tests.
- **Reduce integration friction** — identify interoperability issues before deployment.

## Repository layout

```text
.
├── docs/                               # Shared documentation
├── osb-reference-go/                   # Go reference implementation
├── osb-reference-java-spring-boot/     # Java/Spring Boot implementation
├── osb-checker/                        # Validation and conformance tooling
├── .gitignore
├── LICENSE
└── README.md
```

## Quick start

### 1. Clone the repository

```bash
git clone https://github.com/<your-org>/development-open-service-broker.git
cd development-open-service-broker
```

### 2. Choose a component

```bash
cd osb-reference-go
# or:
cd osb-reference-java-spring-boot
# or:
cd osb-checker
```

### 3. Follow the component guide

Each subproject has its own `README.md` with prerequisites, configuration, startup commands, and tests.

> **Tip:** Start with the checker when integrating a new broker implementation. It provides a practical feedback loop while developing the service endpoints.

## Development workflow

A typical workflow looks like this:

```text
Implement → Run locally → Validate with OSB Checker → Fix → Test → Review
```

Recommended development steps:

1. Read the relevant component README and shared documentation.
2. Implement or change one broker capability at a time.
3. Add or update automated tests.
4. Run the checker against the local service.
5. Document behavior that is important for consumers or operators.
6. Open a pull request with a focused description.

## Project principles

### Clear contracts

API behavior should be explicit, documented, and testable.

### Comparable implementations

The Go and Java reference services should represent equivalent broker concepts wherever practical.

### Automation first

Validation and regression tests belong close to the implementation.

### Small, reviewable changes

Prefer focused commits and pull requests over large, unrelated changes.

## Roadmap

- [ ] Define the shared OSB capability matrix.
- [ ] Add a minimal catalog flow to both reference implementations.
- [ ] Add provisioning and deprovisioning examples.
- [ ] Add service binding and unbinding examples.
- [ ] Expand checker coverage for lifecycle operations.
- [ ] Add container images for local evaluation.
- [ ] Add CI pipelines for build, test, and validation.
- [ ] Publish contribution and release guidelines.

> The roadmap is intentionally incremental. Features can be tracked as GitHub Issues and grouped in a GitHub Project.

## Contributing

Contributions are welcome — whether they improve an implementation, extend the checker, clarify documentation, or add tests.

Before opening a pull request:

- Keep changes scoped to one purpose.
- Add tests for new behavior where applicable.
- Update the appropriate component README or documentation.
- Run the component's build and test commands.
- Explain validation results in the pull request description.

Suggested branch names:

```text
feature/go-catalog-endpoint
feature/java-service-binding
feature/checker-lifecycle-validation
fix/catalog-response-validation

docs/local-development
```

## Documentation

Shared documentation belongs in [`docs/`](./docs). Component-specific instructions belong in the respective subproject directory.

Useful documentation topics include:

- Architecture and component responsibilities.
- Local development and test setup.
- OSB endpoint behavior.
- Configuration and environment variables.
- Checker usage and expected results.
- Release and compatibility notes.

## License

This project is licensed under the terms defined in [`LICENSE`](./LICENSE).

If no license has been selected yet, add one before publishing or distributing the project.

## Status

🚧 **Active development**

The repository is evolving. Interfaces, commands, and directory names may change until the first stable release.

<div align="center">

**Build it. Check it. Improve it.**

</div>
