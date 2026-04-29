# Plan for gh-clone-backend

# THE ULTIMATE BLUEPRINT: 100% FEATURE-PARITY GITHUB REPLICA
**Version:** 8.1 (The Definitive Engineering Whitepaper & Specification)
**Objective:** Architect, implement, and deploy a 100% feature-parity clone of GitHub. This system must act as an exact drop-in replacement for the GitHub ecosystem (`gh` CLI, Actions, APIs, IDE integrations). The architecture guarantees **Universal Portability**: scaling from a statically-compiled, zero-config single binary on a Raspberry Pi up to a globally distributed Kubernetes cluster handling 10,000+ concurrent requests per second.

---

## Phase 0: Foundation, Architecture, & Environment Setup

### 0.1 Codebase Initialization & Dev Environment
- [ ] Initialize `Cargo.toml` specifying Rust 2021 edition.
- [ ] Define module path as `replica-core`.
- [ ] Create `cmd/replica/main.rs` as the primary single-binary entrypoint.
- [ ] Configure `clippy` for security static analysis.
- [ ] Configure `clippy` to enforce unhandled errors.
- [ ] Configure `clippy` for general linting rules.
- [ ] Create `.editorconfig` with `indent_style = space` and `indent_size = 4` for Rust files.
- [ ] Write `Makefile` target `build` executing `cargo build --release`.
- [ ] Write `Makefile` target `test` executing `cargo test`.
- [ ] Write `Makefile` target `lint` executing `cargo clippy`.
- [ ] Write `Makefile` target `clean` to remove `dist/` and `tmp/` directories.
- [ ] Setup `air` (`.air.toml`) for hot-reloading during local development.
- [ ] Create `.dockerignore` excluding `.git`, `tmp`, and local database files.
- [ ] Create multi-stage `Dockerfile`: `builder` stage using `rust:1.76-alpine`.
- [ ] Create multi-stage `Dockerfile`: `final` stage using `scratch` (empty filesystem).
- [ ] Configure Rust target features for SQLite native bindings.
- [ ] Setup static asset embedding via `rust-embed` for the `web/public` directory.
- [ ] Initialize `internal/config` package to hold singleton struct `ServerConfig`.
- [ ] Integrate `figment or config` to parse configuration from `replica.yaml`.
- [ ] Map environment variable prefix `REPLICA_` to all Viper configuration keys.
- [ ] Create struct field `Config.Database.DSN` for connection string.
- [ ] Create struct field `Config.Server.Port` defaulting to `8080`.
- [ ] Create struct field `Config.Git.BinPath` defaulting to `/usr/bin/git`.
- [ ] Create struct field `Config.Security.SecretKey` for HMAC signing.
- [ ] Create standard `.gitignore` targeting standard Rust, OS, and IDE artifacts.
- [ ] Write `cmd/replica/cmd_serve.go` using `clap` to start the HTTP server.
- [ ] Write `cmd/replica/cmd_migrate.rs` for running database migrations manually.
- [ ] Write `cmd/replica/cmd_doctor.rs` to validate system dependencies (Git binary, SSH keys).
- [ ] Write `cmd/replica/cmd_admin_create.rs` to bootstrap the first root user.
- [ ] Setup pre-commit hooks natively via `.git/hooks/pre-commit` to run `cargo clippy`.
- [ ] Create `scripts/seed.rs` to insert 100 fake users and repositories into the local DB.
- [ ] Setup `dependabot.yml` for automated Rust cargo dependency updates.
- [ ] Create `.github/workflows/ci.yml` to trigger `cargo test` on PRs.
- [ ] Create `.github/workflows/build.yml` to test cross-compilation (linux/amd64, linux/arm64, darwin/arm64).
- [ ] Write test helper `testutil.NewDB()` to spin up an in-memory SQLite DB for unit testing.
- [ ] Write test helper `testutil.NewTempRepo()` to initialize an isolated bare git repo per test.

### 0.2 Database Architecture & Schema Definitions (V1 Migrations)
- [ ] Initialize `sqlx-cli` library for executing `.sql` files.
- [ ] Write migration `0001_create_users_table.up.sql`.
- [ ] Define `users.id` as `BIGSERIAL PRIMARY KEY`.
- [ ] Define `users.login` as `VARCHAR(39) UNIQUE NOT NULL` matching GitHub username constraints.
- [ ] Define `users.email` as `VARCHAR(255) UNIQUE NOT NULL`.
- [ ] Define `users.password_hash` as `VARCHAR(255)`.
- [ ] Define `users.type` as `VARCHAR(10)` (ENUM: 'User', 'Organization', 'Bot').
- [ ] Define `users.created_at` as `TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP`.
- [ ] Define `users.updated_at` as `TIMESTAMP WITH TIME ZONE`.
- [ ] Write migration `0002_create_repositories_table.up.sql`.
- [ ] Define `repositories.id` as `BIGSERIAL PRIMARY KEY`.
- [ ] Define `repositories.owner_id` as `BIGINT REFERENCES users(id) ON DELETE CASCADE`.
- [ ] Define `repositories.name` as `VARCHAR(100) NOT NULL`.
- [ ] Create unique composite index on `(owner_id, name)` for repositories.
- [ ] Define `repositories.description` as `TEXT`.
- [ ] Define `repositories.default_branch` as `VARCHAR(255) DEFAULT 'main'`.
- [ ] Define `repositories.is_private` as `BOOLEAN DEFAULT false`.
- [ ] Define `repositories.is_fork` as `BOOLEAN DEFAULT false`.
- [ ] Define `repositories.fork_id` as `BIGINT REFERENCES repositories(id) NULL`.
- [ ] Write migration `0003_create_ssh_keys_table.up.sql`.
- [ ] Define `ssh_keys.id` as `BIGSERIAL PRIMARY KEY`.
- [ ] Define `ssh_keys.user_id` as `BIGINT REFERENCES users(id) ON DELETE CASCADE`.
- [ ] Define `ssh_keys.fingerprint` as `VARCHAR(255) UNIQUE NOT NULL`.
- [ ] Define `ssh_keys.content` as `TEXT NOT NULL`.
- [ ] Write migration `0004_create_access_tokens_table.up.sql`.
- [ ] Define `access_tokens.id` as `BIGSERIAL PRIMARY KEY`.
- [ ] Define `access_tokens.user_id` as `BIGINT REFERENCES users(id) ON DELETE CASCADE`.
- [ ] Define `access_tokens.token_hash` as `VARCHAR(64) UNIQUE NOT NULL`.
- [ ] Define `access_tokens.scopes` as `TEXT[]` (Array of strings).
- [ ] Define `access_tokens.expires_at` as `TIMESTAMP WITH TIME ZONE`.
- [ ] Write migration `0005_create_issues_table.up.sql`.
- [ ] Define `issues.id` as `BIGSERIAL PRIMARY KEY`.
- [ ] Define `issues.repo_id` as `BIGINT REFERENCES repositories(id) ON DELETE CASCADE`.
- [ ] Define `issues.number` as `INTEGER NOT NULL`.
- [ ] Create unique composite index on `(repo_id, number)` for issues.
- [ ] Define `issues.author_id` as `BIGINT REFERENCES users(id)`.
- [ ] Define `issues.title` as `VARCHAR(255) NOT NULL`.
- [ ] Define `issues.body` as `TEXT`.
- [ ] Define `issues.state` as `VARCHAR(20) DEFAULT 'open'` (ENUM: 'open', 'closed').
- [ ] Write migration `0006_create_pull_requests_table.up.sql`.
- [ ] Define `pull_requests.issue_id` as `BIGINT REFERENCES issues(id) PRIMARY KEY` (1:1 with issue).
- [ ] Define `pull_requests.head_repo_id` as `BIGINT REFERENCES repositories(id)`.
- [ ] Define `pull_requests.head_branch` as `VARCHAR(255) NOT NULL`.
- [ ] Define `pull_requests.base_repo_id` as `BIGINT REFERENCES repositories(id)`.
- [ ] Define `pull_requests.base_branch` as `VARCHAR(255) NOT NULL`.
- [ ] Write migration `0007_create_stars_table.up.sql`.
- [ ] Define `stars.user_id` as `BIGINT REFERENCES users(id)`.
- [ ] Define `stars.repo_id` as `BIGINT REFERENCES repositories(id)`.
- [ ] Create unique composite index on `(user_id, repo_id)` for stars.
- [ ] Create abstract `db.Driver` interface to support multiple SQL dialects.
- [ ] Implement `db.SQLiteDriver` utilizing `sqlx`.
- [ ] Configure SQLite PRAGMA `journal_mode=WAL` for high concurrent performance.
- [ ] Configure SQLite PRAGMA `synchronous=NORMAL` for optimized disk I/O.
- [ ] Configure SQLite PRAGMA `foreign_keys=ON` to enforce relations.
- [ ] Implement `db.PostgresDriver` utilizing `sqlx`.
- [ ] Set Postgres connection pool `MaxConns` to 50 for clustered deployments.

### 0.3 Core API HTTP Server & Middleware Strict Parity
- [ ] Initialize `axum::Router` router for strict, zero-allocation path routing.
- [ ] Configure core HTTP server `ReadTimeout` to `10s` to prevent slow-loris attacks.
- [ ] Configure core HTTP server `WriteTimeout` to `30s` to cap maximum response duration.
- [ ] Configure core HTTP server `IdleTimeout` to `120s` for keep-alive connections.
- [ ] Implement `NotFound` handler returning exactly `{"message": "Not Found", "documentation_url": "https://docs.github.com/rest"}`.
- [ ] Implement `MethodNotAllowed` handler returning `{"message": "Not Found"}` (GitHub masks 405s as 404s for security).
- [ ] Write `RequestIDMiddleware` that injects a UUIDv4 into `X-GitHub-Request-Id` response header.
- [ ] Write `TimezoneMiddleware` parsing the `Time-Zone` HTTP header and attaching location data to request context.
- [ ] Write `VersionMiddleware` requiring `X-GitHub-Api-Version: 2022-11-28` for explicit versioned endpoints.
- [ ] Write `ContentNegotiationMiddleware` enforcing `Accept: application/vnd.github+json`.
- [ ] Write `CORSMiddleware` injecting `Access-Control-Allow-Origin: *`.
- [ ] Write `CORSMiddleware` injecting `Access-Control-Allow-Methods: GET, POST, PATCH, PUT, DELETE`.
- [ ] Write `CORSMiddleware` injecting `Access-Control-Expose-Headers: ETag, Link, X-GitHub-OTP, X-RateLimit-Limit, X-RateLimit-Remaining, X-RateLimit-Reset, X-OAuth-Scopes, X-Accepted-OAuth-Scopes, X-Poll-Interval`.
- [ ] Implement `PaginationMiddleware` reading `?page=` and `?per_page=` (max 100).
- [ ] Implement Link header generator constructing exact `<url>; rel="next", <url>; rel="last"` pagination formats.
- [ ] Write `RateLimitMiddleware` using an in-memory token bucket (or Redis fallback).
- [ ] Inject header `X-RateLimit-Limit` natively matching GitHub's 5000/hour authenticated limit.
- [ ] Inject header `X-RateLimit-Remaining` accurately tracking context consumption.
- [ ] Inject header `X-RateLimit-Reset` formatted as absolute Unix epoch seconds.
- [ ] Inject header `X-RateLimit-Resource` categorized as `core`, `search`, or `graphql`.
- [ ] Write `RecoveryMiddleware` that catches panics and returns a generic `500 Internal Server Error` without leaking stack traces to the client.
- [ ] Write `LoggerMiddleware` outputting structured JSON logs for every request (`method`, `path`, `status`, `duration_ms`, `ip`).
- [ ] Write `ETagMiddleware` calculating MD5 of the finalized JSON response payload.
- [ ] Implement 304 Not Modified handler verifying `If-None-Match` request headers against computed ETag.
- [ ] Implement JSON validation strictly returning `422 Unprocessable Entity` for invalid field types.
- [ ] Define standard `APIError` Rust struct matching GitHub error schema: `{ "message": string, "errors": []{ "resource": string, "field": string, "code": string } }`.
- [ ] Map all 401 Unauthorized errors to `{"message": "Bad credentials"}` string exactly.

### 0.7 Authentication, Cryptography, & Security
- [ ] Implement `crypto.HashPassword(password string) (string, error)` utilizing `bcrypt crate` with cost=12.
- [ ] Implement `crypto.VerifyPassword(hash, password string) bool`.
- [ ] Implement PAT generation: generate 40 random cryptographic bytes.
- [ ] Format generated PAT as `ghp_{base58_encoded_bytes}` to mimic exact GitHub prefix formatting.
- [ ] Hash PAT utilizing SHA256 before storing in the `access_tokens.token_hash` database column.
- [ ] Implement OAuth2 Access Token generation formatted as `gho_{base58}`.
- [ ] Implement Server-to-Server token generation formatted as `ghs_{base58}`.
- [ ] Implement Refresh token generation formatted as `ghr_{base58}`.
- [ ] Write `BasicAuthMiddleware` parser extracting standard `Authorization: Basic {base64}`.
- [ ] Write `BearerAuthMiddleware` parser extracting standard `Authorization: Bearer {token}`.
- [ ] Write token validation routing logic: if token starts with `ghp_`, query `access_tokens` table.
- [ ] Inject `*models.User` pointer into `context.Context` upon successful token validation.
- [ ] Implement `X-OAuth-Scopes` header generation based on database token scopes array.
- [ ] Implement strict scope-checking function `RequireScope(ctx, scope string)` matching GitHub's `repo`, `admin:org`, `user:email` logic.
- [ ] Implement cryptographic payload signing for Webhooks: compute HMAC SHA256 of POST body using user-provided secret.
- [ ] Inject webhook signature into `X-Hub-Signature-256` header formatted as `sha256={hex_hmac}`.
- [ ] Write AES-256-GCM encryption helper to securely encrypt sensitive database fields at rest (e.g., OAuth App client secrets, Webhook secrets).
- [ ] Implement JWT parser for GitHub App authentication validating `RS256` signatures against configured App private keys.

### 0.8 Object Modeling, Subsystem Routing, & API Base
- [ ] Define `UserSerializer` struct mapping internal user to GitHub `{ "login": "...", "id": 1, "node_id": "...", "avatar_url": "..." }`.
- [ ] Define `RepoSerializer` struct mapping internal repo to exact GitHub repository JSON object (over 100 specific fields).
- [ ] Generate base64 GraphQL `node_id` strings (e.g., `MDEwOlJlcG9zaXRvcnkxMjk2MjY5` translates to `010:Repository1296269`).
- [ ] Scaffold `GET /users/{username}` route controller calling `UserSerializer`.
- [ ] Scaffold `GET /user` (authenticated user) route controller.
- [ ] Scaffold `GET /repos/{owner}/{repo}` route controller calling `RepoSerializer`.
- [ ] Scaffold `POST /user/repos` to handle repository creation API.
- [ ] Scaffold Git LFS v2 routing: `POST /{owner}/{repo}.git/info/lfs/objects/batch`.
- [ ] Implement LFS batch payload parser checking `operation` (upload/download).
- [ ] Generate presigned URLs or local replica endpoints for LFS object transfer actions.
- [ ] Define internal Job Queue interface `queue.WorkerPool`.
- [ ] Define `queue.Task` struct: `Type`, `Payload`, `MaxRetries`.
- [ ] Implement in-memory channel-based worker pool for single-binary asynchronous jobs (e.g., webhook delivery).
- [ ] Map endpoint `GET /rate_limit` returning exact JSON limit breakdown per category.
- [ ] Scaffold the `ghcr.io` OCI registry equivalent at `GET /v2/` mimicking Docker Registry HTTP API V2.
- [ ] Return `Docker-Distribution-Api-Version: registry/2.0` on OCI base route.
- [ ] Map GitHub Actions remote runner registration endpoint `POST /api/v3/actions/runners/register`.
- [ ] Architect the search sub-router `GET /search/repositories` parsing the specific `?q=language:go+is:public` grammar syntax.
## 1. HEXAGONAL ARCHITECTURE & STATE MANAGEMENT (DEEP DIVE)

The system is strictly divided into `core-domain` (pure Rust, no I/O), `application-services` (orchestration), and `infrastructure-adapters` (I/O). Adapters are dynamically bound at startup via a custom Dependency Injection container built on `tokio::OnceCell` and `Arc<dyn Trait>`.

### 1.1 The "Gitea Killer" (Single Binary Embedded Mode)

#### 1.1.1 Database (SQLite Adapter)
**Connection & Tuning:**
- [ ] Configure `sqlx::sqlite::SqlitePoolOptions::new().max_connections(num_cpus::get() * 2)` for optimal concurrent access.
- [ ] Configure `sqlx::sqlite::SqlitePoolOptions::new().acquire_timeout(std::time::Duration::from_secs(5))`.
- [ ] Execute `PRAGMA journal_mode = WAL;` on pool initialization to bypass read-blocking-write bottlenecks.
- [ ] Execute `PRAGMA synchronous = NORMAL;` to balance ACID compliance with high throughput.
- [ ] Execute `PRAGMA mmap_size = 30000000000;` (30GB) to memory-map the primary database file.
- [ ] Execute `PRAGMA page_size = 4096;` to align with standard OS page sizes.
- [ ] Execute `PRAGMA foreign_keys = ON;` strictly at the connection level via `sqlx` after-connect hooks.
- [ ] Execute `PRAGMA temp_store = MEMORY;` to force temporary tables and indices into RAM.
- [ ] Configure `libsqlite3-sys` crate with features `bundled`, `unlock_notify`, and `v2`.

**Schema Migrations (RBAC & Users):**
- [ ] Create `users` table: `id` (INTEGER PRIMARY KEY), `login` (VARCHAR 39, UNIQUE Collation NOCASE), `email` (UNIQUE), `hashed_password` (VARCHAR 255).
- [ ] Create `users` constraints: Enforce `CHECK(length(login) BETWEEN 1 AND 39)`.
- [ ] Create `users` constraints: Enforce `CHECK(login NOT LIKE '-%' AND login NOT LIKE '%-%-%')` (GitHub username rules).
- [ ] Create `user_emails` table for multi-email support: `user_id`, `email`, `is_primary` (BOOLEAN), `is_verified` (BOOLEAN).
- [ ] Create partial index on `user_emails(user_id)` where `is_primary = TRUE` to guarantee only one primary email.
- [ ] Create `organizations` table: `id`, `login` (UNIQUE), `description`, `company`, `location`.
- [ ] Create `teams` table: `id`, `org_id` (FK), `name`, `slug` (UNIQUE per org), `description`, `privacy` (ENUM: secret, closed).
- [ ] Create `team_memberships` table: `team_id` (FK), `user_id` (FK), `role` (ENUM: maintainer, member).

**Schema Migrations (Git & Repositories):**
- [ ] Create `repositories` table: `id`, `owner_id` (FK: User or Org), `name` (VARCHAR 100), `description`.
- [ ] Create `repositories` constraints: Enforce `UNIQUE(owner_id, name)`.
- [ ] Create `repositories` flags: `is_private`, `is_fork`, `is_mirror`, `is_template`, `is_archived` (all BOOLEAN DEFAULT FALSE).
- [ ] Create `repositories` stats columns: `forks_count`, `stars_count`, `watchers_count`, `size_kb`.
- [ ] Create `repository_topics` table: `repo_id` (FK), `topic_name` (VARCHAR 50).
- [ ] Create `repository_collaborators` table: `repo_id`, `user_id`, `permission` (ENUM: read, triage, write, maintain, admin).
- [ ] Create `git_refs` table: `repo_id`, `ref_type` (ENUM: branch, tag), `ref_name` (VARCHAR 255), `commit_sha` (CHAR 40).
- [ ] Create `git_commits_metadata` table: `repo_id`, `commit_sha`, `author_email`, `committer_email`, `authored_at`, `committed_at`.

**Schema Migrations (Issues & PRs):**
- [ ] Create `issues` table: `id`, `repo_id` (FK), `number` (INTEGER), `title`, `body_markdown`, `state` (ENUM: open, closed).
- [ ] Create `issues` constraints: Enforce `UNIQUE(repo_id, number)`.
- [ ] Create `pull_requests` table: `issue_id` (FK, PRIMARY KEY), `head_repo_id`, `head_branch`, `base_repo_id`, `base_branch`, `merge_commit_sha`.
- [ ] Create `issue_comments` table: `id`, `issue_id` (FK), `user_id` (FK), `body_markdown`, `created_at`, `updated_at`.
- [ ] Create `issue_labels` table: `issue_id` (FK), `label_id` (FK).
- [ ] Create `labels` table: `id`, `repo_id` (FK), `name`, `color` (CHAR 6), `description`.
- [ ] Create `milestones` table: `id`, `repo_id` (FK), `title`, `state`, `due_on`.

**Search & Triggers (FTS5):**
- [ ] Create FTS5 virtual table `issues_fts` using `tokenize='unicode61 remove_diacritics 1'`.
- [ ] Define `issues_fts` columns: `repo_id UNINDEXED`, `issue_number UNINDEXED`, `title`, `body`.
- [ ] Implement SQLite trigger `issues_after_insert`: `INSERT INTO issues_fts(rowid, repo_id, issue_number, title, body) VALUES(new.id, ...)`.
- [ ] Implement SQLite trigger `issues_after_update`: `UPDATE issues_fts SET title = new.title, body = new.body WHERE rowid = new.id`.
- [ ] Implement SQLite trigger `issues_after_delete`: `DELETE FROM issues_fts WHERE rowid = old.id`.
- [ ] Create FTS5 virtual table `code_search_fts` for default branch file indexing.
- [ ] Implement FTS5 rank query generation for exact phrase matching (`"exact phrase"`) vs token matching in Rust.

**Operational Subsystems:**
- [ ] Create `webhook_endpoints` table: `repo_id`, `url`, `content_type`, `secret_token`, `insecure_ssl`.
- [ ] Create `webhook_events` array/JSON table for routing specific events (e.g., `push`, `pull_request`).
- [ ] Create `webhook_deliveries` table: `endpoint_id`, `guid`, `request_headers`, `request_body`, `response_status`, `duration_ms`.
- [ ] Implement background task using `tokio::time::interval` to run `VACUUM INTO 'backup.sqlite'` every 24 hours.
- [ ] Implement SQLite auto-updating `updated_at` triggers for every mutable table.
- [ ] Create `lfs_objects` table: `repo_id`, `oid` (CHAR 64), `size` (BIGINT), `storage_path`.

#### 1.1.2 Memory Bus & Cache (Local Adapter)
**Concurrency & Data Structures:**
- [ ] Initialize `dashmap::DashMap<RepoId, Arc<tokio::sync::Mutex<GitRepository>>>` to multiplex local `git2` bindings safely.
- [ ] Initialize `dashmap::DashMap<UserId, ActiveSession>` for instant session validation without DB lookups.
- [ ] Initialize `tokio::sync::mpsc::channel(100_000)` for the asynchronous webhook dispatch queue.
- [ ] Initialize `tokio::sync::mpsc::channel(50_000)` for the repository statistics rollup queue (stars/forks counters).
- [ ] Implement `tokio::sync::Semaphore::new(num_cpus::get() * 4)` strictly limiting concurrent `git-upload-pack` executions.

**PubSub & WebSockets:**
- [ ] Initialize `tokio::sync::broadcast::channel::<SystemEvent>(1024)` for global system-wide banners/alerts.
- [ ] Implement a `DashMap<RepoId, tokio::sync::broadcast::Sender<RepoEvent>>` for dynamic repository-scoped event channels.
- [ ] Implement Garbage Collection: Wrap repository channels in `Arc` and drop from `DashMap` when `Arc::strong_count` == 1.
- [ ] Wire Axum `WebSocketUpgrade` extractors to subscribe to repository broadcast channels for issue comment streaming.
- [ ] Implement ping/pong keepalive frames in the WebSocket handler (interval: 30s) to prune dead TCP connections.

**Caching (Moka):**
- [ ] Configure `moka::future::Cache` for rendered Markdown HTML, sized to 500MB, with a Time-To-Idle (TTI) of 2 hours.
- [ ] Configure `moka::future::Cache` for Git Tree entries (`git ls-tree`), sized to 1GB, evicting on LRU.
- [ ] Configure `moka::future::Cache` for parsed JWTs (GitHub Apps), max capacity 10,000, TTL 10 minutes.
- [ ] Configure `moka::future::Cache` for user permission matrices (`UserId` + `RepoId` -> `Role`), TTL 5 minutes.
- [ ] Implement Singleflight pattern (`async_cell`) around the Markdown rendering cache to prevent cache stampedes on hot issues.

**Rate Limiting (Governor):**
- [ ] Configure `governor::Quota::per_hour(nonzero!(5000))` for authenticated API requests.
- [ ] Configure `governor::Quota::per_hour(nonzero!(60))` for unauthenticated API requests (keyed by IP).
- [ ] Configure `governor::Quota::per_minute(nonzero!(20))` for resource-intensive GraphQL queries.
- [ ] Implement Axum middleware to extract IP/Token, check the Governor rate limiter, and inject `X-RateLimit-Limit` headers.
- [ ] Implement Axum middleware to inject `X-RateLimit-Remaining` and `X-RateLimit-Reset` headers.

#### 1.1.3 Binary Embedding & Asset Serving (Axum)
**Build-Time Compression (`build.rs`):**
- [ ] Execute `npx @angular/cli build --configuration production` via `std::process::Command` in `build.rs`.
- [ ] Iterate over all output files (`.js`, `.css`, `.svg`, `.html`).
- [ ] Compress each file using `brotli::CompressorReader` with quality level 11 (maximum).
- [ ] Compress each file using `flate2::GzBuilder` with quality level 9 as a fallback.
- [ ] Write compressed byte arrays into a generated `assets.rs` file as `pub static` arrays.
- [ ] Generate SHA-256 hashes of uncompressed contents to serve as immutable ETags.

**Runtime Serving (Axum):**
- [ ] Implement custom Axum `Service` routing specifically for the static perfect hash map (PHF) generated by `rust-embed`.
- [ ] Read `Accept-Encoding` header from client requests.
- [ ] Serve `.br` pre-compressed bytes if client supports `br`, set `Content-Encoding: br`.
- [ ] Serve `.gz` pre-compressed bytes if client supports `gzip`, set `Content-Encoding: gzip`.
- [ ] Set `Cache-Control: public, max-age=31536000, immutable` for all files containing a hash in their filename (e.g., `main.a3b4c5.js`).
- [ ] Set `Cache-Control: no-cache` explicitly for `index.html`.

**Security Headers & Routing:**
- [ ] Implement middleware to inject `Content-Security-Policy: default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'; img-src 'self' data:; connect-src 'self' ws: wss:;`.
- [ ] Implement middleware to inject `X-Frame-Options: DENY`.
- [ ] Implement middleware to inject `X-Content-Type-Options: nosniff`.
- [ ] Implement middleware to inject `Referrer-Policy: strict-origin-when-cross-origin`.
- [ ] Implement Axum fallback route (`Fallback`) that parses the URI; if it's an API route return 404 JSON, otherwise return the embedded `index.html`.

**Dynamic Assets:**
- [ ] Implement deterministic Identicon avatar generator using `identicon-rs`.
- [ ] Hash the user's login name with MD5 to determine grid layout and color.
- [ ] Render the Identicon to a PNG buffer in memory.
- [ Brotli-compress the PNG buffer and cache it in `moka` with a 30-day TTL.
- [ ] Serve source maps (`.map` files) dynamically, executing a permission check to ensure the requester is an Admin/Developer before serving.

### 1.2 The "Enterprise Cloud" (Distributed Microservices Mode)

#### 1.2.1 Database (PostgreSQL 16+ Adapter)
**Partitioning & Schema Design:**
- [ ] Execute DDL: `CREATE TABLE audit_logs (...) PARTITION BY RANGE (created_at);`
- [ ] Execute DDL to create time-based partitions automatically via `pg_cron` (e.g., `audit_logs_y2026m04`).
- [ ] Execute DDL: `CREATE TABLE webhook_deliveries (...) PARTITION BY LIST (tenant_id);`
- [ ] Migrate primary keys from serial `id` to `UUIDv7` via `uuid-ossp` or Rust-side generation to ensure sequential writes without B-Tree bloat.
- [ ] Create Postgres `ENUM` types natively for strict domain values (`issue_state`, `pr_review_state`, `webhook_event_type`).

**Row-Level Security (RLS) & Auth:**
- [ ] Execute DDL: `ALTER TABLE repositories ENABLE ROW LEVEL SECURITY;`
- [ ] Create RLS Policy: `CREATE POLICY repo_read ON repositories FOR SELECT USING (is_private = false OR current_user_id() IN (SELECT user_id FROM repository_collaborators WHERE repo_id = id));`
- [ ] Create RLS Policy: `CREATE POLICY repo_write ON repositories FOR UPDATE USING (current_user_id() IN (SELECT user_id FROM repository_collaborators WHERE repo_id = id AND permission IN ('write', 'maintain', 'admin')));`
- [ ] Implement PostgreSQL `set_config('app.current_user_id', ...)` in the connection pool checkout phase to pass context to RLS.

**Indexing & Advanced Types:**
- [ ] Define column `workflow_definition` in table `actions_workflows` as `JSONB`.
- [ ] Create index: `CREATE INDEX idx_workflows_jsonb ON actions_workflows USING GIN (workflow_definition);`
- [ ] Create covering index: `CREATE INDEX idx_prs_status_assignee ON pull_requests (repo_id, state, assignee_id) INCLUDE (title, merge_commit_sha);` for hot API endpoints.
- [ ] Enable `ltree` extension: `CREATE EXTENSION IF NOT EXISTS ltree;`
- [ ] Add column `path` of type `ltree` to `teams` table to represent nested organization units (e.g., `engineering.backend.api`).
- [ ] Create index: `CREATE INDEX idx_teams_path ON teams USING GIST (path);`

**Replication & CDC:**
- [ ] Enable logical replication in `postgresql.conf`: `wal_level = logical`.
- [ ] Create CDC publication: `CREATE PUBLICATION github_events_pub FOR TABLE issues, pull_requests, repositories, users;`
- [ ] Create CDC replication slot: `SELECT pg_create_logical_replication_slot('elasticsearch_indexer', 'pgoutput');`
- [ ] Configure `sqlx` read replicas: Parse comma-separated database URLs and route all HTTP `GET` queries to the replica pool via application middleware.
- [ ] Utilize `pg_advisory_xact_lock` hash of the user slug when creating a new user to prevent race conditions bypassing unique constraints in highly concurrent setups.

#### 1.2.2 Distributed Cache (Redis Cluster Adapter)
**Cluster Setup & Namespacing:**
- [ ] Initialize `fred::clients::RedisClusterClient` with topology refresh enabled.
- [ ] Establish strict key prefixes: `sys:`, `sess:`, `cache:`, `limit:`, `lock:`, `job:`.
- [ ] Implement Redis connection health checks via Axum `/healthz` probing `PING`.
- [ ] Configure client-side caching in `fred` (Redis 6+ Tracking) for extremely hot keys like global feature flags.

**Distributed Rate Limiting (GCRA):**
- [ ] Write GCRA Lua script `gcra.lua` checking token bucket state entirely server-side.
- [ ] Load `gcra.lua` into Redis via `SCRIPT LOAD` on application boot and store the SHA1 digest.
- [ ] Execute `EVALSHA` passing `limit:user:{id}:api`, rate (e.g., 5000), and burst window to decrement API quotas atomically.
- [ ] Implement dynamic penalty boxes: if `EVALSHA` returns rate-limited > 100 times in 1 minute, write IP to `sys:penalty_box:{ip}` with TTL 3600s.

**Session & Auth State:**
- [ ] Write OAuth session data using `HSET sess:oauth:{token_hash} user_id {id} scope {scopes}`.
- [ ] Apply 24-hour TTL to OAuth sessions: `EXPIRE sess:oauth:{token_hash} 86400`.
- [ ] Write Personal Access Token validations to `cache:pat:{token_hash}` with a 5-minute TTL to bypass Postgres.
- [ ] Implement OAuth session revocation by sending `DEL sess:oauth:{token_hash}` across the cluster.

**Locks & Concurrency (Redlock):**
- [ ] Implement Redlock algorithm logic in Rust.
- [ ] Generate unique random string (UUID) for lock value.
- [ ] Execute `SET lock:pr:{id}:merge {uuid} NX PX 15000` to lock a Pull Request during a Git merge operation.
- [ ] Implement Lua script to safely release Redlock only if the UUID matches (preventing releasing someone else's lock if TTL expired).
- [ ] Use Redlock on `lock:repo:{owner}:{name}:create` to prevent identical simultaneous repository creation requests.

**Analytics & Trackers:**
- [ ] Execute `PFADD track:repo:{id}:views:{date} {user_id}` on repository load to track unique daily views using HyperLogLog.
- [ ] Execute `PFADD track:repo:{id}:clones:{date} {ip_hash}` on `git-upload-pack` hook to track unique daily clones.
- [ ] Merge HyperLogLog metrics over 14 days using `PFMERGE` for the "Traffic" insights tab.
- [ ] Execute `INCRBY notifications:user:{id}:unread 1` when dispatching an issue mention.
- [ ] Store long-running repository import statuses as JSON strings in `job:import:{id}` with UI polling.

#### 1.2.3 Message Broker (Kafka/Redpanda Adapter)
**Topic Architecture:**
- [ ] Initialize `rdkafka::producer::FutureProducer` with `acks=all` and `enable.idempotence=true`.
- [ ] Create topic `github.events.webhooks` with 32 partitions, compacted.
- [ ] Create topic `github.events.audit` with 16 partitions, retention 365 days.
- [ ] Create topic `github.events.ci.dispatch` with 64 partitions, retention 1 day.
- [ ] Create topic `github.events.dlq` for dead-letter messages.
- [ ] Implement partitioning strategy: Always set message key to `repository_id` as a byte array to guarantee strict ordering per repository.

**Schema Registry & Protobuf:**
- [ ] Define `IssueCreated.proto`: `int64 issue_id`, `int64 repo_id`, `int64 actor_id`, `string timestamp`.
- [ ] Define `PullRequestMerged.proto`: `int64 pr_id`, `string base_ref`, `string head_ref`, `string commit_sha`.
- [ ] Define `ActionWorkflowTriggered.proto`: `string workflow_yml`, `string commit_sha`, `map<string, string> env_vars`.
- [ ] Integrate Schema Registry HTTP client to fetch schemas.
- [ ] Serialize all outgoing Kafka messages strictly as Confluent-compatible Protobuf binaries (Magic Byte + Schema ID + Payload).

**Transactional Outbox Pattern (Reliability):**
- [ ] Create Postgres table `outbox_events`: `id` (UUIDv7), `aggregate_type`, `aggregate_id`, `event_type`, `payload_pb` (BYTEA), `published` (BOOLEAN).
- [ ] Modify API handlers to open a Postgres transaction, insert the Domain Model mutation, and insert the `outbox_events` row in the same transaction.
- [ ] Deploy a standalone Rust worker (CDC Relay) that polls/tails `outbox_events` where `published = false`.
- [ ] Have the CDC Relay publish the `payload_pb` to Kafka, wait for Ack, then `UPDATE outbox_events SET published = true`.

**Consumers & Dispatchers:**
- [ ] Initialize `rdkafka::consumer::StreamConsumer` with `group.id="webhook_dispatcher"`.
- [ ] Consumer `webhook_dispatcher`: Read `github.events.webhooks`, execute HTTP POST to target URLs.
- [ ] Consumer `webhook_dispatcher`: If HTTP POST fails (5xx/Timeout), push original message to `github.events.dlq` with custom header `Retry-Count=1`.
- [ ] Initialize Consumer `ci_runner_coordinator`: Read `github.events.ci.dispatch`, hold message until an active GitHub Actions Runner polls for work.
- [ ] Initialize Consumer `search_indexer`: Read general events, extract text payloads, and execute bulk HTTP `POST` to ElasticSearch `_bulk` API.
- [ ] Initialize Consumer `mail_dispatcher`: Aggregate issue comments grouped by user preference, format HTML emails using `askama`, send via AWS SES.
- [ ] Initialize Consumer `security_scanner`: Read `PushEvent` messages, check `package.json`/`Cargo.toml` diffs against OSV (Open Source Vulnerabilities) database API.
- [ ] Initialize Consumer `metrics_aggregator`: Read API usage metrics stream and push to InfluxDB/Prometheus pushgateway for billing calculation.## 2. API PARITY & THE GENERATION COMPILER

We do not write API boilerplate. We build a compiler that reads `api.github.com.yaml` and emits Rust.

### 5.7 Portable Single-Node Executable (Zero-Dependency Builds)
- [ ] **Fully Static Compilation:** Configure the CI/CD pipeline to compile fully statically linked ELF binaries for Linux using `musl` libc to ensure zero host-level shared library requirements.
- [ ] **Frontend Asset Embedding:** Implement build steps using `rust-embed` (or `rust-embed`) to bundle all compiled Angular 21 (Zoneless) frontend assets, CSS, and localized templates directly into the compiled native binary.
- [ ] **Embedded SQLite Datastore:** Integrate SQLite support with connection pooling, Write-Ahead Logging (WAL), and busy-timeout handling as the default, zero-configuration relational database backend for single-node deployments.
- [ ] **In-Process SSH Server:** Embed a pure Go/Rust SSH server (e.g., `russh` or `gliderlabs/ssh`) listening on a custom port to handle `git+ssh://` operations natively without requiring host `sshd` integration or user account creation.
- [ ] **In-Process Background Scheduler:** Implement an internal task supervisor to manage cron jobs, webhook deliveries, and background queue processing (via an embedded queue like SQLite) independent of external message brokers (RabbitMQ/Kafka).
- [ ] **Embedded Search Engine Fallback:** Provide a lightweight, embedded full-text search alternative (e.g., SQLite FTS5 or Bleve/ZincSearch) to fallback from Elasticsearch for code and issue indexing in the portable execution mode.
- [ ] **Auto-Provisioning & Zero-Config:** Build a first-run initialization routine that automatically detects the host environment, generates a secure default configuration (TLS certificates, JWT secrets), and bootstraps the initial admin user.
- [ ] **Cross-Platform Standalone Binaries:** Output standalone `.exe` (Windows), Mach-O (macOS), and ELF (Linux) binaries natively, guaranteeing no external runtime dependencies (Node.js, Python, or Ruby) are required on the target host.
- [ ] **Built-in Daemonization Support:** Add built-in CLI flags (e.g., `github-clone web --daemon`) and generate helper scripts for users who wish to wrap the single binary trivially in systemd or Windows Services later.
## 6. EXHAUSTIVE QA, OBSERVABILITY & THREAT MODELING

### 6.2 Telemetry, Tracing & Context Propagation
- [ ] **Distributed Tracing Implementation (OpenTelemetry)**
    - [ ] Integrate `tracing-opentelemetry` crate into the Rust Axum API backend.
    - [ ] Integrate `@opentelemetry/auto-instrumentations-node` for specific Node.js microservices.
    - [ ] Configure OpenTelemetry Collector to export traces in OTLP format over gRPC to the telemetry backend.
    - [ ] Propagate standard W3C headers (`traceparent`, `tracestate`) across HTTP boundaries.
    - [ ] Propagate trace context inside Kafka record headers for background jobs (e.g., webhook delivery).
    - [ ] Create explicit spans for PostgreSQL transactions, capturing the parameterized SQL query string (excluding values).
    - [ ] Create explicit spans for Redis Cache interactions, categorizing by Hits, Misses, and Evictions.
    - [ ] Create explicit spans for Git subprocess executions (e.g., `git-receive-pack`).
    - [ ] Inject `user.id`, `repository.id`, and `organization.id` as span attributes for tier-based tracing.
    - [ ] Ensure span attributes scrub all Personally Identifiable Information (PII) like email addresses.
- [ ] **Metrics Collection & Prometheus Export**
    - [ ] Expose a `/metrics` endpoint in Prometheus format on a dedicated internal port (e.g., 9090).
    - [ ] Track HTTP request duration in a histogram (`github_api_request_duration_seconds`) with explicit buckets (0.01, 0.05, 0.1, 0.5, 1, 5).
    - [ ] Track HTTP request totals categorized by route, method, and status code.
    - [ ] Track Git-specific metrics: measure `git-rev-list` execution time.
    - [ ] Track Git-specific metrics: measure `git pack-objects` memory consumption and duration.
    - [ ] Track Git-specific metrics: log object delta depth averages per push.
    - [ ] Track background job queue depth, processing latency, and dead-letter queue (DLQ) sizes.
    - [ ] Track PostgreSQL connection pool exhaustion events.
- [ ] **Internal Observability Dashboards (Grafana)**
    - [ ] Create "API Golden Signals" dashboard (Latency, Traffic, Errors, Saturation).
    - [ ] Create "Git Infrastructure" dashboard displaying 99th percentile clone/fetch times.
    - [ ] Create "Database Health" dashboard showing active connections, transaction rollback rates, and slow queries.
    - [ ] Create "WebSocket Connections" dashboard showing active client counts and message throughput.
    - [ ] Create "Storage Nodes" dashboard displaying IOPS, disk utilization, and packfile sizes.
- [ ] **Continuous Profiling in Rust (`pprof-rs`)**
    - [ ] Embed the `pprof` crate in the core Rust API binaries.
    - [ ] Configure an OS signal handler (e.g., `SIGUSR1`) to trigger a 30-second CPU profile.
    - [ ] Write the resulting CPU profile to disk as a collapsed stack format (for flamegraphs).
    - [ ] Implement an admin-only authenticated endpoint `/api/internal/debug/pprof/heap` to dump jemalloc heap profiles.
    - [ ] Write a CI script to automate generating a flamegraph during simulated load testing to catch performance regressions.
- [ ] **Structured Logging & Audit Trails**
    - [ ] Enforce 100% structured JSON logging across all backend services via `tracing-subscriber`.
    - [ ] Define a strict JSON schema requiring `timestamp` (ISO8601), `level`, `target`, `span_id`, and `trace_id`.
    - [ ] Generate an `X-GitHub-Request-Id` (UUIDv4) at the NGINX ingress layer.
    - [ ] Ensure the `X-GitHub-Request-Id` is included in all downstream logs and returned in the HTTP response header.
    - [ ] Implement a regex-based log sanitization middleware to scrub GitHub PATs (`ghp_...`, `gho_...`).
    - [ ] Implement log sanitization to scrub OAuth access tokens and session cookie values before writing to standard out.
    - [ ] Forward sanitized logs to a centralized aggregation service (e.g., Elasticsearch/Loki) via Vector or FluentBit.

### 6.3 Security Threat Model (STRIDE) & GitHub-Parity Mitigations
- [ ] **Authentication, Spoofing & Session Management**
    - [ ] Enforce Ed25519, RSA (minimum 2048-bit), and ECDSA validation for all uploaded SSH keys.
    - [ ] Reject compromised or weak SSH keys using a known-vulnerable key blocklist.
    - [ ] Implement precise WebAuthn (FIDO2) ceremonies using `navigator.credentials.create()` for security key registration.
    - [ ] Implement WebAuthn `navigator.credentials.get()` for two-factor login challenges.
    - [ ] Generate and validate 16-character alphanumeric recovery codes, enforcing a one-time-use policy.
    - [ ] Enforce `Strict` SameSite cookie attributes, `Secure` flags, and `HttpOnly` on all session cookies.
    - [ ] Parse raw GPG signatures from Git commit objects using an internal keyring service.
    - [ ] Validate x.509 (S/MIME) certificates attached to commits against a trusted root CA.
    - [ ] Ensure the UI precisely renders the "Verified", "Unverified", or "Partially Verified" badges based on signature trust chains.
- [ ] **Integrity, Tampering & Branch Protections**
    - [ ] Implement the "Require linear history" branch protection rule by rejecting pushes containing merge commits via `pre-receive` hook.
    - [ ] Implement the "Require signed commits" branch protection rule by rejecting commits with null/invalid signatures in the `pre-receive` hook.
    - [ ] Implement "Require status checks to pass" by verifying required CI contexts exist and are 'success' before allowing a merge.
    - [ ] Configure distributed storage nodes to run `git fsck` periodically.
    - [ ] Verify Git SHA-1/SHA-256 object hashes on disk reads, failing the read closed and triggering an alert on hash mismatch.
    - [ ] Setup strict read-only PostgreSQL database replicas for all `GET` API endpoints to physically prevent tampering during read operations.
- [ ] **Auditability & Repudiation Mitigations**
    - [ ] Design the PostgreSQL `audit_logs` table schema (actor_id, action_id, timestamp, resource_id, metadata JSONB).
    - [ ] Enforce append-only constraints on the `audit_logs` table via database-level triggers or roles.
    - [ ] Partition the `audit_logs` table by month to optimize retention policies and query performance.
    - [ ] Log precise IP address, standard User-Agent, and GeoIP location resolution for all authentication events.
    - [ ] Emit all audit log rows securely to an internal Kafka topic for ingestion by a SIEM (e.g., Splunk, Datadog).
    - [ ] Implement a user-facing "Security log" UI in repository and user settings, perfectly matching GitHub's chronological event history.
- [ ] **Isolation, SSRF, & Information Disclosure Prevention**
    - [ ] Isolate the execution of GitHub-Flavored Markdown (GFM) rendering to prevent memory reading or path traversal.
    - [ ] Run the GFM renderer inside a specific WebAssembly container or `libmountsandbox` environment.
    - [ ] Apply `seccomp` filters to the sandbox to explicitly block `execve`, `open`, and network syscalls.
    - [ ] Implement Server-Side Request Forgery (SSRF) protections for outgoing webhooks (block local IPs, 169.254.x.x, 127.x.x.x).
    - [ ] Implement a pre-receive hook for real-time Secret Scanning on `git push`.
    - [ ] Write exact regex matchers for AWS Access Keys (`AKIA...`), GitHub PATs (`ghp_...`), and Slack Bot Tokens.
    - [ ] Reject pushes containing secrets with a terminal error message matching GitHub's exact push rejection text.
    - [ ] Enforce strict Authorization checks for Repository visibility (Public vs. Private) at the GraphQL node-resolver layer.
    - [ ] Enforce strict Authorization checks at the REST API middleware layer, failing with a generic `404 Not Found` (instead of `403`) to prevent repository existence leakage.
    - [ ] Implement strict Content Security Policy (CSP) headers blocking inline scripts (`script-src 'self'`).
- [ ] **Rate Limiting & Denial of Service (DDoS)**
    - [ ] Configure Cloudflare/AWS Shield Web Application Firewall (WAF) rules for volumetric Layer 3/4 attacks.
    - [ ] Implement exact GitHub REST API Rate Limiting headers: `x-ratelimit-limit`, `x-ratelimit-remaining`, `x-ratelimit-used`, `x-ratelimit-reset`.
    - [ ] Use Redis Lua scripts to ensure atomic token bucket deduction for rate limits.
    - [ ] Implement GraphQL rate limiting based on a complex query cost calculation (nodes + edges), not just request counts.
    - [ ] Implement a custom Axum `ConcurrencyLimitLayer` specifically for the `/git-upload-pack` and `/git-receive-pack` endpoints.
    - [ ] Protect against "Git Bomb" attacks by limiting max tree depth to 50 and rejecting deeply nested Git trees.
    - [ ] Protect against Git bombs by enforcing a strict maximum size (e.g., 2GB) per individual Git object inside a push.
    - [ ] Enforce repository-level storage quotas (e.g., 10GB limit) and reject pushes that exceed the quota.
- [ ] **Elevation of Privilege Protections**
    - [ ] Implement strict Role-Based Access Control (RBAC) mimicking GitHub's exact permission tiers: Read, Triage, Write, Maintain, Admin.
    - [ ] Ensure fine-grained permissions map correctly to actions (e.g., 'Triage' can manage issues but cannot push code).
    - [ ] Provision temporary, short-lived `GITHUB_TOKEN`s for CI/CD pipeline workers.
    - [ ] Scope `GITHUB_TOKEN` permissions strictly to the executing repository unless explicitly overridden in the workflow file.
    - [ ] Ensure `GITHUB_TOKEN`s are automatically revoked and deleted from the database the millisecond the CI job completes.
### 0.9 Enterprise Identity & Marketplaces
- [ ] Define routing and DB tables for SAML Single Sign-On configuration (IdP URL, X.509 Certificate, Issuer).
- [ ] Implement the SAML Assertion Consumer Service (ACS) endpoint parsing XML payloads using `xmlsec`.
- [ ] Validate SAML signatures strictly to prevent XML Signature Wrapping (XSW) attacks.
- [ ] Automatically provision Just-In-Time (JIT) shadow users upon successful SAML authentication into the enterprise tenant.
- [ ] Implement SCIM 2.0 `GET /scim/v2/Users` and `POST /scim/v2/Users` endpoints for automated Okta/Entra ID synchronization.
- [ ] Support SCIM PATCH operations for granular updates (e.g., changing a user's display name or active status).
- [ ] Build the GitHub App Marketplace webhook event multiplexer to dispatch events to third-party integrations.
- [ ] Implement OAuth App and GitHub App granular installation endpoints (per-repository vs all-repositories selection).
- [ ] Enforce rate limits specifically for third-party App tokens, separate from user PAT limits.
- [ ] Track revenue and billing models for GitHub Apps via Stripe Connect webhooks.

### 0.9 Enterprise Identity, Marketplaces & SCIM (Expanded)
- [ ] Define Postgres routing and DB tables for SAML Single Sign-On configuration (IdP URL, X.509 Certificate, Issuer).
- [ ] Implement the SAML Assertion Consumer Service (ACS) endpoint parsing XML payloads natively using the `xmlsec` crate.
- [ ] Validate SAML signatures strictly to prevent XML Signature Wrapping (XSW) attacks and enforce strict temporal bounds (`NotBefore`, `NotOnOrAfter`).
- [ ] Automatically provision Just-In-Time (JIT) shadow users upon successful SAML authentication into the enterprise tenant.
- [ ] Implement SCIM 2.0 `GET /scim/v2/Users` and `POST /scim/v2/Users` endpoints for automated Okta/Entra ID synchronization conforming to RFC 7644.
- [ ] Support SCIM PATCH operations for granular field updates (e.g., changing a user's display name or toggling `active` status).
- [ ] Implement SCIM `DELETE /scim/v2/Users/{id}` to automatically suspend users and invalidate their active sessions.
- [ ] Build the GitHub App Marketplace webhook event multiplexer to reliably dispatch installation events to third-party integrations.
- [ ] Implement OAuth App and GitHub App granular installation endpoints distinguishing between "All Repositories" and "Select Repositories".
- [ ] Enforce dynamic rate limits specifically for third-party App tokens, separate from the standard user PAT quotas.
- [ ] Track revenue and billing models for GitHub Apps via Stripe Connect webhooks, updating database installation states.
- [ ] Implement fine-grained repository permissions scopes specifically for GitHub Apps (e.g., `contents: read`, `issues: write`).
- [ ] Build a robust JWT verification middleware specifically for authenticating incoming requests from GitHub Apps securely.
- [ ] Implement the UI backend for the Marketplace, exposing GraphQL nodes for App Listings, Reviews, and Pricing plans.

### 2.4 Repository Rulesets & AST Engine
- [ ] **Ruleset Database Schema:** Define Postgres tables mapping `repository_rulesets` to specific `ref_names` (branches/tags) and `bypass_actors`.
- [ ] **Abstract Syntax Tree (AST) Rules:** Implement a JSONB-based AST schema to store complex logical conditions (e.g., `Require 2 approvals AND (CI must pass OR Actor is Admin)`).
- [ ] **Ruleset Evaluation Engine:** Build a highly concurrent Rust evaluation engine that intercepts `git push` hooks and dynamically evaluates the incoming commit against all active rulesets.
- [ ] **Commit Signature Enforcement:** Add specific rule evaluators to enforce that every commit in the push is cryptographically signed and verified.
- [ ] **Merge Queue Integration:** Link the Ruleset engine to the Merge Queue, automatically enforcing queue entry requirements if a branch is strictly protected.

### 2.5 Advanced Code Review State Machine
- [ ] **Review Assignment Algorithms:** Implement round-robin and load-balanced reviewer assignment logic for teams inside Postgres/Rust.
- [ ] **Viewed File State Management:** Create a distributed tracking table allowing developers to mark specific files as "Viewed" in a PR, persisting this state across active browser sessions.
- [ ] **Multi-line Suggested Changes:** Extend the PR comment API to support `start_line` and `end_line` parameters to generate "Suggested Changes" diff blocks.
- [ ] **Suggested Changes Commit Engine:** Build a server-side Git tree mutation endpoint that directly applies a user's approved "Suggested Change" without requiring a local clone or push.
- [ ] **Review Dismissal Logic:** Implement the state transition logic to automatically dismiss older pending approvals when new commits are forcefully pushed to a PR branch.

### Tier 9: Missing Ecosystem Integrations (Email, Media, Conflicts, Archives)
- [ ] **Inbound Email Reply Processing:** Build an SMTP ingestion server natively in Rust (utilizing `mailin` or similar) to receive raw RFC 822 email replies to notifications without requiring a heavy external dependency.
- [ ] **MIME Parsing & Markdown Extraction:** Implement a robust multipart MIME parser to traverse email structures, strip signatures, remove quoted historical replies, and discard HTML formatting, extracting only the user's intended raw text reply.
- [ ] **Reply-To Token Cryptography:** Generate and validate stateless, cryptographically signed `Reply-To` addresses (e.g., `reply+{repo_id}+{user_id}+{hmac}@replica.com`) to securely authenticate inbound email replies and map them to the correct Issue/PR thread.
- [ ] **Media Attachment API:** Implement the `POST /upload/policies/assets` API to issue short-lived, pre-signed S3 upload URLs specifically for user-attached media (images, PDFs, videos) inside Markdown text areas, strictly enforcing file type and size quotas.
- [ ] **Antivirus Pipeline:** Implement a Rust background worker that integrates with `clamd` (ClamAV) to scan all user-uploaded media blobs asynchronously before they are officially linked into the Markdown rendered output, quarantining infected files.
- [ ] **Conflict Resolution API:** Build a Rust endpoint that utilizes `libgit2` or `gitoxide` to dynamically compute 3-way merges and return the raw file content injected with standard `<<<<<<< HEAD` conflict markers for web-based resolution.
- [ ] **Conflict Commit Synthesizer:** Implement the backend endpoint to accept a user's manually resolved file content from the UI, generate a new Git tree, and construct the merge commit directly via the backend API without a local clone.
- [ ] **Archive Streaming (ZIP/TAR):** Implement `GET /{owner}/{repo}/archive/refs/heads/{ref}.zip`, executing `git archive` natively on the storage node and dynamically chunking/streaming the compressed bytes to the HTTP response to prevent memory exhaustion on massive repositories.

### Tier 10: Cross-Fork Collaborations & Submodule Resolution
- [ ] Implement the "Allow edits from maintainers" feature specifically for Cross-Fork Pull Requests.
- [ ] Build the authentication override mapping: when a maintainer attempts to push to `refs/heads/feature-branch` on a forked repository, dynamically inject a temporary, scoped authorization token bypassing strict repository ownership.
- [ ] Ensure `git push` access is explicitly limited *only* to the branch tied to the active PR, rejecting pushes to the fork's `main` branch or any other refs.
- [ ] Implement deep Submodule metadata resolution inside the Git Tree API (`GET /repos/{owner}/{repo}/git/trees/{sha}`).
- [ ] Parse `.gitmodules` continuously in the background using `gitoxide` and map the submodule SHA to the exact upstream repository URL.
- [ ] Inject the resolved `submodule_url` dynamically into the Tree API response to allow the Angular frontend to render clickable directory links to the exact commit history of the external repository.
- [ ] Support automated sync mutations: build an API endpoint that executes `git submodule update --remote` purely on the backend and generates an automated PR bumping the submodule pointers.

### Phase 2.7: Ecosystem Edge Cases & Raw Representations
- [ ] **Public Key Discovery Endpoints:** Implement the `GET /{username}.keys` endpoint, querying the `ssh_keys` database and returning a plaintext string of all verified, active public SSH keys separated by newlines.
- [ ] **GPG Key Discovery Endpoints:** Implement the `GET /{username}.gpg` endpoint, returning the user's public GPG keys in armor-encoded plaintext for easy importing via `curl`.
- [ ] **Raw Patch Output Generation:** Implement routing logic allowing users to append `.patch` to any Commit or Pull Request URL. Dynamically execute `git format-patch` natively via `gitoxide` or `libgit2` and stream the pure text response.
- [ ] **Raw Diff Output Generation:** Implement routing logic for `.diff` suffixes on Commits and PRs, streaming the raw `git diff` output with `text/plain` content types.
- [ ] **Actions SVG Build Badges:** Implement `GET /{owner}/{repo}/actions/workflows/{file}/badge.svg`, querying the latest Check Suite state for the default branch and utilizing `resvg` to generate a dynamic "passing/failing" SVG badge for README embeds.
- [ ] **Commit Status Badges:** Expose generic SVG badge endpoints that render arbitrary CI/CD commit statuses (e.g., coverage percentages) based on the latest commit hash.
- [ ] **Redirect Edge Cases:** Implement the explicit redirect mapping logic: if a user navigates to a repository that has been renamed, strictly issue a `301 Moved Permanently` to the new path to prevent breaking existing ecosystem hyperlinks.

### 0.10 Developer Portal & App Lifecycle APIs
- [ ] **App Registration Schema:** Define `github_apps` Postgres table: `id`, `name`, `slug` (unique constraint), `description`, `owner_id`, `public` (boolean), `webhook_url`, `webhook_secret`.
- [ ] **Granular Permissions Storage:** Define the `github_app_permissions` JSONB column to rigidly store the 50+ granular `Read/Write/None` scope assertions defined during app creation.
- [ ] **Installation Mapping Schema:** Define `github_app_installations` Postgres table linking an App to specific user/org accounts, explicitly tracking `repository_selection` (all vs selected) and mapping selected IDs in a join table.
- [ ] **Manifest Conversion API:** Implement the `POST /app-manifests/{code}/conversions` endpoint to support the GitHub App Manifest creation flow natively from web browsers via deep-linking.
- [ ] **PKI & RSA Key Generation:** Build the RSA Keypair generator in Rust utilizing the `rsa` and `pkcs8` crates: dynamically generate 2048-bit RSA keys when an admin requests a new client secret.
- [ ] **Ephemeral Private Key Streaming:** Ensure the backend stores *only* the public key and its SHA-256 fingerprint in Postgres; securely stream the generated `.pem` private key directly to the HTTP response one time, dropping it entirely from server memory.
- [ ] **Public Key Rotation (JWKS):** Expose a `/.well-known/jwks.json` endpoint publishing all active public keys for the App to support OIDC verification by external identity providers.
- [ ] **Client Secret CSPRNG:** Implement the `ClientSecret` generator utilizing cryptographically secure pseudorandom number generators (e.g., `rand::rngs::OsRng`), returning a 40-character hex string.
- [ ] **Secret Hashing:** Store only `argon2` or `bcrypt` hashed versions of the Client Secret in the database to prevent lateral movement if the database is compromised.
- [ ] **App Installation Token Vending:** Implement the `POST /app/installations/{id}/access_tokens` endpoint, generating short-lived JWTs (1 hour max).
- [ ] **Intersection Permissions:** In the token vendor, dynamically intersect the App's global permissions against the specific installation's granted access to compute the final, minimal set of claims for the vended token.
- [ ] **App Deletion & Revocation:** Create administrative endpoints to suspend or delete GitHub App installations, instantly removing them from Postgres and dispatching a Redis broadcast to instantly invalidate all active vended JWTs globally.
- [ ] **Webhook Event Mapping:** Implement the strict validation layer ensuring an App cannot subscribe to webhook events (e.g., `pull_request`) if it has not been granted the prerequisite API permissions (e.g., `pull_requests: read`).
- [ ] **App Rate Limiting Mitigation:** Configure the `governor` rate limit logic in Rust to apply a distinct, significantly higher quota bucket strictly for requests authenticated via GitHub App JWTs vs standard user PATs.
- [ ] **Webhook Delivery Logging:** Architect the `app_webhook_deliveries` table to persist headers, payload hashes, and HTTP response codes for all outbound app webhooks, enforcing an automated 30-day deletion sweep.
### Tier 11: Compare, Sync Fork & Legacy Statuses
- [ ] **Commit Graph Traversal API:** Implement the `GET /repos/{owner}/{repo}/compare/{basehead}` REST endpoint natively using `gitoxide` to rapidly traverse the repository commit graph.
- [ ] **Merge Base Computation:** Compute the common ancestor (merge base) dynamically between the two refs to accurately return the `ahead_by` and `behind_by` commit integer counts.
- [ ] **Sync Fork Background Worker:** Build the asynchronous "Sync Fork" worker in Rust to perform a highly optimized `git fetch` from the upstream repository directly into the fork's bare repo storage.
- [ ] **Safe Fast-Forward Engine:** Execute a safe `git merge --ff-only` or `git reset --hard` (if configured by the user) on the backend to synchronize the fork's default branch.
- [ ] **Webhooks for Synced Forks:** Emit standard `push` webhooks after a successful fork sync so downstream CI systems linked to the fork are triggered correctly.
- [ ] **Legacy Statuses API Endpoint:** Implement the legacy `POST /repos/{owner}/{repo}/statuses/{sha}` endpoint to maintain full backward compatibility with older CI/CD tools (e.g., TravisCI, older Jenkins plugins).
- [ ] **State Translation Logic:** Map the legacy `state` string payload (`pending`, `success`, `error`, `failure`) dynamically into the modern `check_runs` and `check_suites` Postgres database schema.
- [ ] **Virtual Check Suites:** Automatically generate a virtual "Check Suite" wrapper for incoming legacy statuses to ensure they render seamlessly alongside modern GitHub Actions inside the Angular PR UI.
- [ ] **Status Rollup API:** Implement the `GET /repos/{owner}/{repo}/commits/{ref}/status` endpoint, performing a SQL rollup of all legacy statuses and modern checks into a single aggregate `state`.

### Tier 11.1: Compare, Sync Fork & Ref Math Internals
- [ ] **`gitoxide` Merge Base Resolution:** Implement lowest-common-ancestor algorithms natively in Rust. If the references share absolutely no history (unrelated histories), safely return an empty commit list and HTTP 404 instead of panicking.
- [ ] **Extensive Commit Pagination:** For branches that are massive (e.g., 10,000 commits ahead), enforce a strict limit of returning the first 250 commits in the API response payload, returning a pagination URL for subsequent data.
- [ ] **Fast-Forward Evaluation Engine:** During Sync Fork, utilize `gitoxide` to explicitly verify if the upstream commit is a direct descendant of the fork's current `HEAD` before attempting the write operation.
- [ ] **Rejection Context Headers:** If the Sync Fork fast-forward fails, return a `409 Conflict` containing a custom JSON schema describing the exact conflicting files to aid the user in manual resolution.
- [ ] **Fork Sync Branch Protection Override:** Ensure the internal Sync Fork execution strictly respects branch protection rules on the fork (e.g., bypassing review requirements only if the user holds Admin privileges).
- [ ] **Cross-Repository Ref Specifications:** Build the ref parser to correctly identify and fetch revisions located in separate physical repositories (e.g., `upstream_owner:branch` vs `fork_owner:branch`).

### Tier 11.2: Legacy Commit Statuses Emulation
- [ ] **Legacy Schema Translation Mapping:** Explicitly map incoming `POST /statuses/{sha}` payload fields (`state`, `target_url`, `description`, `context`) into a synthesized `check_run` Postgres insertion.
- [ ] **Context Deduplication Override:** Implement Upsert logic: if a legacy status is posted with a `context` string that already exists for that SHA, overwrite the existing status rather than appending a duplicate row.
- [ ] **Virtual Check Suite Instantiation:** Since legacy statuses do not belong to GitHub Apps, dynamically generate an overarching "Legacy System" virtual Check Suite on-the-fly to group these statuses cleanly in the database.
- [ ] **Hard Status Limits:** Enforce a hard ceiling of 1,000 distinct legacy status contexts per commit SHA to prevent database bloating from misconfigured CI loops.
- [ ] **Status Rollup Algorithm:** Implement the `/commits/{ref}/status` endpoint logic in Rust: query all associated checks and legacy statuses, returning `failure` if any single check fails, `pending` if any are running, and `success` only if all are green.
- [ ] **Pagination of Status Contexts:** Provide standard cursor-based pagination for the `GET /commits/{ref}/statuses` endpoint to support UI rendering when a single commit holds hundreds of CI status reports.
