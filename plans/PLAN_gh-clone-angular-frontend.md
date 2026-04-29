# Plan for gh-clone-angular-frontend

### Phase 1: Foundational UI & Core Workflows

**Phase 1.1: Global Shell & Repository Navigation**
- [ ] **App Shell Component:** Build the global navigation header with search input, notifications bell, and user avatar dropdown using strictly Angular primitives.
- [ ] **Repository Layout Wrapper:** Implement the sticky repository header containing the repository name, visibility badge, and metrics (Watch, Fork, Star).
- [ ] **Navigation Tabs:** Build the router-aware repository navigation tabs (Code, Issues, Pull Requests, Actions, Projects, Security, Insights, Settings) with exact active-state CSS matching.
- [ ] **Lazy-Loaded Route Modules:** Structure the Angular router to lazily load each major repository tab to strictly minimize the initial Javascript bundle size.

**Phase 1.2: Source Code Viewer & Tree Navigation**
- [ ] **File Explorer Tree:** Implement the recursive directory traversal component, rendering folder icons and file type specific icons accurately.
- [ ] **File Blob Viewer:** Build the high-fidelity code viewer with line numbers, syntax highlighting (via Prism or Monaco), and a robust copy-to-clipboard button.
- [ ] **Repository Meta Banner:** Render the latest commit message, author avatar, and relative timestamp accurately above the file tree.
- [ ] **Branch/Tag Switcher:** Implement the fuzzy-search dropdown to seamlessly swap the current route context between branches, tags, and specific commits.
- [ ] **Raw File Rendering:** Add support for viewing raw files, rendering images directly, and displaying tabular data (CSV/TSV) in a styled table layout.

**Phase 1.3: Issue Tracker Foundation**
- [ ] **Issue Listing View:** Build the paginated issue list displaying issue title, number, author, creation date, label tags, and comment count.
- [ ] **Issue Filtering Engine:** Implement the complex search bar parsing exact GitHub syntax (e.g., `is:open is:issue author:@me label:bug`) mapped to backend query params.
- [ ] **Issue Detail View:** Render the full issue thread layout with the original description on top, chronologically ordered comments, and the markdown editor at the bottom.
- [ ] **Issue Sidebar:** Build the interactive right sidebar for managing Assignees, Labels, Projects, Milestones, and Development branch links.
- [ ] **New Issue Form:** Implement the creation wizard supporting issue templates, title validation, and the rich Markdown editor for the body.

**Phase 1.4: Basic Pull Request Scaffolding**
- [ ] **PR Listing View:** Mirror the issue listing view but display the branch merging context (e.g., `samuel:feature-branch` into `main`) and CI status checks.
- [ ] **PR Conversation Tab:** Render the PR discussion timeline, integrating code-level review comments seamlessly into the chronological feed.
- [ ] **PR Commits Tab:** List the sequence of commits attached to the pull request with clickable SHAs to view isolated commit diffs.
- [ ] **PR Diff Viewer:** Build the foundational split-pane and unified diff viewers comparing `base` and `head` utilizing precise CSS line-number synchronization.
- [ ] **Merge Box Component:** Implement the terminal PR merge block, displaying branch protection statuses and the "Squash and Merge" action dropdown.

### 2.2 Angular 21: Zoneless SSR & Advanced Hydration

**Phase 2.2.1: Zoneless Core, SSR & Hydration**
- [ ] **Zoneless Provider Configuration:** Bootstrap the application strictly via `bootstrapApplication` using `provideExperimentalZonelessChangeDetection()` to completely eradicate `zone.js`.
- [ ] **Event Replay Buffering Strategy:** Enable `withEventReplay()` in hydration providers to strictly buffer user clicks and keystrokes occurring before Angular hydration completes.
- [ ] **TransferState Payload Injection:** Intercept the initial Node.js SSR request, serialize the Rust API response, and inject it securely into `<script type="application/json" id="transfer-state">`.
- [ ] **HttpTransferCache Interception:** Write an `HttpInterceptorFn` that synchronously resolves outbound HTTP requests from the TransferState cache to achieve Zero-CLS initial hydration.
- [ ] **Lazy Component Chunking:** Utilize `loadComponent` strictly on all route definitions to enforce granular, route-level JavaScript code splitting.
- [ ] **Preload Strategy Configuration:** Implement a custom `PreloadingStrategy` triggering chunk downloads specifically when users hover over primary navigation links for 200ms.
- [ ] **SSR Server Request URL Mapping:** Map incoming Express/Node.js request URLs strictly to Angular's `ServerPlatformLocation` to ensure accurate router initialization on the server.
- [ ] **Hydration Non-Destructive Bailout:** Implement targeted `ngSkipHydration` attributes strictly on third-party ad/tracking elements to prevent hydration mismatch panics.

**Phase 2.2.2: Web Worker, Monaco & Tree-Sitter WASM**
- [ ] **Monaco Worker Isolation:** Instantiate the Monaco editor exclusively inside a dedicated Angular Web Worker (`new Worker()`) to strictly isolate syntax highlighting from the main UI thread.
- [ ] **ArrayBuffer Zero-Copy Transfer:** Implement `postMessage(data, [data.buffer])` zero-copy transfer semantics to move massive 10MB+ file strings to the Monaco worker instantaneously.
- [ ] **Tree-Sitter WASM Instantiation:** Download and initialize the `tree-sitter.wasm` core binary exclusively within the Web Worker context.
- [ ] **IndexedDB Grammar Caching Layer:** Download specific Tree-Sitter grammars (e.g., `tree-sitter-rust.wasm`) and cache them persistently utilizing the native IndexedDB API.
- [ ] **Viewport Intersection Loading:** Delay the instantiation of the Monaco Worker until the DOM wrapper element intersects the viewport via `IntersectionObserver`.
- [ ] **Virtual Scrolling CdkViewport:** Implement `@angular/cdk/scrolling` with a custom `VirtualScrollStrategy` for rendering massive 1000+ line PR diffs at 60 FPS.
- [ ] **Dynamic Row Height Computation:** Compute dynamic virtual scrolling row heights accurately handling line wrapping without triggering layout jitter or scrollbar bouncing.
- [ ] **Diff Gutter Synchronization:** Synchronize scroll positions exactly between left (base) and right (head) split-diff viewports utilizing precise RxJS stream timing.

**Phase 2.2.3: Primer Design System CSS Variables**
- [ ] **Primitive Variable Extraction:** Map Primer primitive colors (`canvas-default`, `fg-default`, `border-muted`) strictly to CSS custom properties (`--color-canvas-default`).
- [ ] **Spacing Scale Enforcement:** Map Primer spacing primitives (4px, 8px, 16px, 24px) strictly to `--primer-space-1`, `--primer-space-2` CSS variables.
- [ ] **Typography Scale Enforcement:** Map Primer typography scales strictly into `--primer-text-body-size`, `--primer-text-title-size`, and `--primer-font-stack`.
- [ ] **Color Mode Native Synchronization:** Synchronize `[data-color-mode="auto"]` with OS-level `window.matchMedia('(prefers-color-scheme: dark)')` event listeners.
- [ ] **Transition Suppression Macro:** Implement a global CSS macro disabling all DOM transitions for exactly 150ms during theme toggling to prevent a "flash of styling".
- [ ] **Focus Ring Strictness:** Enforce global Primer focus rings precisely using `box-shadow: 0 0 0 3px rgba(9,105,218,0.3)` across all interactive inputs and buttons.
- [ ] **Dimmed Theme Variant:** Implement the explicit "Dimmed" dark mode variant (`[data-color-mode="dark"][data-dark-theme="dark_dimmed"]`) exactly matching GitHub's subdued contrast.

**Phase 2.2.4: Component Library Parity (Layout & Navigation)**
- [ ] **Global Header Component:** Replicate the global header layout utilizing flexbox space-between, exactly matching the logo, search bar, and avatar dropdown positioning.
- [ ] **Repository Header Tabs:** Implement the repository navigation tab bar, calculating precise active-state bottom border `border-bottom: 2px solid var(--color-primer-border-active)`.
- [ ] **Breadcrumb Component:** Implement the recursive Breadcrumb component with exact slash `/` SVG separators and correct overflow ellipsis behavior.
- [ ] **ActionMenu Dialog Wrapper:** Replicate the Primer ActionMenu utilizing the native HTML `<dialog>` element combined with `position-fallback` polyfills for tooltip anchoring.
- [ ] **SegmentedControl Animations:** Implement the Primer SegmentedControl mimicking the background slider translation animation using `transform: translateX()`.
- [ ] **AvatarStack Layout Logic:** Build the AvatarStack component calculating exact negative margin overlaps (`margin-left: -8px`) with specific z-index stacking orders.
- [ ] **Sticky File Header Intersection:** Pin PR/File file headers exactly to the viewport top `position: sticky; top: 0; z-index: 100`, hiding them precisely when scrolling down past the container.

**Phase 2.2.5: NgRx SignalStore & Optimistic Mutations**
- [ ] **Entity State Instantiation:** Configure `signalStore` with custom entity methods tracking the complex normalization of nested User, Repo, Issue, and PR entities.
- [ ] **Optimistic Star Mutation:** Immediately patch the local SignalStore repository entity's `starred` state to `true` and increment `stargazers_count` prior to network dispatch.
- [ ] **Optimistic Follow Mutation:** Immediately patch the local User entity `is_following` state to `true` while the `rxMethod` side-effect executes the API call.
- [ ] **Rollback State Recovery:** Cache the previous SignalStore snapshot within the `rxMethod` pipeline, rolling the UI state back automatically if the underlying API HTTP request fails.
- [ ] **RxJS PR Merge Orchestration:** Emulate GitHub's exact "Merge PR" UI flow (Draft -> Ready -> Merging -> Merged) by orchestrating specific RxJS `delay()` and `switchMap` streams.
- [ ] **Debounced Hovercard Streams:** Utilize RxJS `debounceTime(400)` before dispatching API requests to fetch user/repo data specifically for Hovercard tooltips.
- [ ] **Hovercard Cancellation:** Map RxJS `takeUntil(mouseLeave$)` to instantly cancel pending Hovercard HTTP requests if the user's cursor aggressively sweeps across the screen.

**Phase 2.2.6: IndexedDB, Service Worker & PWA Offline Sync**
- [ ] **Dexie.js Inbox Schema:** Instantiate a `Dexie` IndexedDB schema specifically modeled to cache `/notifications` inbox threads and read states locally.
- [ ] **Service Worker Registration:** Register `@angular/service-worker` in the bootstrap pipeline, conditionally bypassed in development environments.
- [ ] **Avatar Cache-First Strategy:** Configure a strict `CacheFirst` Workbox strategy tailored for `avatars.githubusercontent.com` URLs with a 30-day expiration policy.
- [ ] **Primer CSS Stale-While-Revalidate:** Configure a `StaleWhileRevalidate` Workbox strategy targeting static Primer CSS and Web Font payloads.
- [ ] **API Network-First Strategy:** Configure a `NetworkFirst` strategy for `api.github.com` requests with a 3-second network timeout fallback to local IndexedDB caches.
- [ ] **Background Sync Replay Queue:** Bind the Workbox Background Sync API to an IndexedDB queue, recording offline mutations (stars, watches, reads) for silent replay upon network restoration.
- [ ] **Install Prompt Micro-interaction:** Intercept the native `beforeinstallprompt` event and render a GitHub-styled custom PWA installation banner strictly once per user session.

**Phase 2.2.7: Server-Sent Events & Real-Time Sync**
- [ ] **SSE Fetch Implementation:** Establish persistent EventSource connections specifically utilizing `@microsoft/fetch-event-source` to allow custom headers (Authorization).
- [ ] **Live Status Check Streaming:** Bind the SSE stream to the PR component, dynamically appending live CI/CD pipeline check updates into the DOM without triggering full CD runs.
- [ ] **Live Issue Comment Streaming:** Bind the SSE stream to the Issue timeline, instantly appending new comments to the bottom of the thread with a specific yellow highlight fade-out animation.
- [ ] **Exponential Backoff Reconnect:** Implement a robust exponential backoff algorithm with 20% jitter to manage dropped SSE connections without causing server thundering herds.
- [ ] **BroadcastChannel Tab Sync:** Instantiate the native `BroadcastChannel` API exactly to synchronize the local `/notifications` unread counter instantly across multiple open browser tabs.
- [ ] **Cross-Tab LocalStorage Event Sync:** Utilize `window.addEventListener('storage')` to immediately mirror optimistic UI updates (e.g., Star toggles) across concurrent browser windows.

**Phase 2.2.8: GFM, Regex Parsers & Markdown Rendering**
- [ ] **marked.js Instantiation:** Configure `marked.js` strictly enabling `gfm: true` and `breaks: true` matching GitHub Flavored Markdown specifications exactly.
- [ ] **@Mention Regex Plugin:** Implement a custom Markdown parsing tokenization step strictly identifying `@[a-zA-Z0-9-]+` strings and hyperlinking them to `/{username}`.
- [ ] **#Issue Regex Plugin:** Implement a custom Markdown parsing tokenization step identifying `#{number}` and hyperlinking to `/{owner}/{repo}/issues/{number}`.
- [ ] **Commit SHA Regex Plugin:** Implement a custom Markdown parsing token identifying 7-40 character hexadecimal strings, verifying them, and hyperlinking to the commit view.
- [ ] **Syntax Highlighting Injection:** Override the default `marked` code block renderer to apply specific HTML classes that trigger asynchronous Monaco highlighting.
- [ ] **DomSanitizer Strictness Enforcement:** Enforce Angular's native `DomSanitizer.bypassSecurityTrustHtml` strictly *after* running the parsed markdown AST through a rigid DOMPurify white-list to eliminate XSS.
- [ ] **RelativeTime Auto-Updating Component:** Implement `<relative-time>` component observing timestamps via `requestAnimationFrame` and recalculating "X minutes ago" precisely on the minute boundary without global Change Detection.

**Phase 2.2.9: Repository, PR & File Explorer UI Strictness**
- [ ] **Code View Header Banner:** Replicate the latest commit banner above the file explorer exactly: Avatar left, author bolded, commit message truncated, SHA hyperlinked, and time-ago right-aligned.
- [ ] **File Explorer Icon Mapping:** Implement precise mappings from file extensions (`.rs`, `.ts`, `.md`) and names (`Dockerfile`, `Makefile`) to their corresponding Primer SVG icons.
- [ ] **PR Timeline Layout Structure:** Build the PR timeline strictly mirroring GitHub: Main column for conversation left, sticky sidebar for Reviewers/Assignees/Labels right.
- [ ] **PR Diff Unified/Split Toggle:** Implement the logic to seamlessly swap between Unified and Split diff views instantly without re-requesting data from the server.
- [ ] **Clipboard Micro-interaction (Clone URL):** Replicate the Clone URL clipboard click event exactly: execute `navigator.clipboard.writeText`, transition the icon to a green SVG checkmark.
- [ ] **Clipboard Timeout Reset Constraint:** Enforce a strict `setTimeout` resetting the clipboard checkmark back to the default copy icon after precisely 2000ms.
- [ ] **Turbo-Style Layout Swaps:** Implement prefetching on `<a routerLink>` utilizing mouseover to download the next chunk, achieving instantaneous layout swaps perfectly mirroring GitHub's Turbo framework.

**Phase 2.2.10: Accessibility (A11y) & Keyboard Navigation**
- [ ] **Cmd+K Global Command Palette:** Register a global `@HostListener('window:keydown.meta.k')` (and `ctrl.k`) specifically to invoke the fuzzy-finding Command Palette overlay.
- [ ] **WASM SQLite Command Index:** Compile SQLite FTS5 to WebAssembly strictly to power the typo-tolerant local search engine for the Command Palette executing in sub-millisecond time.
- [ ] **Command Palette Focus Traps:** Enforce strict focus-trap logic inside the Command Palette, preventing the `Tab` key from escaping the modal boundary.
- [ ] **ARIA Live Region Announcements:** Inject `aria-live="polite"` strictly into the Command Palette search results counter to guarantee screen readers announce result count changes.
- [ ] **ActionList Arrow Key Navigation:** Implement custom Angular Directives specifically managing `ArrowUp`, `ArrowDown`, `Home`, and `End` focus traversal inside Primer ActionLists.
- [ ] **Escape Key Modal Dismissal:** Enforce a global `Escape` key listener instantly dismissing any open dropdown, ActionMenu, popover, or Command Palette dialog.
- [ ] **Focus Restoration Constraint:** Guarantee browser focus is accurately restored strictly to the original invoking button element when a modal or dialog is dismissed.
- [ ] **Current Page ARIA Sync:** Dynamically map Angular Router active states specifically to the `aria-current="page"` attribute on navigation tab links.## 3. THE DISTRIBUTED GIT DATA PLANE (PROTOCOL LEVEL)

### 6.1 Playwright E2E, UI Fidelity, i18n & a11y
- [ ] **Core POM Architecture & Configuration**
    - [ ] Configure `playwright.config.ts` with 3 retries on CI and 0 locally.
    - [ ] Define explicit browser testing matrix: Chromium, Firefox, WebKit, and Mobile Safari emulation.
    - [ ] Implement `BasePage.ts` with custom `waitForNetworkIdle` and `waitForDOMStable` helpers.
    - [ ] Create `test.extend` fixtures for authenticated state injection to bypass UI login for non-auth tests.
    - [ ] Implement specific API mock fixtures for simulating edge-case server errors (500, 502) in the UI.
    - [ ] Configure video recording and trace viewer retention for failed tests on CI.
- [ ] **Internationalization (i18n) & Localization Implementation**
    - [ ] Integrate an i18n framework (e.g., `@angular/localize` or `@ngx-translate/core`) and configure dynamic loading of translation namespaces.
    - [ ] Extract all hardcoded UI strings into JSON language files (e.g., `en-US.json`, `es-ES.json`, `ja-JP.json`).
    - [ ] Implement user locale detection based on the `Accept-Language` HTTP header, with user settings override.
    - [ ] Implement ICU Message Format for complex pluralization (e.g., "0 commits", "1 commit", "2 commits").
    - [ ] Replace hardcoded dates with `Intl.DateTimeFormat` and `Intl.RelativeTimeFormat` to support localized relative times (e.g., "2 hours ago" vs "hace 2 horas").
    - [ ] Implement Right-to-Left (RTL) support using CSS logical properties (`margin-inline-start`, `padding-block`) and `dir="rtl"` attribute.
- [ ] **i18n Playwright Testing**
    - [ ] Write tests to switch user language settings via the UI and assert translation application without page reloads.
    - [ ] Test date/time rendering across at least 3 distinct timezones and locales using Playwright's `timezoneId` and `locale` context options.
    - [ ] Verify UI layout integrity and overflow handling in RTL mode (e.g., Arabic).
- [ ] **Accessibility (a11y) Implementation**
    - [ ] Enforce `eslint-plugin-jsx-a11y` in the CI pipeline to catch structural accessibility issues at build time.
    - [ ] Implement "Skip to content" anchor links for screen reader navigation bypassing the header.
    - [ ] Implement explicit focus trapping inside all modal dialogs (e.g., Command Palette `Cmd+K`, Review Dialogs).
    - [ ] Ensure all dynamic notifications (Toasts, status check updates) use `aria-live="polite"` or `aria-live="assertive"`.
    - [ ] Ensure all decorative SVG icons have `aria-hidden="true"` and functional icons have `<title>` or `aria-label`.
    - [ ] Enforce WCAG 2.1 AA compliance for color contrast, especially in code syntax highlighting and dark mode diffs.
- [ ] **a11y Playwright Testing & Validation**
    - [ ] Integrate `@axe-core/playwright` and assert 0 violations on all major pages (Repository homepage, PR view, Settings).
    - [ ] Write a dedicated "Keyboard Only" Playwright test: navigate from repo homepage -> create PR -> submit review -> merge PR entirely using `page.keyboard.press()`.
    - [ ] Verify `aria-expanded` state toggles accurately on the "Code" clone dropdown and other nested menus.
    - [ ] Verify `aria-invalid` states on repository creation form when an existing name is typed.
    - [ ] Verify standard keyboard shortcuts: `t` for file finder, `w` for branch switcher, `Cmd+Enter` for submitting comments.
    - [ ] Verify focus rings (`:focus-visible`) are correctly styled with the standard blue outline and never suppressed via `outline: none` without a fallback.
- [ ] **Pull Request Pages (`PullRequestPage.ts`)**
    - [ ] Abstract DOM locators using GitHub's specific `js-*` classes and `data-testid` attributes.
    - [ ] Implement helper to assert the "Reviewers" sidebar autocomplete dropdown behavior and filtering.
    - [ ] Implement helper for adding/removing labels and verifying the exact hex color renders in the DOM.
    - [ ] Write test for Draft PR creation, specifically checking the gray draft icon rendering.
    - [ ] Write test for "Ready for Review" conversion, checking the green open icon and timeline event generation.
    - [ ] Write test for "Merge Pull Request" squashing, verifying the default commit message combines all PR commits.
    - [ ] Write test for cross-repository fork PR creation, verifying the base/head repository dropdowns.
- [ ] **Code Navigation & Blob Viewing (`CodeNavPage.ts`)**
    - [ ] Test the file tree pane state preservation (scroll position and open folders) on browser back navigation.
    - [ ] Verify the "Copy raw contents" button writes the exact file string to the clipboard (`navigator.clipboard.readText`).
    - [ ] Test the "Blame" toggle, asserting the left gutter renders correct commit SHAs and author avatars per line.
    - [ ] Test syntax highlighting applies correct CSS classes (e.g., `.pl-k` for keywords) based on language linguist detection.
    - [ ] Test rich file rendering for CSVs (table view), SVGs, and Images (diff swipe view).
    - [ ] Test the "Find in file" (`Cmd+F`) custom search bar overlay overriding the default browser search.
- [ ] **Review & Commenting Workflows (High-Fidelity)**
    - [ ] Test line-level commenting by hovering over gutter, clicking the blue `+`, and typing in the Markdown editor.
    - [ ] Toggle between "Unified" and "Split" diff views, asserting line comments anchor to the correct side.
    - [ ] Test replying to an existing comment thread, verifying the visual nesting indentation.
    - [ ] Test resolving a conversation thread, asserting the thread collapses and the "Resolved" badge appears.
    - [ ] Test unresolving a conversation, asserting the thread re-expands.
    - [ ] Test reaction emojis (👍, 👎, 🚀) on comments, verifying the counter increments and the active state toggles.
    - [ ] Verify GitHub-flavored Markdown rendering in the preview tab (tables rendering with borders, task lists rendering as checkboxes).
    - [ ] Verify `@` mentions trigger the user auto-suggest popover and render as user profile links.
    - [ ] Test the "Review Changes" dropdown states (Approve, Request Changes, Comment) and form submission.
- [ ] **Real-time Updates (WebSockets/SSE) Replication**
    - [ ] Intercept WebSocket `/cable` or SSE endpoints using Playwright's `page.route` to mock real-time payloads.
    - [ ] Simulate receiving a "CI Pending" payload and assert the yellow circle icon appears without page reload.
    - [ ] Simulate receiving a "CI Success" payload and assert the green checkmark appears and the Merge button enables.
    - [ ] Simulate receiving a new timeline comment payload and assert it injects at the bottom of the timeline dynamically.
    - [ ] Simulate "Review requested" real-time event and verify the reviewer appears in the right sidebar instantly.


### 2.3 Additional Workflows: Releases, Webhooks, & Social
**Phase 2.3.1: Releases & Tags UI**
- [ ] Build the `/releases` listing page with timeline view matching GitHub's specific vertical line layout.
- [ ] Implement the "Draft a new release" form with target branch/tag selector and auto-complete dropdown.
- [ ] Create a "Generate release notes" button that calls the backend to summarize merged PRs since the last tag.
- [ ] Implement chunked, resumable file uploads for release binaries (Assets) using S3 multipart upload APIs directly from the browser.
- [ ] Build the semantic version tag listing page with search/filtering and direct links to tag tree states.
- [ ] Implement a drag-and-drop zone for uploading release assets with a dynamic progress bar for each file.
- [ ] Build the "Compare" UI for releases, highlighting commit differences between two semantic version tags.
- [ ] Add explicit warnings when a user attempts to overwrite an existing release tag.

**Phase 2.3.2: Webhooks Management UI**
- [ ] Build the webhook configuration form supporting Payload URL, Content type (JSON/URL-encoded), and Secret input.
- [ ] Implement the granular event selector list (e.g., Push, Pull Request, Issues) with expandable descriptions.
- [ ] Build the "Recent Deliveries" log table showing HTTP status, execution time, and delivery ID.
- [ ] Implement a split-pane view for delivery details, showing the exact Request Headers/Payload and Response Headers/Body.
- [ ] Wire up the "Redeliver" button to trigger a replay mutation and instantly append the new result to the log.
- [ ] Implement JSON formatting/pretty-printing inside the webhook delivery payload viewer.
- [ ] Add a ping-webhook feature to send a test payload without requiring a real repository event.

**Phase 2.3.3: Social, Explore & Dashboards**
- [ ] Build the user dashboard "For You" timeline feed utilizing Infinite Scroll and intersection observers.
- [ ] Implement the "Trending" repositories page with date range filtering (Today, This week, This month) and language toggles.
- [ ] Build the repository "Insights" dependency graph UI mapping `package.json` / `Cargo.toml` dependencies via a D3.js force-directed graph.
- [ ] Implement the "Network" graph view showing cross-fork commit divergence utilizing a highly optimized HTML5 Canvas renderer.
- [ ] Build the "Stars" page for users, with list sorting, filtering, and custom list creation capabilities.
- [ ] Implement dynamic "Activity Squares" (the contribution graph) rendering SVG rects based on a daily commit count API response.
- [ ] Implement hover tooltips over the contribution graph indicating exact dates and contribution counts.

### 2.3 Additional Workflows: Releases, Webhooks, & Social (Expanded)
**Phase 2.3.1: Releases & Tags UI**
- [ ] Build the `/releases` listing page with a timeline view matching GitHub's specific vertical line and chronological layout.
- [ ] Implement the "Draft a new release" form with a target branch/tag selector utilizing a fuzzy-search auto-complete dropdown.
- [ ] Create a "Generate release notes" API call that automatically summarizes merged PRs since the last semantic tag.
- [ ] Implement chunked, resumable file uploads for release binaries (Assets) using S3 multipart upload APIs directly from the browser to bypass backend limits.
- [ ] Build the semantic version tag listing page with regex search/filtering and direct links to the exact tag tree states.
- [ ] Implement a drag-and-drop dropzone component for uploading release assets with a dynamic bytes-transferred progress bar for each file.
- [ ] Build the "Compare" UI specifically for releases, highlighting commit and file differences between two semantic version tags.
- [ ] Add explicit confirmation modals when a user attempts to overwrite or delete an existing release tag.
- [ ] Implement a "Pre-release" checkbox that visually distincts the release with a specific warning badge.
- [ ] Build the "Latest Release" badge logic for the repository sidebar, dynamically fetching the highest semver tag.

**Phase 2.3.2: Webhooks Management UI**
- [ ] Build the webhook configuration form supporting Payload URL, Content type (`application/json` vs `application/x-www-form-urlencoded`), and Secret inputs.
- [ ] Implement the granular event selector list (e.g., Push, Pull Request, Issues) mapping to the 100+ specific backend event triggers.
- [ ] Build the "Recent Deliveries" log table showing HTTP status codes, execution time in milliseconds, and the exact delivery GUID.
- [ ] Implement a split-pane view for delivery details, showing the exact Request Headers/Payload on top and Response Headers/Body on the bottom.
- [ ] Wire up the "Redeliver" button to trigger a replay GraphQL mutation and instantly append the new delivery result to the log table via Subscription.
- [ ] Implement JSON formatting/pretty-printing utilizing Monaco or Prism.js inside the webhook delivery payload viewer.
- [ ] Add a ping-webhook diagnostic feature to send a test payload (the `zen` event) without requiring a real repository mutation.
- [ ] Implement visual indicators (Red/Green icons) on the webhook listing page reflecting the success state of the last delivery.

**Phase 2.3.3: Social, Explore & Dashboards**
- [ ] Build the user dashboard "For You" timeline feed utilizing `IntersectionObserver` for seamless Infinite Scroll.
- [ ] Implement the "Trending" repositories page with precise date range filtering (Today, This week, This month) and language toggles.
- [ ] Build the repository "Insights" dependency graph UI mapping `package.json` / `Cargo.toml` dependencies via a D3.js force-directed graph.
- [ ] Implement the "Network" graph view showing cross-fork commit divergence utilizing a highly optimized HTML5 Canvas renderer to handle thousands of nodes.
- [ ] Build the "Stars" page for users, with list sorting, language filtering, and custom list creation capabilities.
- [ ] Implement dynamic "Activity Squares" (the contribution graph) rendering SVG `<rect>` elements based on a daily commit count API response.
- [ ] Implement exact hover tooltips over the contribution graph indicating the precise date and contribution count.
- [ ] Build the "Sponsors" modal, rendering a user's `FUNDING.yml` links dynamically in a standardized popover.
r.

### Phase 2.4: Profiles, Organizations, and Submodules
- [ ] **Profile README Implementation:** Detect if a repository exists matching the user's login (`user/user`) and dynamically fetch its `README.md` to render prominently at the top of the User Profile view.
- [ ] **Achievements & Badges UI:** Build the Angular components to render hexagonal achievement badges (e.g., "Pull Shark", "Arctic Code Vault") on the profile sidebar, fetching SVG assets dynamically.
- [ ] **Pinned Repositories:** Implement the interactive "Customize Pins" modal allowing users to drag-and-drop up to 6 repositories, persisting the sorted array via a GraphQL mutation.
- [ ] **Isometric Contribution Graph:** Integrate a 3D WebGL or Canvas-based isometric rendering of the contribution graph specifically for the "GitHub Skyline" or profile review features.
- [ ] **Submodule Tree Rendering:** Modify the File Explorer Tree component to parse `.gitmodules` entries; render specific submodule folder icons.
- [ ] **Submodule Hyperlinking:** Ensure clicking a submodule directory instantly resolves the upstream Git SHA and redirects the Angular Router to the correct external/internal repository URL at that exact commit.
- [ ] **Organization Dashboard:** Build the specific Organization overview page featuring pinned repositories, active people, and associated teams.
- [ ] **Outside Collaborators UI:** Implement the administrative data grid to manage "Outside Collaborators" (users with repo access but not org membership), including bulk removal actions.
- [ ] **Verified Domains Config:** Build the Angular form for Organization Admins to verify custom domains via TXT records, enabling restricted email domains for enterprise routing.
- [ ] **Organization Webhooks UI:** Implement the Webhooks management views specifically scoped to the Organization level, utilizing the existing webhook component library.

### Phase 2.5: Missing Parity Workflows (Rich Files, Media, Code Nav, Conflicts)
- [ ] **Jupyter Notebook Renderer:** Implement an Angular component that parses `.ipynb` JSON cells, rendering Markdown blocks natively and syntax-highlighting Python execution cells and tabular outputs.
- [ ] **3D Model Viewer:** Integrate `three.js` to render `.stl` and `.obj` files directly within the blob explorer, supporting mouse drag-to-rotate, scroll-to-zoom, and wireframe toggles.
- [ ] **GeoJSON Map Viewer:** Integrate `leaflet.js` or `mapbox-gl` to automatically render `.geojson` and `.topojson` files as interactive maps with clustered markers and polygon overlays.
- [ ] **PDF Blob Viewer:** Integrate `pdf.js` into the blob viewer to render PDF files natively within the Angular application without requiring users to download them to their local machine.
- [ ] **Drag-and-Drop Media Dropzone:** Build a global Angular directive applied to all Markdown textareas that intercepts native HTML5 drag-and-drop events and clipboard paste events for images.
- [ ] **Media Upload Orchestration:** Orchestrate the S3 pre-signed URL upload flow directly from the browser, injecting a `[Uploading image.png...]` placeholder in the textarea and replacing it with the final `![image](url)` upon completion.
- [ ] **Web-Based Conflict Editor:** Implement a specialized Monaco Editor component utilizing CodeLens and specific Regex matching to distinctly highlight `<<<<<<< HEAD` / `=======` / `>>>>>>> branch` conflict markers.
- [ ] **Conflict Resolution Actions:** Add "Accept Current Change", "Accept Incoming Change", and "Accept Both Changes" floating action buttons above conflict markers in the web editor to automate resolution.
- [ ] **Semantic "Symbols" Sidebar:** Build the file-level Outline sidebar pane in the code view, querying the Rust Blackbird API to list all functions, classes, and structs within the currently viewed file.
- [ ] **Clickable AST Navigation:** Implement popovers in the blob viewer: when a user clicks a method call, dynamically query the Blackbird semantic backend to display a "Go to definition" / "Find all references" floating dialog mapped to the exact line number.

### Phase 2.6: Consolidated Settings Architecture
- [ ] **Global Settings Shell:** Build the unified Settings navigation sidebar component utilizing `@angular/router` deeply nested child routes (`/settings/profile`, `/settings/security`).
- [ ] **Dynamic Form Generation:** Implement a generic, data-driven Angular Form mapping component to generate simple toggle/input settings natively from a JSON schema to prevent massive component duplication.
- [ ] **Integrations Layout:** Build the specific "Installed GitHub Apps" and "Authorized OAuth Apps" pane, strictly rendering granular permission scopes (e.g., `Read/Write` on `Issues`) requested by the app.
- [ ] **Deploy Keys UI:** Implement the `DeployKeysComponent` managing SSH keys tied exclusively to a repository, tracking the "Last used" timestamp and highlighting it if inactive for >1 year.
- [ ] **Audit Log Viewer:** Render the Organization/Enterprise Audit log interface utilizing `@angular/cdk/table` and infinite scrolling, supporting complex `actor:samuel action:repo.create` search queries.
- [ ] **Danger Zone Architecture:** Build the generic "Danger Zone" block specifically enforcing a red-bordered UI, strictly requiring the user to type the exact repository/username into a confirmation input before enabling destructive API calls.

### Phase 2.7: Complex UI Edge Cases & Visualizations
- [ ] **YAML Issue Forms Renderer:** Implement a robust Angular engine capable of parsing `.github/ISSUE_TEMPLATE/*.yml` schema files and dynamically generating complex Reactive Forms containing dropdowns, checkboxes, and conditionally required text areas.
- [ ] **Dependency Network Graph Canvas:** Build the specific "Network" tab visualizer utilizing an optimized HTML5 `<canvas>` element to render massive, interactive cross-fork commit timelines and branching structures without DOM bloat.
- [ ] **Traffic Insights Graphs:** Implement D3.js or Apache ECharts integrations to render interactive "Clones" and "Visitors" line charts, plotting unique daily/weekly metrics returned from the backend aggregation APIs.
- [ ] **Repository Code Frequency Visualization:** Build the specific Insights graph rendering additions and deletions (green/red stacked bar charts) spanning the entire repository lifecycle.
- [ ] **Punchcard / Commit Time Heatmap:** Render the "Punchcard" view representing commit frequency across days of the week and hours of the day using dynamically shaded SVG circles.
- [ ] **Dynamic Language Bar:** Implement the language statistics bar under the repository description, calculating precise percentage widths based on the backend Linguist byte-count response.

### Phase 2.8: Certifications, Custom Properties & Sponsor Discovery
- [ ] **GitHub Certifications UI:** Build the `/certifications` user profile tab in Angular, rendering high-fidelity, verified badges mapped to the user's credential status stored in the Rust backend.
- [ ] **Repository Custom Properties:** Implement the UI for Enterprise Administrators to define custom `key:value` metadata schemas (e.g., `Data Classification: Confidential`, `Cost Center: 1234`).
- [ ] **Custom Properties Enforcement:** Build the Angular repository settings pane allowing maintainers to assign these custom properties, mapping them directly to Postgres JSONB columns.
- [ ] **Sponsors Explore Feed:** Implement a dedicated `/sponsors/explore` discovery feed utilizing a masonry layout, dynamically highlighting open-source projects relevant to the user's starred repositories.
- [ ] **Interactive Funding UI:** Build the streamlined checkout modal in Angular for the Sponsors feature, rendering Stripe Elements securely within an iframe without breaking the Primer visual aesthetics.
- [ ] **Achievement Animations:** Implement complex CSS keyframe animations and Lottie web player integration for rendering celebratory sequences when a user unlocks a new profile achievement (e.g., "Pull Shark").

### Phase 2.9: Developer Settings & App Registration UI
- [ ] **Developer Settings Shell:** Build the nested navigation for User/Organization Developer Settings (OAuth Apps, GitHub Apps, Personal Access Tokens) utilizing lazy-loaded Angular route modules.
- [ ] **GitHub App Creation Wizard:** Implement the complex multi-step reactive form for creating GitHub Apps, validating Callback URLs, Setup URLs, and Webhook domains strictly via regex.
- [ ] **Granular Permissions Matrix:** Build the expansive UI matrix allowing developers to toggle `Read/Write/No Access` across 50+ API scopes, utilizing Angular Signals to efficiently track dirty form states.
- [ ] **Webhook Event Subscription Selector:** Implement the checkbox grid dynamically mapping selected API permissions to their corresponding subscribable Webhook events (e.g., enabling "Issues" read unlocks "Issue Comment" events).
- [ ] **Private Key Generation UI:** Build the interface to trigger RSA private key generation; intercept the Blob response and trigger a browser-side file download for the `.pem` file, displaying a "one-time only" warning.
- [ ] **OAuth App Management:** Implement the creation and management UI for legacy OAuth Apps, including Client Secret generation, masking/unmasking visibility toggles, and targeted revocation flows.
- [ ] **App Logo Cropper:** Integrate an Angular-native image cropping component allowing developers to upload, zoom, and crop custom SVG/PNG branding logos for their marketplace applications.
- [ ] **Device Flow Authentication Screens:** Build the specific `/login/device` screens rendering the 8-character user code input, validating the format instantly as the user types.
- [ ] **Fine-Grained PAT UI:** Implement the Fine-Grained Personal Access Token generation form, including a strict date-picker that caps the `expiration_date` to a maximum of 365 days from today.
- [ ] **Resource Owner Selection:** Build the repository targeting component for Fine-Grained PATs, utilizing `@angular/cdk/scrolling` to render a virtualized, searchable dropdown for users with 10,000+ repositories.
- [ ] **Token Revocation Data Grid:** Render a sortable table of all active OAuth tokens, Classic PATs, and Fine-Grained PATs, displaying their `last_used` date, origin IP, and a destructive "Revoke" button.
- [ ] **App Installation UI:** Build the targeted installation screens where users select whether to install an App on "All Repositories" or a specific subset, passing the final selection array via GraphQL.
- [ ] **Webhook Delivery Logs View:** Implement a split-pane log viewer for GitHub App Webhooks, rendering the exact JSON request payload, response headers, and HTTP status codes for the last 30 days of deliveries.
- [ ] **Redeliver Webhook Action:** Add a "Redeliver" button to the delivery log UI that executes a backend mutation and optimistically prepends the new delivery result to the Angular data grid.
- [ ] **App Suspension Controls:** Build the Danger Zone UI for Enterprise Admins to globally suspend or completely delete a malicious GitHub App, requiring strict username/app-name confirmation inputs.

### Phase 2.10: Actions Runner Management & Grouping UI
- [ ] **Runner Fleet Dashboard:** Build the Organization/Enterprise Settings view listing all registered self-hosted runners, displaying real-time online/offline status via polling or Server-Sent Events (SSE).
- [ ] **Runner Registration Wizard:** Implement the modal providing dynamic, OS-specific (Linux, macOS, Windows) curl/bash snippets, dynamically injecting short-lived registration tokens fetched from the Rust API.
- [ ] **Runner Labels Management:** Build an interactive Angular tag-editor allowing administrators to assign custom routing labels (e.g., `gpu-a100`, `arm64-large`) to specific runners for targeted workflow dispatch.
- [ ] **Runner Groups Architecture:** Implement the Runner Groups management UI, allowing enterprise admins to partition fleets and strictly bind specific groups to designated organizations or individual repositories.
- [ ] **Ephemeral Runner Toggles:** Add UI switches to explicitly mark runner configurations as ephemeral, visually distinguishing them in the fleet dashboard with a specific Primer SVG icon.
- [ ] **Runner Network Topology UI:** Display connected runner IP addresses, host machine architectures, operating system details, and installed agent versions in a detailed slide-out properties pane.
- [ ] **Active Job Linkage:** Render a hyperlinked progress spinner next to runners that are actively executing a job, allowing admins to click directly into the workflow run logs.
- [ ] **Auto-Update Configuration:** Build the toggle interface allowing admins to enable or disable automatic minor version updates for the self-hosted runner binaries.
- [ ] **Bulk Label Mutation:** Implement a multi-select data grid for runners, exposing an action bar to append or remove specific routing labels across 50+ runners simultaneously.
- [ ] **Offline Runner Pruning:** Add a "Remove offline runners" administrative button that executes a batch deletion mutation for all agents that have missed their heartbeat for >30 days.
- [ ] **Runner Registration Token Revocation:** Provide UI controls to manually revoke an active Runner Registration token before its natural 1-hour expiration.
- [ ] **Runner Queue Metrics:** Integrate Echarts or Chart.js to render a visual histogram of "Queued vs Active" jobs associated with a specific Runner Group over the last 24 hours.

### Phase 2.11: Repository Rulesets & Branch Protection UI
- [ ] **Rulesets Dashboard:** Build the centralized Repository and Organization level UI listing all active Rulesets, their enforcement status (Active, Evaluate, Disabled), and target branches/tags.
- [ ] **Rule Target Configuration:** Implement the complex target selector allowing admins to apply rules via include/exclude regex patterns (e.g., `refs/heads/release-*`) and validate the regex instantly in the browser.
- [ ] **Status Checks Autocomplete:** Implement a debounced RxJS stream that queries historical CI executions to suggest available context names when configuring the "Require status checks to pass" rule.
- [ ] **Require Linear History Toggle:** Build the UI components to enforce linear history, automatically exposing sub-options to enforce squash-merging or rebase-merging on the target branch.
- [ ] **Commit Signature Enforcement:** Add the UI toggle to require cryptographically verified commits (GPG/SSH/S/MIME), rendering warnings about the impact on web-based edits.
- [ ] **Committer Email Restrictions:** Implement a regex-validated input array allowing enterprises to restrict pushes exclusively to commits authored by specific domains (e.g., `*@enterprise.com`).
- [ ] **Bypass Actor Configuration:** Build the RBAC selector allowing admins to explicitly whitelist specific Teams, Apps, or Roles (e.g., "Repository Admin") that can bypass the configured ruleset.
- [ ] **Bypass Mode Selectors:** For each bypassed actor, implement a dropdown specifying if they bypass "Always" or "Only for Pull Requests".
- [ ] **Ruleset Evaluation Insights:** Implement a timeline UI showing exactly which pushes were blocked or allowed by a specific ruleset during its "Evaluate" (dry-run) phase to ensure safe rollouts.
- [ ] **JSON Export/Import:** Provide buttons to download the entire ruleset configuration as a strict JSON schema, and allow uploading JSON to clone rulesets across repositories.
- [ ] **Required Reviewers Matrix:** Build the counter input (1-6) for required approving reviews, alongside checkboxes to "Dismiss stale pull request approvals when new commits are pushed".
- [ ] **Code Owner Review Enforcement:** Add the toggle to explicitly require approval from the mapped CODEOWNERS before the branch can be merged.
- [ ] **Deployment Branch Restrictions:** Implement the environment targeting UI, restricting specific Ruleset branches to only deploy to selected GitHub Environments.
- [ ] **Conflict Resolution Warnings:** Display a dynamic Angular warning banner if an administrator configures two overlapping rulesets with conflicting requirements for the same branch pattern.

### Phase 2.12: Security Advisories & Coordinated Vulnerability Disclosure (CVD) UI
- [ ] **Security Advisory Dashboard:** Build the repository-level "Security Advisories" tab, strictly gating access utilizing Angular Route Guards for maintainers and authorized security collaborators.
- [ ] **Advisory Drafting Wizard:** Implement the rich-text reactive form for CVEs, including fields for affected ecosystem (npm, crates.io), vulnerable version ranges, and patched versions.
- [ ] **Interactive CVSS 3.1 Calculator:** Build a highly interactive Angular component modeling the CVSS 3.1 vector matrix; dynamically calculate and update the Critical/High/Medium score as users click attack vectors (e.g., Network, High Privileges).
- [ ] **CWE Fuzzy Search:** Implement an autocomplete dropdown that queries a static JSON dictionary of Common Weakness Enumerations (CWEs) by ID or description (e.g., `CWE-79 Cross-site Scripting`).
- [ ] **Private Fork Provisioning UI:** Build the "Start a temporary private fork" workflow, rendering a loading spinner while the backend generates a sandboxed, invisible collaboration environment.
- [ ] **CVD Collaboration Workspace:** Implement a localized issue/commenting timeline exclusively attached to the draft advisory for secure communication between the external reporter and internal maintainers.
- [ ] **External Researcher Invitations:** Build a search-by-username input allowing maintainers to invite external security researchers to the private advisory without granting them baseline repository access.
- [ ] **CVE Request Integration:** Add the UI button and status indicator for requesting a formal CVE identification number directly from the internal CNA (CVE Numbering Authority) proxy backend.
- [ ] **Advisory Credit Management:** Build a dynamic array input allowing maintainers to assign multiple users specific credit types (Reporter, Analyst, Coordinator, Remediation Developer).
- [ ] **Bounty Allocation UI:** Implement the internal inputs for enterprise maintainers to attach a USD bug bounty reward amount to the advisory, integrating with the GitHub Sponsors/Stripe payout system.
- [ ] **Advisory Publication Flow:** Implement the final publication modal, transitioning the advisory from Draft to Public, rendering validation warnings if the CVE ID is missing or CVSS score is 0.0.
- [ ] **Dependabot Alert Linkage:** Render a visual list of all internal repositories currently vulnerable to the drafted advisory, indicating which ones will receive automated Dependabot PRs upon publication.
- [ ] **Global Advisory Search:** Implement the public `/advisories` search interface, utilizing infinite scrolling and faceted filtering by ecosystem, severity, and date published.
- [ ] **Embargo Countdown Timer:** Build a localized countdown clock component on the private advisory page indicating exactly when the Coordinated Vulnerability Disclosure embargo lifts.
### Phase 2.13: GitHub Apps Marketplace & Compare Views
- [ ] **Marketplace Landing Page:** Build the `/marketplace` Angular homepage featuring carousel headers, categorical navigation, and a fuzzy-search bar for third-party integrations.
- [ ] **App Listing UI:** Create the detailed App view rendering the developer's marketing Markdown, structured pricing tiers, and required API permissions scopes in a sticky sidebar.
- [ ] **Installation Wizard:** Implement the installation modal allowing users to explicitly select "All repositories" or a granular checklist of specific repositories for the App context.
- [ ] **Stripe Checkout Integration:** Build the "Manage Billing" modal for paid Marketplace Apps, integrating Stripe Elements natively to securely handle credit card tokenization.
- [ ] **User Reviews & Ratings:** Implement the five-star review and commenting components for published Apps, including author reply threading and helpfulness upvotes.
- [ ] **The "Compare" View Foundation:** Build the `/compare` Angular route capable of handling complex ref structures (e.g., `base...head` or `branch1...tag2`) natively.
- [ ] **Comparison Timeline:** Render a specialized timeline summarizing the commits, active PRs, and contributors introduced strictly between the two comparison refs.
- [ ] **Rich Diff Viewer Reuse:** Instantiate the PR diff viewer component outside the PR context to render file changes between the arbitrary compared commits.
- [ ] **Contextual PR Creation:** Add a sticky "Create Pull Request" action bar in the Compare view that pre-fills the base and head refs based on the current comparison state.
- [ ] **Sync Fork Dropdown:** Implement the "Sync Fork" dropdown component on the repository homepage, querying the upstream default branch for divergence stats.
- [ ] **Fast-Forward Action:** Add the "Update branch" button inside the Sync Fork dropdown, triggering the backend fast-forward/reset GraphQL mutation and displaying a success toast.
- [ ] **Dependency Review Rich Diff:** Build a specialized "Rich Diff" view tab specifically for supported lockfiles (`package-lock.json`, `Cargo.lock`, `yarn.lock`) in the Pull Request UI.
- [ ] **Lockfile AST Parsing:** Parse the lockfile diffs directly in the frontend to extract and list added, removed, and updated transitive dependency version strings.
- [ ] **CVE Alert Overlays:** Query the Security Advisory database and render inline warnings (Yellow/Red alert boxes) within the PR diff if a new dependency introduces a known vulnerability.
- [ ] **License Compliance Diffing:** Render license changes (e.g., `MIT` to `GPL-3.0`) natively inside the lockfile diff view, utilizing a backend license evaluation endpoint.

### Phase 2.14: Marketplace Storefront & Installation UX
- [ ] **Category Masonry Layout:** Implement a `@angular/cdk/scrolling` masonry grid for the Marketplace homepage, dynamically rearranging App cards based on window viewport width without causing reflows.
- [ ] **Dynamic Faceted Search:** Build the left-sidebar faceted filtering UI (Categories, Free/Paid, Verification Status), utilizing Angular Signals to trigger instant search query updates.
- [ ] **Verified Creator Badges:** Render distinct, high-fidelity SVGs (blue checkmarks) on App listings dynamically linked to organizations that have passed the Stripe Identity/KYB verification checks.
- [ ] **Markdown App Descriptions:** Render the App's detailed marketing description utilizing the internal strict Markdown parser, blocking external `iframe` or dangerous CSS injections.
- [ ] **Sticky Pricing Table:** Implement a sticky pricing grid component highlighting features, seat limits, and recurring intervals (Monthly/Annually) with smooth CSS transition toggles.
- [ ] **Installation Target Selector:** Build a custom modal UI utilizing a virtualized list allowing users to search and explicitly select 100+ individual repositories for a granular App installation.
- [ ] **Permissions Matrix Consent View:** Render a bold, red-highlighted matrix explicitly detailing the exact read/write API scopes the App is requesting before the user can click "Authorize".

### Phase 2.15: The Compare View & Sync Fork Automation
- [ ] **Fuzzy Ref Selector Dropdown:** Implement the specialized branch/tag selector dropdown combining `RxJS` debounce mapping against the backend Git references API to find specific commits globally.
- [ ] **Three-Dot vs Two-Dot Logic UI:** Render dynamic visual explainers indicating whether the comparison is utilizing the merge-base (three-dot `...`) or a direct diff (two-dot `..`) based on the URL.
- [ ] **Empty State Resolution:** Detect when `ahead_by` and `behind_by` are both zero and render a specific "There isn't anything to compare" Primer empty state component with guidance.
- [ ] **Commit Pagination Integration:** Integrate infinite scrolling on the Compare view's "Commits" tab to gracefully handle branches that are thousands of commits ahead.
- [ ] **Sync Fork Status Poller:** Implement an RxJS polling mechanism on fork homepages that checks the upstream repository state every 60 seconds to detect new commits.
- [ ] **Sync Fork Resolution UI:** Render the "Fetch and Merge" dropdown dynamically; if a fast-forward is impossible due to conflicts, dynamically replace the primary CTA with "Open Pull Request".
- [ ] **Optimistic UI on Fast-Forward:** When "Update branch" is clicked, optimistically increment the branch's commit count and update the 'latest commit' hash in the UI before the backend confirms success.

### Phase 2.16: Dependency Review & Rich Diff Overlays
- [ ] **Lockfile Diffs Navigation:** Add specific file-tree icons marking lockfiles (`Cargo.lock`, `package-lock.json`) with a distinct shield icon within the Pull Request files changed view.
- [ ] **Transitive Dependency Tree View:** Build an Angular recursive component mapping added/removed top-level dependencies directly to their nested transitive dependencies in a collapsible tree.
- [ ] **Vulnerability Alert Overlay:** Inject deep-red alert panels directly below the specific lockfile diff line if a newly added dependency version maps to a known OSV vulnerability.
- [ ] **CVSS Score Badge Renderer:** Build an Angular pipe generating color-coded badges (Low: Gray, Medium: Yellow, High: Orange, Critical: Red) based on the vulnerability's CVSS severity score.
- [ ] **License Compliance Badges:** Render SPDX license tags (e.g., `MIT`, `Apache-2.0`) next to every added dependency, utilizing a red strikethrough if the license violates the repository's configured allowlist.
- [ ] **"View Advisory" Sidebar:** Implement an off-canvas Angular sidebar that slides in when a vulnerability alert is clicked, rendering the full markdown description, patched version, and mitigation steps.
