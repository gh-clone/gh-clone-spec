# Plan for gh-clone-docs

### Tier 1: SSG Core, Routing & Offline Resilience (AnalogJS/Angular)
- [ ] Scaffold the `/docs` portal utilizing AnalogJS (or Angular SSR) to support robust Static Site Generation (SSG) for thousands of markdown pages.
- [ ] Implement file-system-based routing in the frontend to dynamically map the `content/` directory structure directly to URL paths (e.g., `content/actions/index.md` -> `/docs/actions`).
- [ ] Configure the Angular application to support localized route prefixes (e.g., `/en/actions`, `/ja/actions`) aligned with translation workflows.
- [ ] Build a robust hierarchical sidebar navigation component that automatically reads the directory structure and frontmatter `weight` attributes to generate nested menus.
- [ ] Implement the "On this page" right-hand Table of Contents (ToC) component, utilizing `IntersectionObserver` to highlight the active heading as the user scrolls.
- [ ] Design the layout strictly using the Primer CSS framework to ensure visual consistency between the main application and the documentation portal.
- [ ] Implement global dark/light theme toggles specifically for the Docs portal, persisting the preference to `localStorage` and synchronizing with OS settings.
- [ ] Build automated integration testing specifically asserting that no top-level documentation sections throw 404s after the SSG build phase completes.
- [ ] Configure `@angular/service-worker` to aggressively cache top-level documentation sections, enabling robust offline reading capabilities.
- [ ] Implement hover-based preloading: when a user's cursor rests on a sidebar navigation link for >200ms, prefetch the underlying HTML/JSON payload.
- [ ] Build a Breadcrumb generation service that traverses the active route tree to render accessible, schema-compliant `BreadcrumbList` JSON-LD data.
- [ ] Implement specific layout templates for Hub Pages (e.g., `/docs/actions`), containing expanded card grids and category overviews rather than standard article formatting.

### Tier 2: Advanced Rust Markdown Compiler (`pulldown-cmark`)
- [ ] Architect the backend compilation pipeline in Rust utilizing `pulldown-cmark` to parse complex GitHub Flavored Markdown (GFM) into an Abstract Syntax Tree (AST).
- [ ] Implement a custom AST visitor to inject deep-linking anchor tags (`<a id="...">`) into every generated HTML heading element (`h1` - `h6`).
- [ ] Build a custom Markdown extension specifically for documentation "Callouts" (Notes, Warnings, Tips), mapping them to Primer CSS Alert components.
- [ ] Implement dynamic Liquid-style variable interpolation within the Markdown parsing step (e.g., rendering `{{ site.version }}` dynamically into the HTML output).
- [ ] Write a Rust plugin to automatically syntax highlight code blocks at build-time using `syntect` to avoid shipping heavy JavaScript syntax highlighters to the client.
- [ ] Add Mermaid.js support: parse ```mermaid``` codeblocks, extracting the content and wrapping it in specialized DOM nodes for the Angular frontend to lazily render.
- [ ] Implement KaTeX integration in Rust to parse and render complex mathematical equations (`$$...$$`) into pure HTML/CSS strings at build-time.
- [ ] Build an automated broken-link checker in Rust that traverses the entire Markdown content directory and validates all relative internal `href` and `src` attributes.
- [ ] Implement an automated "Edit this page on GitHub" link generator, dynamically appending the current page's Git path to the bottom of the article.
- [ ] Store the compiled HTML and extracted metadata (Title, Description, ToC) in Postgres or Redis to serve to the Angular SSR engine with sub-millisecond latency.

### Tier 3: Rich Interactive Components & Context Toggles
- [ ] Implement the "Product Version" dropdown (Free/Pro/Team, Enterprise Cloud, Enterprise Server 3.x), which modifies the URL scope (e.g., `/enterprise-server@3.11/actions`).
- [ ] Build a context-aware rendering engine: allow Markdown authors to wrap content in conditional blocks (e.g., `if: enterprise-server`) which the Rust backend evaluates based on the URL context.
- [ ] Implement the "Platform Picker" toggles (Mac, Windows, Linux) across documentation pages, persisting the user's OS preference globally via cookies to dynamically filter code snippets.
- [ ] Build custom Markdown directives for nested Tabbed content (e.g., selecting between `npm`, `yarn`, or `pnpm` instructions) using native Angular state management.
- [ ] Support explicit REST API versioning dropdowns (e.g., `2022-11-28` vs legacy versions) ensuring the documentation strictly matches the selected API schema.
- [ ] Build the Tool Picker toggles (Web UI, GitHub CLI, Desktop, Curl), dynamically switching the visible instruction sets and code blocks across the entire page based on the selected tool.
- [ ] Handle explicit deprecation banners based on version frontmatter, warning users dynamically if they are viewing documentation for an unsupported Enterprise release.
- [ ] Implement the "Copy Code Block" Angular directive that attaches a native copy-to-clipboard button strictly over hovered `<pre>` elements.
- [ ] Build interactive tooltip popovers over specific glossary terms (e.g., hovering over "Commit" shows a short definition bubble).

### Tier 4: OpenAPI 3.1 & REST Reference Generation
- [ ] Ingest the generated `api.github.com.yaml` OpenAPI specification into a specialized Rust processor strictly designed to generate API reference documentation.
- [ ] Dynamically generate the REST API navigation tree based on OpenAPI tags (e.g., `Repos`, `Issues`, `Pulls`).
- [ ] Render complex OpenAPI schemas into highly readable, nested HTML tables for Path Parameters, Query Parameters, and JSON Body schemas.
- [ ] Parse and generate specific documentation for OpenAPI `callbacks` and `webhooks`, documenting the exact payload structures emitted by the platform.
- [ ] Implement automated SDK code snippet generation: parse the OpenAPI spec to generate accurate example payloads for Rust, Go, Python, and TypeScript SDKs dynamically.
- [ ] Implement the "Status Codes" response viewer, rendering example JSON payloads for `200 OK`, `404 Not Found`, and `422 Unprocessable Entity` exactly as defined in the spec.
- [ ] Inject dynamic rate limit documentation derived from the OpenAPI extension fields (`x-github-ratelimit`) directly into the endpoint descriptions.
- [ ] Render OAuth permission scope requirements precisely based on the endpoint's `security` definitions.
- [ ] Automatically generate deep-links linking referenced schema models (e.g., `PullRequestSimple`) directly to their definitions in the schema glossary.

### Tier 5: Interactive GraphQL Explorer (GraphiQL + Rust)
- [ ] Build an Angular wrapper around a native GraphQL IDE component (e.g., porting GraphiQL to Angular or using Monaco Editor with GraphQL language server capabilities).
- [ ] Connect the GraphiQL Explorer directly to the live Rust `async-graphql` backend endpoint.
- [ ] Configure the GraphQL introspection client to dynamically fetch the schema and populate the integrated documentation explorer sidebar.
- [ ] Build the "Query History" local storage cache, ensuring developers do not lose their drafted GraphQL queries upon page refresh.
- [ ] Implement one-click "Copy as cURL" and "Copy as JavaScript" utilities directly within the GraphiQL Explorer interface.
- [ ] Pre-populate the Explorer with a dropdown of "Example Queries" (e.g., "Fetch top 10 repos", "Get issue comments") fetched from the Rust backend.
- [ ] Implement a custom HTTP Headers injection UI specifically for the GraphiQL explorer, allowing developers to manually set `Accept` or custom preview headers.
- [ ] Build WebSocket support into the GraphQL Explorer to allow developers to natively test and listen to GraphQL Subscriptions.
- [ ] Implement automated GraphQL schema diffing: generate visual changelogs highlighting exactly which fields and mutations were added/removed between API versions.

### Tier 6: Authenticated Developer Context
- [ ] Build the isolated "Sign in to see your repositories" OAuth flow specifically sandboxed for the documentation portal to prevent session leakage.
- [ ] Implement the "Auth Injection" mechanism: seamlessly pass the current user's HTTP session cookie or OAuth token to the GraphiQL Explorer to enable live data querying.
- [ ] Build the "Run in Browser" functionality: embed a secure HTTP client interface utilizing the native browser Fetch API to allow authenticated users to test REST endpoints directly against the live API from the documentation.
- [ ] Implement an automated Personal Access Token (PAT) generator flow integrated into the Docs UI, scoping the token strictly for the "Run in Browser" execution.
- [ ] Scrutinize payload validation in the API runner, providing real-time feedback if the user's manual JSON body fails the OpenAPI schema constraints.
- [ ] Scope API examples dynamically to the authenticated user's context (e.g., if signed in, replace `<YOUR_USERNAME>` with their actual handle in code snippets).
- [ ] Dynamically warn authenticated users if their active PAT lacks the required scopes necessary to execute the currently viewed REST endpoint.

### Tier 7: Semantic Search & Vector Discovery
- [ ] Implement the search ingestion pipeline in Rust: tokenize all compiled Markdown pages and synchronize the index to Elasticsearch/Algolia.
- [ ] Implement vector (semantic) search utilizing `pgvector` in PostgreSQL: embed documentation chunks using an ML model to answer natural language queries (e.g., "how to fix merge conflicts").
- [ ] Build the global `/docs` search bar in Angular, implementing debounced keystrokes and rendering rich auto-complete results categorized by product area.
- [ ] Enforce focus management: implement the `Cmd+K` / `Ctrl+K` global keyboard shortcut to instantly focus the search modal from any documentation page.
- [ ] Implement "Typo Tolerance" and "Synonym Matching" in the search backend (e.g., mapping "webhook" to "web hooks", "pr" to "pull request").
- [ ] Build contextual search scoping: if a user is inside the `/actions` directory, default the search filter to prioritize GitHub Actions documentation results.
- [ ] Highlight the exact search term matches dynamically within the Angular search result dropdown preview snippets.

### Tier 8: Feedback, Analytics & Open Source Contributions
- [ ] Build the "Was this page helpful?" feedback widget at the bottom of every documentation article, utilizing a simple Thumbs Up / Thumbs Down interface.
- [ ] Implement the "Provide detailed feedback" modal, securely piping user text responses into internal tracking systems (Jira/GitHub Issues).
- [ ] Plumb the feedback widget payloads directly to the internal Data Warehouse (TimescaleDB) to track documentation quality metrics and flag outdated articles.
- [ ] Expose an administrative dashboard for the Docs team to analyze "Zero Result Searches" (search terms users tried that yielded no results) to guide future content creation.
- [ ] Build a seamless "Open a PR to fix this doc" workflow: pre-fill a Pull Request against the `docs` repository modifying the exact markdown file the user was viewing.
- [ ] Implement an internal translation synchronization pipeline in Rust: detect changes to English markdown source files and automatically dispatch payloads to CrowdIn or Transifex for localization.
- [ ] Build automated markdown linter CI checks (e.g., `markdownlint`, `vale`) enforcing corporate style guides and inclusive language across all documentation files.
