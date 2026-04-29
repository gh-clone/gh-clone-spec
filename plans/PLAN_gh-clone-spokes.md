# Plan for gh-clone-spokes

### 0.4 Git Smart HTTP Protocol Implementation
- [ ] Register base route: `GET /{owner}/{repo}.git/info/refs`.
- [ ] Register base route: `POST /{owner}/{repo}.git/git-upload-pack`.
- [ ] Register base route: `POST /{owner}/{repo}.git/git-receive-pack`.
- [ ] Implement protocol parser for `/info/refs`: read `?service=` query parameter.
- [ ] Enforce 403 Forbidden if `service` is missing or not `git-upload-pack` or `git-receive-pack`.
- [ ] Set `Content-Type: application/x-git-{service}-advertisement` on `/info/refs` response.
- [ ] Implement pkt-line formatting: calculate exactly 4-byte hex lengths for every protocol line.
- [ ] Write initial service packet: `001e# service=git-upload-pack\n0000`.
- [ ] Call `git {service} --stateless-rpc --advertise-refs {repo_path}` and stream output directly to HTTP response.
- [ ] Implement capability advertisement parser to detect Git Wire Protocol v2 requests (`Git-Protocol: version=2` header).
- [ ] Validate `Content-Type: application/x-git-upload-pack-request` on POST bodies.
- [ ] Parse POST body stream and pipe directly to `git upload-pack --stateless-rpc {repo_path}`.
- [ ] Validate `Content-Type: application/x-git-receive-pack-request` on POST bodies.
- [ ] Parse POST body stream and pipe directly to `git receive-pack --stateless-rpc {repo_path}`.
- [ ] Implement zero-copy byte streaming using `tokio::io::copy` between HTTP request body and Git standard input.
- [ ] Implement zero-copy byte streaming using `tokio::io::copy` between Git standard output and HTTP response writer.
- [ ] Write HTTP middleware specific to `.git` paths to bypass standard JSON authentication and use Basic Auth (Username + PAT).
- [ ] Implement HTTP 401 challenge: `WWW-Authenticate: Basic realm="GitHub"` if `.git` request lacks auth headers.
- [ ] Implement parser for `git push -o <option>` capability over HTTP.
- [ ] Configure `http.Flusher` to immediately flush Git progression packets (e.g., `Counting objects: 100%`) to prevent client timeouts.

### 0.5 SSH Server & Git Multiplexer
- [ ] Import `russh` to construct the embedded SSH daemon.
- [ ] Configure SSH server to listen on port `22` (or configurable alternative).
- [ ] Generate default ed25519 host key for the server on first boot if `ssh_host_ed25519_key` doesn't exist.
- [ ] Define `PublicKeyHandler` callback to authenticate incoming connections.
- [ ] Extract public key from incoming connection, calculate SHA256 fingerprint.
- [ ] Query `ssh_keys` database table for matching fingerprint.
- [ ] Reject connection if key is not found or associated user is suspended.
- [ ] Map authenticated user struct into the SSH `Session.Context()`.
- [ ] Define `SessionHandler` to parse the SSH "exec" payload (the command sent by the git client).
- [ ] Parse exec payload matching `git-upload-pack '{repo_path}'`.
- [ ] Parse exec payload matching `git-receive-pack '{repo_path}'`.
- [ ] Enforce strict regex validation on `{repo_path}` to prevent directory traversal (`^[a-zA-Z0-9_.-]+/[a-zA-Z0-9_.-]+$`).
- [ ] Parse the `{owner}` and `{repo}` from the `{repo_path}` string.
- [ ] Query database to check if user has read permission (for upload-pack) or write permission (for receive-pack) on the requested repo.
- [ ] Drop SSH connection with explicit error message if permissions are insufficient.
- [ ] Execute `git-upload-pack {actual_disk_path}` via `tokio::process::Command`.
- [ ] Pipe SSH `Session` (which implements `AsyncRead/AsyncWrite`) directly to the Git subprocess `Stdin`, `Stdout`, and `Stderr`.
- [ ] Wait for Git subprocess to exit and capture its exit code.
- [ ] Send corresponding SSH exit status packet to client to terminate cleanly.
- [ ] Implement pseudo-TTY allocation rejection (PTY requests should fail gracefully, as this is a Git server, not a shell).
- [ ] Implement `KeepAlive` mechanism sending empty packets to prevent load balancers from dropping long-running git clones.
- [ ] Implement fallback interactive shell message: if user connects without an exec command (`ssh git@replica`), print "Hi {username}! You've successfully authenticated, but Replica does not provide shell access." and close connection.

### 0.6 Storage Interfaces & Bare Repo Management
- [ ] Define `storage.GitProvider` interface.
- [ ] Define `storage.GitProvider.InitBareRepo(ctx context.Context, owner, name string) error`.
- [ ] Define `storage.GitProvider.DeleteRepo(ctx context.Context, owner, name string) error`.
- [ ] Define `storage.GitProvider.GetRepoPath(owner, name string) string`.
- [ ] Implement `storage.LocalGitProvider` which stores repositories in a configured `/var/lib/replica/data/repositories` directory.
- [ ] Ensure `InitBareRepo` executes `git init --bare --initial-branch=main {path}`.
- [ ] Execute `git config core.logallrefupdates true` on newly initialized bare repos.
- [ ] Implement directory hashing to avoid filesystem limits: store repo `user/project` at `data/repositories/u/s/user/p/r/project.git`.
- [ ] Define `storage.BlobProvider` interface for avatars, attachments, and Git LFS objects.
- [ ] Define `storage.BlobProvider.Put(ctx context.Context, bucket, key string, r io.Reader) error`.
- [ ] Define `storage.BlobProvider.Get(ctx context.Context, bucket, key string) (io.ReadCloser, error)`.
- [ ] Implement `storage.LocalBlobProvider` storing files to disk.
- [ ] Implement `storage.S3BlobProvider` utilizing `aws-sdk-rust`.
- [ ] Write pre-receive Git hook binary (`replica-hook-pre-receive`).
- [ ] Write update Git hook binary (`replica-hook-update`).
- [ ] Write post-receive Git hook binary (`replica-hook-post-receive`).
- [ ] Configure `LocalGitProvider.InitBareRepo` to symlink all hooks in the new repo's `hooks/` directory to the replica hook binaries.
- [ ] Design internal RPC endpoint `POST /internal/api/pre-receive` for hooks to query the replica server for branch protection validation.
- [ ] Design internal RPC endpoint `POST /internal/api/post-receive` for hooks to trigger webhook deliveries and database ref synchronization.

### 3.1 Gitaly RPC & Storage Sharding (Spokes/Dgit Architecture)
- [ ] **Define `UploadPack` gRPC Protobuf schema** for fetching operations (pull/clone/fetch).
- [ ] **Define `ReceivePack` gRPC Protobuf schema** for mutation operations (push).
- [ ] **Define `InfoRefs` gRPC Protobuf schema** for reference and capability discovery.
- [ ] **Define `ObjectInfo` and `CommitGraph` gRPC Protobuf schemas** for internal server-side metadata queries.
- [ ] **Implement gRPC mutual TLS (mTLS)** authentication to encrypt traffic between API nodes and Storage nodes.
- [ ] **Configure gRPC connection pooling and keep-alives** (TCP keepalive, HTTP/2 PING) to prevent idle connections from dropping during long git operations.
- [ ] **Implement gRPC interceptors for distributed tracing** (OpenTelemetry) to trace multi-node repository requests.
- [ ] **Implement gRPC interceptors for Prometheus metrics** (request duration, payload size, error rates per RPC).
- [ ] **Implement an RPC-level rate-limiting middleware** dynamically driven by tenant/repository usage quotas.
- [ ] **Integrate `jump-consistent-hash`** algorithm for calculating logical partition IDs from repository hashes.
- [ ] **Implement a murmur3 hashing function** to convert repository UUID strings into numerical keys for the jump ring.
- [ ] **Develop an in-memory routing table** that maps logical partition IDs to physical storage node gRPC endpoints.
- [ ] **Set up etcd/ZooKeeper client integration** for dynamic storage node discovery and cluster topology management.
- [ ] **Implement heartbeat mechanisms** for storage nodes to register their presence and disk capacity in etcd.
- [ ] **Create a 3-node replica quorum topology** per repository (1 Primary, 2 Secondaries) based on GitHub Spokes design.
- [ ] **Implement the Reference Transaction Hook** (`reference-transaction`) for 3-phase commits during `git push`.
- [ ] **Implement Phase 1 (Pre-Vote):** Verify disk space, lock the repository path on all 3 nodes concurrently.
- [ ] **Implement Phase 2 (Write):** Stream the packfile payload concurrently over RPC to all 3 nodes.
- [ ] **Implement Phase 3 (Commit):** Execute reference updates only if >= 2 nodes (quorum) report successful pack indexing.
- [ ] **Implement rollback and cleanup logic** if the quorum fails during Phase 1 or 2 to prevent detached objects.
- [ ] **Implement etcd-based Leader Election** to designate a Primary replica for each partition dynamically.
- [ ] **Implement RPC fencing tokens** to reject writes from deposed Primaries, guaranteeing split-brain protection.
- [ ] **Build a Read-Repair heuristic** triggered during `UploadPack` to compare replica checksums of `packed-refs`.
- [ ] **Implement asynchronous `git fetch` tasks** to heal diverging Secondaries detected via the Read-Repair mechanism.
- [ ] **Configure Kafka topics (`repository_mutations`)** with schema registries (Avro/Protobuf) for all successful pushes.
- [ ] **Implement Kafka Producer logic on Storage nodes** to emit events (RepoID, TransactionID, UpdatedRefs).
- [ ] **Implement an Idempotency Key system** to ensure Kafka events are not processed multiple times.
- [ ] **Build asynchronous Kafka Consumer workers** (the "Replicator") to continuously synchronize offline or newly provisioned nodes.
- [ ] **Implement an exponential backoff strategy** for Replicator failures (e.g., target node still down).
- [ ] **Implement a Dead Letter Queue (DLQ)** for replication jobs that fail repeatedly after maximum retries.
- [ ] **Implement an automated Git Garbage Collection (`git gc`) scheduler** that runs safely on storage nodes during off-peak hours.
- [ ] **Implement automated multi-pack-index (MIDX)** and bitmap generation during background GC for faster graph queries.
- [ ] **Implement commit-graph file updates** during background GC to accelerate future `have`/`want` traversals.
- [ ] **Implement a health-check endpoint on storage nodes** that rigorously verifies read/write capabilities to the underlying filesystem.
- [ ] **Implement failover routing logic** in the API tier to instantly and transparently switch to a Secondary if the Primary times out.

### 3.2 Smart HTTP (`git-httpd`) Wire Protocol Mechanics
- [ ] **Configure Axum routing** to handle `GET /<user>/<repo>/info/refs` with dynamic URL path parameter extraction.
- [ ] **Implement HTTP Basic Auth and OAuth Bearer token validation** middleware specifically for Git API routes.
- [ ] **Strictly validate the `service` query parameter** to only allow `git-upload-pack` and `git-receive-pack`.
- [ ] **Enforce `Content-Type: application/x-git-upload-pack-advertisement`** on the response for `info/refs`.
- [ ] **Write a Pkt-Line encoder function** to perfectly format exactly 4-byte hex lengths (e.g., `001e`).
- [ ] **Implement Pkt-Line flush packet (`0000`)** and delimiter packet (`0001`) encoding structures.
- [ ] **Dynamically inject the `service=<service_name>` Pkt-Line header** exactly at the start of the `info/refs` stream.
- [ ] **Implement Git Protocol v2 capability advertisement**, actively checking the `GIT_PROTOCOL` HTTP header.
- [ ] **Expose `ls-refs`, `fetch`, `server-option`, and `object-format=sha1/sha256` capabilities** in the v2 payload.
- [ ] **Expose the `filter` capability** to support partial clones (e.g., `--filter=blob:none`, `--filter=tree:0`).
- [ ] **Configure Axum routing for `POST /<user>/<repo>/git-upload-pack`**.
- [ ] **Enforce `Content-Type: application/x-git-upload-pack-request`** header validation.
- [ ] **Handle gzip/deflate `Content-Encoding` unpacking** to seamlessly support compressed incoming POST bodies.
- [ ] **Implement an asynchronous streaming Pkt-Line parser** to decode the incoming HTTP request body chunk-by-chunk without full memory loading.
- [ ] **Implement state machine logic to parse standard capabilities:** `want <SHA>`, `have <SHA>`, and `deepen <depth>`.
- [ ] **Implement Git Protocol v2 specific `command=fetch`** argument parsing.
- [ ] **Implement DoS protection limit:** Cap maximum `want` declarations to 100,000 per request.
- [ ] **Implement DoS protection limit:** Cap maximum `have` declarations and limit graph traversal depths to prevent CPU exhaustion.
- [ ] **Implement an absolute timeout** (e.g., 60 seconds) strictly for the packet negotiation phase.
- [ ] **Integrate `gitoxide` commit-graph loading** directly into the memory space of the RPC nodes.
- [ ] **Implement fast ancestry graph traversal using `gitoxide`** by mapping `want` OIDs as targets and `have` OIDs as uninteresting boundary commits.
- [ ] **Implement Bloom filter lookups** to rapidly bypass full commit parsing for paths/OIDs that obviously don't exist.
- [ ] **Identify the exact minimal object set** (commits, trees, blobs) required to satisfy the negotiation graph difference.
- [ ] **Implement dynamically generated thin-packs using `gitoxide`**, specifically omitting base objects already present in client `have` commits.
- [ ] **Enable `--reuse-delta` fast-paths** to aggressively copy existing delta-compressed chunks directly from disk to the network buffer.
- [ ] **Implement the Sideband-64k multiplexing state machine**.
- [ ] **Wrap binary packfile chunks** in strict `\x01` sideband headers (maximum 65520 bytes per chunk).
- [ ] **Implement dynamic, real-time progress stream calculation** (e.g., "Counting objects: 100% (50/50), done.").
- [ ] **Wrap textual progress logs in `\x02` sideband headers** and stream them concurrently with pack data.
- [ ] **Implement a panic/error capture mechanism** to cleanly send fatal errors via `\x03` sideband headers before closing the connection.
- [ ] **Implement keep-alive packet emission** (`0005\x02`) during slow graph traversals to prevent intermediate load-balancer timeouts.
- [ ] **Stream the final multiplexed byte stream** back through Axum utilizing `Transfer-Encoding: chunked`.
- [ ] **Disable Nginx/HAProxy/ALB HTTP buffering** via headers (e.g., `X-Accel-Buffering: no`) to ensure real-time progress UX for the client.
- [ ] **Implement graceful client disconnect handling** (TCP RST) to immediately interrupt and terminate backend gitoxide/packing threads.

### 3.3 Git LFS & S3 Handoff
- [ ] **Compile a standalone Rust binary** explicitly designed to act as the `pre-receive` hook inside the Storage nodes' `.git/hooks` directories.
- [ ] **Implement STDIN reading inside the pre-receive hook** to reliably capture `old-oid new-oid refname` triples.
- [ ] **Implement strict environment variable parsing** within the hook to securely capture `GL_ID`, `GL_REPOSITORY`, and API routing tokens injected by Gitaly.
- [ ] **Implement `git rev-list --objects $old..$new` execution** within the hook to iterate over all newly pushed blobs.
- [ ] **Implement a fast heuristic scanner** that checks only the first 100 bytes of new blobs for the LFS signature (`version https://git-lfs.github.com/spec/v1`).
- [ ] **Implement strict Regex parsing for the OID** (`oid sha256:[a-f0-9]{64}`) from successfully detected LFS pointers.
- [ ] **Implement strict Regex parsing for the object size** (`size [0-9]+`) from detected LFS pointers.
- [ ] **Ensure pushed files matching the LFS signature but exceeding 1KB are automatically rejected** (LFS pointer spoofing protection).
- [ ] **Perform a synchronous HTTP RPC call from the pre-receive hook** to the API node to rigorously validate LFS pointer existence and available storage quotas.
- [ ] **Implement `POST /<user>/<repo>.git/info/lfs/objects/batch`** routing in Axum.
- [ ] **Enforce strict validation of `Accept: application/vnd.git-lfs+json`** and `Content-Type` HTTP headers.
- [ ] **Deserialize the Git LFS Batch JSON payload** into strict, type-safe Rust structs (`operation: upload/download`, `transfers`, `objects`).
- [ ] **Set up a dedicated PostgreSQL connection pool** specifically optimized for LFS metadata queries.
- [ ] **Implement an efficient bulk SQL query** utilizing `SELECT oid FROM lfs_objects WHERE oid = ANY($1) AND repo_id = $2`.
- [ ] **Partition the client's requested OIDs into two definitive vectors:** `existing_objects` and `missing_objects`.
- [ ] **Initialize the `aws-sdk-s3` client** securely with IAM roles, custom endpoint support (for MinIO/R2 dev parity), and correct region configs.
- [ ] **Generate S3 Pre-Signed PUT URLs for all `missing_objects`** utilizing the AWS SDK.
- [ ] **Configure the PUT pre-signed URL to strictly enforce** `Content-Type: application/octet-stream`.
- [ ] **Configure the PUT pre-signed URL to cryptographically enforce** exact file sizes matching the client's reported `size`.
- [ ] **Set a strict, non-negotiable 15-minute expiration timestamp** (`expires_at`) for all generated pre-signed URLs.
- [ ] **Generate S3 Pre-Signed GET URLs for `existing_objects`** during standard `download` operations.
- [ ] **Serialize the complex LFS Batch API JSON response** accurately (mapping `actions.upload.href`, `header`, and precise `expires_in` times).
- [ ] **Implement Redis-based atomic quota increments** for the user dynamically based on the aggregate byte sizes of `missing_objects`.
- [ ] **Implement a `POST /info/lfs/objects/verify` endpoint** to handle client post-upload verifications per the LFS specification.
- [ ] **Validate the uploaded object in S3** using a `HeadObject` API call to definitively ensure the `Content-Length` matches the PG metadata.
- [ ] **Atomically mark the object as `verified = true`** in the PostgreSQL `lfs_objects` table.
- [ ] **Build a nightly cron worker to garbage-collect** (hard delete from S3) any uploaded LFS objects older than 24 hours where `verified = false`.
## 4. THE COMPUTATIONAL ROADMAP & SEARCH ENGINE

### 5.4 Git Routing, Proxying & Storage (Spokes / Babeld)
- [ ] **`babeld` Proxy Implementation:** Develop a highly concurrent Go-based proxy (`babeld` equivalent) to handle incoming Git over SSH and HTTPS traffic multiplexing.
- [ ] **SNI & Protocol Termination:** Configure the `babeld` proxy to terminate SSL/TLS and route traffic based on SNI (Server Name Indication) and SSH ALPN headers.
- [ ] **Spokes Replication Engine:** Implement a Spokes-like repository replication orchestrator to ensure every Git repository maintains a strict three-node consensus replica.
- [ ] **Distributed Lock Manager:** Deploy ZooKeeper or a Consul-backed locking mechanism to serialize concurrent Git pushes to the same repository across the global cluster.
- [ ] **Git Hook Interceptors:** Implement highly optimized `pre-receive` and `post-receive` Git hooks in Rust to intercept commits, validate GPG signatures, and enforce branch protection rules before writing objects.
- [ ] **`git-daemon` Optimization:** Tune the internal `git-daemon` to maximize throughput for anonymous, unauthenticated fetch traffic on public repositories.
- [ ] **Storage Anti-Affinity:** Define strict Kubernetes pod anti-affinity rules for `storage-tier` StatefulSets to guarantee repository replicas are physically spread across distinct AWS Availability Zones.
- [ ] **Distributed Tracing Headers:** Inject W3C Trace Context headers into all payloads at the proxy layer to maintain distributed traces through the entire Git lifecycle.


### 3.4 Deploy Keys & Fine-Grained Authentication
- [ ] **Deploy Keys Database Schema:** Define the `deploy_keys` Postgres table linking a specific SSH public key exclusively to a single `repository_id`.
- [ ] **Read/Write Scopes for Deploy Keys:** Implement the boolean `read_only` flag on the deploy key, enforcing this state strictly during the Git push (`receive-pack`) authorization phase.
- [ ] **SSH Multiplexer Deploy Key Override:** Modify the SSH `PublicKeyHandler` to search the `deploy_keys` table if the fingerprint is not found in the global user `ssh_keys` table.
- [ ] **Deploy Key Context Mapping:** Ensure a connection authenticated via a Deploy Key creates an anonymous session context specifically bound to the repository, bypassing normal user identity checks.
- [ ] **Fine-Grained PAT Infrastructure:** Define the `fine_grained_pats` Postgres schema to support resource-specific permissions (e.g., Read on `repo:123`, Write on `repo:456`).
- [ ] **Permission Matrix Compilation:** Implement a Rust engine to compile the Fine-Grained PAT's JSON payload of rules into an efficient bitmask or fast-lookup hash map stored in Redis.
- [ ] **Endpoint Level Authorization Middleware:** Update the Axum REST middleware to evaluate Fine-Grained PAT claims against the requested resource (e.g., verifying `issues:write` scope strictly for the targeted repository).
- [ ] **Token Expiration & Rotation Enforcement:** Build background workers that automatically invalidate Fine-Grained PATs based on their strictly required expiration dates (maximum 1 year).

### 3.4 Archive Generation & Streaming
- [ ] **Git Archive RPC:** Define a specific Gitaly/Spokes gRPC interface (`GetArchive`) that requests a specific repository and commit SHA to be compressed.
- [ ] **Streaming Compression:** Implement the storage-node execution of `git archive --format=zip` or `tar`, piping stdout directly into the gRPC bidirectional stream to bypass memory buffering.
- [ ] **In-Flight Caching:** Configure the API edge nodes to transparently cache the resulting ZIP payloads for frequently accessed release tags (e.g., `v1.0.0.zip`), bypassing the storage nodes entirely during viral traffic spikes.

### 3.5 Git LFS File Locking Protocol
- [ ] Implement the complete Git LFS File Locking API (`POST /info/lfs/locks`, `GET /info/lfs/locks`, `POST /info/lfs/locks/{id}/unlock`, `POST /info/lfs/locks/verify`).
- [ ] Define the `lfs_locks` Postgres table: `id`, `repo_id`, `path`, `owner_id`, `created_at`.
- [ ] Implement the `POST /info/lfs/locks` endpoint to securely lock a file path, ensuring atomic inserts to prevent race conditions during concurrent lock attempts by game developers.
- [ ] Build the strict verification engine inside the `pre-receive` Git hook: when a user pushes commits, strictly verify that any modified files matching LFS paths are either unlocked, or explicitly locked by the pushing user.
- [ ] Reject the `git push` directly in the `pre-receive` hook with a standard LFS locking error message if a file is locked by a different user.
- [ ] Implement the "Force Unlock" capability (`POST /info/lfs/locks/{id}/unlock`), enforcing RBAC to ensure only the lock owner or a Repository Admin can forcefully drop a lock.
- [ ] Render an Angular UI inside the repository "Settings" or "Code" view displaying all actively locked LFS paths, their owners, and timestamps.

### Tier 4: Subversion (SVN) Client Bridge & Custom Hooks
- [ ] **SVN Protocol Gateway:** Implement an SVN network protocol parser natively in Rust, listening on port 3690 (svn://) and multiplexing over HTTP (WebDAV).
- [ ] **WebDAV Support:** Handle SVN-specific XML namespaces and HTTP methods (`PROPFIND`, `REPORT`, `OPTIONS`, `MKACTIVITY`, `CHECKOUT`, `PUT`) required by standard SVN clients.
- [ ] **Dynamic Revision Mapping:** Map linear SVN revision numbers to non-linear Git commit SHAs dynamically, utilizing a fast Redis caching layer for the translation table.
- [ ] **On-the-Fly Tree Generation:** Translate SVN `checkout` and `update` commands into virtualized `git archive` operations or rapid tree-walks via `gitoxide`.
- [ ] **Delta Payload Processing:** Support SVN `commit` by parsing the SVN delta payloads and converting them directly into Git Blob and Tree objects in-memory before writing the Git commit.
- [ ] **Ignore Rule Translation:** Parse `.gitignore` rules in the Git tree and enforce them during SVN addition commands to maintain repository consistency.
- [ ] **Branch/Tag Emulation:** Emulate SVN's `branches/` and `tags/` directory structure conventions by mapping them natively to actual Git branches and tags on the backend.
- [ ] **Enterprise Pre-Receive Hook Execution:** Extend the native `replica-hook-pre-receive` binary to dynamically query and fetch globally enforced Enterprise hooks from the API.
- [ ] **Hook Sandboxing:** Initialize a strict `libmountsandbox` isolated execution context for each custom Bash/Python hook to prevent host breakout and lateral movement.
- [ ] **Read-Only Context:** Mount the target Git repository strictly as read-only within the hook sandbox to ensure the hook cannot secretly mutate the repository state.
- [ ] **Resource Quotas:** Enforce hard timeouts (e.g., 5000ms) and CPU limits to prevent poorly written custom Bash scripts from causing denial-of-service during Git pushes.
- [ ] **Sideband Streaming:** Pipe standard error (`stderr`) and output (`stdout`) from the sandboxed hook directly back to the Git client's terminal using Git Sideband-64k packets.

### Tier 4.1: Subversion (SVN) Client Bridge (Deep Implementation)
- [ ] **XML-RPC / WebDAV Engine:** Utilize `quick-xml` and `axum` to build a highly concurrent HTTP-based WebDAV engine parsing SVN `PROPFIND` and `REPORT` requests without heavy memory allocation.
- [ ] **Delta-V Versioning Protocol:** Implement Delta-V HTTP extensions mapping `MKACTIVITY`, `CHECKOUT`, and `MERGE` methods directly to virtual Git staging index creations.
- [ ] **`vcc` (Version-Controlled Configuration) Resolution:** Map SVN VCC paths securely to backend repository bare paths, enforcing authentication on the initial `OPTIONS` handshake.
- [ ] **SVN Revision Mapping Index:** Build a secondary Postgres table `svn_revisions` mapping sequential integer SVN revisions to 40-character Git SHAs to enable O(1) history lookups.
- [ ] **Git to SVN Property Emulation:** Read Git attributes (`.gitattributes`) and file modes (`100755`) to synthesize equivalent `svn:executable` and `svn:mime-type` properties in the WebDAV response.
- [ ] **`update-report` Translation Engine:** Implement the SVN `update-report` command by executing a `git diff-tree` between the requested SVN revision's corresponding Git SHA and the `HEAD` SHA.
- [ ] **`log-report` Commit History:** Translate SVN `log-report` requests into highly optimized `gitoxide` commit graph traversals, formatting Git commit authors, dates, and messages into standard SVN XML formats.
- [ ] **Inline Delta Compression (svndiff):** Implement the `svndiff` (v0/v1) binary encoding format natively in Rust to stream compressed file diffs back to the SVN client during `update` operations.
- [ ] **TxDelta Payload Ingestion:** Parse incoming `svndiff` delta payloads from SVN `PUT` requests, reconstructing the full file blob in-memory and hashing it for Git object insertion.
- [ ] **SVN Directory Layout Emulation:** Map the implicit Git repository root to the SVN `trunk/` directory, and map Git branches natively to the `branches/` virtual directory.
- [ ] **SVN Locking Mechanism Emulation:** Intercept SVN `LOCK` requests and translate them strictly to the Git LFS File Locking API, ensuring locks are respected across both protocols.

### Tier 4.2: Advanced Pre-Receive Hook Sandboxing
- [ ] **gRPC Hook Broker:** Implement a local gRPC server within the storage nodes specifically to receive and execute custom enterprise hook payloads requested by the `pre-receive` binary.
- [ ] **`libmountsandbox` cgroups Profiling:** Apply strict Linux `cgroups v2` resource controls via the sandbox: restrict custom bash/python hooks to 1 CPU core and a maximum of 256MB RAM.
- [ ] **OverlayFS Read-Only Mounts:** Utilize `overlayfs` to mount the entire target bare Git repository into the sandbox namespace strictly as `ro` (read-only) to guarantee hooks cannot bypass Git object immutability.
- [ ] **Network Namespace Dropping:** Explicitly drop the `CLONE_NEWNET` namespace within the hook sandbox to strictly prevent custom enterprise hooks from making unauthorized outbound HTTP requests (data exfiltration).
- [ ] **Standard IO Multiplexing:** Capture `stdout` and `stderr` streams from the sandboxed hook process via Unix Domain Sockets, chunking the output securely into Git Sideband-64k packets (`\x02`).
- [ ] **Execution Chain Short-Circuiting:** Implement fail-fast logic for execution chains: if Hook A exits with a non-zero status code, instantly terminate the push without evaluating Hook B or Hook C.
- [ ] **Push Option Variable Injection:** Parse Git push options (`git push -o key=value`) and securely inject them into the hook's sandbox environment variables (`GIT_PUSH_OPTION_0=key=value`).
- [ ] **Hook Timeout Watchdog:** Implement a strict `tokio::time::timeout` wrapper on the hook execution future; if the hook hangs beyond 10 seconds, send a `SIGKILL` to the sandbox and reject the Git push with a timeout error.
