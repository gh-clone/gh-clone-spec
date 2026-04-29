# Plan for gh-clone-security (GHAS & Dependabot)

### Tier 1: CodeQL Static Application Security Testing (SAST)
- [ ] Provide pre-compiled CodeQL CLI binaries bundled securely as tool caches within the GitHub Actions runner images.
- [ ] Implement the `autobuild` logic capable of automatically intercepting compiler toolchains (gcc, javac, msbuild) to generate relational CodeQL databases for compiled languages.
- [ ] Execute standard CodeQL query packs (`codeql/cpp-queries`, `codeql/javascript-queries`) specifically filtering for CWE-associated security vulnerabilities.
- [ ] Build the backend parser to ingest standard SARIF (Static Analysis Results Interchange Format) v2.1.0 JSON payloads.
- [ ] Map SARIF `physicalLocation` artifacts directly to the repository's Git Blob SHAs and specific line numbers.
- [ ] Implement the "Code Scanning Alerts" UI, overlaying vulnerability data flows and taint-tracking paths directly in the code viewer.
- [ ] Build resolution logic: automatically transition alerts to "Fixed" state if the vulnerable code is removed in the default branch.
- [ ] Implement branch protection rules configured via the GraphQL API to block Pull Request merges if new `Critical` or `High` severity SARIF alerts are introduced.
- [ ] Expose the `/code-scanning/sarifs` REST API to allow third-party scanners (Trivy, Semgrep, Snyk) to inject their results seamlessly into the GitHub UI.

### Tier 2: Dependabot Vulnerability Scanning (SCA)
- [ ] Implement a background chron worker that fetches and ingests the latest OSV (Open Source Vulnerabilities) JSON dumps into a Postgres JSONB database.
- [ ] Implement strict Semantic Versioning (SemVer) resolution parsers natively in Rust for the core ecosystems: `npm`, `pip`, `cargo`, `maven`, and `rubygems`.
- [ ] Build a dependency graph extraction worker that clones repositories, parses `package.json` and `Cargo.toml`, and builds a relational tree.
- [ ] Parse lockfiles (`package-lock.json`, `yarn.lock`, `Cargo.lock`) to accurately extract deeply nested transitive dependencies.
- [ ] Implement the comparison engine: intersect the extracted repository dependency graph with the local OSV database to detect vulnerable versions.
- [ ] Render the "Dependency Graph" UI tab, allowing users to visually inspect their explicit and transitive dependencies.
- [ ] Generate "Dependabot Alerts" for detected vulnerabilities, mapping the exact CVE and CVSS scores to the UI.

### Tier 3: Dependabot Automated Pull Requests
- [ ] Build the Dependabot PR generator engine: dynamically compute the required version bump to resolve a specific vulnerability.
- [ ] Execute package manager lockfile updates securely within an isolated sandbox (e.g., running `npm install` inside a restricted Firecracker microVM).
- [ ] Construct the Git Commit and Git Tree purely via the GitHub REST API/libgit2 to generate the PR without requiring a local workspace clone.
- [ ] Auto-generate comprehensive Pull Request descriptions containing the CVE details, Release Notes, and Commits from the upstream patched package.
- [ ] Implement automated rebase logic: if the target branch advances, Dependabot must automatically `git rebase` its open PR to resolve conflicts.
- [ ] Support `dependabot.yml` parsing to handle scheduled Version Updates (non-security), allowing maintainers to specify target ecosystems, directories, and schedules.
- [ ] Implement Dependabot grouped updates, allowing users to configure PR batching (e.g., group all `devDependencies` into a single weekly PR).

### Tier 4: Secret Scanning & Push Protection
- [ ] Integrate the `hyperscan` high-performance regex matching engine natively into the `pre-receive` Git hook binary.
- [ ] Define over 250+ highly specific regex signatures targeting active cloud provider tokens (AWS, Azure, GCP, Stripe, Slack, RSA Private Keys).
- [ ] Implement heuristic entropy checks specifically designed to filter out high-entropy false positives (e.g., standard UUIDs vs actual secrets).
- [ ] Implement "Push Protection": reject the `git push` command entirely if a secret is matched, outputting the exact filename and line number to `stderr`.
- [ ] Build the bypass mechanism: allow developers to push the commit if they explicitly visit a generated URL or append `?bypass=secret-scanning` to the push options.
- [ ] Develop a background historical scanning queue: recursively scan the entire commit history of newly imported repositories for legacy secrets.
- [ ] Build the "Secret Scanning Alerts" UI, allowing organization admins to review exposed tokens and mark them as "False Positive" or "Revoked".
- [ ] Implement specific Webhook partnerships with major cloud providers (AWS, Azure) to automatically dispatch revocation events when a valid token is pushed publicly.

### Tier 5: Telemetry, Audit & High Availability
- [ ] Instrument the `hyperscan` pre-receive hook with `tracing` to guarantee secret scanning adds < 50ms to the `git push` critical path.
- [ ] Store all "Secret Scanning Bypass" events in the immutable `audit_logs` table for compliance and SOC2 reporting.
- [ ] Implement a Dead-Letter Queue (DLQ) in Kafka for OSV ingestion failures, ensuring no vulnerabilities are missed during upstream API outages.
- [ ] Configure the Dependabot runner cluster with ephemeral, aggressively firewalled nodes to prevent lateral movement during malicious package installation.
- [ ] Export Prometheus metrics tracking the total number of scanned commits, discovered secrets, and Dependabot PR merge success rates.

### Tier 6: Advanced Security & Vulnerability Coordination
- [ ] **Private Vulnerability Reporting:** Implement a secure, isolated channel for external researchers to report vulnerabilities privately to repository maintainers.
- [ ] **Coordinated Vulnerability Disclosure (CVD):** Build the workflow to transform a private report into an embargoed CVE, securely coordinating patches across forks before public release.
- [ ] **Custom Secret Scanning Patterns:** Implement the UI and backend regex compiler to allow organizations to define their own proprietary token signatures.
- [ ] **Regex Performance Profiling:** Build an internal guardrail that runs User-Defined Regex patterns against a test corpus, strictly rejecting patterns that cause catastrophic backtracking (ReDoS).
- [ ] **CodeQL Threat Models:** Support `codeql-workspace.yml` definitions, allowing repositories to specify custom taint-tracking sources, sinks, and threat models specific to their architecture.
- [ ] **Security Advisory GraphQL API:** Expose the complete lifecycle of Security Advisories via the GraphQL endpoint to allow automated patch generation by third-party tools.

### Tier 7: Artifact Attestations & Supply Chain Security (Sigstore)
- [ ] Architect the internal integration with the Sigstore ecosystem (Fulcio for PKI, Rekor for transparency logs) entirely in Rust.
- [ ] Implement the OIDC claims verification engine: ensure Actions workflows minting attestations explicitly bind the workflow SHA, repository ID, and trigger event to the generated certificate.
- [ ] Implement the `POST /repos/{owner}/{repo}/attestations` REST endpoint to accept and validate signed provenance bundles (in-toto format).
- [ ] Integrate NPM Provenance logic: validate Sigstore bundles generated by `npm publish --provenance` during the package registry upload phase.
- [ ] Build the "Attestations" Angular UI on the Release page: display cryptographically verifiable badges indicating a release binary was genuinely produced by the listed Actions workflow.
- [ ] Expose the CLI verification endpoints, allowing `gh attestation verify` to query the internal Rust transparency log proxy to confirm artifact integrity before deployment.
- [ ] Build strict branch protection constraints: require that any release binary pushed to the default branch is accompanied by a valid, mathematically verifiable attestation.

### Tier 8: Security Advisories & Coordinated Vulnerability Disclosure (CVD)
- [ ] **Advisories Database Schema:** Define the Postgres tables `repository_advisories` mapping to CVE IDs, CVSS scores, vulnerable ecosystem packages, patched version strings, and disclosure embargo timestamps.
- [ ] **Private Fork Git Isolation:** Build the backend Git logic via `spokes`/`gitoxide` to instantiate a "Temporary Private Fork"; implement hard constraints in the GraphQL/REST routing to ensure this fork is completely invisible to global `/search` and public APIs.
- [ ] **CVD Access Control Middleware:** Implement strict RBAC middleware in Rust ensuring only the original vulnerability reporter, repository maintainers, and explicitly invited security researchers can resolve the advisory thread IDs.
- [ ] **CVSS 3.1 Scoring Engine:** Implement the Common Vulnerability Scoring System (CVSS) v3.1 calculator mathematically in Rust, deriving the exact `Critical/High/Medium/Low` severity and base score dynamically from the vector strings.
- [ ] **CVE Numbering Authority (CNA) Integration:** Build a secure API connector utilizing `reqwest` to interface directly with the official MITRE CVE Services API.
- [ ] **Automated CVE Reservation:** Implement the backend mutation allowing repository maintainers to click "Request CVE", triggering the Rust backend to securely reserve an ID from MITRE and persist it to the `repository_advisories` table.
- [ ] **CVE Payload Synthesis:** Construct the highly specific JSON schema required by MITRE to publish the vulnerability data, dynamically mapping the Markdown description and affected versions into the external payload.
- [ ] **Embargo State Machine:** Implement a `tokio` cron worker that tracks the specific `embargo_date` of drafted advisories; immediately halting publication or warning maintainers if they attempt to merge patches prematurely.
- [ ] **Private-to-Public Merge Strategy:** Build a specialized Git merge execution flow in Rust: upon advisory publication, automatically pull the commits from the isolated Private Fork, rebase/merge them into the public upstream repository, and forcefully delete the private fork.
- [ ] **Security Advisory Webhooks:** Expose `security_advisory.published` webhook events from the backend, generating HMAC signatures so downstream consumers and internal enterprise teams can automatically trigger internal remediations.
- [ ] **Automated Dependabot OSV Linking:** Upon publication of a Repository Security Advisory, automatically format the payload into the Open Source Vulnerability (OSV) JSON standard and inject it directly into the internal Dependabot tracking database.
- [ ] **Dependabot PR Fan-out:** Trigger the Dependabot background worker to immediately fan-out and generate automated Pull Requests for all internal repositories depending on the newly published vulnerable package.
- [ ] **Advisory Credit Synchronization:** Implement backend logic to explicitly map credited security researchers from the advisory payload to their user profiles, updating their public "Security Bug Bounty" metadata and badges.
- [ ] **Bounty Payout Integration:** Expose an internal API allowing the enterprise billing engine to read bounty values assigned to specific researchers and trigger payouts via the Stripe Connect API upon successful disclosure.
- [ ] **Advisory Markdown Sanitization:** Enforce a specialized, ultra-strict markdown rendering pipeline for Security Advisories, stripping complex HTML embeddings to prevent malicious payloads within the advisory descriptions themselves.
### Tier 9: Dependency Review & License Compliance in Pull Requests
- [ ] **Lockfile Interceptor Worker:** Implement an asynchronous Rust worker that intercepts Pull Request `synchronize` events and isolates diffs exclusively for supported lockfiles (e.g., `Cargo.lock`, `package-lock.json`, `yarn.lock`).
- [ ] **Semantic Version Extraction:** Build a robust lockfile AST parser in Rust to extract exact transitive dependency version bumps introduced strictly by the PR's commits.
- [ ] **Real-Time OSV Cross-Reference:** Cross-reference the extracted, newly introduced dependencies against the internal OSV (Open Source Vulnerabilities) database with sub-millisecond latency.
- [ ] **Dependency Graph Compare API:** Expose the `/repos/{owner}/{repo}/dependency-graph/compare/{basehead}` REST endpoint returning structured JSON detailing added/removed dependencies and their associated vulnerabilities.
- [ ] **License Compliance Engine:** Intersect the newly added dependencies against the Enterprise's configured "Allowed Licenses" policy (e.g., flagging `GPL-3.0` or `AGPL-3.0` violations).
- [ ] **Automated PR Blocking:** Integrate the Dependency Review engine with Branch Protection rules, allowing administrators to automatically block PR merges if an unapproved license or a `Critical` vulnerability is introduced in a lockfile.
- [ ] **Reviewer Context Webhooks:** Dispatch specific `pull_request_review_thread` webhooks when the Dependency Review engine flags a vulnerability, alerting security teams instantly.

### Tier 9.1: Lockfile Parsing & Transitive Vulnerability Engine
- [ ] **Native Lockfile Parsers in Rust:** Build dedicated, zero-copy Rust parsers utilizing the `nom` framework specifically targeting the highly complex structures of `Cargo.lock`, `package-lock.json` (v1/v2/v3), and `yarn.lock` (v1/berry/v4).
- [ ] **Submodule Dependency Resolution:** Extend the dependency graph extractor to accurately resolve `git submodule` references, linking specific submodule commit hashes to potential underlying OSV vulnerabilities.
- [ ] **Environment Segregation (Dev vs Prod):** Parse lockfile scopes strictly; ensure development-only dependencies (e.g., `devDependencies` in `package.json`) are tagged distinctly from production dependencies in the Postgres database.
- [ ] **CVSS Fallback Calculator:** Implement a heuristic in the backend: if an incoming OSV report lacks a formal CVSS base score, utilize a localized NLP model against the vulnerability description to assign a preliminary severity estimate.
- [ ] **SPDX License Database Intersect:** Implement a nightly background worker that fetches the latest official SPDX license list, maintaining an internal lookup table to instantly categorize parsed dependency licenses.
- [ ] **Fuzzy License Matching:** Build a heuristic in Rust utilizing Sørensen–Dice coefficients to match non-standard license string declarations in `package.json` against known SPDX identifiers.
- [ ] **"Ignored" Vulnerability Auditing:** Implement the workflow allowing developers to "Dismiss" a dependency vulnerability as "Risk Accepted" or "Inaccurate", writing this state to a persistent `vulnerability_dismissals` table scoped to the repository.

### Tier 10: Dependabot Private Registry Authentication
- [ ] Define the `dependabot_secrets` Postgres table, mapping encrypted registry credentials strictly to specific repositories or organizations via envelope encryption.
- [ ] Build the Angular UI allowing administrators to securely input credentials for external registries (e.g., JFrog Artifactory, AWS CodeArtifact, private NPM, Docker Hub).
- [ ] Modify the Dependabot Rust orchestrator to securely inject these decrypted credentials into the isolated Dependabot update worker container as short-lived environment variables.
- [ ] Implement `dependabot.yml` parsing extensions to recognize and map the `registries` configuration blocks to the stored database secrets.
- [ ] Architect the network isolation boundaries ensuring the Dependabot worker can safely route to internal enterprise VPN/VPC IPs when attempting to resolve dependencies from self-hosted private registries.

### Tier 11: Fallback SAST Engine Integration (CodeQL License Mitigation)
- [ ] Architect a drop-in SAST replacement pipeline utilizing open-source engines (e.g., Semgrep) to provide immediate Code Scanning capabilities without relying on proprietary CodeQL licensing.
- [ ] Build a Rust translation layer that maps Semgrep output schemas and rule IDs into the internal, standardized SARIF format expected by the security dashboard.
- [ ] Provide a curated, continuously updated library of default Semgrep rulesets mimicking the most critical CodeQL security queries (SQLi, XSS, Path Traversal, Command Injection).
- [ ] Implement the "Advanced Security" UI toggles, allowing Enterprise users to seamlessly switch the backend engine from the default Semgrep fallback to their own licensed CodeQL runners if they provide a valid license key.
- [ ] Integrate Trivy for high-speed container scanning: automatically intercept pushed `Dockerfile` and built images in the Packages registry to execute immediate vulnerability scans against OS-level packages.
