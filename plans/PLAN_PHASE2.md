## 2. API PARITY & THE GENERATION COMPILER

We do not write API boilerplate. We build a compiler that reads `api.github.com.yaml` and emits Rust.

### 2.1 The `openapi-bindgen` AST Compiler

**Phase 2.1.1: OpenAPI 3.1 Lexical Parsing & Validation**
- [ ] **Recursion-Safe YAML Ingestion:** Configure `serde_yaml::Deserializer` with `recursion_limit(1024)` to safely ingest the 150,000+ line `api.github.com.yaml` document without stack overflows.
- [ ] **Draft 2020-12 Strict Validation:** Instantiate `jsonschema` crate strictly enforcing Draft 2020-12 to validate the OpenAPI spec payload before processing begins.
- [ ] **Path Parameter Normalization:** Scan all OpenAPI path strings (e.g., `/repos/{owner}/{repo}`) and enforce a unified internal regex representation `^/[a-zA-Z0-9_.-]+$`.
- [ ] **`$ref` Pointer Canonicalization:** Resolve all intra-document JSON pointers (e.g., `#/components/schemas/Commit`) into absolute, globally unique memory addresses within the compiler.
- [ ] **Extension Key Extraction:** Traverse the AST to extract and map all custom `x-github-*` extensions (e.g., `x-github-media-type`, `x-github-scopes`) into a dedicated metadata HashMap.
- [ ] **Orphaned Schema Detection:** Implement a dead-code elimination pass to detect and purge schemas in `components.schemas` that are never referenced by any endpoint.
- [ ] **Type Coercion Logging:** Emit verbose compiler warnings whenever a non-standard OpenAPI type combination is detected (e.g., `type: string, format: integer`).
- [ ] **Summary to Rustdoc Pre-pass:** Extract `summary` and `description` fields, stripping HTML tags and converting them to pure Markdown for future Rustdoc injection.
- [ ] **Deprecation Hardening:** Map `deprecated: true` flags globally to an internal AST state that triggers `#![allow(deprecated)]` and `#[deprecated]` emissions correctly.
- [ ] **Empty Schema Fallbacks:** Identify empty schema objects `{}` and coerce them into `serde_json::Value` strictly for untyped arbitrary payload mapping.
- [ ] **Global Parameter Resolution:** Resolve all `components.parameters` injected directly into path arrays, merging them with operation-level parameters to compute the final required signature.

**Phase 2.1.2: AST Graph Construction & Cycle Resolution**
- [ ] **Directed Graph Instantiation:** Initialize `petgraph::DiGraph` to represent all schemas as nodes, with fields and references acting as directed edges.
- [ ] **Edge Weight Classification:** Assign weights to edges to distinguish between `1:1` (object property), `1:N` (array items), and `Polymorphic` (`oneOf`/`anyOf`) relationships.
- [ ] **Tarjan's SCC Algorithm:** Execute Tarjan's Strongly Connected Components algorithm to precisely identify cyclic subgraphs (e.g., `Issue` -> `User` -> `Issue`).
- [ ] **Recursive Entrypoint Identification:** Within each cycle, identify the "entrypoint" node with the highest in-degree to act as the primary break point.
- [ ] **`Box<T>` Injection for 1:1 Cycles:** Dynamically mutate the AST graph to inject a `Box` wrapper type modifier exactly at the identified 1:1 cycle boundaries.
- [ ] **`Vec<T>` Cycle Bypass:** Validate that `1:N` cyclic relationships inherently break recursion via Rust's `Vec`, requiring no `Box` injection.
- [ ] **`Arc<T>` Injection for Read-Only Cycles:** Configure compiler flags to optionally use `Arc<T>` instead of `Box<T>` for schemas designated as immutable read-only models.
- [ ] **Inline Schema Flattening Pass:** Traverse all operation request/response definitions, extracting anonymous inline objects and lifting them to top-level schemas with generated names (e.g., `PostReposOwnerRepoBody`).
- [ ] **AllOf Composition Resolution:** Flatten `allOf` inheritance arrays by deep-merging properties, required arrays, and descriptions into a single unified struct definition.
- [ ] **AnyOf Heuristic Simplification:** Analyze `anyOf` arrays; if they represent nullable unions (e.g., `anyOf: [{type: string}, {type: null}]`), simplify them purely to Rust `Option<T>`.

**Phase 2.1.3: Rust Struct & Trait Synthesis**
- [ ] **Struct Name PascalCase Normalization:** Enforce strict `PascalCase` conversions for all struct names utilizing the `heck` crate (e.g., `pull-request` -> `PullRequest`).
- [ ] **Field Name snake_case Normalization:** Enforce strict `snake_case` conversions for struct fields, checking against Rust reserved keywords (e.g., `type` -> `r#type`).
- [ ] **Integer Format Mapping:** Map OpenAPI `integer` with `format: int32` strictly to Rust `i32`.
- [ ] **Long Integer Format Mapping:** Map OpenAPI `integer` with `format: int64` strictly to Rust `i64` to prevent user/repo ID overflow panics.
- [ ] **Float Format Mapping:** Map OpenAPI `number` with `format: float` to `f32` and `format: double` to `f64`.
- [ ] **Strict Date-Time Mapping:** Map OpenAPI `string` with `format: date-time` to `time::OffsetDateTime` instead of `chrono` to prevent panic-on-leap-second vulnerabilities.
- [ ] **URL String Semantic Typing:** Map OpenAPI `string` with `format: uri` strictly to `url::Url` to ensure payload hyperlinks are valid upon deserialization.
- [ ] **Base64 Payload Typing:** Synthesize a custom `Base64String` wrapper type for properties explicitly documented as holding base64-encoded file content.
- [ ] **Struct Derivation Injections:** Inject `#[derive(Debug, Clone, PartialEq, Serialize, Deserialize)]` automatically onto every generated Rust struct.
- [ ] **Builder Trait Synthesis:** Utilize `derive_builder` macro generation to create fluent `::builder()` initializers for all complex request bodies.
- [ ] **String Enum Synthesis:** Detect parameters with `enum` arrays and generate explicit Rust `enum` types with `#[serde(rename_all = "snake_case")]` configurations.
- [ ] **Display Trait for Enums:** Synthesize explicit `std::fmt::Display` trait implementations for all generated string-enums to support URL formatting.
- [ ] **FromStr Trait for Enums:** Synthesize `std::str::FromStr` trait implementations for all string-enums to support robust Axum path/query extraction.
- [ ] **Default Trait Generation:** Synthesize `impl Default` blocks, securely mapping OpenAPI `default` values directly into the Rust initialization logic.
- [ ] **Null vs Absent Field Handling:** For optional fields that can be explicitly set to null, synthesize `Option<Option<T>>` to distinguish `null` from omitted JSON keys.

**Phase 2.1.4: Serde & Polymorphic Deserialization**
- [ ] **Discriminator Key Extraction:** Identify OpenAPI `discriminator.propertyName` mappings (e.g., `event` on Timeline payloads) to construct efficient `oneOf` parsers.
- [ ] **Untagged Union Visitor Synthesis:** Generate custom `serde::de::Visitor` implementations for untagged `oneOf` unions using `serde_json::value::RawValue` peeking.
- [ ] **O(1) Discriminator Routing:** Implement exact string matching within the untagged visitors to route payload deserialization to the correct struct in `O(1)` time without backtracking.
- [ ] **Fallback Layout Matching:** For `oneOf` payloads lacking a discriminator, generate sequential heuristic fallback attempts, trying to deserialize into the most restrictive struct first.
- [ ] **Sparse Payload Optimization:** Inject `#[serde(skip_serializing_if = "Option::is_none")]` strictly on all non-required fields to match GitHub's sparse PATCH payloads.
- [ ] **Custom Boolean Deserializer:** Generate a `deserialize_github_bool` function handling legacy GitHub payloads where booleans are occasionally serialized as `"true"` or `"false"` strings.
- [ ] **Custom Nullable Date Deserializer:** Generate a `deserialize_github_datetime` function that specifically handles empty strings `""` or `"0000-00-00T00:00:00Z"` gracefully as `None`.
- [ ] **Empty Array Coercion:** Inject `#[serde(default)]` on all `Vec<T>` properties to ensure omitted array fields are deserialized as `[]` rather than returning an error.
- [ ] **Catch-All Additional Properties:** If `additionalProperties` is true, inject a `#[serde(flatten)] pub extra: std::collections::HashMap<String, serde_json::Value>` field to capture arbitrary keys.
- [ ] **Renaming Keywords:** Map OpenAPI property names containing dashes (e.g., `foo-bar`) utilizing `#[serde(rename = "foo-bar")]` while maintaining `foo_bar` struct identifiers.

**Phase 2.1.5: Axum Router & Extractor Generation**
- [ ] **Route Tree Flattening:** Flatten the OpenAPI path definitions into a deterministic vector sorted alphabetically to prevent Axum route collision panics.
- [ ] **Axum Method Routing:** Map OpenAPI `get`, `post`, `patch`, `put`, `delete` strictly to their corresponding `axum::routing::{get, post, patch, put, delete}` macros.
- [ ] **Path Variable Regex Conversion:** Convert OpenAPI `{owner}` template syntax directly into Axum `/:owner` path segment syntax within the Router definitions.
- [ ] **Strongly-Typed Path Extractors:** Synthesize explicit `axum::extract::Path<RepoParams>` structs mapping directly to the specific variables required by each endpoint.
- [ ] **Strict Query Extractor Synthesis:** Synthesize `axum::extract::Query<ListRepoParams>` structs, enforcing required vs optional pagination/filtering parameters via Serde.
- [ ] **JSON Body Extraction Validation:** Synthesize `axum::extract::Json<CreateIssueBody>` extractors specifically for `application/json` operations, enforcing exact structural requirements.
- [ ] **Raw Body Extraction Mapping:** Synthesize `axum::body::Bytes` extractors for endpoints explicitly declaring `application/octet-stream` or raw file uploads.
- [ ] **Multipart Extractor Synthesis:** Implement `axum::extract::Multipart` parsing specifically mapped to endpoints matching `multipart/form-data` schemas.
- [ ] **Handler Function Signatures:** Generate async handler function skeletons dynamically returning `Result<Json<ResponseT>, GithubApiError>`.
- [ ] **VND Accept Header Guard Extraction:** Generate a custom `FromRequestParts` extractor requiring the precise `application/vnd.github+json` Accept header strictly on all API operations.
- [ ] **Sub-Router Modularization:** Generate distinct Axum `Router` blocks per top-level API domain (e.g., `repos_router()`, `issues_router()`) for crate modularity.

**Phase 2.1.6: Middleware, Rate Limits & Security Guards**
- [ ] **Token Bucket State Injection:** Inject a high-performance `Arc<Mutex<TokenBucket>>` state extractor globally into the Axum Router for rate limit tracking.
- [ ] **Rate Limit Headers Generator:** Implement Axum middleware that dynamically appends `X-RateLimit-Limit`, `X-RateLimit-Remaining`, and `X-RateLimit-Reset` to every HTTP response.
- [ ] **Abuse Limit Header Injection:** Configure middleware to intercept secondary rate limit breaches and automatically inject a `Retry-After: 60` HTTP header.
- [ ] **Redis Rate Limit Backing:** Implement a Redis connection pool adapter for the Token Bucket to synchronize rate limits across distributed backend instances.
- [ ] **UUIDv7 Request-ID Generation:** Write Axum middleware generating UUIDv7 strings and injecting them specifically into the `X-GitHub-Request-Id` response header.
- [ ] **Request-ID Tracing Span:** Bind the generated `X-GitHub-Request-Id` directly into `tracing::info_span!` to guarantee correlated logs across the execution boundary.
- [ ] **App Token JWT Guard:** Generate an Axum `FromRequestParts` guard that strictly parses `Bearer` tokens and validates GitHub App JWT signatures via `jsonwebtoken`.
- [ ] **Installation Token Scopes Guard:** Extract `x-github-scopes` from the OpenAPI spec and generate middleware asserting the incoming token holds the exact required repository bits.
- [ ] **Classic PAT Prefix Validation:** Implement fast-path middleware validating `ghp_` prefixes for Classic PATs before committing to database authentication lookups.
- [ ] **Fine-Grained PAT Prefix Validation:** Implement fast-path middleware validating `github_pat_` prefixes for Fine-Grained PATs.
- [ ] **Sudo-Mode Extractor:** Generate an extractor for `X-GitHub-OTP` headers, returning a `401 Unauthorized` with the `X-GitHub-OTP: required; :2fa-type` header if missing.
- [ ] **Strict CORS Implementation:** Generate `tower_http::cors::CorsLayer` specifically allowing `*` origins but strictly constraining `Access-Control-Allow-Headers` to exact GitHub specifications.
- [ ] **Preflight Caching Headers:** Inject `Access-Control-Max-Age: 86400` into CORS preflight `OPTIONS` responses to minimize redundant browser checks.
- [ ] **Content-Length Guardrail:** Enforce a strict `100MB` payload limit middleware using `tower_http::limit::RequestBodyLimitLayer` for release asset endpoints.
- [ ] **406 Error Schema Mapping:** Guarantee `406 Not Acceptable` responses map precisely to a generated `BasicError` struct containing `message` and `documentation_url`.
- [ ] **422 Error Schema Mapping:** Map validation failures directly to `422 Unprocessable Entity` payloads, populating the `errors` array with the exact `resource`, `field`, and `code`.
- [ ] **500 Error Scrubbing:** Implement a global panic catcher returning a scrubbed `500 Internal Server Error` ensuring internal stack traces never leak into the JSON payload.

**Phase 2.1.7: Pagination, HATEOAS & HTTP Headers**
- [ ] **Cursor Pagination Structs:** Generate strongly-typed cursor structs parsing `before` and `after` base64 query parameters for GraphQL-style endpoints mapped to REST.
- [ ] **Offset Pagination Validation:** Generate query extractors strictly bounding `page` variables and clamping `per_page` maximum limits to 100 as dictated by the OpenAPI spec.
- [ ] **RFC 5988 Link Header Formatting:** Synthesize robust URL formatters combining the current path and query state to generate RFC 5988 compliant `<url>; rel="next"` Link headers.
- [ ] **HATEOAS Relation Calculation:** Dynamically compute the presence of `rel="first"`, `rel="last"`, `rel="prev"`, and `rel="next"` based on row counts from the database layer.
- [ ] **Farmhash ETag Computation:** Implement middleware computing `farmhash::fingerprint64` over serialized JSON response bodies to generate strong `ETag` HTTP headers.
- [ ] **304 If-None-Match Interception:** Implement early-return middleware comparing the incoming `If-None-Match` header to the ETag, immediately halting execution with `304 Not Modified`.
- [ ] **If-Modified-Since Optimization:** Extract `If-Modified-Since` headers into `time::OffsetDateTime` and pass them to the underlying handlers for optimized database query pruning.
- [ ] **Cache-Control Synthesis:** Read OpenAPI caching extensions to dynamically output `Cache-Control: public, max-age=60, s-maxage=60` headers for specific public endpoints.

**Phase 2.1.8: Webhook Validation Subsystem**
- [ ] **Subtle Constant-Time Comparison:** Implement webhook payload validation strictly using the `subtle` crate's `ConstantTimeEq` to defeat cryptographic timing attacks.
- [ ] **HMAC SHA-256 Signature Extractor:** Generate middleware extracting `X-Hub-Signature-256` and computing the HMAC SHA-256 hash of the raw `Bytes` payload against a shared secret.
- [ ] **Delivery GUID Extractor:** Extract and validate the `X-GitHub-Delivery` header, enforcing strict UUID formats before accepting webhook payloads.
- [ ] **Event Routing Discriminator:** Route webhook payloads dynamically by extracting the `X-GitHub-Event` header and dispatching to the specific generated Rust handler.
- [ ] **100+ Payload Structs Generation:** Map all `components.schemas.webhook-*` definitions directly into 100+ strictly typed Rust structs guaranteeing 100% schema parity.
- [ ] **Ping Event Auto-Responder:** Synthesize a default handler for the `ping` webhook event, automatically responding with the required `zen` payload success state.

**Phase 2.1.9: Rust Code Emission & Toolchain Integration**
- [ ] **Crate Directory Scaffolding:** Implement the compiler to dynamically generate a complete `Cargo.toml`, `src/lib.rs`, and modular `src/api/` directories.
- [ ] **`#![forbid(unsafe_code)]` Injection:** Hardcode the `#![forbid(unsafe_code)]` attribute at the top of every generated Rust module for absolute memory security.
- [ ] **Clippy Configuration Injection:** Auto-generate a `clippy.toml` file alongside the output configuring strict linting rules tailored to generated codebase constraints.
- [ ] **`cargo fmt` Invocation:** Shell out to `cargo fmt --edition 2021` programmatically to format the generated abstract syntax tree code strings into idiomatic Rust.
- [ ] **Integration Test Scaffolding:** Auto-generate `tests/` integration directories containing baseline `tokio::test` functions that instantiate the generated Router against a mock database.
- [ ] **Cargo Check Validation:** Execute `cargo check` against the generated output directory as the final compiler step, panicking the generator if the output fails to compile.

---

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
- [ ] **Current Page ARIA Sync:** Dynamically map Angular Router active states specifically to the `aria-current="page"` attribute on navigation tab links.