# Plan for gh-clone-desktop

### Tier 1: Tauri V2 & Angular Zoneless Foundation
- [ ] Scaffold the native desktop application utilizing Tauri V2 (Rust) replacing the legacy Electron architecture to drastically reduce RAM consumption and bundle size.
- [ ] Bootstrap the frontend strictly using Angular 21 with Zoneless Change Detection (`provideExperimentalZonelessChangeDetection`) for maximum rendering performance.
- [ ] Configure `tauri.conf.json` to enforce strict Content Security Policy (CSP) and disable external OS command execution outside of specific allowed Git binaries.
- [ ] Implement the `tauri-plugin-window-state` to persistently save and restore the exact window dimensions and positions across application restarts.
- [ ] Map the GitHub Primer CSS design system specifically into the desktop viewport, supporting fluid resizing and native window controls via custom frameless windows (`titleBarStyle: "transparent"` on macOS).
- [ ] Implement custom draggable titlebars in the Angular UI that correctly trigger native OS window dragging utilizing `data-tauri-drag-region`.
- [ ] Implement native OS theme synchronization: automatically toggle the Angular dark/light mode based on the host OS preference (Windows 11/macOS/Linux) via Tauri's Window API.
- [ ] Build the native OS menubar/tray integration via Rust, exposing global shortcuts for "Clone Repository", "New Issue", and "Open Settings".
- [ ] Register application-level protocol handlers (`github-mac://` or `github-windows://`) utilizing Tauri's deep link plugin to capture "Open in Desktop" actions directly from the Web UI.
- [ ] Implement secure Inter-Process Communication (IPC) utilizing Tauri Commands, strictly typing the JSON payload requests and responses between Angular and the Rust backend.
- [ ] Build an Angular `TauriIpcService` that wraps asynchronous Rust commands into RxJS Observables or Angular Signals for seamless UI state updates.
- [ ] Integrate native OS contextual menus (Right-Click) utilizing the Tauri Menu API, bypassing HTML-based mock menus for superior native performance.

### Tier 2: Authentication, SAML & OS Keychain
- [ ] Implement the OAuth Web Flow for desktop: spawn a temporary localhost Rust `axum` server, open the system browser, and capture the callback token payload securely.
- [ ] Handle Enterprise SAML SSO redirects seamlessly within the OAuth flow, gracefully capturing IdP assertion responses.
- [ ] Integrate the `keyring` Rust crate to securely store the generated Personal Access Token (PAT) natively in macOS Keychain, Windows Credential Manager, or Linux Secret Service.
- [ ] Implement biometric prompt integration (TouchID, Windows Hello) via Rust plugins to require user presence before accessing highly privileged enterprise repositories.
- [ ] Build the multi-account manager UI in Angular, allowing developers to authenticate with both their Enterprise clone instance and the public GitHub instance simultaneously.
- [ ] Implement an Angular HTTP Interceptor that automatically retrieves the active profile's PAT from the Tauri backend via IPC and injects it into all outbound REST/GraphQL requests.
- [ ] Build a background Rust worker to automatically rotate and refresh short-lived OAuth access tokens using stored refresh tokens.
- [ ] Implement automatic SSH key generation (Ed25519) within Rust, executing `ssh-keygen` bindings, registering the public key via the backend API, and adding the private key to the local `ssh-agent`.
- [ ] Implement a credential helper integration: automatically configure `git config credential.helper` to route CLI Git authentication through the desktop application's stored keychain.

### Tier 3: Local Git Interoperability (`gitoxide` & `libgit2`)
- [ ] Embed the `gitoxide` Rust crate natively within the Tauri backend to perform rapid, thread-safe Git operations (status, branch, log) without spawning heavy external shell processes.
- [ ] Implement a fallback execution engine: dynamically spawn the system's native `git` executable (`std::process::Command`) for complex edge-cases (e.g., interactive rebases) not yet supported by `gitoxide`.
- [ ] Build a highly optimized, asynchronous file-system watcher in Rust utilizing the `notify` crate to actively monitor the `.git/index` and working tree of the active repository.
- [ ] Implement debounce logic on the file-system watcher (e.g., 200ms) to prevent flooding the IPC channel during massive operations like `npm install`.
- [ ] Push file-system change events via Tauri WebSockets/Events to the Angular UI to update the "Changes" tab instantly without manual user refreshing.
- [ ] Implement local repository discovery: recursively scan the user's configured base directory (e.g., `~/Documents/GitHub`) in Rust and populate the UI's repository list, utilizing multi-threading for speed.
- [ ] Build the "Clone Repository" workflow: invoke `gitoxide` fetch operations, parsing the sideband progress packets to stream exact percentage and throughput (MB/s) back to the Angular progress bar.
- [ ] Implement seamless submodule initialization: detect `.gitmodules` post-clone and automatically execute recursive fetching and updating transparently.
- [ ] Handle Git LFS (Large File Storage) pointer resolution natively within the Rust clone pipeline to download binary assets without requiring the external `git-lfs` extension to be pre-installed on the host.

### Tier 4: Working Tree, Commits & Diff Visualization
- [ ] Build the "Changes" Angular view, listing modified, added, deleted, and untracked files with corresponding Primer SVG icons and dynamic counts.
- [ ] Implement the unified and split-pane diff viewer components specifically for local uncommitted changes.
- [ ] Offload diff computation to a WebWorker or the Rust backend, passing only the final structured diff chunks (Hunks) to the Angular UI to prevent blocking the main thread.
- [ ] Integrate a lightweight syntax highlighter (e.g., Prism.js or Monaco) into the diff viewer, mapping file extensions to the correct language grammar.
- [ ] Support granular, line-by-line staging (hunks): allow users to select specific lines in the diff and generate a partial index update directly via `gitoxide`.
- [ ] Implement the "Discard Changes" action, utilizing Rust to safely restore a specific file to its `HEAD` state, requiring a strict confirmation modal for destructive actions.
- [ ] Build the Commit UI: enforce conventional commit validation (if configured via repository settings), support Co-Authored-By trailers, and construct the commit tree via the Rust backend.
- [ ] Support amending the previous commit (`git commit --amend`), pre-filling the text box with the `HEAD` commit message and transparently merging the newly staged index.
- [ ] Implement the "Undo Commit" action: execute a soft reset (`git reset --soft HEAD~1`) to bring the last commit's changes back into the staging area.

### Tier 5: Branches, Syncing & Merge Conflicts
- [ ] Build the Branch Switcher UI: list local branches, remote-tracking branches, and active Pull Request branches utilizing Angular Signals for instant filtering.
- [ ] Implement the "Publish Branch" workflow, pushing the local ref to the remote and automatically configuring the upstream tracking branch.
- [ ] Implement the "Fetch" background worker: periodically poll the remote repository (e.g., every 5 minutes) via `gitoxide` and display an unobtrusive badge if there are incoming commits to pull.
- [ ] Build the "Pull" functionality, allowing users to configure default behaviors (Merge vs Rebase vs Fast-Forward Only) via the application settings schema.
- [ ] Implement the Merge Conflict UI: when a pull or merge operation fails, parse the unmerged index paths and render a dedicated "Resolve Conflicts" screen listing conflicting files.
- [ ] Integrate an inline conflict resolution editor allowing developers to visually choose "Accept Current", "Accept Incoming", or manually edit the `<<<<<<< HEAD` code block.
- [ ] Support the "Stash" workflow: allow users to stash changes with an optional text message, view a list of historical stashes, and pop/apply them directly to the working directory.
- [ ] Implement safe branch deletion: explicitly check if a branch is fully merged into the default branch before allowing deletion, warning the user otherwise.

### Tier 6: Pull Request & Issue Integration
- [ ] Integrate the GraphQL API client within Rust to securely fetch the current repository's open Pull Requests and Issues, piping the serialized JSON to the Angular UI.
- [ ] Build the "Checkout PR" feature: clicking a PR automatically adds the remote (if originating from a fork), fetches the specific `refs/pull/ID/head`, and checks out a detached/tracking local branch.
- [ ] Render the PR Review UI natively in the desktop app: fetch inline review comments from the API and map them directly onto the local diff viewer gutters.
- [ ] Support submitting PR Review comments (Approve, Request Changes) completely from within the Desktop application, queuing mutations in IndexedDB if offline.
- [ ] Implement the "Create Pull Request" wizard: after publishing a branch, auto-open a modal pulling the local commit messages to intelligently pre-fill the PR description.
- [ ] Add "Issue Autocomplete" within the commit message text area: dynamically search open issues via GraphQL and insert `#ID` syntax.
- [ ] Implement status check badges (CI/CD): poll the REST API to display the real-time success/failure icons of the checked-out branch's Actions workflow directly in the desktop header.
- [ ] Stream GitHub Actions build logs directly into a native desktop terminal overlay when a user clicks on a failing status check.

### Tier 7: Advanced Git Workflows & History Re-writing
- [ ] Implement the "History" tab, rendering the chronological commit graph. Utilize Rust to efficiently parse `git log --graph` into a structured JSON array representing nodes and edges.
- [ ] Build infinite scrolling for the commit history using `@angular/cdk/scrolling` to handle massive repositories (e.g., Linux kernel) seamlessly.
- [ ] Build the "Cherry-Pick" feature: allow users to right-click any commit in the History tab and selectively cherry-pick it onto their current active branch.
- [ ] Implement "Squash Commits" interactively: provide a UI to select multiple adjacent commits in the history view and squash them into a single commit without launching a terminal editor.
- [ ] Support "Revert Commit": automatically generate a reverse patch for a historical commit and apply it to the working tree safely.
- [ ] Implement a native "Reflog Viewer": expose the `git reflog` output in a specialized recovery UI to allow users to rescue accidentally deleted branches or hard resets.
- [ ] Build the interactive rebase drag-and-drop UI: allow users to reorder commits, mark them for rewording, or drop them entirely utilizing `@angular/cdk/drag-drop`.
- [ ] Implement Git Worktree management: provide UI options to checkout branches into separate, linked working directories to allow simultaneous multi-branch development.

### Tier 8: OS Integrations & Toolchain
- [ ] Build OS native file explorer integrations: "Reveal in Finder" (macOS) / "Show in Explorer" (Windows) mapped directly via Tauri shell commands.
- [ ] Integrate with native external editors (VS Code, IntelliJ, Neovim), parsing the OS `PATH` environment variable in Rust to automatically configure the "Open in Editor" shortcut button.
- [ ] Implement automatic GPG/SSH commit signing detection: parse the user's `~/.gitconfig` and pipe the commit payload to the local `gpg` or `ssh-keygen` binary for signature generation automatically.
- [ ] Expose an integrated terminal view within the desktop app utilizing `xterm.js`, automatically initialized to the current repository's working directory.
- [ ] Provide proxy configuration support: allow users to explicitly set HTTP/HTTPS proxies, reading system-level PAC files or Windows Registry proxy settings via Rust.

### Tier 9: Packaging, Auto-Updates & Telemetry
- [ ] Configure `tauri-builder` to generate highly optimized, distinct binaries for `x86_64-pc-windows-msvc`, `aarch64-apple-darwin` (Apple Silicon), `x86_64-apple-darwin`, and `x86_64-unknown-linux-gnu`.
- [ ] Implement the Tauri Auto-Updater (`tauri://update`): periodically check an internal API endpoint for new semver tags, download the binary delta patch, and prompt the user to restart.
- [ ] Generate MSIX / AppX installers specifically for distribution through the Microsoft Store.
- [ ] Generate fully signed `.dmg` and `.pkg` installers utilizing Apple's Notary Service to bypass macOS Gatekeeper warnings entirely.
- [ ] Generate `.AppImage`, `.deb`, and `.rpm` packages for comprehensive Linux distribution support.
- [ ] Integrate crash reporting using `sentry-rust` in the backend and `@sentry/angular-ivy` in the frontend to capture native memory panics and unhandled JS exceptions securely.
- [ ] Build a strictly opt-in, anonymized telemetry pipeline to track specific feature usage (e.g., "Conflict UI opened", "PR checked out") to guide UX decisions.
- [ ] Automate the entire CI/CD release pipeline via GitHub Actions to cross-compile matrices and attach the signed native binaries directly to a specific repository Release tag.
