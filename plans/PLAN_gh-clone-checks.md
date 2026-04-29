# Plan for gh-clone-checks

### Tier 1: Check Suites & Check Runs Data Model
- [ ] Define the `check_suites` Postgres table: `id`, `repo_id`, `head_sha`, `head_branch`, `status` (queued, in_progress, completed), `conclusion` (success, failure, neutral, cancelled, skipped, timed_out, action_required).
- [ ] Define the `check_runs` Postgres table: `id`, `suite_id`, `name`, `status`, `conclusion`, `started_at`, `completed_at`, `external_id`, `details_url`.
- [ ] Create a composite Postgres index on `(repo_id, head_sha)` to optimize the highly trafficked PR status checks queries.
- [ ] Build the Rust Axum REST API endpoints: `POST /repos/{owner}/{repo}/check-runs` and `PATCH /repos/{owner}/{repo}/check-runs/{check_run_id}`.
- [ ] Implement payload validation: ensure `name` is required and `status` enforces strict state machine transitions (queued -> in_progress -> completed).
- [ ] Implement cryptographic authorization: verify the issuing GitHub App JWT strictly possesses the `checks:write` permission scope for the target repository.
- [ ] Build the "Check Suite Request" webhook dispatcher: emit `check_suite.requested` and `check_run.created` events instantly via Kafka to external integrations.
- [ ] Support the `rerequested` action: map the UI "Re-run all checks" button to a backend workflow that wipes existing suite states and re-emits webhooks.

### Tier 2: Output Logs & Cold Storage Handoff
- [ ] Define the `check_run_outputs` Postgres JSONB schema for metadata: `title`, `summary` (max 65535 chars), and `text`.
- [ ] Architect the raw log storage: route all multi-megabyte step execution logs strictly to an S3/MinIO backend, never storing raw build text in Postgres.
- [ ] Build the Rust log ingestion endpoint accepting chunked `PATCH` requests to continuously append to an active check run's S3 object utilizing multipart uploads.
- [ ] Implement an active-log Redis buffer: buffer incoming fast-streaming log chunks in Redis for 5 seconds before flushing to S3 to reduce object storage API costs.
- [ ] Implement the Angular `CheckRunLogViewer` component using `@angular/cdk/scrolling` virtual scrolling to render 50,000+ line logs without browser DOM freezing.
- [ ] Implement an ANSI color code parser in the Angular UI to accurately convert terminal color escapes (e.g., `\x1b[31m`) into styled HTML spans.
- [ ] Add regex-based deep-linking to the log viewer, allowing users to generate shareable URLs that automatically scroll to and highlight specific log lines (e.g., `#step:4:128`).
- [ ] Build a Server-Sent Events (SSE) log streaming router in Rust to multiplex active log writes directly to connected Angular clients for a real-time terminal experience.
- [ ] Implement strict log sanitization on the Rust backend: utilize Aho-Corasick automaton to redact known repository secrets and PATs from the raw byte stream *before* S3 persistence.

### Tier 3: Annotations & Code Diff Integration
- [ ] Define the `check_run_annotations` table: `check_run_id`, `path`, `start_line`, `end_line`, `start_column`, `end_column`, `annotation_level` (notice, warning, failure), `message`, `raw_details`.
- [ ] Implement the `POST` payload parser for annotations, enforcing GitHub's strict limit of 50 annotations per API request.
- [ ] Build an asynchronous `tokio` worker in Rust to batch-insert massive arrays of annotations into Postgres efficiently using bulk `INSERT` statements.
- [ ] Map the `check_run_annotations` directly to the "Files Changed" PR diff viewer utilizing the specific `path` and `start_line` coordinates.
- [ ] Render floating inline annotation blocks within the Angular code viewer, displaying the exact error message, level icon (Red X or Yellow Triangle), and the originating Check Run name.
- [ ] Implement the "Resolve Annotation" interaction, allowing reviewers to visually collapse a linter warning block after it has been discussed.
- [ ] Build the Pull Request Merge Gate validation engine: strictly disable the "Merge" button if *any* Check Run associated with the `head_sha` has a `failure` conclusion and is marked as "Required" in branch protection.
- [ ] Implement the UI summary badge: roll up the highest severity annotation level to the top-level PR conversation timeline (e.g., "3 annotations from 1 check").

### Tier 4: Code Coverage, Security & Advanced Actions
- [ ] Define the `check_run_coverage` Postgres table to store line-by-line coverage percentage rollups per file path.
- [ ] Implement a native parser in Rust for standard coverage formats (Cobertura, LCOV, JaCoCo) submitted as XML/JSON attachments to the Checks API.
- [ ] Render the Coverage UI in Angular: inject green (covered) and red (uncovered) vertical indicator lines directly into the gutters of the PR diff viewer.
- [ ] Implement "Coverage Drop" protection logic: calculate the absolute delta between the base branch coverage and the head branch coverage, failing a suite if the drop exceeds configured thresholds.
- [ ] Implement the `action_required` conclusion state: render a custom Angular "Resolve" button that redirects the user to the external CI platform URL defined in `details_url` for manual deployment approvals.
- [ ] Build the Check Suite "Cancel" workflow: expose an API to forcefully terminate an `in_progress` suite, emitting the `cancelled` webhook and updating the DB state.
- [ ] Implement strict data retention cron jobs using `pg_cron`: automatically delete all S3 raw logs and Postgres annotations older than 400 days to comply with standard platform storage limits.
- [ ] Export Prometheus metrics tracking Check API latency and insertion rates to ensure the database can withstand massive monorepo CI bursts.

### Tier 5: Enterprise Compliance & Metrics
- [ ] Add the `check_suite_retries` table to track how many times a suite was restarted to accurately calculate flakiness metrics.
- [ ] Expose GraphQL queries for Check Run durations, allowing Enterprise Admins to build Grafana dashboards of CI queue times and execution delays.
- [ ] Implement RBAC checks for log viewing: ensure sensitive Check Run logs inside Private repositories are strictly gated from public users or unauthorized App tokens.
- [ ] Build an automatic Dead-Letter Queue (DLQ) in the webhook dispatcher to ensure external CI systems receive delayed `check_suite.requested` payloads if they suffer an outage.
