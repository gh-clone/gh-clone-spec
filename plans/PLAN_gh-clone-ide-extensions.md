# Plan for gh-clone-ide-extensions

### Tier 1: Extension Architecture, Auth & Security
- [ ] Initialize the VS Code Extension (`vscode-replica`) utilizing strict TypeScript, structured with `esbuild` for ultra-fast, single-file bundling.
- [ ] Implement the `AuthenticationProvider` API in VS Code to handle OAuth 2.0 securely with the Rust backend.
- [ ] Support the OAuth Device Authorization Flow (`/login/device`) natively for headless environments or Remote SSH/WSL connections.
- [ ] Integrate with VS Code's `SecretStorage` API to securely persist OAuth access and refresh tokens natively in the OS keychain (macOS Keychain, Windows Credential Manager, Linux Secret Service).
- [ ] Build the Enterprise Host configuration wizard, prompting users on first load to specify if they are targeting the public instance or a private `replica.enterprise.com` URL.
- [ ] Implement proxy support: explicitly read VS Code's `http.proxy` and `http.proxyStrictSSL` settings and inject them into the underlying `node-fetch` or `undici` HTTP clients.
- [ ] Handle enterprise custom Certificate Authorities (CAs): parse OS-level trust stores so the extension does not reject self-signed certificates on internal enterprise clones.
- [ ] Implement robust error handling interceptors: automatically catch `401 Unauthorized` responses and silently attempt to exchange the refresh token before prompting the user to re-authenticate.
- [ ] Expose an administrative "Sign Out" command that actively sends a `DELETE /authorizations/{token}` request to the Rust backend before purging the local SecretStorage.
- [ ] Implement deep linking (`vscode://`) utilizing `vscode.env.openExternal` to intercept "Open in VS Code" buttons clicked directly from the Angular web frontend.
- [ ] Support Multi-Account Management: allow users to authenticate against both the public instance and their enterprise instance simultaneously, intelligently routing requests based on the active Git remote.

### Tier 2: API Client, Caching & Data Synchronization
- [ ] Build the API client wrapper in TypeScript, strictly generated from the Rust backend's OpenAPI/GraphQL schema definitions via code generation.
- [ ] Implement a local SQLite caching layer (via `better-sqlite3` compiled with `node-gyp`) to persist issue lists, PR metadata, and comment threads for offline viewing.
- [ ] Implement periodic background polling (with exponential backoff and jitter) to fetch notification counts, PR CI statuses, and review requests without exceeding API rate limits.
- [ ] Subscribe to the Rust backend's WebSocket/SSE endpoint when the extension is active to receive push-based invalidation events (e.g., "PR #123 merged").
- [ ] Implement ETags and `If-None-Match` header injection for all `GET` requests to heavily reduce bandwidth and latency during synchronization.
- [ ] Debounce high-frequency API mutations: if a user types multiple review comments rapidly, aggregate them locally and perform a single GraphQL batch mutation upon clicking "Submit Review".

### Tier 3: Pull Request Tree View, Filtering & Discovery
- [ ] Register the custom `TreeDataProvider` to render the primary "Pull Requests" sidebar container in VS Code.
- [ ] Implement categorical PR queries by default: "Local Pull Request Branches", "Created by Me", "Waiting for My Review", and "All Open".
- [ ] Map fetched PR nodes to custom `TreeItem` objects, utilizing dynamically colored Primer SVG icons for Open (Green), Draft (Gray), Merged (Purple), and Closed (Red).
- [ ] Implement cursor-based GraphQL pagination within the Tree View, adding a "Load More..." node that dynamically fetches the next page of results.
- [ ] Support custom "Search Queries" configured in `settings.json` (e.g., `"replica.queries": [{"label": "Bugs", "query": "label:bug is:open"}]`).
- [ ] Implement a rich hover provider (`vscode.Hover`) in the tree view that renders the PR description markdown, author avatar, and the exact CI/CD status directly in a native tooltip.
- [ ] Add context menu (`menus: view/item/context`) actions to PR nodes: "Checkout PR", "Copy PR URL", "Copy PR Number", and "Open on Web".
- [ ] Implement a multi-select action in the tree view: allow developers to shift-click multiple PRs and assign a label or milestone in bulk.
- [ ] Build a "Refresh" command attached to the view title bar that instantly flushes the local SQLite cache and executes a hard network fetch.

### Tier 4: Virtual File Systems & High-Fidelity Diff Review
- [ ] Register a custom `FileSystemProvider` (`replica-pr-fs`) to read virtual file contents directly from the Rust backend's Git Tree without requiring a local `git fetch`.
- [ ] Implement the "View Changes" flow: clicking a PR dynamically constructs file URIs for the `base` and `head` SHAs and opens VS Code's native `vscode.diff` multi-file diff view.
- [ ] Implement image diffing support: if the virtual file is an image (`png`, `jpg`), securely pass the binary buffer to VS Code's native image preview viewer instead of treating it as text.
- [ ] Handle massive PRs gracefully: if a diff exceeds 10,000 lines or 10MB, display a warning and provide a button to "View Full Diff on Web" to prevent VS Code memory crashes.
- [ ] Implement conflict marker detection: visually parse `<<<<<<< HEAD` blocks and inject CodeLens actions ("Accept Current", "Accept Incoming") to resolve them locally before committing.
- [ ] Implement file tree collapsing inside the PR view: group changed files by directory and overlay the count of additions/deletions on the parent folder node.
- [ ] Support the "Mark as Viewed" workflow: track viewed files locally and sync the state back to the Rust backend to collapse them automatically in future review sessions.

### Tier 5: Inline Code Review & Conversation Management
- [ ] Implement the `CommentingProvider` API to render PR review comments natively within the IDE's editor gutters alongside the virtual diff.
- [ ] Map the Rust backend's line-level comment coordinate system precisely to VS Code's 0-indexed `Range` and `Position` objects.
- [ ] Provide Code Actions to add, edit, reply to, and delete single comments directly from the editor margin.
- [ ] Implement the "Start Review" and "Submit Review" batching system, rendering a temporary badge on the Activity Bar indicating the number of pending comments.
- [ ] Render standard GitHub markdown in comments (including tables, task lists, and syntax-highlighted code blocks) utilizing VS Code's `MarkdownString`.
- [ ] Implement "Suggest Changes": provide a specific button in the comment box that automatically wraps the highlighted text in a ```suggestion markdown block.
- [ ] Add a CodeLens on suggestion blocks allowing the PR author to click "Apply Suggestion" and instantly mutate their local working tree.
- [ ] Support resolving conversations: add a "Resolve" button to the comment thread UI that mutates the backend state and collapses the thread in the editor.
- [ ] Implement reaction support: allow users to click a smiley face icon on a comment and select an emoji (👍, ❤️, 🚀) via the backend GraphQL mutation.

### Tier 6: PR Checkout, SCM & Branch Automation
- [ ] Implement the "Checkout PR" command: automate the parsing of the fork remote, securely fetching the ref (`refs/pull/ID/head`), and executing `git checkout`.
- [ ] Integrate with VS Code's built-in Source Control Management (SCM) API to dynamically decorate the active branch name with its linked PR number and status icon.
- [ ] Implement the "Create Pull Request" workflow natively: parse the current branch commits and upstream target, and open an integrated Angular 21 (Zoneless) Webview to draft the title, body, reviewers, and labels.
- [ ] Auto-generate the PR description in the Webview by scraping the local commit messages and executing a regex pass against configured PR templates (`.github/PULL_REQUEST_TEMPLATE.md`).
- [ ] Implement merge capabilities within the extension UI: support "Merge Commit", "Squash and Merge", and "Rebase and Merge", executing the corresponding REST mutations against the Rust backend.
- [ ] Add post-merge automation: prompt the user to automatically delete the remote branch and local tracking branch upon a successful PR merge.
- [ ] Implement "Draft PR" toggles inside the Create PR Webview to prevent triggering full CI pipelines prematurely.

### Tier 7: Issues, Hover Providers & Editor Autocompletion
- [ ] Implement a repository-wide `HoverProvider`: whenever the user hovers over a string matching `#\d+` (e.g., `#123`) in a markdown file or commit message, fetch and display the Issue details in a tooltip.
- [ ] Build the "Issues" sidebar `TreeDataProvider`, listing assigned issues, mentioned issues, and a "Create Issue" action button.
- [ ] Implement a `CompletionItemProvider` (Autocomplete) triggered by typing `#` in the SCM commit input box, returning a list of open issues and their titles.
- [ ] Implement an `@` mention autocomplete provider, querying the Rust backend for repository collaborators and injecting their usernames into issue bodies or comments.
- [ ] Build the "Start Working on Issue" command: right-click an issue in the sidebar, which automatically creates a new local branch named `issue-123-slugified-title` and links it.
- [ ] Render Mermaid diagrams automatically within issue hover tooltips if the issue body contains ` ```mermaid ` blocks.
- [ ] Parse and execute task lists (`- [ ]`) natively in the issue sidebar, allowing developers to check/uncheck tasks and sync the markdown modification to the backend.

### Tier 8: Actions, CI/CD Logs & Artifact Management
- [ ] Implement the "Actions" Tree View container, listing all active workflow runs and historical workflow executions for the current repository.
- [ ] Fetch and display the hierarchical workflow jobs, distinct matrix executions, and individual steps as nested tree items.
- [ ] Implement the "View Logs" command: fetch the raw chunked log stream from the Rust backend and pipe it directly into a custom, read-only VS Code `OutputChannel`.
- [ ] Implement ANSI color code parsing within the OutputChannel to ensure build logs render with exact terminal syntax highlighting and error formatting.
- [ ] Build the "Trigger Workflow" (`workflow_dispatch`) command, opening an Angular Webview or QuickPick menu to accept custom workflow inputs before dispatching the POST request.
- [ ] Implement the "Re-run Failed Jobs" command, targeting specific failed matrix jobs via the backend API instead of restarting the entire workflow.
- [ ] Add the "Download Artifacts" action to completed workflow nodes, downloading the `.zip` archive via the Rust S3 proxy and extracting it to the local workspace.

### Tier 9: JetBrains / IntelliJ Plugin Platform Parity
- [ ] Scaffold the JetBrains plugin utilizing Kotlin, Gradle, and the IntelliJ Platform Plugin SDK.
- [ ] Implement the standard OAuth flow utilizing JetBrains' `BrowserUtil` and secure credential store (`PasswordSafe`).
- [ ] Build the "Pull Requests" tool window using standard Swing / Kotlin UI DSL components, replicating the exact VS Code sidebar functionality.
- [ ] Map JetBrains' `DiffRequest` and `Document` APIs to securely render PR diffs fetched from the Rust backend virtual file system.
- [ ] Implement inline code review comments utilizing IntelliJ's `EditorCustomElementRenderer` and `InlayModel` for precise line-level annotations.
- [ ] Build the local SQLite caching layer within the JetBrains plugin structure utilizing JDBC bindings.
- [ ] Mirror the branch checkout automation, mapping JetBrains' internal Git4Idea API to manage the local remotes and tracking branches safely.
- [ ] Implement the "View Actions Logs" feature inside the JetBrains `Run/Debug Tool Window`, translating ANSI codes into standard `ConsoleView` text attributes.
- [ ] Ensure any complex, embedded Webviews (like the Create PR flow) utilized in the JetBrains plugin are driven by the exact same compiled Angular 21 (Zoneless) application logic used in the VS Code Webview to ensure 100% UI consistency.
- [ ] Register `GotoDeclarationHandler` to allow developers to `Ctrl+Click` an issue number (`#123`) in a commit message and jump directly to the Issue tool window.