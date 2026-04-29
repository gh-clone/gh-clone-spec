# Plan for gh-clone-enterprise-management

### Tier 1: Enterprise Account Hierarchy & Organization Mapping
- [ ] Define the `enterprises` Postgres table: `id`, `slug`, `name`, `billing_email`, `created_at`.
- [ ] Define the `enterprise_organizations` mapping table, strictly linking multiple standalone `organizations` under a single parent `enterprise`.
- [ ] Implement the Enterprise GraphQL API node (`Enterprise`), allowing traversal of all nested organizations, users, and billing data.
- [ ] Build the Enterprise Dashboard Angular UI (`/enterprises/{slug}`), separate from standard repository views, to provide a high-level administrative interface.
- [ ] Implement cross-organization user search and management, allowing enterprise admins to globally suspend users across all mapped organizations.
- [ ] Configure RBAC specifically for Enterprise Administrators, distinguishing them from Organization Owners (e.g., an Enterprise Admin can force-add themselves as an Org Owner).
- [ ] Build the "Managed Users" (EMU) provisioning logic, ensuring identities are strictly federated from an external IdP (Azure AD/Okta) and cannot be modified via the standard UI.

### Tier 2: Global Policies & Ruleset Enforcement
- [ ] Define the `enterprise_policies` JSONB schema to store strict true/false and constraint data (e.g., `require_two_factor_authentication: true`).
- [ ] Implement the inheritance engine in Rust: whenever an Organization attempts an action, calculate the effective policy by recursively merging Org policies with the parent Enterprise policies.
- [ ] Build the Angular UI for Enterprise Policies, allowing admins to enforce "Allowed IP Addresses" across all API and Web access globally.
- [ ] Implement the "Repository Creation Policy" allowing enterprises to restrict creation to specific visibility types (e.g., internal and private only, public disabled).
- [ ] Implement the "Forking Policy" to explicitly block users from forking enterprise-owned repositories into their personal namespaces.
- [ ] Build the Base Ruleset feature: allow enterprise admins to define a standard Repository Ruleset (e.g., always require 1 reviewer) that is immutably applied to every repository in every organization.
- [ ] Configure global SSH Certificate Authority requirements, forcing developers to use short-lived, IdP-issued SSH certificates instead of static keys.

### Tier 3: Audit Log Streaming & Retention
- [ ] Architect the `audit_logs` table for hyper-scale partitioning using Postgres `pg_partman`, partitioned daily by enterprise ID.
- [ ] Implement the "Audit Log Streaming" configuration UI, allowing admins to input AWS S3 credentials, Datadog API keys, or Splunk HEC endpoints.
- [ ] Build a highly concurrent Rust background worker utilizing `rdkafka` to consume audit events and stream them in real-time to the configured external destinations.
- [ ] Implement strict failure retry mechanisms for the streamer: backoff exponentially and alert enterprise admins if the external Splunk/S3 endpoint is unreachable for >24 hours.
- [ ] Provide the Enterprise Audit Log Search UI, exposing the Elasticsearch/OpenSearch backend to query across millions of events using the standard `action:` and `actor:` syntax.
- [ ] Enforce immutability protocols: ensure the Rust API strictly forbids any `DELETE` or `UPDATE` statements against the `audit_logs` tables.
- [ ] Configure automated GDPR "Right to Erasure" scrubbing workers that cryptographically shred PII from legacy audit logs while preserving the raw event structural integrity.

### Tier 4: Consolidated Billing & Metering
- [ ] Build the "Enterprise Billing Rollup" engine: execute daily aggregations of Actions minutes, LFS storage, and Packages egress across all child organizations.
- [ ] Implement the Enterprise Billing Dashboard in Angular, rendering comprehensive cost breakdowns per organization using Echarts or D3.js.
- [ ] Implement Cost Centers: allow admins to attach specific metadata tags to organizations to group billing exports by department.
- [ ] Generate detailed CSV exports containing line-item usage metrics for ingestion into enterprise ERP systems.
- [ ] Configure the Stripe API integration to invoice the single Enterprise Customer object for all aggregated seat licenses and usage, rather than billing individual organizations.
- [ ] Implement spending limits and alerting: automatically send emails to Enterprise Admins if the aggregated Actions usage exceeds a configured monthly budget threshold.

### Tier 5: SAML SSO & SCIM Directory Sync
- [ ] Implement the Enterprise-level SAML IdP configuration, enforcing a single SSO gateway for all child organizations simultaneously.
- [ ] Build the SCIM 2.0 API gateway explicitly for the Enterprise entity (`/enterprises/{slug}/scim/v2/Users`).
- [ ] Map complex Okta/Azure AD groups directly into Enterprise Teams, which then sync down to specific Repository roles across child organizations.
- [ ] Ensure SCIM `DELETE` or `active: false` updates instantly revoke the user's sessions, PATs, and SSH keys globally across the entire cluster.
- [ ] Implement the "SSO Recovery Codes" workflow, securely generating and displaying offline bypass codes for Enterprise Admins in case the external IdP suffers an outage.
- [ ] Ensure all API endpoints (`REST` and `GraphQL`) explicitly validate the `X-GitHub-SSO` headers and HTTP session claims against the enterprise SAML assertion before returning payload data.
### Tier 6: Compliance, Legal Holds & eDiscovery
- [ ] Define the `legal_holds` Postgres table to enforce immutable archiving of users, organizations, or specific repositories during compliance audits or litigation.
- [ ] Implement the backend guardrails for Legal Holds: strictly block all `git push --force`, repository deletions, and issue modifications for flagged entities, overriding all Admin privileges.
- [ ] Build the "eDiscovery Export" asynchronous pipeline: orchestrate a massive Rust worker task to compile every Git commit, Issue, PR, and Wiki page into a compressed ZIP archive.
- [ ] Integrate AWS KMS (Key Management Service) or HashiCorp Vault to enforce field-level encryption on PII (Email addresses, Usernames) across the entire enterprise database schema.
- [ ] Implement the "Data Residency" UI, allowing enterprise admins to select specific geographical regions (EU vs US) for storing their organization's Git blob data and databases.
- [ ] Provide automated GDPR Export compliance APIs: allow end-users to request their data and orchestrate the aggregation of their contributions across all enterprise repositories.
- [ ] Enforce "IP Allowlist" inheritance conflicts accurately: if a developer accesses via a compliant IP for Org A, but a non-compliant IP for Org B, strictly restrict cross-org GraphQL queries to Org A data only.
- [ ] Build the "Terms of Service" enforcement gate: require all new users provisioned via SAML/SCIM to explicitly accept a custom Enterprise EULA before accessing the dashboard.

### Tier 7: Advanced Network, Auth & Security Governance
- [ ] Architect the network layer to support AWS PrivateLink and Azure ExpressRoute, allowing enterprises to connect their VPCs directly to the instance without traversing the public internet.
- [ ] Implement the "Strict WebAuthn" global policy: force all organization members to register a FIDO2 hardware key, disabling TOTP (Authenticator App) fallbacks entirely.
- [ ] Build the Angular Security Center UI, providing a bird's-eye view of all CodeQL, Dependabot, and Secret Scanning alerts aggregated across all thousands of enterprise repositories.
- [ ] Implement cross-organization Forking networks: allow code to be securely forked between an Enterprise's internal organizations while remaining invisible to the public network.
- [ ] Configure dynamic CAPTCHA thresholds: utilize Redis to track failed login attempts globally across the enterprise and dynamically enforce hCaptcha on suspicious IP blocks.
- [ ] Implement precise SCIM mapping attributes: support extracting custom attributes (e.g., `department`, `cost_center`) from Azure AD payloads and mapping them to internal billing schemas.
- [ ] Build the SCIM Error Recovery Dashboard: expose detailed logs of failed SCIM synchronizations to enterprise admins to debug IdP group mapping errors.
- [ ] Implement the "Pat Revocation" global kill switch: allow Enterprise Admins to instantly invalidate all Personal Access Tokens across the entire enterprise with a single GraphQL mutation during a breach.
- [ ] Design the High-Availability (HA) failover logic for the Enterprise Management Plane, ensuring the administrative APIs remain available even if specific organization shards suffer an outage.

### Tier 8: Enterprise Federation (GitHub Connect Parity)
- [ ] **Federation Protocol Definition:** Define the secure gRPC/mTLS protocol linking an isolated, on-premise Enterprise Server clone instance securely to the public Cloud instance infrastructure.
- [ ] **Unified Search Sync:** Implement the indexing bridge allowing Enterprise users to search across both their private on-premise repositories and public Cloud repositories simultaneously from a single Angular interface.
- [ ] **Dependabot Advisory Mirroring:** Build a daily synchronization worker that pulls the latest Open Source Vulnerabilities (OSV) databases and Dependabot Advisory feeds from the Cloud instance down to the air-gapped Enterprise instance.
- [ ] **License Usage Aggregation:** Implement the automatic telemetry ping that securely transmits anonymized active-user counts from the on-premise Enterprise server up to the Cloud billing engine for unified invoice calculation.
- [ ] **Federated Actions Marketplace:** Configure the on-premise Actions orchestrator to seamlessly proxy `uses: actions/checkout@v4` requests up to the public Cloud marketplace, caching the tarballs locally.
- [ ] **Cross-Environment Authentication:** Implement OAuth token bridging, allowing users to securely link their enterprise corporate identity with their public developer identity to share specific achievements or sponsorships.
- [ ] **Federated Audit Logging:** Allow enterprise administrators to automatically ship on-premise audit events up to the Cloud instance's unified audit log sink for consolidated SIEM ingestion.

### Tier 9: Custom Pre-Receive Hooks Management
- [ ] **Global Hooks Schema:** Define the `enterprise_pre_receive_hooks` Postgres table storing the raw bash/python script payloads, enforcement states, and global repository exclusions.
- [ ] **Hook Administration UI:** Build the Enterprise Admin UI in Angular to author, upload, test, and deploy global pre-receive hooks with inline syntax highlighting.
- [ ] **Simulated Dry-Run Execution:** Implement a dry-run feature: allow admins to execute a newly drafted hook against a sample of recent historical pushes to estimate false-positive failure rates before global enforcement.
- [ ] **Granular Targeting:** Support granular enforcement policies: configure hooks to run on "All repositories", "Specific organizations", or explicitly "Exclude specific repositories".
- [ ] **Hook Execution Ordering:** Build a priority weighting system allowing administrators to dictate the exact sequential execution order of multiple overlapping pre-receive hooks.
- [ ] **Audit Logging for Hooks:** Emit detailed audit log events whenever a custom pre-receive hook is created, modified, or bypassed by an administrator.

### Tier 9.1: Advanced Pre-Receive Hooks Management
- [ ] **Monaco Editor Integration:** Embed the Monaco Editor specifically within the Hook Administration Angular view, configuring it for Python and Bash syntax validation.
- [ ] **Dynamic Environment Variable Allowlisting:** Build a UI panel allowing Enterprise Admins to securely define static environment variables or inject specific API tokens specifically isolated to the hook's execution context.
- [ ] **Failure Telemetry Dashboard:** Implement an internal telemetry dashboard plotting hook execution times, timeout rates, and non-zero exit codes to detect when a custom hook is slowing down developer productivity globally.
- [ ] **A/B Testing & Phased Rollout:** Implement a routing configuration allowing admins to deploy a new version of a pre-receive hook to a randomized 5% subset of pushes for canary testing.
- [ ] **Global Error Log Viewer:** Build an Angular view that aggregates the `stderr` outputs of failed hook executions centrally, allowing enterprise admins to debug user-reported push rejections without querying server logs.
- [ ] **Admin Bypass Webhook Trigger:** Configure a distinct internal webhook (`hook_bypassed.alert`) to instantly notify enterprise security teams via Slack/Teams if an administrator utilizes their override privileges to push past a failing hook.
