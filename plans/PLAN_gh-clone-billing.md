# Plan for gh-clone-billing

### Tier 1: Stripe Integration & Subscription Modeling
- [ ] Integrate the official `stripe-rust` SDK into the core backend services.
- [ ] Establish exact mapping logic: map GitHub Organizations to Stripe `Customer` objects, and map individual GitHub Users to Stripe `Customer` objects.
- [ ] Implement checkout sessions utilizing Stripe Checkout for seamless upgrades to `GitHub Pro` or `GitHub Team` tiers.
- [ ] Define internal Subscription Models mapping directly to Stripe Products (Free, Team, Enterprise) and their associated metadata.
- [ ] Implement the core Stripe Webhook endpoint `POST /webhooks/stripe`, rigorously validating the `Stripe-Signature` header to prevent spoofing.
- [ ] Handle `invoice.payment_succeeded` webhook payloads to instantly reset billing cycle constraints and unlock tenant access.
- [ ] Handle `customer.subscription.deleted` or `invoice.payment_failed` payloads to systematically lock organizations and downgrade users to the Free tier.
- [ ] Build logic to automatically generate and dispatch PDF receipts and invoices based on Stripe's payload data to the organization's billing email.
- [ ] Implement idempotent webhook processing using a Postgres `processed_webhooks` table, ensuring duplicate Stripe network events do not overcharge users.

### Tier 2: Metering, Quotas & Usage Analytics
- [ ] Implement the `metering_events` Postgres table to log highly granular usage events (e.g., `actions_minutes_linux`, `actions_minutes_macos`, `lfs_bytes_stored`, `packages_egress_bytes`).
- [ ] Build the Action Runners meter: tally `runs_duration_ms` at the exact moment a runner job completes and insert it into the metering table.
- [ ] Build the Storage meter: execute a daily cron job that queries the S3 bucket sizes for Git LFS usage and updates the user's quota cache.
- [ ] Build a background synchronization worker that aggregates un-billed `metering_events` daily and dispatches them to the Stripe Usage-Based Billing API (`v1/subscription_items/{id}/usage_records`).
- [ ] Implement strict pre-flight quota checks: instantly block `git push` (LFS) or Actions job dispatch if the current billing cycle's usage has exceeded the Stripe subscription limit.
- [ ] Design the UI dashboard for Billing Settings, rendering usage charts for Storage, Actions, and Packages, predicting the month's final bill.

### Tier 3: Enterprise Identity (SAML/SCIM)
- [ ] Implement SAML 2.0 Identity Provider (IdP) integration settings for Enterprise organizations (IdP SSO URL, Issuer, X.509 Public Certificate).
- [ ] Build the SAML Assertion Consumer Service (ACS) endpoint in Rust, verifying XML cryptographic signatures via the `xmlsec` library.
- [ ] Enforce "SSO Authorized" states on OAuth/PAT tokens: explicitly require users to click "Authorize" via SSO before their tokens can access enterprise-protected repositories.
- [ ] Implement the SCIM 2.0 Server API strictly conforming to RFC 7643/7644 (`GET /scim/v2/Users`, `POST`, `PATCH`, `DELETE`).
- [ ] Wire SCIM `DELETE` payloads to automatically suspend users or forcefully remove them from the organization in real-time (integrating directly with Okta/Entra ID).
- [ ] Handle group synchronization via SCIM (`/scim/v2/Groups`) to automatically map external IdP groups dynamically to internal GitHub Teams.

### Tier 4: GitHub Sponsors & Payouts
- [ ] Build the Stripe Connect custom account integration to allow open-source developers to securely onboard, verify identity (KYC), and receive payouts.
- [ ] Implement the Sponsors Tier configuration UI (defining monthly recurring vs one-time amounts, custom perks, and welcome messages).
- [ ] Build the `sponsor` GraphQL mutation to process a payment method and cryptographically link a Sponsor user to a Sponsored Developer.
- [ ] Render the "Sponsor" button dynamically on repository headers if a `.github/FUNDING.yml` file is present in the default branch.
- [ ] Implement a specific internal webhooks engine to notify sponsored developers instantly when a new sponsorship is created, edited, or cancelled.
- [ ] Build the Sponsor Dashboard, highlighting recurring monthly revenue (MRR), one-time revenue analytics, and exporting sponsor CSVs.
- [ ] Implement the "GitHub Sponsors Matching Fund" internal logic to temporarily multiply specific sponsorship tiers during promotional periods.

### Tier 5: Financial Compliance & Resilience
- [ ] Implement the Circuit Breaker pattern on all outbound calls to the Stripe API to gracefully degrade UI functionality during Stripe outages.
- [ ] Build a daily automated reconciliation cron job that compares the internal `metering_events` totals against the actual usage billed in Stripe.
- [ ] Enforce strict SOC2 compliant audit logging for any administrative mutation of a user's billing tier or manual invoice adjustment.
- [ ] Utilize distributed tracing (Jaeger/OpenTelemetry) to track the full lifecycle of a Stripe Webhook from ingress to database commit.
- [ ] Implement an alerting system (PagerDuty) triggered instantly if the Stripe webhook signature validation failure rate exceeds 1%.

### Tier 6: GitHub Apps Marketplace Billing & Revenue Sharing
- [ ] **Marketplace Subscriptions Schema:** Define the `marketplace_subscriptions` Postgres table tracking active user and organization subscriptions strictly tied to third-party Apps.
- [ ] **Stripe Connect Integration:** Implement Stripe Connect routing in Rust to securely and automatically split recurring revenue between the platform (Replica) and the third-party App developer.
- [ ] **Marketplace Checkout Webhooks:** Handle Stripe `checkout.session.completed` webhooks specifically for Marketplace App purchases, automatically triggering the App Installation REST workflows.
- [ ] **Consolidated Invoicing:** Architect the billing engine to generate consolidated monthly invoices that combine the platform's base cost (e.g., Enterprise seats) with all third-party App subscriptions.
- [ ] **Proration Logic:** Implement precise proration algorithms for mid-cycle App upgrades, downgrades, or seat expansions to ensure accurate chargebacks.
- [ ] **Subscription Cancellation Flow:** Implement the cancellation API, ensuring the third-party App receives a `marketplace_purchase.cancelled` webhook immediately to revoke external access.
- [ ] **Developer Financial Dashboard:** Build the specific Stripe Connect financial dashboard for App Developers to monitor active subscriptions, MRR, and upcoming payout schedules.

### Tier 6.1: Stripe Connect & Financial Reconciliation (Deep Dive)
- [ ] **Stripe Custom Connect Onboarding:** Implement the internal API routes to generate Stripe Account Links, routing App Developers through the Stripe hosted onboarding flow to gather KYC/KYB compliance data.
- [ ] **Platform Fee Calculation:** Build a precise Rust math engine utilizing `rust_decimal` to calculate the platform revenue split (e.g., 5% fee) to prevent floating-point rounding errors during financial transactions.
- [ ] **Tax/VAT Automation via Stripe Tax:** Integrate the Stripe Tax API payload securely within checkout sessions, dynamically computing European VAT or US State Sales Tax based on the purchaser's billing address.
- [ ] **App Proration Engine:** Implement complex upgrade/downgrade handlers: if an organization switches from an App's "Pro" ($10/mo) to "Enterprise" ($50/mo) mid-cycle, calculate the exact unused time credits and apply them to the next invoice line item.
- [ ] **Webhook Idempotency Layer:** Utilize Redis to store Stripe Webhook Event IDs (`evt_xxx`) with a 7-day TTL, strictly ignoring duplicate webhook deliveries that could erroneously provision extra licenses.
- [ ] **Automated Chargeback Suspension:** Intercept Stripe `charge.dispute.created` webhooks; instantly suspend the associated Marketplace App installation and notify both the purchaser and the developer.
- [ ] **Payout Schedule Configuration:** Expose an administrative GraphQL mutation allowing platform operators to define payout delays (e.g., T+30 days) before transferring collected funds to App Developer Stripe accounts.
- [ ] **Enterprise Consolidated Invoicing:** Architect the monthly invoice generation cron job: fetch all base Enterprise seat usage, add Actions minute overages, and append all active Marketplace App subscriptions onto a single unified Stripe Invoice.
- [ ] **Branded PDF Receipts Generation:** Configure Stripe to utilize platform branding, but build an internal fallback Rust endpoint to generate custom, line-itemized PDF invoices utilizing `printpdf` for complex enterprise accounting requirements.
- [ ] **Refund APIs & Audit Logging:** Implement an internal API endpoint allowing App Developers to trigger full or partial refunds for their apps, generating an immutable audit log row tagged with the requesting developer's ID.
