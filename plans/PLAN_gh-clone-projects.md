# Plan for gh-clone-projects (Memex)

### Tier 1: CRDT Engine & Synchronization Pipeline
- [ ] Implement the core operational transformation / CRDT engine utilizing `yjs` to guarantee conflict-free state resolution across concurrent clients.
- [ ] Define the `memex_projects` Postgres table, strictly using `JSONB` columns to store the binary-encoded `Y.Doc` state natively.
- [ ] Implement a custom `y-websocket` backend server in Rust (`tokio-tungstenite`) to handle high-frequency WebSocket connections.
- [ ] Build the WebSocket handshake authentication phase, validating session JWTs or PATs before upgrading the HTTP connection.
- [ ] Implement the `y-protocols/awareness` specification to broadcast real-time user presence (mouse cursors, active cell selection) without persisting to the document.
- [ ] Configure a Redis PubSub backplane to multiplex CRDT update binaries across multiple backend instances for horizontal scaling.
- [ ] Implement an asynchronous `tokio` background worker that runs `Y.encodeStateAsUpdate` hourly to compact the CRDT history and prevent unbounded memory growth.
- [ ] Implement a strict validation layer on the server to prevent malicious clients from injecting arbitrary properties into the `Y.Doc` map.
- [ ] Design an Event Sourcing log (stored in Postgres) to track non-collaborative metadata (e.g., Project renaming, deletion, visibility changes).
- [ ] Configure an `IndexedDB` caching strategy on the frontend via `y-indexeddb` to allow full offline editing capabilities.
- [ ] Implement automatic sync-reconciliation logic to flush IndexedDB mutations to the server immediately upon `navigator.onLine` restoration.
- [ ] Expose an administrative GraphQL mutation to forcibly rollback a project CRDT state to a specific historical timestamp.
- [ ] Build binary diffing logic to only transmit the `stateVector` differences between the client and server during initial page load.

### Tier 2: Frontend Virtualization & Data Grid
- [ ] Instantiate `react-window` (or `@tanstack/react-virtual`) to render grids containing up to 50,000 items without DOM degradation.
- [ ] Implement dynamic dynamic column width recalculation utilizing ResizeObserver to ensure smooth dragging interactions.
- [ ] Build the "Single Select" cell type renderer, persisting custom color codes and dropdown options as `Y.Map` configurations.
- [ ] Build the "Iteration" cell type renderer, calculating automated sprint durations (e.g., 2 weeks) and rolling over incomplete items.
- [ ] Implement precise keyboard navigation strictly matching Excel semantics (Arrow keys, `F2` to edit, `Esc` to cancel, `Enter` to save).
- [ ] Implement a robust cross-cell Copy/Paste clipboard integration parsing tab-separated values (TSV) directly into CRDT mutations.
- [ ] Build the "Group By" rendering engine, dynamically inserting collapsible virtual rows that compute real-time sum/count aggregates.
- [ ] Implement "Filter by" logic building an Abstract Syntax Tree (AST) for the custom query language (e.g., `status:done -assignee:@me`).
- [ ] Implement column visibility toggles and strict ordering logic, saving these preferences to a dedicated `Y.Array` within the layout settings.
- [ ] Build the "Item Title" renderer, which queries the Relay store to dynamically display the linked Issue/PR's current state icon (Open/Closed/Merged).

### Tier 3: Kanban Board & Roadmap Visualizations
- [ ] Implement the Kanban Board layout deriving its columns strictly from a user-selected "Single Select" or "Iteration" field.
- [ ] Integrate `@hello-pangea/dnd` for smooth, accessible Drag-and-Drop item transitions between Kanban columns.
- [ ] Implement the background mutation logic: dropping an item in a new column instantly applies the corresponding CRDT update to the item's field value.
- [ ] Build the Roadmap (Gantt) visualization, mapping horizontal time blocks to specific "Date" fields on project items.
- [ ] Implement an SVG or Canvas overlay to draw dependency arrows between items in the Roadmap view.
- [ ] Add draggable resizing handles to the left/right edges of Roadmap items to visually mutate their start and end dates.
- [ ] Implement a semantic zooming scale (Days, Weeks, Months, Quarters) that dynamically recalculates the CSS Grid template columns for the Roadmap.
- [ ] Implement the "Save View" abstraction, allowing users to persist the current combination of Layout, Filters, and Sorting into a named tab (e.g., "Sprint 42 Board").
- [ ] Design a specific "Archive" state for items, removing them from the active CRDT document and storing them in cold Postgres storage to optimize client load times.

### Tier 4: Automations, Workflows & Graph APIs
- [ ] Implement the GitHub GraphQL API extensions, specifically defining `ProjectV2`, `ProjectV2Item`, and `ProjectV2Field` nodes.
- [ ] Implement the `addProjectV2ItemById` GraphQL mutation, handling the complex translation of a repository Issue into a Memex item.
- [ ] Build a distributed Workflow Rule Engine that listens to Kafka events (`issue_closed`, `pr_merged`) to trigger Project mutations.
- [ ] Implement default workflow templates (e.g., "Set item status to 'Done' when pull request is merged").
- [ ] Expose an API for GitHub Actions to arbitrarily query and modify project item fields via the `gh project` CLI.
- [ ] Implement automatic sync for title and assignee changes: if an issue is renamed on the repository, the linked Memex item must reflect the change instantly without page reload.
- [ ] Build the Insights charts generator, parsing historical CRDT snapshots to render Burn-up, Burn-down, and Velocity charts using `recharts` or `d3.js`.
- [ ] Implement Organization-level RBAC (Role-Based Access Control) for projects, differentiating between Project Admin, Write, and Read roles independent of repository access.

### Tier 5: Enterprise Scaling & Observability
- [ ] Instrument `yjs` CRDT merge operations with OpenTelemetry spans to monitor P99 resolution latency.
- [ ] Implement strict WebSocket connection rate limits (e.g., max 5 concurrent connections per user) to prevent resource exhaustion.
- [ ] Configure Prometheus metrics to track active WebSocket connections, CRDT document sizes, and sync conflict rates.
- [ ] Enforce PostgreSQL Snapshot Isolation levels when performing background CRDT compaction to avoid corrupting live sessions.
- [ ] Implement an automated Data Loss Prevention (DLP) scanner over Project custom fields to detect inadvertently pasted secrets.
