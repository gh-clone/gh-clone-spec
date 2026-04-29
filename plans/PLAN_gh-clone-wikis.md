# Plan for gh-clone-wikis

### Tier 1: Gollum Git Backend & Storage
- [ ] Intercept the `repository_created` backend event to optionally initialize a parallel `{repo}.wiki.git` bare Git repository automatically.
- [ ] Implement a Gollum-compatible backend parser in Rust to read the wiki repository tree and extract page blobs directly into memory.
- [ ] Execute `git log --pretty=format` dynamically to generate the Wiki revision history UI without requiring database metadata.
- [ ] Expose the wiki repository via the standard Git Smart HTTP endpoints, strictly ensuring users can execute `git clone https://domain/user/repo.wiki.git` locally.
- [ ] Map Gollum's standard directory structure, ensuring top-level markdown files act as primary web routes (e.g., `Home.md` -> `/wiki/Home`).
- [ ] Implement automated Git commit wrapping: all web UI edits must automatically construct a `git commit` structurally attributed to the specific editing user's email.
- [ ] Implement a 3-way merge algorithm (`libgit2`) explicitly for Wiki web edits to support concurrent web editing and safely merge non-conflicting changes.
- [ ] Handle merge conflicts elegantly, generating UI alerts and preventing the save if a user tries to push changes over a conflicting updated `HEAD`.

### Tier 2: Rendering Engine & UI
- [ ] Integrate a Markdown and Asciidoc rendering pipeline specifically configured for standard Wiki-style internal links (`[[Page Name]]` or `[[Link Text|Page Name]]`).
- [ ] Automatically generate the dynamic Wiki Sidebar navigation by parsing a `_Sidebar.md` file if it exists in the repository root.
- [ ] Generate the Custom Header and Footer components by parsing `_Header.md` and `_Footer.md`.
- [ ] Build the Wiki Editor UI utilizing a split-pane layout containing a raw Markdown editor on the left and a live-updating HTML preview on the right.
- [ ] Implement an exact page-comparison view (diff viewer) comparing the AST/line changes between two specific wiki page commits.
- [ ] Support rendering MathJax/KaTeX math blocks (`$$...$$`) natively and securely inside the Wiki markdown renderer.
- [ ] Support Mermaid.js state diagrams parsing natively within Wiki codeblocks (` ```mermaid `) executing the rendering in a secure frontend iframe.

### Tier 3: Security, Search & Access Control
- [ ] Map standard repository Read/Write permissions directly to the Wiki UI (Read permission -> View Wiki, Write permission -> Edit Wiki).
- [ ] Implement a repository-level setting to restrict Wiki editing strictly to repository collaborators (explicitly disabling public wiki edits on public repos).
- [ ] Render extremely strict CSP (Content Security Policy) headers on Wiki endpoints to prevent XSS attacks via maliciously crafted wiki markdown.
- [ ] Apply a rigorous HTML sanitizer (DOMPurify equivalent configured for GitHub) to strip dangerous `iframe`, `script`, or `object` tags specifically from Wiki outputs.
- [ ] Enable Elasticsearch/ZincSearch indexing specifically for Wiki contents to allow precise phrase searching within the repository "Wiki" tab context.
- [ ] Ensure relative image links (`[[images/logo.png]]`) resolve correctly to the binary blobs stored inside the `.wiki.git` repo.
- [ ] Track Wiki page view counts and unique visitors via Redis HyperLogLog, exporting the data to the repository Insights dashboard.

### Tier 4: Asset Management & Telemetry
- [ ] Implement native Git LFS (Large File Storage) support specifically within `.wiki.git` repositories for managing large embedded videos and PDFs.
- [ ] Configure `moka` or Redis caching for rendered Wiki HTML outputs to prevent re-parsing complex Markdown/Mermaid ASTs on every page load.
- [ ] Instrument the Gollum-compatible Markdown renderer with performance tracing to identify and timeout regex denial-of-service (ReDoS) attacks.
- [ ] Implement a strict rate limit on Wiki `POST` edits to prevent automated vandalism by compromised collaborator accounts.
- [ ] Output detailed structured logs for every Wiki page edit, feeding into the main repository's aggregated activity stream.
