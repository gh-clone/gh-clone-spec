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
| [`gh-clone-actions-runner`](https://github.com/gh-clone/gh-clone-actions-runner) | CI/CD execution engine utilizing isolated sandboxes and containers. | Rust / .NET | `gh-clone-backend` |
| [`gh-clone-angular-frontend`](https://github.com/gh-clone/gh-clone-angular-frontend) | 100% feature-parity UI, SSR, and real-time streaming components. | Angular 21, RxJS | None |
| [`gh-clone-authz`](https://github.com/gh-clone/gh-clone-authz) | Zanzibar-inspired high-speed authorization and permissions graph. | Rust, gRPC | None |
| [`gh-clone-backend`](https://github.com/gh-clone/gh-clone-backend) | Core API application server, orchestrator, and single-binary entrypoint. | Rust, Axum, sqlx | `gh-clone-openapi-compiler`, `gh-clone-angular-frontend` |
| [`gh-clone-billing`](https://github.com/gh-clone/gh-clone-billing) | Stripe integrations, metering, enterprise SCIM, and Sponsors payouts. | Rust, Stripe API | `gh-clone-backend` |
| [`gh-clone-cdn-raw`](https://github.com/gh-clone/gh-clone-cdn-raw) | High-speed edge routing proxy for raw content, avatars, and strictly isolated domains. | Rust, OpenResty | None |
| [`gh-clone-checks`](https://github.com/gh-clone/gh-clone-checks) | Check Suites, rich log streaming, and line-level code annotations. | Rust, Angular | `gh-clone-backend` |
| [`gh-clone-classroom`](https://github.com/gh-clone/gh-clone-classroom) | LTI 1.3 LMS integrations, student rosters, and autograding pipelines. | Rust, Angular | `gh-clone-backend` |
| [`gh-clone-codespaces`](https://github.com/gh-clone/gh-clone-codespaces) | Cloud-based development environment orchestrator parsing devcontainers into remote compute. | Rust, K8s | `gh-clone-backend` |
| [`gh-clone-copilot`](https://github.com/gh-clone/gh-clone-copilot) | High-performance AI gateway for LLM integrations, prompt proxying, and chat capabilities. | Rust | `gh-clone-backend` |
| [`gh-clone-deployments`](https://github.com/gh-clone/gh-clone-deployments) | CI/CD environment protections, release gating, and deployment tracking mechanisms. | Rust | `gh-clone-backend` |
| [`gh-clone-desktop`](https://github.com/gh-clone/gh-clone-desktop) | Native desktop application built with Tauri V2 and Angular for managing local repositories. | Rust, Tauri, Angular | None |
| [`gh-clone-discussions`](https://github.com/gh-clone/gh-clone-discussions) | GraphQL API for threaded Q&A and Redis-backed upvoting. | Rust, Redis | `gh-clone-backend` |
| [`gh-clone-docs`](https://github.com/gh-clone/gh-clone-docs) | Robust documentation portal utilizing AnalogJS/Angular SSR with file-system routing. | AnalogJS, Angular, Rust | None |
| [`gh-clone-enterprise-management`](https://github.com/gh-clone/gh-clone-enterprise-management) | Centralized administration, compliance mapping, and multi-organization hierarchy engine. | Rust, Angular | `gh-clone-backend` |
| [`gh-clone-gists`](https://github.com/gh-clone/gh-clone-gists) | Headless Git storage for snippets and Monaco editor UI. | Rust, libgit2 | `gh-clone-spokes` |
| [`gh-clone-graphql-engine`](https://github.com/gh-clone/gh-clone-graphql-engine) | Dedicated async-graphql server providing complete compatibility with the v4 schema. | Rust, async-graphql | `gh-clone-backend` |
| [`gh-clone-ide-extensions`](https://github.com/gh-clone/gh-clone-ide-extensions) | VS Code and IDE integrations bridging local development with the clone ecosystem. | TypeScript, VS Code | `gh-clone-backend` |
| [`gh-clone-infrastructure`](https://github.com/gh-clone/gh-clone-infrastructure) | Deployment automation, Helm charts, and appliance builder (AMIs/VHDs). | Terraform, K8s, Packer | All repositories |
| [`gh-clone-marketing`](https://github.com/gh-clone/gh-clone-marketing) | High-performance public-facing landing pages, enterprise sales portals, and SEO assets. | Angular, Rust | None |
| [`gh-clone-migrations`](https://github.com/gh-clone/gh-clone-migrations) | High-fidelity archives, and GitLab/Bitbucket/SVN importer pipelines. | Rust, Angular | `gh-clone-spokes` |
| [`gh-clone-mobile`](https://github.com/gh-clone/gh-clone-mobile) | Native iOS and Android application shell with swipe-to-review PR flows. | Ionic, Angular | `gh-clone-backend` |
| [`gh-clone-models`](https://github.com/gh-clone/gh-clone-models) | AI model registry managing inference providers, token billing, and model parameters. | Rust | `gh-clone-backend` |
| [`gh-clone-moderation`](https://github.com/gh-clone/gh-clone-moderation) | Trust & Safety, automated spam heuristics, CSEM scanning, and DMCA holds. | Rust, Angular | `gh-clone-backend` |
| [`gh-clone-notifications`](https://github.com/gh-clone/gh-clone-notifications) | Real-time webhooks, email dispatch, and user inbox management. | Rust | `gh-clone-backend` |
| [`gh-clone-openapi-compiler`](https://github.com/gh-clone/gh-clone-openapi-compiler) | AST compiler generating the Rust API routing/types from the OpenAPI spec. | Rust, petgraph | None |
| [`gh-clone-packages`](https://github.com/gh-clone/gh-clone-packages) | Native registry protocol emulation (npm, Maven, Docker v2). | Rust | `gh-clone-backend` |
| [`gh-clone-pages`](https://github.com/gh-clone/gh-clone-pages) | Distributed edge routing and CI deployment pipelines for static sites. | OpenResty, Rust | `gh-clone-infrastructure` |
| [`gh-clone-projects`](https://github.com/gh-clone/gh-clone-projects) | Real-time reactive Memex Kanban/Spreadsheet engine. | Rust, yjs, React | `gh-clone-backend` |
| [`gh-clone-search`](https://github.com/gh-clone/gh-clone-search) | High-performance code search indexer ("Blackbird" clone). | Rust, Hyperscan | `gh-clone-backend` |
| [`gh-clone-security`](https://github.com/gh-clone/gh-clone-security) | CodeQL SARIF ingestion, Dependabot, and Push Protection engines. | Rust, Hyperscan | `gh-clone-backend` |
| [`gh-clone-spokes`](https://github.com/gh-clone/gh-clone-spokes) | Distributed Git storage, RPC layer, and replica consensus engine. | Rust, gRPC, gitoxide | `gh-clone-authz` |
| [`gh-clone-status`](https://github.com/gh-clone/gh-clone-status) | Physically isolated incident management dashboard and TimescaleDB metrics API. | Rust, TimescaleDB, Angular | None |
| [`gh-clone-web-ide`](https://github.com/gh-clone/gh-clone-web-ide) | In-browser VS Code experience (github.dev) via virtual FileSystem API. | TypeScript, Rust | `gh-clone-backend` |
| [`gh-clone-wikis`](https://github.com/gh-clone/gh-clone-wikis) | Gollum-compatible backend and wiki rendering pipeline. | Rust, libgit2 | `gh-clone-spokes` |


---

## The Master Blueprint (`/plans`)

This project is governed by a highly rigorous, multi-phased engineering blueprint located in the [`plans/`](plans) directory. This blueprint details the systematic approach required to achieve our goal of 100% feature parity with the GitHub ecosystem, from foundational architecture to exhaustive quality assurance. 

The master plan is divided into 35 core repository-specific plans, each tackling a critical dimension of the project:

| Component / Original Project | Description | Tasks | Status |
| :--- | :--- | :--- | :--- |
| **gh-clone-actions-runner**<br>[Repository](https://github.com/gh-clone/gh-clone-actions-runner) • [Tasks](plans/PLAN_gh-clone-actions-runner.md) | CI/CD execution engine utilizing isolated sandboxes and containers. | 0/154 | ⏳ 0.0% |
| **gh-clone-angular-frontend**<br>[Repository](https://github.com/gh-clone/gh-clone-angular-frontend) • [Tasks](plans/PLAN_gh-clone-angular-frontend.md) | 100% feature-parity UI, SSR, and real-time streaming components. | 0/330 | ⏳ 0.0% |
| **gh-clone-authz**<br>[Repository](https://github.com/gh-clone/gh-clone-authz) • [Tasks](plans/PLAN_gh-clone-authz.md) | Zanzibar-inspired high-speed authorization and permissions graph. | 0/29 | ⏳ 0.0% |
| **gh-clone-backend**<br>[Repository](https://github.com/gh-clone/gh-clone-backend) • [Tasks](plans/PLAN_gh-clone-backend.md) | Core API application server, orchestrator, and single-binary entrypoint. | 0/497 | ⏳ 0.0% |
| **gh-clone-billing**<br>[Repository](https://github.com/gh-clone/gh-clone-billing) • [Tasks](plans/PLAN_gh-clone-billing.md) | Stripe integrations, metering, enterprise SCIM, and Sponsors payouts. | 0/50 | ⏳ 0.0% |
| **gh-clone-cdn-raw**<br>[Repository](https://github.com/gh-clone/gh-clone-cdn-raw) • [Tasks](plans/PLAN_gh-clone-cdn-raw.md) | High-speed edge routing proxy for raw content, avatars, and strictly isolated domains. | 0/40 | ⏳ 0.0% |
| **gh-clone-checks**<br>[Repository](https://github.com/gh-clone/gh-clone-checks) • [Tasks](plans/PLAN_gh-clone-checks.md) | Check Suites, rich log streaming, and line-level code annotations. | 0/37 | ⏳ 0.0% |
| **gh-clone-classroom**<br>[Repository](https://github.com/gh-clone/gh-clone-classroom) • [Tasks](plans/PLAN_gh-clone-classroom.md) | LTI 1.3 LMS integrations, student rosters, and autograding pipelines. | 0/29 | ⏳ 0.0% |
| **gh-clone-codespaces**<br>[Repository](https://github.com/gh-clone/gh-clone-codespaces) • [Tasks](plans/PLAN_gh-clone-codespaces.md) | Cloud-based development environment orchestrator parsing devcontainers into remote compute. | 0/50 | ⏳ 0.0% |
| **gh-clone-copilot**<br>[Repository](https://github.com/gh-clone/gh-clone-copilot) • [Tasks](plans/PLAN_gh-clone-copilot.md) | High-performance AI gateway for LLM integrations, prompt proxying, and chat capabilities. | 0/68 | ⏳ 0.0% |
| **gh-clone-deployments**<br>[Repository](https://github.com/gh-clone/gh-clone-deployments) • [Tasks](plans/PLAN_gh-clone-deployments.md) | CI/CD environment protections, release gating, and deployment tracking mechanisms. | 0/56 | ⏳ 0.0% |
| **gh-clone-desktop**<br>[Repository](https://github.com/gh-clone/gh-clone-desktop) • [Tasks](plans/PLAN_gh-clone-desktop.md) | Native desktop application built with Tauri V2 and Angular for managing local repositories. | 0/76 | ⏳ 0.0% |
| **gh-clone-discussions**<br>[Repository](https://github.com/gh-clone/gh-clone-discussions) • [Tasks](plans/PLAN_gh-clone-discussions.md) | GraphQL API for threaded Q&A and Redis-backed upvoting. | 0/34 | ⏳ 0.0% |
| **gh-clone-docs**<br>[Repository](https://github.com/gh-clone/gh-clone-docs) • [Tasks](plans/PLAN_gh-clone-docs.md) | Robust documentation portal utilizing AnalogJS/Angular SSR with file-system routing. | 0/70 | ⏳ 0.0% |
| **gh-clone-enterprise-management**<br>[Repository](https://github.com/gh-clone/gh-clone-enterprise-management) • [Tasks](plans/PLAN_gh-clone-enterprise-management.md) | Centralized administration, compliance mapping, and multi-organization hierarchy engine. | 0/69 | ⏳ 0.0% |
| **gh-clone-gists**<br>[Repository](https://github.com/gh-clone/gh-clone-gists) • [Tasks](plans/PLAN_gh-clone-gists.md) | Headless Git storage for snippets and Monaco editor UI. | 0/27 | ⏳ 0.0% |
| **gh-clone-graphql-engine**<br>[Repository](https://github.com/gh-clone/gh-clone-graphql-engine) • [Tasks](plans/PLAN_gh-clone-graphql-engine.md) | Dedicated async-graphql server providing complete compatibility with the v4 schema. | 0/40 | ⏳ 0.0% |
| **gh-clone-ide-extensions**<br>[Repository](https://github.com/gh-clone/gh-clone-ide-extensions) • [Tasks](plans/PLAN_gh-clone-ide-extensions.md) | VS Code and IDE integrations bridging local development with the clone ecosystem. | 0/74 | ⏳ 0.0% |
| **gh-clone-infrastructure**<br>[Repository](https://github.com/gh-clone/gh-clone-infrastructure) • [Tasks](plans/PLAN_gh-clone-infrastructure.md) | Deployment automation, Helm charts, and appliance builder (AMIs/VHDs). | 0/55 | ⏳ 0.0% |
| **gh-clone-marketing**<br>[Repository](https://github.com/gh-clone/gh-clone-marketing) • [Tasks](plans/PLAN_gh-clone-marketing.md) | High-performance public-facing landing pages, enterprise sales portals, and SEO assets. | 0/77 | ⏳ 0.0% |
| **gh-clone-migrations**<br>[Repository](https://github.com/gh-clone/gh-clone-migrations) • [Tasks](plans/PLAN_gh-clone-migrations.md) | High-fidelity archives, and GitLab/Bitbucket/SVN importer pipelines. | 0/49 | ⏳ 0.0% |
| **gh-clone-mobile**<br>[Repository](https://github.com/gh-clone/gh-clone-mobile) • [Tasks](plans/PLAN_gh-clone-mobile.md) | Native iOS and Android application shell with swipe-to-review PR flows. | 0/37 | ⏳ 0.0% |
| **gh-clone-models**<br>[Repository](https://github.com/gh-clone/gh-clone-models) • [Tasks](plans/PLAN_gh-clone-models.md) | AI model registry managing inference providers, token billing, and model parameters. | 0/56 | ⏳ 0.0% |
| **gh-clone-moderation**<br>[Repository](https://github.com/gh-clone/gh-clone-moderation) • [Tasks](plans/PLAN_gh-clone-moderation.md) | Trust & Safety, automated spam heuristics, CSEM scanning, and DMCA holds. | 0/32 | ⏳ 0.0% |
| **gh-clone-notifications**<br>[Repository](https://github.com/gh-clone/gh-clone-notifications) • [Tasks](plans/PLAN_gh-clone-notifications.md) | Real-time webhooks, email dispatch, and user inbox management. | 0/35 | ⏳ 0.0% |
| **gh-clone-openapi-compiler**<br>[Repository](https://github.com/gh-clone/gh-clone-openapi-compiler) • [Tasks](plans/PLAN_gh-clone-openapi-compiler.md) | AST compiler generating the Rust API routing/types from the OpenAPI spec. | 0/94 | ⏳ 0.0% |
| **gh-clone-packages**<br>[Repository](https://github.com/gh-clone/gh-clone-packages) • [Tasks](plans/PLAN_gh-clone-packages.md) | Native registry protocol emulation (npm, Maven, Docker v2). | 0/46 | ⏳ 0.0% |
| **gh-clone-pages**<br>[Repository](https://github.com/gh-clone/gh-clone-pages) • [Tasks](plans/PLAN_gh-clone-pages.md) | Distributed edge routing and CI deployment pipelines for static sites. | 0/36 | ⏳ 0.0% |
| **gh-clone-projects**<br>[Repository](https://github.com/gh-clone/gh-clone-projects) • [Tasks](plans/PLAN_gh-clone-projects.md) | Real-time reactive Memex Kanban/Spreadsheet engine. | 0/45 | ⏳ 0.0% |
| **gh-clone-search**<br>[Repository](https://github.com/gh-clone/gh-clone-search) • [Tasks](plans/PLAN_gh-clone-search.md) | High-performance code search indexer ("Blackbird" clone). | 0/26 | ⏳ 0.0% |
| **gh-clone-security**<br>[Repository](https://github.com/gh-clone/gh-clone-security) • [Tasks](plans/PLAN_gh-clone-security.md) | CodeQL SARIF ingestion, Dependabot, and Push Protection engines. | 0/88 | ⏳ 0.0% |
| **gh-clone-spokes**<br>[Repository](https://github.com/gh-clone/gh-clone-spokes) • [Tasks](plans/PLAN_gh-clone-spokes.md) | Distributed Git storage, RPC layer, and replica consensus engine. | 0/214 | ⏳ 0.0% |
| **gh-clone-status**<br>[Repository](https://github.com/gh-clone/gh-clone-status) • [Tasks](plans/PLAN_gh-clone-status.md) | Physically isolated incident management dashboard and TimescaleDB metrics API. | 0/63 | ⏳ 0.0% |
| **gh-clone-web-ide**<br>[Repository](https://github.com/gh-clone/gh-clone-web-ide) • [Tasks](plans/PLAN_gh-clone-web-ide.md) | In-browser VS Code experience (github.dev) via virtual FileSystem API. | 0/48 | ⏳ 0.0% |
| **gh-clone-wikis**<br>[Repository](https://github.com/gh-clone/gh-clone-wikis) • [Tasks](plans/PLAN_gh-clone-wikis.md) | Gollum-compatible backend and wiki rendering pipeline. | 0/27 | ⏳ 0.0% |
| **Total** | | **0/2758** | **⏳ 0.0%** |

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
