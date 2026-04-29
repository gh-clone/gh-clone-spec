# Architecture Overview

This document outlines the architecture for the 100% feature-parity replica of GitHub. The system is designed to support both a highly scalable, globally distributed enterprise cloud model and a zero-configuration, single-binary deployment.

Per the organization's standards, this project explicitly avoids a monorepo design. The architecture is decomposed into focused, loosely coupled repositories managed under the `gh-clone` GitHub organization. 

**Core Stack Constraints:**
- **Backend:** Rust (Axum, sqlx, Tokio)
- **Frontend:** Angular 21 (Zoneless, SSR, signals)

---

## 1. Organization Repositories (`https://github.com/gh-clone`)

The ecosystem is distributed across several targeted repositories.

### `gh-clone-angular-frontend`
The Angular 21 user interface. 
- **Tech:** Angular 21, Zoneless change detection (`provideExperimentalZonelessChangeDetection`), SSR, RxJS, Primer Design System.
- **Role:** Handles hydration, optimistic UI mutations via NgRx SignalStore, Monaco editor integration within Web Workers, and real-time streaming via Server-Sent Events (SSE). 
- **Compilation:** Builds static assets compressed via Brotli/Gzip, which are either served via a CDN or embedded directly into the Rust backend for the single-binary deployment.

### `gh-clone-backend`
The central API application server and orchestrator.
- **Tech:** Rust, Axum, sqlx, Tokio.
- **Role:** Strict implementation of Hexagonal Architecture. It houses the application services, domain models, and HTTP routes. Handles user sessions, PR logic, webhooks, and connects to the database (PostgreSQL/SQLite) and cache (Redis/Memory).

### `gh-clone-openapi-compiler`
The API specification compiler.
- **Tech:** Rust, petgraph, jsonschema.
- **Role:** Reads the massive `api.github.com.yaml` specification and dynamically generates strictly-typed Rust structs, Serde deserializers, and Axum routing code. It ensures complete API parity without writing manual boilerplate.

### `gh-clone-spokes` (Git Data Plane)
The distributed Git storage and RPC layer.
- **Tech:** Rust, gRPC, `gitoxide`, `libgit2`.
- **Role:** Implements the Smart HTTP Git protocol, SSH multiplexing, and distributed quorum commits. In Enterprise mode, it uses a Raft state machine for 3-node replica consensus. Houses the `pre-receive`, `update`, and `post-receive` hooks.

### `gh-clone-actions-runner`
The CI/CD runner and execution engine.
- **Tech:** Rust or .NET, `libmountsandbox`, Docker daemon.
- **Role:** Provides an isolated environment (Firecracker MicroVMs, Windows Job Objects, or Docker containers) to execute CI pipelines securely. Handles real-time log streaming back to the API.

### `gh-clone-search`
The high-performance code search indexer ("Blackbird" clone).
- **Tech:** Rust, `tree-sitter`, Hyperscan, RocksDB.
- **Role:** Consumes Kafka push events, extracts semantic symbols using `tree-sitter`, generates n-grams, and provides sub-millisecond regex code search capabilities.

### `gh-clone-authz`
The Zanzibar-inspired authorization graph.
- **Tech:** Rust, gRPC.
- **Role:** Manages the intricate web of permissions (e.g., "does User X have push access to Repo Y via Team Z?"). Implements the `Check`, `Read`, and `Expand` APIs with heavy SIMD and caching optimizations.

### `gh-clone-infrastructure`
Deployment automation and appliance generation.
- **Tech:** Terraform, Kubernetes Operator, Packer, Ansible.
- **Role:** Responsible for building the Amazon AMIs, Azure VHDs, VMware OVAs, and KVM disks. Contains Helm charts for the Enterprise distributed deployment.

---

## 2. Subsystem Integrations & Moving Parts

### 2.1 Storage & Data Tiers
- **Relational Databases:** 
  - *Enterprise:* PostgreSQL 16+ partitioned by time (audit logs) and tenant (webhooks) using Row-Level Security (RLS) and logical replication.
  - *Single-Binary:* SQLite utilizing `journal_mode=WAL` and `mmap_size`, embedded in-process.
- **Cache & Concurrency:**
  - *Enterprise:* Redis Cluster running GCRA rate limiters and Redlock.
  - *Single-Binary:* In-memory `moka` cache and `DashMap`.
- **Message Broker:**
  - *Enterprise:* Kafka/Redpanda utilizing Protobuf schemas for reliable outbox events, CI dispatch, and search indexing.
  - *Single-Binary:* In-memory `tokio::sync::mpsc` channels.
- **Blob & Git LFS Storage:**
  - MinIO / AWS S3 via pre-signed URLs for LFS uploads and CI artifact storage.

### 2.2 The Generation Compiler
The backend does not manually define routes. `gh-clone-openapi-compiler` converts the GitHub OpenAPI 3.1 YAML spec into a graph, resolves cyclic references by injecting `Box<T>`, and emits pure Rust with strict `axum` extractors.

### 2.3 Angular 21 UI 
Provides an identical replica of GitHub's visual interface.
- Utilizes an Event Replay buffering strategy and TransferState payloads for zero-CLS hydration.
- The Monaco code editor and Tree-Sitter WASM parsers run strictly inside Web Workers.
- Implements Workbox Service Workers and IndexedDB (`Dexie.js`) for offline sync and background replay of notifications and stars.

### 2.4 Computational Code Search
Built around a Kafka consumer tailing push events. It parses files into ASTs using `tree-sitter`, stores structural trigrams in RocksDB, and processes global queries via finite state transducers before dropping to raw `hyperscan` regex matching.

---

## 3. Deployment Architectures

This architecture is designed to scale from a Raspberry Pi up to a global cloud via dynamic infrastructure adapters.

### 3.1 The "Gitea Killer" (Single Binary Embedded Mode)
Designed to be an incredibly portable, zero-configuration executable that requires no external dependencies—aimed squarely at replacing Gitea and GitLab instances for small teams.

**How it works:**
1. During the build process of `gh-clone-backend`, a `build.rs` script triggers the Angular 21 build in `gh-clone-angular-frontend`.
2. The compressed (Brotli/Gzip) Angular assets are embedded directly into the Rust binary utilizing `rust-embed`.
3. At runtime, the binary binds to port 80/443. 
4. **Data:** It initializes an embedded high-performance SQLite database (WAL mode) and stores Git repositories directly on the local filesystem.
5. **Services:** It bypasses Redis and Kafka entirely, using in-memory `DashMap` for sessions, `moka` for caching, and `tokio::sync` channels for background webhook delivery.
6. **SSH:** It spins up a pure-Rust embedded SSH server bound to port 22 to natively handle `git push`/`git pull` without requiring host `sshd` integration.

### 3.2 The Enterprise Cloud (Distributed Microservices Mode)
Designed for extreme scale (10,000+ requests per second) and High Availability.

**How it works:**
1. **API Tier:** Stateless Rust backend instances run as Kubernetes Deployments, auto-scaling based on CPU/Queue metrics.
2. **Git Data Plane (Spokes):** Traffic is multiplexed via `babeld` proxy to stateful storage nodes. Every repository has 1 Primary and 2 Secondaries, synchronized via Raft.
3. **Data:** Backed by Vitess/PostgreSQL and Redis clusters.
4. **Asynchronous Jobs:** Managed via Kafka. `gh-clone-actions-runner` agents connect via gRPC to pull CI jobs off the distributed queues.
5. **Observability:** Strict OpenTelemetry trace propagation from Nginx ingress -> Axum Middleware -> Postgres -> Kafka -> Spokes gRPC.
