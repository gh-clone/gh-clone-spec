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
- [ ] Brotli-compress the PNG buffer and cache it in `moka` with a 30-day TTL.
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
- [ ] Initialize Consumer `metrics_aggregator`: Read API usage metrics stream and push to InfluxDB/Prometheus pushgateway for billing calculation.