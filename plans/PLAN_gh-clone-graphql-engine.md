# Plan for gh-clone-graphql-engine

### Tier 1: Schema Ingestion & Introspection (async-graphql)
- [ ] Initialize the core Rust GraphQL server utilizing `async-graphql` and `axum`.
- [ ] Implement a schema generator that parses the official `schema.graphql` (v4) to guarantee structurally identical Types, Enums, and Inputs.
- [ ] Configure `async-graphql` to automatically generate the Introspection Query API `__schema` matching GitHub's exact introspection output.
- [ ] Implement strict scalar mappings: map GraphQL `DateTime` strictly to `time::OffsetDateTime` and `URI` to `url::Url` in Rust.
- [ ] Build the `HTML` scalar type, explicitly enforcing that strings returned via this scalar have passed through the backend DOMPurify/HTML sanitization layer.
- [ ] Map GitHub's specialized `GitObjectID` and `GitTimestamp` scalars to native `gitoxide` or `libgit2` object types.
- [ ] Implement schema stiching or modularization: break the monolithic schema down into domains (e.g., `IssueQuery`, `RepositoryQuery`) using `async-graphql`'s `MergedObject`.

### Tier 2: Relay Compliant Node & Global Identification
- [ ] Implement the `Node` interface strictly requiring an `id: ID!` field for all global entities (User, Repository, Issue, PullRequest).
- [ ] Build the Global ID encoder/decoder in Rust: generate Base64 strings formatted as `010:Repository1296269` merging the entity type and the internal Postgres BigInt ID.
- [ ] Implement the top-level `node(id: ID!): Node` query: parse the Base64 ID, match the entity type, and dynamically route the fetch to the correct database service.
- [ ] Implement the top-level `nodes(ids: [ID!]!): [Node]!` query, enforcing batch fetching across multiple distinct entity tables simultaneously.
- [ ] Create Rust traits enforcing that all Database models must explicitly implement a `to_global_id()` method prior to serialization.

### Tier 3: Connections, Edges & Cursor Pagination
- [ ] Implement Relay-compliant `Connection` types for all array responses (e.g., `IssueConnection`, `StargazerConnection`).
- [ ] Enforce the presence of `edges { node, cursor }`, `nodes`, and `pageInfo { hasNextPage, endCursor, ... }` strictly on all Connection objects.
- [ ] Implement Base64 cursor encoding: encode Postgres keyset pagination parameters (e.g., `cursor:v2:timestamp:id`) securely without leaking internal SQL structures.
- [ ] Build the Connection Argument validator: strictly enforce that clients provide either `first` or `last` arguments, returning a generic error if both or neither are supplied.
- [ ] Implement maximum node constraints: restrict `first` and `last` to a maximum of 100 elements per query block to prevent database DoS.
- [ ] Build a generic Rust macro or utility function `paginate_query!` that automatically translates GraphQL connection arguments into an optimized `ORDER BY ... LIMIT ... OFFSET ...` or Keyset SQL query.

### Tier 4: DataLoaders & N+1 Query Elimination
- [ ] Integrate the `dataloader` Rust crate to intercept and batch nested GraphQL resolvers across the entire query tree.
- [ ] Implement the `UserLoader`: batch resolve User objects from Postgres `users` table by ID when querying `Issue.author` across 100 distinct issues.
- [ ] Implement the `RepositoryLoader`: batch resolve Repository objects to prevent N+1 queries when fetching `Organization.repositories`.
- [ ] Implement the `GitTreeLoader`: batch resolve Git blobs and trees by grouping requested OIDs and querying the `Spokes` backend via a single gRPC multiplexed call.
- [ ] Configure DataLoader caching to persist strictly for the duration of a single HTTP request context to ensure strong consistency without cross-request state leakage.
- [ ] Bind DataLoaders dynamically to the `async-graphql` Context (`ctx.data::<UserLoader>()`) inside the Axum middleware pipeline.

### Tier 5: Query Cost Analysis & Rate Limiting
- [ ] Implement an AST pre-flight analysis phase: before executing the query, traverse the parsed GraphQL AST to calculate the theoretical query cost.
- [ ] Map node weights: assign 1 point for single objects (`User`, `Repository`) and multiplier points for connections (e.g., `first: 100` = 100 points).
- [ ] Implement the 5,000 point maximum: reject queries that exceed the calculated maximum node threshold with a specific GraphQL error payload `MAX_NODE_LIMIT_EXCEEDED`.
- [ ] Integrate the AST cost result directly with the Redis Token Bucket rate limiter, deducting points based on query complexity rather than mere HTTP request counts.
- [ ] Expose the `rateLimit { cost, limit, remaining, resetAt }` object dynamically in the GraphQL response payload so clients can track usage.
- [ ] Implement query depth limitation: reject any GraphQL query that nests deeper than 10 levels to prevent stack exhaustion attacks.

### Tier 6: Mutations & Standardized Payloads
- [ ] Define standardized Mutation payloads: ensure every mutation returns a distinct payload type (e.g., `AddStarPayload`) containing a `clientMutationId`.
- [ ] Enforce Input Object single-arguments: every mutation must accept a single `input` argument strictly typed to an Input object (e.g., `AddStarInput!`).
- [ ] Return the mutated subject explicitly: `AddStarPayload` must return the updated `starrable` interface node so Apollo/Relay clients can automatically normalize their local cache.
- [ ] Ensure all mutation payloads include a standard `errors` array alongside the Top-Level GraphQL errors, mapping database constraints to friendly UI text.
- [ ] Implement transactional integrity: ensure mutations that modify multiple entities (like `MergePullRequest`) execute entirely within a single Postgres `BEGIN ... COMMIT` block.
- [ ] Build specific input validators inside Rust (e.g., string length checks, regex formatting) executing *before* the Postgres transaction opens.

### Tier 7: Telemetry, Profiling & Deprecation
- [ ] Implement Apollo Tracing extensions: optionally include execution timing data for every resolver in the `extensions` block of the response for internal admins.
- [ ] Inject OpenTelemetry spans strictly around the `async-graphql` execution phase, logging the raw query string and extracted operation name.
- [ ] Add `#[graphql(deprecation = "message")]` macros to legacy fields, matching GitHub's rolling API deprecation schedules.
- [ ] Log telemetry explicitly when a client utilizes a deprecated field, aggregating metrics to safely drop the field in future API versions.