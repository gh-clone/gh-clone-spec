# Plan for gh-clone-openapi-compiler

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

