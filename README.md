gh-clone-spec
=============

[![License](https://img.shields.io/badge/license-Apache--2.0%20OR%20MIT-blue.svg)](https://opensource.org/licenses/Apache-2.0)

The goal of this project is a precise replication of GitHub. Including its API, and an Angular (TypeScript+SCSS+HTML) implementation of its frontend.

Pointing any official GitHub project to a hosted version of this project should work out of the box with no changes. E.g., [`gh`](https://cli.github.com) CLI tool.

---

## Architecture & Deployment

`gh-clone` is built with unparalleled deployment flexibility (see [`ARCHITECTURE.md`](ARCHITECTURE.md) for deep technical details):

- **Single Binary (Like Gitea):** For small teams or personal use, `gh-clone` can be run as a standalone, zero-dependency single binary. This "single node" mode includes everything needed (database, web server, git backend) for an instant, frictionless setup.
- **Enterprise Scale:** The exact same core logic scales up to massive enterprise workloads. It can be deployed as a highly-available, distributed architecture across Kubernetes clusters, seamlessly handling millions of repositories and concurrent users.
- **Cross-Platform:** True to its flexible nature, `gh-clone` provides first-class support and runs flawlessly natively on **Linux**, **macOS**, and **Windows**.

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

The master plan is divided into 8 core repository-specific plans, each tackling a critical dimension of the project:

| Repository Plan | Description | Completed | Total | Percentage |
| :--- | :--- | :--- | :--- | :--- |
| **[gh-clone-backend](plans/PLAN_gh-clone-backend.md)** | Core API application server, orchestrator, and single-binary entrypoint. | 0 | 406 | 0.0% |
| **[gh-clone-angular-frontend](plans/PLAN_gh-clone-angular-frontend.md)** | 100% feature-parity UI, SSR, and real-time streaming components. | 0 | 135 | 0.0% |
| **[gh-clone-openapi-compiler](plans/PLAN_gh-clone-openapi-compiler.md)** | AST compiler generating the Rust API routing/types from the OpenAPI spec. | 0 | 94 | 0.0% |
| **[gh-clone-spokes](plans/PLAN_gh-clone-spokes.md)** | Distributed Git storage, RPC layer, and replica consensus engine. | 0 | 165 | 0.0% |
| **[gh-clone-actions-runner](plans/PLAN_gh-clone-actions-runner.md)** | CI/CD execution engine utilizing isolated sandboxes and containers. | 0 | 97 | 0.0% |
| **[gh-clone-search](plans/PLAN_gh-clone-search.md)** | High-performance code search indexer ("Blackbird" clone). | 0 | 19 | 0.0% |
| **[gh-clone-authz](plans/PLAN_gh-clone-authz.md)** | Zanzibar-inspired high-speed authorization and permissions graph. | 0 | 19 | 0.0% |
| **[gh-clone-infrastructure](plans/PLAN_gh-clone-infrastructure.md)** | Deployment automation, Helm charts, and appliance builder (AMIs/VHDs). | 0 | 51 | 0.0% |
| **Total** | | **0** | **986** | **0.0%** |

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
