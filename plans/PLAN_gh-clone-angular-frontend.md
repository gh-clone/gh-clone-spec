# Plan for gh-clone-angular-frontend

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
    - [ ] Integrate an i18n framework (e.g., `i18next` or `react-intl`) and configure dynamic loading of translation namespaces.
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

