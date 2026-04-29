# Plan for gh-clone-actions-runner

### Tier 3: GitHub Actions Control Plane & Runner Isolation

#### Workflow Parsing & Validation
- [ ] Build a YAML parser with custom anchors/aliases restriction for strict security enforcement.
- [ ] Implement schema validation against a formal JSON Schema for `.github/workflows/*.yml` files.
- [ ] Build syntax checking for GitHub expression language (`${{ ... }}`) detecting invalid object paths.
- [ ] Implement static analysis to detect circular dependencies in `needs` arrays prior to DAG construction.
- [ ] Resolve and inject reusable workflows (`workflow_call`) across repository boundaries considering access controls.
- [ ] Parse and evaluate `on:` triggers to match incoming webhook payloads precisely (e.g., specific branches, paths, or tags).
- [ ] Implement `matrix` strategy expansion, dynamically generating an N-dimensional Cartesian product of job definitions.
- [ ] Handle `exclude` and `include` directives within matrix generation to prune or append specific job variants.
- [ ] Resolve composite actions by fetching `action.yml`, recursively parsing and injecting nested steps.
- [ ] Perform static type checking on inputs and outputs defined in composite and reusable workflows.

#### API & YAML Conformance Testing
- [ ] **Schema Ingestion:** Automatically ingest and sync the official GitHub Actions JSON schema from SchemaStore to ensure baseline YAML parser compatibility.
- [ ] **Public Corpus Fuzzing:** Clone 10,000+ `.github/workflows/*.yml` files from the most starred public GitHub repositories to use as a fuzzer corpus for the parser.
- [ ] **Expression Evaluation Parity:** Implement a test suite specifically targeting edge cases in the `${{ }}` expression language (e.g., `contains()`, `startsWith()`, `toJSON()`, `fromJSON()`, and exact object property filtering logic).
- [ ] **Matrix Expansion Verification:** Write assertions to guarantee that `matrix` expansion (including the notoriously complex `include`/`exclude` overriding rules) yields the exact same Cartesian output as GitHub's scheduler.
- [ ] **REST API Mocking:** Implement integration tests against mock payloads mimicking the official GitHub REST APIs (e.g., `/repos/{owner}/{repo}/actions/runs`, `/repos/{owner}/{repo}/actions/jobs/{job_id}`).
- [ ] **Toolkit Command Parsing:** Validate 100% compliance with the undocumented behaviors of the `@actions/core` toolkit, ensuring `::set-output`, `::add-mask`, `::stop-commands`, and multi-line delimiter (`<<EOF`) parsing matches upstream exactly.
- [ ] **Cache/Artifact API Parity:** Construct HTTP conformance tests ensuring our internal `@actions/cache` and `@actions/artifact` endpoints perfectly mirror the undocumented HTTP verbs, headers, and status codes expected by the official GitHub javascript packages.
- [ ] **OIDC Claims Validation:** Test OIDC token issuance to guarantee claims (`aud`, `sub`, `iss`, `repository`, `job_workflow_ref`) exactly map to the standard claims expected by AWS/GCP/Azure federation gateways.
- [ ] **Lifecycle Hook Parity:** Ensure the strict sequential execution order of `pre:`, `main:`, and `post:` blocks in `action.yml` files (including when steps fail or time out) precisely matches GitHub Actions' behavior.

#### DAG Scheduling & Execution
- [ ] Construct an in-memory job graph tracking node status (pending, running, success, failure, cancelled).
- [ ] Implement `concurrency` groups evaluating `cancel-in-progress` flags to automatically preempt older jobs.
- [ ] Build an environment protection rule engine requiring manual approvals before job execution transitions.
- [ ] Integrate with external deployment gate APIs to hold job execution pending an external status check.
- [ ] Implement conditional execution evaluation (`if: always()`, `if: failure()`) based on upstream graph states.
- [ ] Manage workflow-level timeouts and job-level timeouts, actively killing jobs exceeding the SLA.
- [ ] Implement a distributed lock mechanism to enforce concurrency limits across multiple scheduler nodes.
- [ ] Design an anti-entropy sweep process to reap "zombie" jobs whose runners disconnected without completing.
- [ ] Build a rate-limiter for job dispatch based on organization-level concurrency quotas.
- [ ] Maintain a highly available Postgres table for real-time workflow run status querying.

#### Job Dispatch & Queueing
- [ ] Create a multi-tenant queueing system using Postgres `FOR UPDATE SKIP LOCKED` for reliable, lock-free dequeueing.
- [ ] Implement a long-polling API for self-hosted runners using HTTP/1.1 `Transfer-Encoding: chunked` or WebSockets.
- [ ] Implement label-matching routing, ensuring jobs only target runners possessing the exact requested `runs-on` labels.
- [ ] Build ephemeral runner support, ensuring a single job consumes a runner before automatically unregistering it.
- [ ] Generate unique registration tokens (JWTs) for users to securely connect self-hosted runners to a repository or organization.
- [ ] Track runner heartbeat timestamps to dynamically mark unresponsive agents as offline and requeue their active jobs.
- [ ] Emit metrics on queue wait times and runner availability to inform horizontal pod autoscalers (e.g., KEDA).

#### The Runner Agent Application
- [ ] Implement the core runner loop in .NET or Rust, handling job acceptance, execution, and reporting.
- [ ] Provide an isolated execution environment for Javascript actions utilizing a bundled Node.js runtime.
- [ ] Build the Docker execution engine to `docker pull`, `docker create`, and execute steps directly inside containers.
- [ ] Inject the `${GITHUB_WORKSPACE}` directory into container steps via bind mounts, mapping user permissions correctly.
- [ ] Implement a command parser to intercept stdout magic commands (e.g., `echo "::set-output name=foo::bar"`).
- [ ] Manage the `GITHUB_ENV` file, allowing steps to dynamically append environment variables for subsequent steps.
- [ ] Handle pre- and post-step phases defined in `action.yml` to support cleanup logic (e.g., stopping background processes).
- [ ] Manage the `GITHUB_PATH` file, mutating the runtime `$PATH` to expose newly installed binaries to later steps.
- [ ] Mask job secrets from all runner output streams before transmission back to the control plane.
- [ ] Handle SIGINT and SIGTERM gracefully, attempting to clean up Docker containers and orphan processes before exit.

#### Live Log Streaming & Processing
- [ ] Design a gRPC streaming API for the runner to continuously push chunked stdout/stderr data.
- [ ] Build an in-memory ring buffer on the backend to ingest highly concurrent log streams.
- [ ] Implement line-level timestamping and ANSI color code preservation for accurate UI rendering.
- [ ] Build a high-performance Aho-Corasick automaton to perform secondary secret redaction on the backend in real-time.
- [ ] Flush aggregated log chunks to Azure Blob Storage or S3, utilizing indexed byte offsets for fast retrieval.
- [ ] Expose an HTTP API allowing the web frontend to tail active logs via Server-Sent Events (SSE).
- [ ] Generate a final, compressed text archive of the entire job execution upon completion for permanent storage.

#### Artifacts & Cache Storage
- [ ] Implement the `@actions/artifact` API endpoints for uploading multi-part chunks of arbitrary files.
- [ ] Design a zstd-compressed TAR archiving strategy to minimize artifact storage footprint.
- [ ] Implement deduplication hashing on artifact chunks to prevent storing identical files multiple times.
- [ ] Build the `@actions/cache` endpoints utilizing cache keys, restore keys, and version hashes.
- [ ] Implement an LRU eviction policy service to delete the least recently used caches when storage quotas are exceeded.
- [ ] Handle parallel cache restore operations using aggressive pre-signed URL redirection directly to cloud storage.
- [ ] Enforce repository-scoped and branch-scoped boundaries, preventing cross-branch cache poisoning.
- [ ] Develop background TTL sweepers to permanently delete expired artifacts after 90 days.

#### Secrets, Variables & OIDC
- [ ] Implement AES-256-GCM encryption for all secrets at rest in the database via an envelope encryption scheme.
- [ ] Build an hierarchical resolution system for overriding Organization -> Repository -> Environment variables.
- [ ] Expose a standard OIDC discovery endpoint (`/.well-known/openid-configuration`).
- [ ] Generate ephemeral JWKS keys periodically to sign outgoing OIDC identity tokens securely.
- [ ] Mint short-lived JWTs (OIDC tokens) containing standard claims (`sub`, `aud`) mapped to the repository and branch context.
- [ ] Decrypt secrets *only* at the moment of job dispatch, sending them over TLS directly into the runner's memory.
- [ ] Implement restricted scopes, ensuring jobs running from forks cannot access sensitive repository secrets without approval.
- [ ] Manage permissions scopes (e.g., `GITHUB_TOKEN`) with granular read/write access defined in the workflow YAML.

#### Runner Host Execution & Sandboxing (Powered by libmountsandbox)

##### Universal Sandbox Integration
- [ ] **Cross-Platform Engine:** Integrate `libmountsandbox` via FFI (Rust/Go/.NET) as the core unified sandboxing abstraction across all runner environments.
- [ ] **Dynamic Engine Selection:** Route execution context to the optimal `sandbox_engine_t` backend (`native`, `docker`, `podman`, `wasmtime`) based on the job step's requirements.
- [ ] **Unified Resource Quotas:** Utilize the cross-platform `sandbox_config_t` to enforce hard CPU, memory, and millisecond-polled timeout limits on steps irrespective of the host OS.
- [ ] **Granular Mount Policies:** Implement a "deny-by-default" file policy via `libmountsandbox`. Mount only the `${GITHUB_WORKSPACE}` as read-write, strictly setting system libraries and runner binaries to read-only.
- [ ] **Network Toggle:** Utilize the universal network "kill switch" in `libmountsandbox` to drop network namespaces/interfaces for offline build steps to prevent data exfiltration.

##### Linux Host & Firecracker MicroVMs
- [ ] **bwrap Namespaces:** Utilize the `libmountsandbox` native Linux engine to spawn Bubblewrap (`bwrap`) unprivileged User, Mount, PID, and Network namespaces instead of raw `clone()`.
- [ ] **Seccomp Filters:** Attach the CI-safe Seccomp BPF profile provided by `libmountsandbox` to automatically block `mount`, `ptrace`, and kernel modifications.
- [ ] **Cgroups Enforcement:** Delegate CPU (`cpu.max`) and memory (`memory.max`) constraints to the `libmountsandbox` Cgroups implementation.
- [ ] **Firecracker CI:** Automate the creation of Linux tap devices (`TUNSETIFF`) and bridge networking prior to microVM boot.
- [ ] **Firecracker CI:** Generate a highly minimal `vmlinux` kernel config, stripping out unneeded modules to achieve sub-20ms microVM boot times.
- [ ] **Firecracker CI:** Interface with the Firecracker REST API via Unix Domain Sockets to configure `boot-source` and block `drives`.
- [ ] **Firecracker CI:** Utilize sparse ext4 filesystem templates overlaid with device mapper snapshots for instantaneous block device provisioning.

##### Native Windows Support
- [ ] **Job Objects:** Leverage the `libmountsandbox` Windows native engine to isolate processes within Job Objects, tracking total memory limits and CPU constraints without heavy VM overhead.
- [ ] **AppContainer Execution:** Configure `libmountsandbox` to spin up AppContainers for strict filesystem and network isolation in Windows runner steps.
- [ ] **Runner Agent:** Compile the core runner daemon natively for `x86_64-pc-windows-msvc` and `aarch64-pc-windows-msvc`.
- [ ] **Windows Containers:** Automate the provisioning of Host Compute Service (HCS) via `hcsshim` for executing Windows Server Core/Nano containers.
- [ ] **Filesystem:** Utilize ReFS block cloning to achieve near-instant copy-on-write workspace restoration between sequential jobs.

##### Native macOS Support
- [ ] **Seatbelt Isolation:** Utilize the `libmountsandbox` macOS native engine, wrapping command executions in `sandbox-exec` with dynamically generated Scheme profiles.
- [ ] **Hypervisor Lifecycle:** Automate the Apple `Virtualization.framework` (`VZVirtualMachine`) lifecycle for ephemeral Xcode CI jobs.
- [ ] **Filesystem Isolation:** Utilize APFS Copy-on-Write (CoW) snapshots of base macOS `VZMacRestoreImage` bundles to instantly reset the OS state between jobs.
- [ ] **Guest Communication:** Implement a high-speed daemon to proxy Xcode build logs from the macOS guest to the host control plane via isolated `virtio-vsock` (vSockets).
- [ ] **Developer Tooling:** Inject Apple developer certificates, provisioning profiles, and code-signing identities dynamically into the guest Keychain at startup.

##### Docker Container Actions & Services
- [ ] **Container Engine:** Proxy the Docker Engine configuration through the `libmountsandbox` Docker backend to uniformly handle mounts, network toggles, and memory limits via a unified API.
- [ ] **Job Services:** Parse `services:` declarations to spin up detached background containers (e.g., PostgreSQL, Redis) mapped to the job's dedicated bridge network.
- [ ] **Health Checks:** Implement polling routines evaluating the `HEALTHCHECK` status of service containers before allowing workflow steps to proceed.
- [ ] **Permissions Mapping:** Dynamically inject `USER` and map the host UID/GID to the container user to prevent root-owned artifact creation.
- [ ] **Network Isolation:** Provision a unique `docker network create` per workflow execution to prevent parallel workflows on the same host from sniffing traffic.
- [ ] **Cleanup:** Rely on `libmountsandbox` bounded execution contexts and timeout polling to rigidly enforce cleanup of orphan containers and resources upon exit.## 5. OMNI-PLATFORM DEPLOYMENT & PACKAGING DEEP DIVE

