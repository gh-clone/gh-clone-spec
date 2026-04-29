# Plan for gh-clone-gists

### Tier 1: Headless Git Storage Architecture
- [ ] Provision a separate logical storage shard namespace (`gists`) distinct from standard repositories to handle millions of miniature Git repos.
- [ ] Map the `POST /gists` API call directly to the internal `storage.GitProvider.InitBareRepo` routine to instantly initialize a headless Git repository.
- [ ] Translate multi-file REST JSON payloads into native `git hash-object`, `git mktree`, and `git commit-tree` operations programmatically via `libgit2` or `gitoxide`.
- [ ] Ensure gists function as fully compliant Git repositories, supporting standard `git clone https://gist.github.com/{hash}.git` operations over the Smart HTTP router.
- [ ] Implement a UUID v4 or high-entropy hash generation schema (e.g., `a1b2c3d4e5f6g7h8i9j0`) specifically for Secret Gists to prevent sequential discovery.
- [ ] Implement the fork mechanism: execute a highly optimized server-side shallow clone (`git clone --bare --shared`) of the gist's repo and link it to the new owner's database record.
- [ ] Enforce a strict max size limit on gists (e.g., 1MB per file, maximum 10 files per gist) to prevent users from abusing the namespace as a free CDN.
- [ ] Support downloading the entire gist revision history as a zip archive by dynamically executing `git archive HEAD` and piping to the HTTP response.

### Tier 2: Frontend Editor & Revision UI
- [ ] Integrate the Monaco Editor (or CodeMirror 6) to support robust, multi-file editing capabilities within a single browser viewport.
- [ ] Implement server-side language detection utilizing `github-linguist` based on user-provided file extensions to drive the syntax highlighter.
- [ ] Build the specific `gist.github.com/{username}` user profile page, listing all public gists sorted by creation date or star count.
- [ ] Implement "Secret Gists" authorization logic: ensure they are completely stripped from all public GraphQL and REST `List` queries and user profiles.
- [ ] Build the "Revisions" split-diff view, comparing specific commit SHAs using the standard Patience diff algorithm and highlighting line-level changes.
- [ ] Implement the commenting UI specifically for gists, storing comments in a relational `gist_comments` table entirely outside of the Git repository data model.
- [ ] Render `.md`, `.csv`, and `.geojson` files natively within the Gist view utilizing the standard GitHub rich-content renderers.

### Tier 3: Embeds & Social Integrations
- [ ] Implement the dynamic Javascript embed endpoint: `GET /{user}/{hash}.js`.
- [ ] Construct the Javascript response to utilize `document.write` to securely inject a fully styled, syntax-highlighted HTML snippet of the gist code directly into third-party blogs.
- [ ] Host the specific `gist-embed.css` stylesheet via the edge CDN to ensure external embeds match the GitHub Primer design system aesthetics.
- [ ] Create a specific `gist_stars` join table to handle the "Star" interaction, separate from repository stars.
- [ ] Build webhook event triggers specifically for gists to notify external services on update or comment creation.
- [ ] Implement a utility API endpoint to natively convert a complex Gist directly into a full-fledged GitHub Repository.
- [ ] Apply extremely strict rate limiting and CAPTCHA integrations to gist creation APIs to prevent automated spam and SEO manipulation attacks.

### Tier 4: Search, Abuse Prevention & Telemetry
- [ ] Synchronize public gist text content into the Elasticsearch cluster to support global cross-gist code search.
- [ ] Implement an automated ML heuristic or regex-based scanner to detect and automatically hide gists containing stolen credentials or PII.
- [ ] Set up aggressive edge caching (Varnish/CDN) specifically for the `/{user}/{hash}.js` embed endpoint to withstand massive traffic spikes from popular blogs.
- [ ] Configure Prometheus counters to track `git clone` operations specifically targeting the gist storage shard.
- [ ] Implement an internal audit log tracking when a "Secret Gist" URL is accessed from an unrecognized IP address to detect potential link leakage.
