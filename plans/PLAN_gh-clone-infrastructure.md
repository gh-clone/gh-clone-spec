# Plan for gh-clone-infrastructure

### 5.1 Virtual Appliance Build & Base OS (Packer)
- [ ] **AWS AMI Generation:** Develop HashiCorp Packer HCL templates to generate an AWS AMI leveraging an EBS volume layout partitioned specifically for `/` (root), `/data/user/repositories`, and `/var/log`.
- [ ] **Azure VHD Packaging:** Configure Packer to build an Azure-compatible VHD with the WALinuxAgent pre-installed and configured for accelerated networking.
- [ ] **VMware OVA/OVF Template:** Create a VMware vSphere-compatible OVA using Packer, including open-vm-tools and specific `.ovf` properties for network IP injection on first boot.
- [ ] **KVM QCOW2 Creation:** Build a QCOW2 disk image optimized for KVM/libvirt deployments via cloud-init data sources.
- [ ] **Hardened Ubuntu LTS Base:** Apply CIS (Center for Internet Security) Level 1 benchmarks to the base Ubuntu LTS image used for all virtual appliances.
- [ ] **Disk Encryption Hooks:** Implement custom `initramfs` hooks to support LUKS full-disk encryption with passphrase provisioning over SSH on appliance boot.
- [ ] **Network Interface Tuning:** Pre-configure `systemd-networkd` to automatically support network interface bonding (LACP) and VLAN tagging.
- [ ] **SSH Daemon Hardening:** Configure `sshd_config` to explicitly disable root login, require Ed25519 keys, and utilize `AuthorizedKeysCommand` to fetch internal service keys dynamically.
- [ ] **Kernel Parameter Tuning:** Apply custom `sysctl.conf` rules to optimize TCP congestion control (BBR), increase ephemeral port ranges, and raise max file descriptors (`fs.file-max`) for high-connection Git traffic.
- [ ] **NTP Synchronization:** Pre-configure `chronyd` to sync with enterprise-provided or geo-local Stratum 2 NTP servers to ensure timestamp consistency across distributed commits.

### 5.2 Appliance Configuration Management (`ghe-config`)
- [ ] **`ghe-config` CLI:** Implement the Go-based `ghe-config` CLI wrapper for setting key-value configuration flags dynamically on the appliance.
- [ ] **Consul State Backend:** Deploy a local Consul agent to serve as the distributed key-value store backing the `ghe-config` configuration state.
- [ ] **`ghe-cluster-config` Topology:** Build the topology generation engine that reads Consul state to dynamically provision High Availability (HA) cluster nodes (Active/Passive or Active/Active).
- [ ] **HAProxy Generation:** Write scripts to automatically template and reload `haproxy.cfg` based on the active cluster topology and backend node health checks.
- [ ] **Nginx Routing Rules:** Implement an automatic config templater for `nginx.conf` that translates application routing policies into Nginx server blocks.
- [ ] **Let's Encrypt / ACME:** Integrate an ACME client to allow automatic provisioning and renewal of TLS certificates for the appliance's external hostname.
- [ ] **Custom CA Injection:** Provide a CLI utility (`ghe-ssl-certificate-setup`) to inject and trust enterprise internal Root Certificate Authorities into the system trust store.
- [ ] **Management Console Frontend:** Develop a React/Next.js based administration web interface (served on port 8443) for visual cluster status and configuration.
- [ ] **Management Console Backend:** Implement a PAM-integrated authentication backend for the Management Console to support localized OS admin credentials.
- [ ] **SMTP Outbound Configuration:** Implement the subsystem to configure outbound SMTP relays for platform notifications, including STARTTLS and authentication mechanisms.

### 5.3 Cross-Platform Client Toolchain & Native Packages
- [ ] **`zig cc` Toolchain Matrix:** Configure `zig cc` as the universal linker in CI to flawlessly cross-compile Rust/C dependencies (like `libgit2` and `openssl`) for Windows, macOS, and Linux from a single runner.
- [ ] **WiX Windows Installer (`.msi`):** Write WiX XML definitions to generate a `.msi` installer for Windows clients and background services.
- [ ] **Windows Service Registration:** Embed logic in the `.msi` to automatically register the background binary as an auto-starting Windows Service.
- [ ] **Windows ETW Integration:** Implement and register an Event Tracing for Windows (ETW) provider for high-performance, low-level system logging on Windows.
- [ ] **macOS DMG Creator:** Configure `create-dmg` to bundle macOS binaries into a user-friendly, drag-and-drop `.dmg` installer.
- [ ] **macOS PKG Generation:** Build a `.pkg` installer that includes pre-install and post-install shell scripts to provision system paths and permissions.
- [ ] **macOS LaunchDaemons:** Generate and register a `com.github-clone.plist` LaunchDaemon to ensure background services persist across macOS reboots.
- [ ] **Apple Notarization Pipeline:** Integrate `codesign` and `xcrun altool` into GitHub Actions to securely sign and submit macOS artifacts to Apple's Notary Service, preventing Gatekeeper blocks.
- [ ] **Debian Package (`.deb`):** Automate `dpkg-buildpackage` to create Debian packages containing `systemd` unit files and `logrotate.d` configurations.
- [ ] **RPM Package (`.rpm`):** Write RPM SPEC files to build packages for RHEL/CentOS distributions, including SELinux context restoration hooks (`restorecon`) in the post-install phase.

### 5.5 Global Kubernetes Orchestration & Data Stores
- [ ] **Vitess Operator Deployment:** Deploy the Vitess Kubernetes Operator to orchestrate highly available, sharded MySQL topologies.
- [ ] **Vitess VSchema Configuration:** Define Vitess VSchemas to shard the `users`, `issues`, and `pull_requests` tables based on a hashed repository ID or user ID index.
- [ ] **Elasticsearch ECK Integration:** Provision a multi-node OpenSearch/Elasticsearch cluster using the Elastic Cloud on Kubernetes (ECK) operator for code tokenization.
- [ ] **Elasticsearch ILM Policies:** Configure Index Lifecycle Management (ILM) policies to automatically roll over, compress, and delete aged search index data.
- [ ] **Redis Rate Limiting Cluster:** Deploy a Redis cluster fronted by Twemproxy specifically tuned for high-velocity API rate limit counter increments and decrements.
- [ ] **Redis Session Cache:** Deploy a separate, persistent Redis instance for managing OAuth tokens, SAML assertions, and web UI sessions.
- [ ] **Object Storage Gateway:** Deploy a MinIO or Ceph Object Gateway to provide an S3-compatible API for handling Git Large File Storage (LFS) and release binaries.
- [ ] **GitHub Packages Storage Backend:** Integrate the S3-compatible backend with a Docker Registry v2 API implementation to serve GitHub Packages (npm, Maven, Docker).
- [ ] **Service Mesh (Istio/Linkerd):** Inject a service mesh sidecar to enforce strict mTLS between all microservices and automatically capture L7 networking metrics.

### 5.6 CI/CD, Actions, & Observability
- [ ] **ARC Controller Implementation:** Deploy a custom Kubernetes controller (mimicking Actions Runner Controller) to dynamically scale ephemeral runner pods based on queue length.
- [ ] **Actions Mutating Webhook:** Create a Mutating Admission Webhook that intercepts Actions job pod submissions and automatically injects required `initContainers` and sidecars.
- [ ] **Docker-in-Docker (`dind`) Sidecar:** Implement the `dind` sidecar pattern for Actions runners to allow developers to build and publish Docker containers within their CI workflows securely.
- [ ] **`libmountsandbox` Isolation:** Utilize customized seccomp profiles and `libmountsandbox` to ensure rigorous tenant isolation between concurrent Actions job executions.
- [ ] **Calico/Cilium Network Policies:** Enforce default-deny network policies using Cilium, specifically whitelisting gRPC port 50051 ingress to the `storage-tier` only from the verified `api-tier` service account.
- [ ] **Prometheus ServiceMonitors:** Write custom Custom Resource Definitions (CRDs) for `ServiceMonitors` to automatically discover and scrape `/metrics` endpoints from all deployed pods.
- [ ] **OpenTelemetry Jaeger Tracing:** Deploy a global OpenTelemetry Collector DaemonSet to ingest, sample, and forward distributed tracing spans to a Jaeger backend.
- [ ] **Fluent Bit Log Forwarding:** Configure Fluent Bit to capture container stdout/stderr, parse JSON structured logs, and forward them to external enterprise SIEMs (Splunk, Datadog).
- [ ] **`ghe-migrator` Utility:** Bundle a data transformation utility capable of converting massive exported tarballs (issues, PRs, Git data) into the internal database schema for enterprise migrations.
- [ ] **`ghe-backup-utils` Snapshotting:** Implement an automated snapshot mechanism utilizing rsync over SSH to perform delta backups of the entire distributed appliance state.
- [ ] **Air-Gapped Licensing Service:** Develop an offline-capable enterprise licensing validation module utilizing Ed25519 cryptographic signatures for secure, air-gapped license file verification.
- [ ] **OTA Rolling Upgrades:** Build an Over-The-Air (OTA) orchestrator that parses `.pkg` upgrade payloads, cordons nodes, and applies zero-downtime rolling updates to the virtual appliance cluster.

