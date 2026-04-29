# Plan for gh-clone-deployments

### Tier 1: Environments & Protection Rules
- [ ] Define the `environments` Postgres table: `id`, `repo_id`, `name` (e.g., `production`, `staging`), `created_at`.
- [ ] Define the `environment_protection_rules` table to support required reviewers, wait timers, and custom branch deployment restrictions.
- [ ] Implement the Rust REST API for creating, updating, and deleting environments (`/repos/{owner}/{repo}/environments/{name}`).
- [ ] Build the Angular UI for Environment settings, allowing users to configure protection rules, deployment branches, and environment-specific secrets.
- [ ] Implement the "Wait Timer" execution logic: when a deployment is triggered, pause the workflow execution in the backend queue until the exact `wait_timer` duration has elapsed.
- [ ] Implement the "Required Reviewers" state machine: transition a deployment to `waiting` and dispatch notifications to specified users or teams.
- [ ] Build the Angular Approval UI on the workflow run page, allowing authorized users to click "Approve" or "Reject" to unblock the deployment.
- [ ] Support dynamic branch policies: configure environments to only accept deployments from the `main` branch or specific release tags (`v*`).

### Tier 2: Deployment APIs & Tracking
- [ ] Define the `deployments` table: `id`, `repo_id`, `environment_id`, `ref`, `sha`, `task`, `payload` (JSONB), `creator_id`.
- [ ] Define the `deployment_statuses` table to track state transitions (`queued`, `in_progress`, `success`, `failure`, `inactive`).
- [ ] Implement the `POST /repos/{owner}/{repo}/deployments` endpoint to allow external CI/CD systems to trigger deployments.
- [ ] Implement the `POST /repos/{owner}/{repo}/deployments/{id}/statuses` endpoint for updating the active state of a deployment.
- [ ] Ensure the creation of a new deployment status automatically transitions older active deployments for the same environment to the `inactive` state.
- [ ] Implement transient environments: allow API requests to automatically create temporary environments (e.g., for PR review apps) and mark them for automatic deletion upon PR closure.
- [ ] Build the "Deployments" timeline UI on the repository homepage, rendering the current active deployment and its associated commit status.
- [ ] Generate comprehensive webhook payloads (`deployment`, `deployment_status`) and dispatch them to subscribed listeners for chatops integrations.

### Tier 3: Merge Queue State Machine
- [ ] Define the `merge_queues` and `merge_queue_entries` Postgres tables to orchestrate high-velocity Pull Request merging.
- [ ] Implement the "Add to Merge Queue" Angular UI action, replacing the standard "Merge" button when the branch protection rule requires it.
- [ ] Build the Rust background orchestrator that continuously evaluates the state of the `merge_queues` table.
- [ ] Implement the dynamic temporary branch creation logic (`gh-readonly-queue/main/pr-123-456`), combining the base branch with the queued PR commits.
- [ ] Trigger CI/CD workflows automatically against the temporary merge queue branch.
- [ ] Implement the state machine transition: if CI passes, automatically fast-forward/squash the base branch to exactly match the temporary branch and mark the PR as Merged.
- [ ] Implement the eviction logic: if CI fails, remove the PR from the queue, delete the temporary branch, and notify the author via email/webhooks.
- [ ] Implement queue batching: squash multiple queued PRs into a single temporary branch test run to exponentially increase merge throughput for massive monorepos.

### Tier 4: Environment Secrets & Variables
- [ ] Define the `environment_secrets` and `environment_variables` tables, isolated strictly to their corresponding environment ID.
- [ ] Implement AES-256-GCM envelope encryption at rest specifically for `environment_secrets`.
- [ ] Ensure the Actions Runner control plane injects environment-scoped secrets *only* if the job explicitly declares `environment: {name}` and all protection rules have passed.
- [ ] Build the Angular UI for managing environment secrets, mimicking the repository-level secrets UI but isolated to the environment view.
- [ ] Implement hierarchical variable resolution: during a workflow run, ensure Environment variables correctly override Repository variables, which override Organization variables.
- [ ] Provide an audit log specifically tracking when an environment secret is updated, tracking the actor and timestamp for compliance.

### Tier 5: Telemetry & DORA Metrics
- [ ] Track precise timestamps for deployment state transitions (`created` -> `in_progress` -> `success`) to calculate Deployment Lead Time.
- [ ] Aggregate deployment frequencies per environment and expose this data via a GraphQL API for internal reporting.
- [ ] Implement the DORA Metrics Insights Dashboard in Angular, rendering Deployment Frequency and Lead Time for Changes charts.
- [ ] Calculate the Change Failure Rate by analyzing deployment failures versus successes within a given rolling 30-day window.
- [ ] Implement integration points for incident management tools (e.g., PagerDuty) to track Mean Time to Restore (MTTR) and correlate it with specific deployment SHAs.
### Tier 6: External Gates, OIDC & Custom Integrations
- [ ] Implement "Custom Deployment Protection Rules": allow environments to block deployments pending an HTTP 200 OK response from external webhook payloads (e.g., Jira, ServiceNow).
- [ ] Build the cryptographic payload signing mechanism (HMAC-SHA256) for outbound deployment webhooks to ensure third-party systems can verify the origin of the approval request.
- [ ] Implement dynamic OIDC (OpenID Connect) Trust Policies: strictly bind cloud provider roles (AWS IAM/GCP) to specific Environment IDs rather than just the repository.
- [ ] Build the Angular UI for rendering complex custom payload responses from external gates (e.g., displaying a ServiceNow Ticket ID and Status directly inside the deployment timeline).
- [ ] Implement the "Bypass Protection" RBAC configuration: allow Organization Admins to force-push a deployment through a failing external gate during a critical Sev-1 incident.
- [ ] Expose a GraphQL mutation `createDeploymentStatus` explicitly designed for third-party GitOps controllers (like ArgoCD or Flux) to report sync health back to the UI.
- [ ] Implement the Deployment Matrix: support deploying to multiple environments in parallel (e.g., `staging-us-east`, `staging-eu-west`) mapping to a single Actions matrix job.
- [ ] Build a continuous deployment polling worker to automatically detect and flag "Drift" if the deployed SHA in a tracked environment diverges from the recorded deployment state without an API trigger.

### Tier 7: Advanced Merge Queue Orchestration
- [ ] Implement the backend Rebase vs Squash engine specifically for Merge Queues: allow administrators to enforce linear history by configuring the queue to strictly `git rebase` instead of merge.
- [ ] Build the "Flaky Test Auto-Retry" mechanism: if a PR fails in the merge queue due to a known flaky CI job, automatically re-queue it once before ejecting it entirely.
- [ ] Implement Queue Concurrency Limits: constrain the maximum number of PRs actively running matrix CI builds in the queue to prevent exhausting the organization's runner pool.
- [ ] Build the "Jump the Queue" feature: allow users with Admin permissions to flag a hotfix PR with high priority, instantly moving it to the front of the queue and preempting running tests.
- [ ] Implement exact Git tree caching for batching: when batching 5 PRs, dynamically compute intermediate Git trees to avoid completely rebuilding the branch if PR #3 fails.
- [ ] Build the Angular UI for Merge Queue insights, rendering estimated time to merge (ETM) based on current CI durations and queue depth.
- [ ] Expose webhooks for `merge_group` events, ensuring third-party CI systems (like Jenkins or Buildkite) can trigger correctly against the temporary `gh-readonly-queue` branch.
- [ ] Implement strict stale PR eviction: if a PR has been in the queue for >24 hours due to failing/missing status checks, automatically remove it and leave an automated comment.
- [ ] Synchronize the Merge Queue state machine with the Code Review State Machine to instantly eject PRs from the queue if a required reviewer dismisses their approval while the PR is waiting.

### Tier 8: GitHub Pages Environment Linkage
- [ ] Define the implicit `github-pages` Environment dynamically during repository initialization to support automated deployment gates.
- [ ] Implement linkage in the Actions runner: when a workflow utilizes the `deploy-pages` action, automatically trigger the state machine for the `github-pages` environment.
- [ ] Map the completed deployment's output (`page_url`) directly to the repository's `homepage` UI and the active Deployments timeline.
- [ ] Support custom Environment Protection Rules specifically for the Pages environment, ensuring internal enterprise sites require a manager's explicit approval before routing the edge DNS to the new build.
