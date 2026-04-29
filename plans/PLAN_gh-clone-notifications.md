# Plan for gh-clone-notifications

### Tier 1: Watch States & Relational Schema
- [ ] Define the `repository_watches` Postgres table: `user_id`, `repo_id`, `state` (ignored, subscribed, custom).
- [ ] Define the `thread_subscriptions` Postgres table for explicit issue/PR follows: `user_id`, `thread_id`, `thread_type`, `subscribed` (boolean), `reason` (mention, author, comment, manual).
- [ ] Define the `notification_threads` Postgres table: `id`, `user_id`, `repository_id`, `subject_title`, `subject_url`, `subject_type` (Issue, PullRequest, Release, Discussion), `unread` (boolean), `reason`.
- [ ] Implement the "Custom Watch" DB mapping to allow users to subscribe strictly to Releases or Pull Requests while ignoring Issues on a per-repository basis.
- [ ] Build the "Participating" state heuristic: ensure any user who comments on an issue automatically receives a `thread_subscriptions` entry overriding their repository `ignored` state.
- [ ] Add an indexing strategy on `notification_threads(user_id, unread, updated_at DESC)` to drastically speed up the global inbox fetch query.

### Tier 2: The Routing & Dispatch Engine (Rust & Kafka)
- [ ] Build the `NotificationRouter` Rust worker that continuously consumes from the Kafka `repository_events` topic (Issues opened, PRs merged, etc.).
- [ ] Implement the fan-out algorithm: for a given event, query `repository_watches` to resolve all observing users, while explicitly filtering out users where `state = ignored`.
- [ ] Inject explicit `thread_subscriptions` checks into the fan-out loop to include participants or manually subscribed users.
- [ ] Parse Markdown AST dynamically during the dispatch to resolve all `@username` and `@team` mentions, inserting explicit `mention` reason notifications.
- [ ] Implement deduplication: ensure a user who is both a repository watcher and specifically `@mentioned` only receives a single notification record favoring the `mention` reason.
- [ ] Execute bulk `INSERT` statements into `notification_threads` using `sqlx` to minimize database transaction overhead during massive repository events.
- [ ] Emit the finalized notifications back to a Redis PubSub channel (`user:notifications:{id}`) for real-time WebSocket delivery.

### Tier 3: Angular Inbox UI & Real-Time Client
- [ ] Build the `/notifications` Angular Inbox view strictly adhering to the Primer CSS specific layout (left sidebar filters, main list).
- [ ] Implement Server-Sent Events (SSE) or WebSockets in the Angular frontend to listen to the Redis PubSub channel and dynamically insert new notifications without refresh.
- [ ] Implement exact GitHub keyboard shortcuts within the Angular app: `e` (Mark as done), `Shift+i` (Mark as read), `Shift+u` (Mark as unread), `j/k` (Move cursor).
- [ ] Build the grouped list view logic: dynamically group active notifications by Repository Name and Organization in the frontend rendering layer.
- [ ] Implement the "Done" state mutation: clicking "Done" executes a GraphQL mutation that archives the notification and instantly sweeps it from the UI using Angular Signals.
- [ ] Build the cross-tab synchronization feature utilizing `BroadcastChannel` to automatically sync the "Unread Count" badge across all open browser windows.
- [ ] Implement the "Save" feature, allowing users to move specific notifications into a dedicated "Saved for later" database table and Angular view.

### Tier 4: Inbound Email Parsing & Cryptography
- [ ] Deploy an edge Mail Transfer Agent (MTA) like Postfix configured to route all inbound emails matching `reply+*@reply.gh-clone.com` directly into a Rust ingestion API.
- [ ] Generate cryptographically secure `Reply-To` addresses using HMAC-SHA256: `reply+{base64(thread_id)}+{hmac(user_id + thread_id + secret)}@reply.gh-clone.com`.
- [ ] Build the Rust `mailparse` ingestion worker to extract the raw text and HTML body from the multipart MIME payload.
- [ ] Implement strict DKIM/SPF verification on the inbound email payload; silently drop any emails failing origin authentication to prevent comment spoofing.
- [ ] Build a regex-based stripper to remove email signatures, quoted historical reply text, and "Sent from my iPhone" footers, isolating the user's actual reply.
- [ ] Verify the HMAC signature in the `Reply-To` address string perfectly matches the sender's known `user_id` before processing.
- [ ] Insert the cleaned, extracted plaintext directly into the corresponding Issue/PR as a new comment attributed to the authenticating user.

### Tier 5: Email Rendering & Outbound Delivery
- [ ] Build the `OutboundEmailDispatcher` Rust worker that polls newly created notifications and triggers delivery based on the user's explicit email routing preferences.
- [ ] Integrate the `askama` Rust template engine to natively compile highly-optimized, GitHub-style HTML email templates securely without runtime overhead.
- [ ] Implement the specific Primer Email CSS subset, inlining all CSS directly into the HTML payload via a pre-processing compilation step.
- [ ] Inject standard `List-Unsubscribe` headers into the outbound SMTP payload to ensure compliance and prevent deliverability penalization.
- [ ] Implement the dynamic email `Subject` generator (e.g., `Re: [org/repo] Issue Title (#123)`).
- [ ] Ensure specific metadata headers (e.g., `Message-ID`, `In-Reply-To`, `References`) are precisely constructed so email clients (Gmail/Outlook) thread the notifications correctly.
- [ ] Connect the dispatcher to AWS SES or an internal SMTP relay utilizing `lettre`, enforcing a concurrent sending pool to handle millions of daily dispatches.
- [ ] Handle SMTP bounce notifications (Hard Bounces/Complaints) via webhooks to automatically disable email notifications for failing user addresses and flag their account.