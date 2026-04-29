# Plan for gh-clone-status

### Tier 1: TimescaleDB Core Architecture & Metrics
- [ ] Scaffold the standalone `status-api` microservice entirely in Rust utilizing `axum` and `tokio`, decoupling it completely from the primary replica API to ensure it remains available during total system outages.
- [ ] Deploy a dedicated, highly-available PostgreSQL cluster specifically for the status application, physically isolated from the main application databases to prevent cascading failures.
- [ ] Enable the `TimescaleDB` extension on the Postgres cluster to natively support high-volume time-series metric insertion for uptime and latency tracking.
- [ ] Define the `metrics_hypertable` in TimescaleDB: `time` (TIMESTAMPTZ), `component_id`, `region`, `metric_name` (e.g., `latency_ms`, `error_rate`), `value` (DOUBLE PRECISION).
- [ ] Configure `chunk_time_interval` on the hypertable explicitly to 1 day to optimize memory usage and query performance for the recent 24-hour window.
- [ ] Implement TimescaleDB continuous aggregate policies to automatically downsample raw 1-second metrics into 1-minute, 1-hour, and 1-day averages.
- [ ] Configure TimescaleDB compression policies on chunks older than 7 days, utilizing delta-of-delta and Gorilla compression to reduce storage footprint by 90%.
- [ ] Define the `components` Postgres table: `id`, `name` (e.g., "Git Operations", "API Requests", "Webhooks"), `description`, `current_status` (operational, degraded, partial_outage, major_outage), `group_id`.
- [ ] Define the `incidents` table: `id`, `name`, `status` (investigating, identified, monitoring, resolved), `impact_override`, `created_at`, `resolved_at`.
- [ ] Define the `incident_updates` table to store chronological communication logs, Markdown bodies, and timestamps associated with a specific incident.
- [ ] Implement the core REST API endpoints: `GET /api/v1/summary`, `GET /api/v1/status`, `GET /api/v1/incidents/unresolved`, and `GET /api/v1/metrics`.
- [ ] Enforce strict caching HTTP headers (`Cache-Control: public, s-maxage=30, stale-while-revalidate=60`) on all public REST endpoints to ensure edge CDNs absorb 99.9% of outage traffic spikes.

### Tier 2: Automated Probing & Synthetic Monitoring
- [ ] Build the `probe-worker` daemon in Rust: a distributed synthetic monitoring agent deployed across multiple geographic availability zones (e.g., US-East, EU-West, AP-South).
- [ ] Implement the API Synthetic Probe: execute an authenticated GraphQL query against the main replica every 10 seconds via `reqwest`, explicitly asserting HTTP 200 responses and JSON schema validation.
- [ ] Implement the Git SSH Synthetic Probe: execute an automated `git ls-remote ssh://git@replica/test.git` via `libssh2-sys` to continuously validate SSH handshake connectivity and Gitaly/Spokes backend health.
- [ ] Implement the Git HTTPS Synthetic Probe: execute an automated `git fetch` over the Smart HTTP protocol to monitor edge proxy routing latency, HTTP/2 multiplexing, and TLS negotiation times.
- [ ] Implement the Webhook Delivery Probe: securely trigger a dummy webhook event on the main API and measure the exact milliseconds elapsed until the payload hits a dedicated external validation endpoint.
- [ ] Implement DNS resolution probes utilizing the `trust-dns-resolver` crate to continuously track root domain DNS lookup latencies globally.
- [ ] Build the "Anomaly Detection Engine": calculate the rolling 5-minute standard deviation of latency metrics; if latency spikes beyond 3-sigma, automatically flag the component as `degraded_performance`.
- [ ] Implement strict quorum logic: explicitly require at least 2 out of 3 geographic probes to report a failure concurrently before automatically creating an incident, preventing false positives from localized network blips.
- [ ] Write probe metrics directly into the TimescaleDB `metrics_hypertable` via asynchronous bulk `INSERT` statements utilizing `sqlx` every 5 seconds.

### Tier 3: Angular Zoneless Public Dashboard
- [ ] Bootstrap the frontend dashboard strictly using Angular 21 with Zoneless Change Detection (`provideExperimentalZonelessChangeDetection`) for maximum rendering performance on mobile devices.
- [ ] Configure Angular Server-Side Rendering (SSR) via `@angular/ssr` and deploy as an Edge Function (e.g., Cloudflare Workers) to guarantee immediate time-to-interactive HTML delivery during an incident.
- [ ] Design the global "Status Banner": dynamically render a Green (Operational), Yellow (Minor Outage), or Red (Major Outage) hero block based on the aggregated component health.
- [ ] Implement the Component List UI: render collapsible groups (e.g., "Core Services" -> "API", "Webhooks", "Git LFS") with exact text descriptions and SVG icons.
- [ ] Build the Uptime History Bar: utilize SVG to render 90 days of historical uptime blocks (green/yellow/red rects) for each component, precisely matching the GitHub Status aesthetic.
- [ ] Implement accessible tooltip interactions on the Uptime History Bar to explicitly display the date and calculated uptime percentage (e.g., `99.98%`) for that specific day to screen readers.
- [ ] Build the active "Incidents" feed, rendering unresolved incidents chronologically with their associated updates (Investigating, Identified, Monitoring, Resolved) utilizing Angular Control Flow (`@for`, `@if`).
- [ ] Implement the historical "Past Incidents" view, providing offset-paginated access to resolved incidents grouped by month and year.
- [ ] Integrate Echarts or D3.js (wrapped in a native Angular directive) to render real-time, interactive line charts plotting API latency and webhook delivery times over the last 24 hours.
- [ ] Implement Progressive Web App (PWA) offline support via `@angular/service-worker` to cache the last known status payload if the user loses network connectivity.
- [ ] Enforce strict Primer Design System adherence, explicitly extracting and compiling CSS variables independently so the status page visual identity matches the core product natively.
- [ ] Implement automatic dark/light/high-contrast mode synchronization based on the user's `prefers-color-scheme` CSS media query.

### Tier 4: Incident Management & SRE Console
- [ ] Build a secure, internal Angular application specifically for Site Reliability Engineers (SREs) to manually manage incidents, accessible exclusively via enterprise VPN or strict OAuth + FIDO2 policies.
- [ ] Implement the "Create Incident" wizard: allow SREs to define the Title, select impacted components, override the algorithmically calculated status, and draft the initial "Investigating" communication.
- [ ] Support rich-text Markdown editing for incident updates, utilizing `marked.js` to allow linking to internal post-mortems, Grafana dashboards, or external runbook documentation.
- [ ] Build the "Component Override" toggle: allow admins to manually force a component into a `major_outage` state indefinitely, overriding synthetic probe data during complex partial outages.
- [ ] Implement an automated "Resolved" state machine: when an incident is explicitly marked resolved, automatically revert overridden component statuses back to the real-time probe states.
- [ ] Build the "Scheduled Maintenance" workflow: allow SREs to post upcoming downtime windows, automatically generating yellow warning banners on the public dashboard exactly 48 hours in advance.
- [ ] Integrate a rigorous audit log within the admin console, strictly recording exactly which SRE mutated a component's status or published an incident update to PostgreSQL.
- [ ] Implement a "Post-Mortem Generator": automatically aggregate the incident timeline, metrics peaks, and update logs into a formatted Markdown document for internal compliance review post-resolution.

### Tier 5: Subscriptions, PubSub & Notifications
- [ ] Define the `subscribers` Postgres table: `id`, `endpoint` (email address, webhook URL, phone number), `protocol` (email, webhook, sms), `verified_at`.
- [ ] Implement the Public Subscription UI: allow users to click "Subscribe to Updates" and provide their contact endpoint via a secure, rate-limited Angular form.
- [ ] Implement the verification loop: dispatch a cryptographically signed confirmation link via email/SMS, requiring the user to explicitly click to activate the subscription to prevent spam abuse.
- [ ] Build the asynchronous `NotificationDispatcher` worker in Rust, subscribing to an internal Redis PubSub channel for new `incident_update` events.
- [ ] Implement Email Dispatching: integrate with AWS SES or SendGrid APIs via Rust to fan-out HTML and plaintext incident emails to tens of thousands of subscribers concurrently.
- [ ] Implement SMS Dispatching: integrate with the Twilio REST API to dispatch short, critical outage alerts to verified phone numbers, adhering to strict carrier rate limits.
- [ ] Implement Webhook Dispatching: execute parallel `HTTP POST` requests to subscriber webhooks utilizing `reqwest`, strictly enforcing a 5-second timeout and generating HMAC-SHA256 signatures for payload verification.
- [ ] Build a robust Dead-Letter Queue (DLQ) in Kafka/Redis to catch failed webhook deliveries, implementing an exponential backoff retry mechanism (max 5 retries).
- [ ] Design an opt-out/unsubscribe architecture: embed a secure, one-click unsubscribe link containing a JWT at the bottom of every dispatched email or SMS.
- [ ] Allow granular subscriptions: enable users to subscribe exclusively to specific components (e.g., only receive alerts if "Actions" goes down, ignoring "Codespaces").

### Tier 6: SLA Reporting, Analytics & API Integrations
- [ ] Build a background Rust worker that executes daily aggregations over the TimescaleDB continuous aggregates to compute exact fractional Service Level Agreement (SLA) uptime percentages (e.g., `99.995%`).
- [ ] Expose an administrative endpoint to generate monthly PDF/CSV uptime reports specifically designed for Enterprise customers requiring formal SLA compliance verification and penalty calculations.
- [ ] Implement native integration with PagerDuty / Opsgenie: automatically create a draft incident on the Status Page if a critical PagerDuty alert is triggered for the primary replica ecosystem.
- [ ] Build the X/Twitter API integration: optionally auto-tweet short updates to a dedicated `@GitHubCloneStatus` account whenever an incident transitions to a new state.
- [ ] Implement a Slack/Microsoft Teams incoming webhook integration, allowing the status backend to push rich-formatted alerts directly into the enterprise's internal `#devops` or `#engineering` channels.
- [ ] Provide an `atom.xml` and `rss.xml` feed endpoint natively generated in Rust, mapping unresolved incidents and recent updates into standard XML syndication formats for RSS readers.
- [ ] Provision specific, read-only API tokens to allow enterprise monitoring tools (Datadog, New Relic) to securely ingest the raw `status-api` health metrics.

### Tier 7: Edge Protection & Security
- [ ] Implement an aggressive Token Bucket rate limiter in Rust to cap public API `/status` and `/metrics` endpoint requests by IP address to mitigate L7 DDoS attacks.
- [ ] Deploy Cloudflare Web Application Firewall (WAF) rules specifically tuned to drop malicious payloads targeting the subscriber registration endpoints.
- [ ] Enforce strict mTLS (Mutual TLS) between the distributed `probe-workers` and the central `status-api` to prevent malicious actors from spoofing synthetic health metrics.
- [ ] Integrate Let's Encrypt ACME automated certificate renewal natively within the Rust edge server to ensure `status.replica.com` TLS certificates never expire during an incident.
- [ ] Implement CORS (Cross-Origin Resource Sharing) policies explicitly whitelisting the Angular frontend domain while allowing `GET` access from anywhere for API consumption.
