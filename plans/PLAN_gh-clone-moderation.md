# Plan for gh-clone-moderation

### Tier 1: Trust & Safety Data Models and User Reporting
- [ ] Define the `user_reports` Postgres table: `id`, `reporter_id`, `reported_user_id`, `reported_repo_id`, `reason` (spam, abuse, malware), `context_url`, `status` (open, resolved, dismissed).
- [ ] Build the Angular "Report Abuse" flow: a multi-step modal allowing users to categorize the abuse and provide a markdown-formatted detailed explanation.
- [ ] Implement the `user_blocks` table, enforcing strict database-level constraints: blocked users cannot comment on, star, fork, or view the blocking user's repositories.
- [ ] Define the `organization_blocks` table, mapping blocking logic to apply globally across all repositories owned by the organization.
- [ ] Build the Trust & Safety Administrative Dashboard in Angular: provide internal moderators a queue of open reports with quick-action buttons (Suspend, Hide Content, Dismiss).
- [ ] Implement the "Shadow Ban" (Spammy) user state flag: the user's interface appears fully functional to themselves, but their repositories, issues, and comments return HTTP 404 to the public network.
- [ ] Build a robust Audit Log specifically for Trust & Safety actions, tracking exactly which moderator suspended an account or hid a comment, ensuring internal accountability.

### Tier 2: Automated Spam Mitigation & Heuristics
- [ ] Integrate a high-speed Rust heuristic engine to monitor PR and Issue creation rates; automatically shadow-ban accounts creating > 50 identical PRs across different repositories in under an hour.
- [ ] Build a background worker integrating with the Akismet API (or an internal Rust-bound ONNX ML model) to score the text of every newly created Issue and Comment for spam likelihood.
- [ ] Configure the ML pipeline to automatically flag high-scoring spam content, queue it for manual moderator review, and temporarily collapse the comment in the Angular UI.
- [ ] Utilize Postgres `pg_trgm` (trigram matching) to detect and flag identical, copy-pasted comments deployed across multiple unrelated repositories simultaneously.
- [ ] Implement IP reputation tracking utilizing Redis: track registration and authentication velocity; automatically trigger hCaptcha or block CIDR ranges exhibiting known botnet behavior.
- [ ] Integrate PhotoDNA or a similar perceptual hashing API to continuously scan all user-uploaded avatars and issue attachments for CSEM (Child Sexual Abuse Material).
- [ ] Implement the CSEM emergency protocol: instant, irreversible global account termination, payload quarantining, and automated JSON report generation for NCMEC.
- [ ] Enforce dynamic "Interaction Limits": allow repository maintainers to temporarily restrict comments/PRs to "Existing Users" or "Collaborators Only" for 24h, 7 days, or 6 months during viral harassment events.

### Tier 3: DMCA, Legal Takedowns & Trade Compliance
- [ ] Define the `dmca_takedowns` Postgres table: `id`, `repository_id`, `claimant_name`, `original_work_url`, `legal_document_s3_key`, `status`.
- [ ] Build the internal legal intake pipeline UI, allowing the legal team to process signed DMCA notices and apply immediate restrictions to the target repository.
- [ ] Implement the "Repository Disabled" UI state in Angular: replace the codebase and file explorer views with a stark legal notice explaining the DMCA takedown.
- [ ] Enforce the takedown at the Git networking layer: intercept and drop any `git clone`, `git fetch`, or `git push` requests targeting the disabled repository directly at the `babeld`/`spokes` level.
- [ ] Implement the "Counter Notice" workflow: build a secure Angular form allowing the repository owner to submit a legally binding counter-claim (with digital signature) directly through the platform.
- [ ] Build the "Fork Network Quarantine" heuristic: automatically traverse the disabled repository's fork tree and optionally apply the DMCA takedown to all downstream forks containing the infringing commit.
- [ ] Implement OFAC / Trade Sanctions compliance: utilize GeoIP resolution at the edge proxy to block access to private repositories and paid features from comprehensively sanctioned regions.

### Tier 4: Account Recovery, Appeals & GDPR Erasure
- [ ] Build the "Suspended Account" login experience: allow the user to authenticate but strictly restrict their navigation to a single "Appeal Suspension" Angular page.
- [ ] Implement the `account_appeals` database schema and REST endpoints for suspended users to submit textual explanations or request manual review.
- [ ] Provide Trust & Safety moderators an "Impersonation View" (with strict, non-bypassable audit logging) to view the platform exactly as the suspended user sees it to verify claims.
- [ ] Implement the automated revocation trigger: immediately wipe all active Personal Access Tokens, OAuth grants, and SSH keys the millisecond an account is transitioned to `suspended`.
- [ ] Implement the GDPR "Right to be Forgotten" destructive worker: completely obliterate a user's PII, anonymize their username, and delete their repositories.
- [ ] Ensure the GDPR worker implements a strict 30-day delayed soft-delete phase before executing the irreversible `DELETE` commands against the Postgres master.

### Tier 5: Transparency & Legal Holds
- [ ] Implement the `legal_holds` Postgres table: provide APIs to permanently lock an account's data (preventing the GDPR worker from deleting it) during active litigation.
- [ ] Automate the "Transparency Report" data pipeline: write Rust aggregate queries to automatically calculate the total number of DMCA takedowns, spam suspensions, and CSEM reports per year.
- [ ] Implement an automated notification webhook system to alert internal legal teams instantly if an appeal includes specific keywords (e.g., "lawsuit", "attorney").
- [ ] Configure Axum middleware to permanently return `451 Unavailable For Legal Reasons` HTTP status codes for endpoints that correspond to a successful DMCA takedown.
