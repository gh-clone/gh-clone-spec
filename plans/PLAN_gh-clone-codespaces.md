# Plan for gh-clone-codespaces

### Tier 1: Devcontainer Parsing & Image Orchestration
- [ ] Implement a strict JSON/JSONC parser in Rust to ingest `devcontainer.json` files, supporting comments and trailing commas.
- [ ] Build the `features` resolution engine to dynamically download and inject Devcontainer Features (e.g., `ghcr.io/devcontainers/features/rust:1`) into the base image.
- [ ] Parse and execute `updateContentCommand`, `postCreateCommand`, and `postStartCommand` lifecycle hooks strictly within the generated container.
- [ ] Implement a distributed image caching layer: pre-build and cache common `devcontainer.json` configurations (e.g., standard Node/Python templates) to an internal registry to achieve <10s start times.
- [ ] Build an AST generator that translates `devcontainer.json` configurations into highly optimized Dockerfiles for the builder orchestrator.
- [ ] Implement `docker-compose.yml` parsing within the devcontainer context to support multi-container developer environments.
- [ ] Build a background Rust worker that automatically pre-builds devcontainers on `git push` to the default branch to enable instant Codespace creation for all repository contributors.

### Tier 2: Compute Isolation & Firecracker Orchestration
- [ ] Develop the `codespace-orchestrator` service in Rust to interface with AWS EC2 Bare Metal or Kubernetes for fleet management.
- [ ] Integrate Firecracker microVMs to provide hardware-level isolation for each Codespace instead of relying solely on Docker namespaces.
- [ ] Implement instantaneous volume snapshotting using `ext4` sparse files and devicemapper to persist the `/workspaces` directory across Codespace suspends/resumes.
- [ ] Design the Resource Quota engine: enforce hard RAM (e.g., 4GB, 8GB, 32GB) and CPU limits (e.g., 2, 4, 16 cores) mapped directly to billing tiers.
- [ ] Implement an idle detection daemon inside the Codespace that monitors SSH/HTTP traffic and `ttys` to automatically suspend the microVM after 30 minutes of inactivity.
- [ ] Configure `systemd` inside the microVM to manage the VS Code Server lifecycle, ensuring it restarts automatically on crash.
- [ ] Build a distributed rate-limiter and allocation queue to prevent compute-exhaustion DDoS attacks during concurrent Codespace provisioning.

### Tier 3: Network Routing & Secure Ingress
- [ ] Deploy a globally distributed edge proxy (using Rust and `hyper`) to handle wildcard DNS routing for active Codespaces (e.g., `[name]-[port].preview.gh-clone.com`).
- [ ] Implement mutual TLS (mTLS) between the edge proxy and the internal Firecracker microVM to ensure end-to-end encryption of port-forwarded traffic.
- [ ] Build the dynamic port forwarding API: allow the VS Code Server to publish available ports (e.g., 3000, 8080) and instantly map them to public or private subdomains.
- [ ] Enforce access controls on forwarded ports: support `Private` (owner only, requires session cookie) and `Public` (open to internet) visibility states.
- [ ] Inject OAuth access tokens and `.git-credentials` automatically into the Codespace environment to seamlessly authenticate git operations with the host platform.
- [ ] Implement a dedicated WebSocket proxy explicitly tuned to handle the high-throughput, low-latency traffic required by the VS Code remote extension protocol.

### Tier 4: Angular Frontend & VS Code Web Integration
- [ ] Build the "Create Codespace" Angular component, rendering machine-type selectors, region dropdowns, and advanced devcontainer override options.
- [ ] Implement the active Codespace management UI in Angular, tracking state transitions (Starting, Available, Suspended, Rebuilding) via Server-Sent Events.
- [ ] Host the open-source VS Code Web Server (`code-server` or Microsoft's official web release) directly within the frontend architecture.
- [ ] Embed the VS Code Web UI inside a strictly sandboxed `iframe`, utilizing `postMessage` for authenticated handshakes and lifecycle commands.
- [ ] Implement the "Open in Desktop VS Code" deep link protocol (`vscode://`), generating one-time connection tokens for the local Remote - Tunnels extension.
- [ ] Build the "Secrets" management UI, allowing users to define encrypted environment variables that are securely injected into the Codespace at boot.
- [ ] Implement a dotfiles synchronization service: automatically clone a user's `dotfiles` repository and execute `install.sh` upon Codespace creation.

### Tier 5: Enterprise Controls & Telemetry
- [ ] Implement Enterprise Policies restricting Codespace creation to specific regions (e.g., EU-only for GDPR compliance) and maximum machine types.
- [ ] Add idle-timeout policy enforcements: allow enterprise admins to override user-level idle timeouts (e.g., force suspend after 15 minutes).
- [ ] Implement outbound network restriction policies within the microVM (via iptables/eBPF) to prevent data exfiltration from enterprise Codespaces.
- [ ] Export Prometheus metrics tracking Codespace boot times, resume latencies, and microVM memory saturation.
- [ ] Integrate Codespace compute seconds directly into the Billing Engine, categorizing usage by machine tier and storage consumption.
### Tier 6: Advanced Storage, Pre-builds & Hibernation
- [ ] Implement OverlayFS/Btrfs copy-on-write (CoW) snapshots specifically for instantly cloning base devcontainer images into user-specific Workspaces.
- [ ] Build the "Pre-build Warm Pool" orchestrator: maintain a configurable number of pre-started Firecracker microVMs for heavily trafficked repositories to achieve zero-wait starts.
- [ ] Implement automatic uncommitted state detection: before a Codespace is permanently deleted due to inactivity, automatically `git commit` and push a recovery branch.
- [ ] Design the tiered storage eviction policy: move suspended Codespaces from high-IOPS NVMe drives to cheaper object storage (S3) after 7 days of inactivity.
- [ ] Build the network-attached storage (NAS) auto-mounter: securely mount the recovered workspace tarball into a fresh microVM upon a user requesting a resume.
- [ ] Implement background keep-alive probes using WebSockets to ensure temporary network drops (e.g., closing a laptop) do not instantly trigger the 30-minute idle suspension protocol.
- [ ] Expose an Angular UI component for "Disk Usage", allowing developers to view exactly which node modules or rust target directories are consuming their 32GB quota.
- [ ] Build a background cron worker to aggressively prune dangling Docker images, volumes, and networks within the microVM to prevent ENOSPC errors during active development.
- [ ] Implement an exact parity CLI command `gh codespace ssh` utilizing a Rust-based proxy to bypass the browser entirely and drop users into a native terminal.

### Tier 7: Extensions, Settings Sync & Advanced Networking
- [ ] Implement the VS Code Settings Sync backend API in Rust, storing `settings.json`, `keybindings.json`, and UI state in Postgres JSONB columns.
- [ ] Build a transparent reverse-proxy to the Open VSX Registry (or an internal extension marketplace) to allow extensions to be installed directly within the sandboxed Codespace.
- [ ] Enforce "Allowed Extensions" enterprise policies by intercepting and rejecting installation requests for unapproved VS Code extensions at the network layer.
- [ ] Implement WebRTC-based X11/Wayland forwarding to allow developers to render Linux desktop GUI applications seamlessly within an Angular Canvas element in the browser.
- [ ] Configure custom `resolv.conf` injection per Codespace to allow routing specific corporate subdomains through internal enterprise VPN gateways rather than the public internet.
- [ ] Build the "Port Forwarding" access control list (ACL): track exactly which GitHub users have authenticated and accessed a shared private port.
- [ ] Implement an inbound webhook proxy daemon inside the microVM to allow developers to receive external webhooks (e.g., Stripe, Slack) securely on localhost without ngrok.
- [ ] Export granular network egress metrics to the billing subsystem, charging enterprises for cross-region bandwidth consumed by their active Codespaces.
- [ ] Implement rigorous CPU Cgroup constraints dynamically responding to the "Boost" feature, temporarily allowing a 2-core Codespace to burst to 8 cores during a heavy Rust compilation.
