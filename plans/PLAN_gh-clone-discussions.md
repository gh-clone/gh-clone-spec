# Plan for gh-clone-discussions

### Tier 1: GraphQL API & Core Data Model
- [ ] Define the Postgres table `discussions`: `id`, `repo_id`, `category_id`, `title`, `body_html`, `author_id`, `upvote_count`, `answer_id`.
- [ ] Define the Postgres table `discussion_comments`: `id`, `discussion_id`, `parent_id` (for threading), `author_id`, `body_html`, `upvote_count`.
- [ ] Implement strict GraphQL types matching the Relay specification: `DiscussionConnection`, `DiscussionEdge`, `DiscussionNode`.
- [ ] Define GraphQL mutations: `createDiscussion`, `updateDiscussion`, `deleteDiscussion`, `addDiscussionComment`, `markDiscussionCommentAsAnswer`.
- [ ] Map GraphQL `connection` cursors precisely to Postgres Keyset pagination (sorting by updated_at or upvotes) for high-performance deep fetching.
- [ ] Implement a highly concurrent Redis ZSET (Sorted Set) backend specifically for tracking and ranking Discussion Upvotes to prevent database locking under heavy engagement.
- [ ] Expose all Discussions functionality exclusively via GraphQL APIs (no legacy REST endpoints) to enforce modern data fetching patterns.
- [ ] Synchronize discussion creation and updates with the Elasticsearch/Blackbird indexer for global repository search.

### Tier 2: Threading, Q&A, and Categories
- [ ] Implement nested threaded comments logic, specifically limiting UI and database depth to exactly 1 level (parent comment -> child replies) matching GitHub's specific design constraint.
- [ ] Implement the `markDiscussionCommentAsAnswer` logic: automatically pinning the selected comment node to the top of the GraphQL response and applying the "Answered" badge.
- [ ] Create repository-level categories (e.g., "Q&A", "General", "Announcements") with customizable emoji icons, descriptions, and specific discussion formats.
- [ ] Build "Announcements" categories that enforce strict RBAC, restricting discussion creation exclusively to repository maintainers.
- [ ] Build the "Issue to Discussion" conversion tool: write a complex Postgres transaction to safely rewrite issue comments into discussion comments and redirect the original issue URL.
- [ ] Implement "Discussions Polls", adding specific database schemas for poll options, tracking user votes, and enforcing single-vote uniqueness.
- [ ] Support pinning discussions to the top of the repository index (enforcing a maximum limit, e.g., 4 pins per repo).

### Tier 3: UI Implementation & Moderation
- [ ] Build the Discussions landing page mirroring GitHub's dual-pane layout: categories sidebar on the left, discussion list main pane on the right.
- [ ] Implement the threaded comment UI utilizing `@angular/cdk/scrolling` infinite scrolling for extremely long Q&A threads (100+ replies).
- [ ] Build the upvote interaction with optimistic UI state updates (instant number increment) and transparent background error rollback.
- [ ] Integrate specific reaction emojis (👍, 🎉, 🚀, 👀) onto discussion threads and comments, distinct from the primary upvote counter.
- [ ] Implement a custom Markdown renderer extension in the frontend to support automatically embedding and expanding rich Discussion links via URL.
- [ ] Build the "Top Contributors" leaderboard component summarizing the total upvotes gathered by users within the specific repository context.
- [ ] Implement moderation tooling: allow maintainers to Lock discussions (preventing new comments), Transfer discussions across repositories, or hide comments as "Spam".
- [ ] Build the notification engine integration: automatically subscribe participants, `@mentioned` users, and the original author to new thread comments.

### Tier 4: Search Indexing & Anti-Abuse
- [ ] Implement a Kafka-driven CDC (Change Data Capture) pipeline synchronizing PostgreSQL `discussions` tables directly to Elasticsearch/OpenSearch.
- [ ] Enforce GraphQL query complexity analysis to prevent deeply nested comment fetching from executing denial-of-service (DoS) attacks on the database.
- [ ] Integrate with an external spam detection API (e.g., Akismet) to automatically flag suspicious discussions containing known malicious links.
- [ ] Implement a strict rate limit on Discussion creation (e.g., max 10 per hour per user) to mitigate automated bot spam.
- [ ] Instrument the Redis upvote ZSET operations with tracing to ensure cache latency remains under 2ms during viral community events.

### Tier 5: Organization-Level Discussions
- [ ] **Organization Discussions Schema:** Extend the `discussions` Postgres table to support a nullable `organization_id` distinct from the `repo_id`.
- [ ] **GraphQL API Extensions:** Add `Organization.discussions` and `Organization.discussionCategories` nodes to the GraphQL schema for fetching org-level threads.
- [ ] **Cross-Repo Linkage:** Implement logic allowing Organization Discussions to reference Issues or Pull Requests across any repository owned by the organization.
- [ ] **Organization Discussion Categories:** Build the admin UI for Organization Owners to define global discussion categories (e.g., "Company Announcements", "Engineering All-Hands").
- [ ] **RBAC for Org Discussions:** Implement strict access control, ensuring internal organization discussions are hidden from Outside Collaborators or public viewers depending on the org's visibility settings.
- [ ] **Org-wide Notifications:** Integrate the notification engine to optionally broadcast "Announcement" category posts to all members of the Organization automatically.
