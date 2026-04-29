# Plan for gh-clone-migrations

### Tier 1: High-Fidelity Data Export (User & Org Takeout)
- [ ] Architect the `MigrationArchiveWorker` in Rust to orchestrate asynchronous, long-running export tasks without blocking HTTP threads.
- [ ] Build the Git data exporter module: execute `git clone --mirror` locally on the storage node to bundle the raw repository objects.
- [ ] Implement the Postgres-to-JSON serialization engine: deeply join and extract all Issues, PRs, Comments, Releases, Labels, and Milestones associated with a repository.
- [ ] Map the extracted JSON strictly to the schema utilized by the official GitHub Enterprise Importer to guarantee structural cross-compatibility.
- [ ] Develop the LFS and Attachment exporter: download all associated binaries from the S3 backend and organize them into an `attachments/` subdirectory.
- [ ] Compress the entire staging directory into a highly optimized `tar.gz` archive using streaming compression (`flate2`) to minimize RAM overhead on the worker node.
- [ ] Upload the finalized archive to a secure, isolated S3 bucket specifically designated for exports.
- [ ] Generate a cryptographically signed, 7-day expiring pre-signed URL for the S3 object.
- [ ] Dispatch an automated HTML email notification to the user containing the secure download link and the archive's SHA-256 checksum.

### Tier 2: The Core Ingestion Engine (Importer)
- [ ] Build the `MigrationIngestionWorker` in Rust to handle the reverse process: downloading a provided archive URL and unpacking it safely into a temporary sandbox.
- [ ] Implement rigorous JSON schema validation on the uploaded `export.json` payload, rejecting archives that are malformed or suspiciously large.
- [ ] Build the ID Translation Engine: as legacy Users, Issues, and PRs are inserted into Postgres, maintain a persistent translation table mapping `legacy_id` -> `new_id`.
- [ ] Execute a Git push-mirror to internal Spokes storage to instantiate the repository's source code and commit history on the new instance.
- [ ] Implement "Ghost User" attribution logic: if an exported comment belongs to an email address not registered on the new instance, attribute it to the system `@ghost` user but prepend the original author's name in the comment body.
- [ ] Safely rewrite all internal Markdown issue references (e.g., `#123` -> `#456`) across all comments using the ID Translation Engine to ensure cross-links remain valid.
- [ ] Upload all extracted attachments to the internal S3 storage and execute regex replacements in the markdown bodies to update image URLs to the new domain.
- [ ] Reconstruct the Git Pull Request branch references (`refs/pull/{id}/head`) natively in `gitoxide` to explicitly match the newly generated Postgres PR IDs.

### Tier 3: Third-Party VCS API Connectors
- [ ] Build the GitLab API connector: utilize the user's provided GitLab Personal Access Token to traverse and paginate through `gitlab.com/api/v4` endpoints.
- [ ] Implement state mapping: translate GitLab Merge Requests (and their specific threaded resolution states) precisely to GitHub Pull Requests and Review Comments.
- [ ] Build the Bitbucket API connector: support distinct pagination and authentication strategies for both Bitbucket Cloud (REST API) and Bitbucket Server (legacy).
- [ ] Map Bitbucket Workspaces to GitHub Organizations, translating Bitbucket repository access control lists into GitHub Teams.
- [ ] Build the Subversion (SVN) connector: execute `git-svn` within a rigorously restricted sandbox to perform bidirectional translation of SVN revisions into Git commits.
- [ ] Implement the Team Foundation Server (TFS) connector utilizing `git-tfs` to clone legacy .NET enterprise codebases into standard Git histories.
- [ ] Implement an exponential backoff and retry mechanism for all external API connectors to gracefully handle upstream rate limits during massive migrations.

### Tier 4: Angular Migration Dashboard & Real-Time Tracking
- [ ] Build the Angular Migration Dashboard: provide a wizard for users to input source credentials (e.g., GitLab Token), validate them, and select repositories to migrate in bulk.
- [ ] Implement Server-Sent Events (SSE) streaming from the Rust `MigrationIngestionWorker` to the Angular UI to render real-time, granular progress bars (e.g., "Importing issue 450/1000").
- [ ] Build an "Error Resolution" UI: if an import task fails (e.g., due to a missing attachment or malformed commit), allow the administrator to explicitly skip the failing entity and resume the migration.
- [ ] Provide Organization-level migration tracking views, giving Enterprise Admins a consolidated status board during a massive 500-repository migration weekend.
- [ ] Implement automated "Dry Run" capabilities: allow administrators to execute a simulated migration that only validates credentials, maps schemas, and calculates API limits without writing to Postgres.
- [ ] Generate a final "Migration Audit Report" PDF/CSV detailing exact counts of imported entities, skipped records, and warnings for compliance sign-off.

### Tier 5: Subsystem Exports & Limit Mitigations
- [ ] Extend the export worker to serialize GitHub Projects (Memex) CRDT tables into readable JSON arrays so Kanban boards are not lost during instance migrations.
- [ ] Implement logic to automatically export associated Wikis (`repository.wiki.git`) and package them neatly into the root of the `.tar.gz` export.
- [ ] Construct the `governor` rate limit overrides in Rust: ensure the importer backend bypasses standard rate limiters while executing bulk Postgres insertions.
- [ ] Build an automated pause-and-resume state machine: if a massive repository import times out or a node reboots, the worker must resume from the exact last inserted `legacy_id`.

### Tier 6: Actions Importer & CI/CD Translation Engine
- [ ] **Translation Engine Architecture:** Build the `actions-importer` microservice in Rust specifically designed to parse and transpile competing CI/CD pipeline definitions into valid GitHub Actions YAML.
- [ ] **GitLab CI/CD AST Parser:** Implement a robust AST parser in Rust utilizing `serde_yaml` to deeply ingest `.gitlab-ci.yml` files, handling nested `includes` and YAML anchors.
- [ ] **GitLab Translation Mapping:** Translate GitLab `stages`, `script` blocks, `rules` (if/else), and `artifacts` accurately into Actions `jobs`, `steps`, `if` conditionals, and `@actions/upload-artifact` directives.
- [ ] **GitLab Variables to Expressions:** Implement a regex-based substitution engine to convert GitLab variable syntax (e.g., `$CI_COMMIT_SHA`) into Actions expression syntax (`${{ github.sha }}`).
- [ ] **Jenkinsfile Groovy Parser:** Build a Groovy DSL parser utilizing a native Rust parsing combinator library (e.g., `nom` or `pest`) to extract declarative Jenkins pipelines.
- [ ] **Jenkins Translation Mapping:** Map Jenkins `agent any`, `stages`, `steps`, and `post` blocks into the corresponding Actions `runs-on`, job sequence, and `always()`/`failure()` conditionals.
- [ ] **Jenkins Credentials Mapping:** Implement intelligent secret scanning: map Jenkins `withCredentials` blocks automatically to `${{ secrets.MAPPED_NAME }}` syntax.
- [ ] **CircleCI Config Parser:** Implement a `.circleci/config.yml` parser in Rust, translating custom `executors` and `commands` into native Actions matrix strategies and reusable composite workflows.
- [ ] **CircleCI Orbs Translation:** Build a lookup table in Postgres mapping popular CircleCI Orbs (e.g., `circleci/node@5.0`) to their exact equivalent GitHub Actions in the Marketplace.
- [ ] **Travis CI Transpiler:** Parse `.travis.yml` language matrixes and lifecycle phases (`before_install`, `script`, `after_success`) into a flattened, sequential Actions job execution graph.
- [ ] **Custom Action Substitution Engine:** Implement a configurable mapping dictionary allowing enterprise admins to automatically replace a specific legacy Jenkins plugin with a designated internal Marketplace Action during bulk translation.
- [ ] **Dry-Run Audit API:** Expose a REST API endpoint that performs an in-memory translation and returns a comprehensive JSON report highlighting unsupported features, manual intervention requirements, and the proposed YAML output.
- [ ] **Confidence Scoring Algorithm:** Calculate a "translation confidence score" (0-100%) based on the number of un-translatable directives (e.g., obscure Jenkins plugins) to help admins triage manual migrations.
- [ ] **Actions YAML Serialization:** Utilize `serde_yaml` to format the finalized abstract syntax tree securely into idiomatic, human-readable Actions YAML, explicitly preserving comments where possible.
- [ ] **Bulk CI Migration Worker:** Build an asynchronous `tokio` background task that traverses hundreds of imported repositories, identifies legacy CI configurations, transpiles them, generates a new Git branch via `gitoxide`, and automatically opens Pull Requests proposing the migration.