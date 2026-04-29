gh-clone-spec
=============

[![License](https://img.shields.io/badge/license-Apache--2.0%20OR%20MIT-blue.svg)](https://opensource.org/licenses/Apache-2.0)

The goal of this project is a precise replication of GitHub. Including its API, and an Angular (TypeScript+SCSS+HTML) implementation of its frontend.

Pointing any official GitHub project to a hosted version of this project should work out of the box with no changes. E.g., [`gh`](https://cli.github.com) CLI tool.

---

## Organization Repositories

This project utilizes a multi-repo architecture under the [`gh-clone`](https://github.com/gh-clone) organization.

| Repository | Description | Primary Tech Stack | Dependencies (Internal) |
| :--- | :--- | :--- | :--- |
| [`gh-clone-backend`](https://github.com/gh-clone/gh-clone-backend) | Core API application server, orchestrator, and single-binary entrypoint. | Rust, Axum, sqlx | `gh-clone-openapi-compiler`, `gh-clone-angular-frontend` |
| [`gh-clone-angular-frontend`](https://github.com/gh-clone/gh-clone-angular-frontend) | 100% feature-parity UI, SSR, and real-time streaming components. | Angular 21, RxJS | None |
| [`gh-clone-openapi-compiler`](https://github.com/gh-clone/gh-clone-openapi-compiler) | AST compiler generating the Rust API routing/types from the OpenAPI spec. | Rust, petgraph | None |
| [`gh-clone-spokes`](https://github.com/gh-clone/gh-clone-spokes) | Distributed Git storage, RPC layer, and replica consensus engine. | Rust, gRPC, gitoxide | `gh-clone-authz` |
| [`gh-clone-actions-runner`](https://github.com/gh-clone/gh-clone-actions-runner) | CI/CD execution engine utilizing isolated sandboxes and containers. | Rust / .NET | `gh-clone-backend` |
| [`gh-clone-search`](https://github.com/gh-clone/gh-clone-search) | High-performance code search indexer ("Blackbird" clone). | Rust, Hyperscan | `gh-clone-backend` |
| [`gh-clone-authz`](https://github.com/gh-clone/gh-clone-authz) | Zanzibar-inspired high-speed authorization and permissions graph. | Rust, gRPC | None |
| [`gh-clone-infrastructure`](https://github.com/gh-clone/gh-clone-infrastructure) | Deployment automation, Helm charts, and appliance builder (AMIs/VHDs). | Terraform, K8s, Packer | All repositories |

---

## The Master Blueprint (`/plans`)

This project is governed by a highly rigorous, multi-phased engineering blueprint located in the [`plans/`](plans) directory. This blueprint details the systematic approach required to achieve our goal of 100% feature parity with the GitHub ecosystem, from foundational architecture to exhaustive quality assurance. 

The master plan is divided into 7 core phases (Phases 0 through 6), each tackling a critical dimension of the project:

- **[Phase 0: Foundation, Architecture, & Environment Setup](plans/PLAN_PHASE0.md)**  
  Establishes the core structural foundation, development environments, CI/CD pipelines, and initial language choices for a high-performance single-binary entry point.
- **[Phase 1: Hexagonal Architecture & State Management](plans/PLAN_PHASE1.md)**  
  Details a strict separation of concerns utilizing core domains, application services, and infrastructure adapters, alongside high-performance database tuning (e.g., optimal SQLite/WAL configurations).
- **[Phase 2: API Parity & The Generation Compiler](plans/PLAN_PHASE2.md)**  
  Focuses on the precise replication of the GitHub API, detailing the code generation strategies needed to maintain exact endpoint and schema parity.
- **[Phase 3: The Distributed Git Data Plane (Protocol Level)](plans/PLAN_PHASE3.md)**  
  Dives into the protocol-level implementation of the distributed Git data plane, ensuring perfect emulation of Git operations.
- **[Phase 4: The Computational Roadmap & Search Engine](plans/PLAN_PHASE4.md)**  
  Outlines advanced functionalities, including GitHub Actions emulation, background workers, and full-text code search mechanics.
- **[Phase 5: Omni-Platform Deployment & Packaging](plans/PLAN_PHASE5.md)**  
  Covers the deployment strategies to ensure universal portability, scaling from a standalone zero-config binary (a "Gitea Killer") up to globally distributed Kubernetes clusters.
- **[Phase 6: Exhaustive QA, Observability & Threat Modeling](plans/PLAN_PHASE6.md)**  
  Ensures absolute reliability, security, accessibility (a11y), internationalization (i18n), and pixel-perfect UI fidelity using comprehensive end-to-end testing architectures.

These documents act as the source of truth for architectural decisions, technical implementation details, task tracking, and milestone completion.

---

## License

Licensed under either of

- Apache License, Version 2.0 ([LICENSE-APACHE](LICENSE-APACHE) or <https://www.apache.org/licenses/LICENSE-2.0>)
- MIT license ([LICENSE-MIT](LICENSE-MIT) or <https://opensource.org/licenses/MIT>)

at your option.

### Contribution

Unless you explicitly state otherwise, any contribution intentionally submitted
for inclusion in the work by you, as defined in the Apache-2.0 license, shall be
dual licensed as above, without any additional terms or conditions.
