# Plan for gh-clone-authz

### Tier 1: Identity & The Zanzibar Graph (GitHub AuthZ)
- [ ] Define protobuf schema for `RelationTuple` (`namespace`, `object_id`, `relation`, `subject_namespace`, `subject_id`).
- [ ] Create Postgres migrations for the master tuple store using composite primary keys to guarantee uniqueness and enable prefix scanning.
- [ ] Implement Spanner-style read-write transactions using snapshot isolation to guarantee linearizability for tuple inserts/deletes.
- [ ] Define the Namespace Configuration Language (NCL) parser to load and compile AuthZ rules (e.g., `viewer = owner + team_member`).
- [ ] Implement the `Check` API gRPC endpoint to evaluate binary authorization questions (`user:X can push to repo:Y?`).
- [ ] Implement the `Read` API for fetching direct relationships and their corresponding consistency tokens.
- [ ] Implement the `Expand` API to return a structural tree of all paths proving (or disproving) access.
- [ ] Build the `Check` Engine's concurrent Breadth-First Search (BFS) graph traversal mechanism.
- [ ] Implement bounded-depth graph cycle detection in the BFS traversal algorithm to prevent infinite loops on recursive group structures.
- [ ] Create a multi-tier caching strategy using L1 memory (local to AuthZ node) and L2 Redis (distributed edge cache).
- [ ] Encode "Zookies" (consistency tokens) as base64url encoded protobufs containing the Hybrid Logical Clock (HLC) timestamp.
- [ ] Implement Zookie validation logic: determine if a read can safely hit the cache or if it must query the master database to respect read-after-write.
- [ ] Implement the Leopard index mapping subjects to bitsets of all directly and indirectly accessible objects.
- [ ] Implement SIMD-optimized bitset intersections for fast `ListObjects` queries.
- [ ] Deploy a Debezium connector on Postgres to capture tuple mutations via logical decoding.
- [ ] Route Debezium Change Data Capture (CDC) events into a Kafka `acl-events` stream.
- [ ] Build a Rust cache-invalidation worker consuming `acl-events` to selectively drop or update Redis keys.
- [ ] Implement request hedging (sending duplicated read requests to multiple replicas after a short timeout) to guarantee 99th percentile latency targets (<1ms).
- [ ] Implement a distributed rate-limiter based on the Token Bucket algorithm specifically for the AuthZ API tier.


### Tier 2: CODEOWNERS Enforcement
- [ ] Implement a highly optimized Rust parser for `.github/CODEOWNERS` files, accurately processing standard gitignore-style path globbing.
- [ ] Build the resolution engine: dynamically map extracted paths (e.g., `src/**/*.rs`) to corresponding users (`@samuel`) or teams (`@org/backend`).
- [ ] Integrate the CODEOWNERS resolution step into the Pull Request state machine: when a PR is created or synchronized, automatically calculate the diff paths and assign the required reviewers.
- [ ] Translate CODEOWNERS rules into strict Branch Protection checks, blocking the "Merge" button unless an explicit approval from the exact mapped Owner exists.
- [ ] Implement team expansion logic: if a CODEOWNER is a team, recursively query the Zanzibar Graph to identify all valid members and track any single member's approval as fulfilling the team requirement.

### Tier 3: Custom Roles & RBAC Expansion
- [ ] Define the `custom_repository_roles` and `custom_organization_roles` Postgres schema, extending the default Read/Triage/Write/Maintain/Admin hierarchy.
- [ ] Build the Angular UI allowing Organization Administrators to define custom roles by selecting from a granular list of >40 specific permission bits (e.g., `bypass_branch_protection`, `manage_webhooks`, `delete_issues`).
- [ ] Compile the user-defined custom roles down into the NCL (Namespace Configuration Language) schema representing the exact structural permissions within the Zanzibar graph.
- [ ] Modify the Authorization middleware (`RequireScope`) to dynamically evaluate the expanded permission bitmasks in real-time instead of checking static string roles.
- [ ] Ensure custom role mutations (e.g., changing a role's permissions) instantly trigger a re-compilation of the associated ACLs and flush the distributed Redis authorization cache.
