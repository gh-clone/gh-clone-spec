# Plan for gh-clone-web-ide

### Tier 1: VS Code Web Shell & Initialization
- [ ] Embed the official `@vscode/vscode-web` package within an Angular component strictly utilizing an isolated `iframe` to prevent DOM styling conflicts.
- [ ] Configure the VS Code initialization payload to inject the current user's theme preference (Light/Dark/Dimmed) directly into the `workbench.colorTheme` setting.
- [ ] Implement secure cross-origin message passing (`postMessage`) between the Angular host shell and the VS Code `iframe` to handle authentication handshakes.
- [ ] Pass short-lived, narrowly scoped OAuth tokens to the iframe, specifically restricted to the `repo` scope required for virtual file operations.
- [ ] Build the URL state parser in Angular: automatically infer the target repository, branch, and file path from URLs matching `github.dev/{owner}/{repo}/blob/{branch}/{path}`.
- [ ] Implement a loading splash screen in the Angular wrapper that gracefully transitions out only after VS Code fires the `workbench.lifecycle.restored` event.
- [ ] Disable native VS Code telemetry by explicitly overriding the `telemetry.telemetryLevel` configuration to `off` during instantiation.
- [ ] Map the "Return to Repository" Angular router link to a custom VS Code status bar item utilizing the Extension API.

### Tier 2: The Virtual FileSystemProvider (Rust <-> Browser)
- [ ] Implement the VS Code `FileSystemProvider` interface in TypeScript specifically to map internal IDE read/write requests to the Rust backend's Git Tree REST endpoints.
- [ ] Implement the `stat` method: execute an optimized GraphQL query to resolve if a given path is a Blob (file) or a Tree (directory).
- [ ] Implement the `readDirectory` method: parse the backend Git Tree response and return arrays of `[name, FileType]` tuples.
- [ ] Implement the `readFile` method: fetch the Blob SHA from the Rust backend and decode the base64 payload into a `Uint8Array` for VS Code.
- [ ] Support binary file handling natively: ensure images and compiled assets are passed as raw byte buffers without corrupting string encoding.
- [ ] Implement the `writeFile` method: capture modified `Uint8Array` data, convert to base64, and stage the mutation entirely within the browser's local memory.
- [ ] Build an IndexedDB caching layer utilizing Dexie.js to persist the virtual file system's "dirty" state, preventing data loss on accidental browser refresh.
- [ ] Implement cache invalidation logic: if the target branch's HEAD SHA changes on the Rust backend, flush the local IndexedDB cache and prompt the user to reload.
- [ ] Implement the `delete` and `rename` FileSystemProvider methods, staging these tree structural changes within the local IndexedDB virtual tree.

### Tier 3: Search, Workspace & Navigation
- [ ] Implement the `FileSearchProvider` API to route the IDE's "Cmd+P" (Quick Open) file searches directly to the backend's `git ls-tree` or Blackbird Elasticsearch index.
- [ ] Implement the `TextSearchProvider` API to route "Cmd+Shift+F" global regex searches to the Rust code search endpoint, paginating results natively into the IDE sidebar.
- [ ] Implement Workspace Trust configurations: default the web IDE to a "Trusted" state since code execution is impossible within the pure browser sandbox.
- [ ] Build a WebWorker pool to handle large AST parsing for the "Outline" view without blocking the VS Code UI thread.
- [ ] Configure `search.exclude` defaults to automatically hide common build artifacts (`node_modules/`, `target/`, `dist/`) from the virtual search results.
- [ ] Implement deep-linking to specific line numbers: parse `#L12-L24` from the URL and execute the VS Code `revealLine` command upon file load.
- [ ] Support the "Go to Definition" provider for basic languages by integrating lightweight, WebAssembly-compiled Language Servers (e.g., `rust-analyzer-wasm`).

### Tier 4: Git Source Control Management (SCM)
- [ ] Implement the VS Code `SourceControl` API natively, registering a "GitHub Web SCM" provider in the IDE sidebar.
- [ ] Build a virtual working tree diff engine in TypeScript: perform a local Patience diff comparing the IndexedDB modified state against the pristine backend Git Tree.
- [ ] Populate the SCM "Changes" view dynamically, accurately reflecting Modified (M), Added (A), and Deleted (D) file states.
- [ ] Implement the SCM "Discard Changes" action: surgically remove the specific file's dirty state from IndexedDB and reload the pristine blob from the Rust backend.
- [ ] Implement the "Commit" action: aggregate all IndexedDB modifications into a bulk JSON payload.
- [ ] Build the Rust REST endpoint `POST /repos/{owner}/{repo}/git/commits/web` to receive the bulk payload.
- [ ] Implement the backend commit synthesizer: the Rust endpoint must construct a new Git Tree, write the objects, and move the branch ref atomically utilizing `libgit2` or `gitoxide`.
- [ ] Implement concurrent modification detection: reject the web commit on the backend if the base SHA no longer matches the current branch HEAD (HTTP 409 Conflict).
- [ ] Build the Angular modal to handle HTTP 409 Conflicts, offering the user the option to "Create a new branch and PR" to save their work.
- [ ] Implement branch switching directly from the VS Code status bar, fetching the new branch's tree and wiping the local uncommitted cache (with user confirmation).

### Tier 5: Extensions, PR Context & Integrations
- [ ] Host an internal Open VSX compatible extension registry proxy on the Rust backend to serve `.vsix` files securely.
- [ ] Pre-configure the web editor to automatically inject and activate web-compatible extensions (e.g., Prettier, ESLint, GitLens Web).
- [ ] Implement a strict extension sandboxing policy: explicitly disable and hide any extensions requiring Node.js APIs (`fs`, `child_process`) from the web marketplace.
- [ ] Implement the "Review Mode" context: if the IDE is launched from a Pull Request URL, lock the virtual file system strictly to the PR's head branch.
- [ ] Integrate the VS Code `Comments` API: render existing PR review comments directly inside the IDE editor gutters.
- [ ] Support creating new line-level PR review comments from the IDE, dispatching GraphQL mutations back to the Rust backend.
- [ ] Render GitHub Actions status badges directly into the VS Code status bar, polling the `/actions/runs` endpoint for the active branch via RxJS.
- [ ] Build the "Continue on Codespace" integration: extract the IndexedDB dirty state, compress it, and securely transfer the patch to a newly provisioned Firecracker microVM for seamless transition to full compute.

### Tier 6: Advanced Editing Features & LFS
- [ ] Implement Git LFS pointer resolution: if the FileSystemProvider encounters an LFS pointer, automatically fetch the raw binary from the backend S3 proxy.
- [ ] Build the `xterm.js` Web Terminal wrapper component within the Angular UI, hidden by default but available for remote Codespace connections.
- [ ] Configure `vscode-web` snippets: dynamically sync the user's personal VS Code snippets stored in their backend profile into the web editor.
- [ ] Implement WebWorker-based Prettier auto-formatting on save, mapping to the repository's `.prettierrc` automatically loaded into IndexedDB.
- [ ] Expose an Angular-driven "Keyboard Shortcuts" modal that synchronizes keybindings between standard GitHub.com pages and the `github.dev` editor context.
- [ ] Map the "Blame" UI directly into the web editor utilizing a specialized Rust endpoint that computes blame dynamically without a local Git clone.
