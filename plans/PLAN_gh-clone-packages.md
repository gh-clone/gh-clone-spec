# Plan for gh-clone-packages

### Tier 1: Multi-Protocol Registry Emulation
- [ ] **OCI / Docker V2:** Fully implement the Docker Distribution specification. Support `GET /v2/`, manifest uploads (`application/vnd.oci.image.manifest.v1+json`), and layered chunked blob uploads via `PATCH /v2/<name>/blobs/uploads/<uuid>`.
- [ ] **NPM:** Implement the CouchDB-compatible API expected by the `npm` CLI. Handle `GET /{package}` for metadata, resolve specific tags (`latest`, `beta`), and process `PUT` requests for tarball publication.
- [ ] **Maven / Gradle:** Implement standard directory listing and artifact resolution required by Java build tools. Dynamically generate `maven-metadata.xml` strictly from the Postgres database state.
- [ ] **NuGet:** Implement the NuGet V3 JSON API endpoint (`/v3/index.json`). Support the Search Query Service, Registrations Base URL (SemVer 2.0.0), and Package Base Address functionality.
- [ ] **RubyGems:** Implement the RubyGems API (`/api/v1/gems`). Handle native `.gem` file unpacking and parse the internal `metadata.gz` (Gemspec) securely.
- [ ] **Cargo:** Implement the Cargo Sparse Registry protocol (RFC 3143) to support native Rust `cargo publish` and `cargo fetch` directly over HTTP.

### Tier 2: Unified Storage & Binary Handling
- [ ] Route all massive binary artifact uploads to bypass the API server memory by generating pre-signed S3/MinIO URLs for direct client-to-storage transfer.
- [ ] Implement an asynchronous metadata extraction worker: download newly uploaded artifacts (e.g., `.tgz`, `.jar`), extract their manifest files (`package.json`, `pom.xml`), and index dependencies into Postgres.
- [ ] Enforce absolute immutability: strictly reject HTTP `PUT` requests if a package namespace, name, and specific version string already exist in the database.
- [ ] Generate and validate cryptographic checksums (SHA-256, MD5, SHA-1) dynamically during the upload stream to ensure artifact integrity at rest.
- [ ] Design the Postgres schema using `JSONB` columns to flexibly store arbitrary package metadata (tags, authors, licenses) distinct to each package manager's format.
- [ ] Implement highly efficient cursor-based pagination for Registry APIs (e.g., Docker Catalog endpoints, NPM Search) to handle organizations with thousands of packages.

### Tier 3: UI, Access Control & Lifecycle
- [ ] Build the repository "Packages" UI tab, listing all associated packages with distinct, recognizable SVG icons for Docker, npm, Maven, etc.
- [ ] Implement detailed package detail pages displaying the exact installation commands (e.g., `npm install @org/pkg`), version history timelines, and aggregate download statistics.
- [ ] Integrate GitHub Packages authentication seamlessly with standard Personal Access Tokens (PATs) enforcing the specific `read:packages` and `write:packages` scopes.
- [ ] Implement authentication mapping: allow specific package managers (like RubyGems) to authenticate using base64 encoded PATs as API keys.
- [ ] Enforce strict visibility synchronization: if a linked repository transitions from Public to Private, automatically cascade the Private visibility state to all associated packages.
- [ ] Implement Data Lifecycle Policies (Retention): build background cron jobs to automatically delete untagged container images older than X days.
- [ ] Build a rigorous garbage collection worker to physically execute `s3:DeleteObject` commands, permanently removing orphaned blob layers that are no longer referenced by any active manifest.
- [ ] Integrate package download events with the internal Billing & Metering engine to precisely count egress bandwidth usage for enterprise invoicing.

### Tier 5: CDN Distribution & Enterprise Controls
- [ ] Integrate a global CDN for all public package downloads, utilizing the `Immutable` Cache-Control directive to maximize offload.
- [ ] Implement distributed locking (e.g., Redlock) during package uploads to prevent race conditions when pushing the identical version simultaneously.
- [ ] Enforce strict cryptographic validation: reject uploads if the client-provided `Content-Length` does not exactly match the finalized S3 object size.
- [ ] Build an administrative API to instantly quarantine and yank malicious packages (e.g., dependency confusion attacks) across all edge nodes.
- [ ] Emit detailed audit events to the enterprise log stream whenever a package is published, downloaded (if private), or deleted.

### Tier 6: NPM Provenance & Sigstore Verification
- [ ] Extend the NPM registry emulation in Rust to strictly parse and validate `provenance` signatures embedded inside standard `npm publish` tarballs.
- [ ] Implement OIDC JWT verification: intercept the token provided by the GitHub Actions runner and cryptographically verify the issuer, audience, and repository subject claims.
- [ ] Resolve the provenance signature against the public Rekor transparency log (or an internally hosted Rust Rekor instance for air-gapped environments).
- [ ] Map the validated provenance identity strictly to the workflow run that generated it, writing the linkage to a `package_provenance` Postgres table.
- [ ] Render the "Provenance" badge dynamically in the Angular Package UI, verifying that the package was built by the exact expected source repository and workflow file.
- [ ] Implement NPM policy enforcement: allow organization admins to toggle "Require Provenance", strictly rejecting any `PUT` to the package registry lacking valid Sigstore attestations.

### Tier 7: Actions Marketplace Artifact Storage
- [ ] **Immutable Blob Paths:** Implement the immutable Blob storage schema specifically for published Native Actions (e.g., routing to `s3://packages/actions/{namespace}/{action_name}/{version}.tar.gz`).
- [ ] **Tarball Synthesis Engine:** Build a Rust worker utilizing `gitoxide` and `flate2` to dynamically clone a repository at a specific tag, strip the `.git` directory, and compress it into a `.tar.gz` artifact upon marketplace publication.
- [ ] **Streaming S3 Uploads:** Pipe the dynamically generated tarball stream directly from the `flate2` compressor into the `aws-sdk-s3` multi-part upload client to prevent memory exhaustion on massive action repositories.
- [ ] **Internal Runner API Handoff:** Expose an internal, heavily optimized API endpoint (`GET /internal/runner/actions/{name}/{version}`) strictly for the Actions Control Plane to fetch action artifacts during job initialization.
- [ ] **Runner Rate Limit Bypass:** Ensure the internal runner API completely bypasses standard user HTTP rate limits and governor checks to guarantee CI/CD scale.
- [ ] **Edge Caching Replication:** Build a background replication task utilizing Redis PubSub ensuring that highly requested marketplace actions (like `actions/checkout`) are actively pushed to geographic edge nodes (Varnish/CDN) for sub-second runner initialization.
- [ ] **Semantic Version Resolution:** Enforce strict semantic versioning parsing on the backend using the `semver` crate: guarantee that `uses: action@v2` dynamically resolves to the highest `v2.x.x` immutable artifact in storage.
- [ ] **Hash Pinning Verification:** Implement endpoint logic to fetch an action by absolute SHA (`uses: action@sha`); dynamically generating the tarball exclusively from that exact commit hash.
- [ ] **Cryptographic Checksum Generation:** Calculate the SHA-256 digest of the generated `.tar.gz` action artifact during the upload phase and persist it in the `actions_marketplace_listings` Postgres table.
- [ ] **Checksum Validation API:** Expose the checksum in the artifact metadata payload so the Actions Runner agent can cryptographically verify the downloaded tarball before executing it.
- [ ] **Storage Quota Exemption:** Architect the billing engine to explicitly exempt public Action tarballs from consuming the publisher's S3 storage quota to encourage ecosystem growth.
- [ ] **Artifact Eviction Locks:** Implement object-level locks in the S3 bucket configuration to prevent accidental deletion or modification of a published Action artifact, ensuring deterministic builds for downstream consumers.
- [ ] **Deprecated Action Warning Injection:** If a runner requests an Action artifact that has been marked as deprecated in Postgres, inject a specific custom HTTP header (`X-Action-Deprecated: true`) which the runner parses to output a yellow warning in the build logs.
- [ ] **Private Marketplace Routing:** Extend the artifact resolution logic to support "Internal" actions, dynamically verifying the runner's injected OIDC token against organization membership before granting access to the private S3 artifact.
- [ ] **Dependency Pre-caching:** Build a Rust heuristic that scans the `action.yml` during publication; if it requires `node20`, pre-warm the download of the Node.js toolcache archive into the same edge location as the action artifact.