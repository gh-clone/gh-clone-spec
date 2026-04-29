# Plan for gh-clone-marketing

### Tier 1: Edge, SSR & Core Web Vitals (Angular 21 + Rust)
- [ ] Scaffold the marketing frontend utilizing Angular 21 with Zoneless Change Detection (`provideExperimentalZonelessChangeDetection`) to eliminate `zone.js` overhead.
- [ ] Implement `@angular/ssr` to ensure all public-facing marketing pages are fully pre-rendered for instant Time-to-First-Byte (TTFB).
- [ ] Implement `TransferState` to pass initial Rust API response payloads securely from the Node.js SSR context to the client browser, achieving Zero-CLS hydration.
- [ ] Configure strict Content Security Policy (CSP) headers at the edge router explicitly for marketing domains, minimizing permitted external script executions.
- [ ] Build a native XML Sitemap generator in Rust that crawls active product pages, dynamic blog posts, and changelogs to produce `/sitemap.xml`.
- [ ] Implement a `robots.txt` dynamic generator to manage search engine crawling permissions, specifically separating marketing domains from application subdomains.
- [ ] Enforce W3C accessibility compliance across all marketing templates, ensuring semantic HTML5 landmarks and ARIA roles for screen reader navigation.
- [ ] Inject dynamic JSON-LD structured data (e.g., `Organization`, `SoftwareApplication`, `BreadcrumbList`) into the Angular `<head>` via the `Meta` service to optimize SERP display.
- [ ] Configure Brotli (`.br`) and Zstandard (`.zst`) compression at the edge proxy for all pre-compiled Angular assets to minimize bundle payload size.
- [ ] Implement DNS prefetching and `preconnect` directives for critical external domains (e.g., CDN, Stripe) to reduce TLS handshake latencies during navigation.
- [ ] Utilize Angular's `NgOptimizedImage` directive comprehensively to enforce lazy loading, WebP format fallback, and exact `fetchpriority` attributes for LCP elements.
- [ ] Expose an internal Rust endpoint to collect Core Web Vitals (LCP, FID, CLS, INP) directly from client browsers and aggregate them into TimescaleDB.

### Tier 2: The Homepage & Interactive WebGL/WebGPU Globe
- [ ] Build the iconic unauthenticated homepage layout utilizing Primer CSS typography, fluid grid layouts, and high-contrast dark mode aesthetics.
- [ ] Integrate `three.js` specifically within an Angular Directive to construct the interactive 3D WebGL globe visualization.
- [ ] Implement a WebGPU fallback renderer pipeline within the Angular directive to future-proof the globe for modern browser hardware acceleration.
- [ ] Build a WebSocket/SSE connection in the Angular globe component that consumes a live feed of public Pull Request creation events from the Rust backend.
- [ ] Implement the math projection logic to map geographic coordinates (lat/long derived from PR author IP/GeoLite2) to spherical 3D vectors on the globe.
- [ ] Create rendering shaders (GLSL/WGSL) to draw glowing connection arcs, atmosphere scattering, and pulse animations representing merge events.
- [ ] Optimize globe geometry using LOD (Level of Detail) algorithms, reducing polygon counts automatically for mobile device viewports.
- [ ] Implement `IntersectionObserver` to pause WebGL rendering and WebSocket ingestion when the globe scrolls out of the viewport to save battery life.
- [ ] Build staggered, scroll-linked animations for homepage sections (e.g., sliding code windows, fading feature lists) using Angular's native Web Animations API.
- [ ] Implement a fallback 2D SVG rendering of the globe specifically for devices with disabled WebGL capabilities or "Reduce Motion" OS preferences.
- [ ] Construct the "Fake Terminal" interactive component rendering an animated typing effect (`git commit`, `git push`) synchronized with visual UI updates.
- [ ] Implement aggressive payload compression (e.g., binary protobufs instead of JSON) for the high-frequency WebGL WebSocket data stream.

### Tier 3: Immersive Product Pages (Actions, Security, Copilot)
- [ ] Build the `/features` overview page, implementing sticky sidebar navigation and deeply nested scroll-spy highlighting for major feature categories.
- [ ] Implement the `/enterprise` page, focusing on high-fidelity enterprise compliance diagrams, contact sales forms, and SAML/SSO feature highlights.
- [ ] Build the `/security` page, featuring interactive code-scanning vulnerability flow diagrams and Dependabot dependency tree showcases.
- [ ] Build the `/copilot` marketing page, utilizing advanced CSS gradients, blurred backdrop filters, and embedded looping `<video>` tags demonstrating code completions.
- [ ] Implement the "Interactive Copilot Diff" component: an Angular component allowing users to slide a visual handler comparing manual code vs AI-generated code.
- [ ] Develop the "Customer Stories" (Case Studies) template, rendering dynamic quotes, company logos, and specific metric highlights fetched from the Rust headless CMS.
- [ ] Implement the "Compare" matrix pages (e.g., GitHub vs GitLab), rendering expansive tabular comparisons powered by structured JSON data.
- [ ] Build reusable Angular components for common marketing primitives: Hero Sections, Call-to-Action (CTA) Bands, Logo Marquees, and Feature Cards.
- [ ] Implement targeted route preloading: eagerly fetch JavaScript chunks for the `/signup` and `/login` routes when users hover over primary CTA buttons.
- [ ] Ensure all marketing `<video>` assets utilize HTTP Live Streaming (HLS) or DASH, dynamically serving 1080p, 720p, or 480p chunks based on the user's bandwidth.
- [ ] Build the `/mobile` marketing page, featuring CSS-driven 3D device mockups framing the Ionic mobile app interfaces.

### Tier 4: Pricing, Subscriptions & Checkout Flows
- [ ] Define the `marketing_pricing_tiers` schema in Postgres to centrally manage tier features, prices, and limits (Free, Team, Enterprise).
- [ ] Build the `/pricing` Angular page, rendering the dynamic tier selection cards, monthly vs. yearly toggle states, and currency localization.
- [ ] Implement the expansive pricing comparison matrix table, featuring sticky table headers and precise feature-availability checkmarks.
- [ ] Build the "Calculate your ROI" interactive slider component, dynamically adjusting predicted savings based on developer seat counts and CI minutes.
- [ ] Connect the pricing tier CTAs directly to the Stripe Checkout initiation endpoint in the Rust backend, passing the selected plan, frequency, and currency.
- [ ] Implement the "Add-ons" calculator specifically for estimating GitHub Actions minutes and LFS storage overages on top of base plan costs.
- [ ] Build the Stripe Billing Portal integration to allow users to securely manage their payment methods and download PDF invoices.
- [ ] Implement 3D Secure (SCA) authentication handling in the Angular frontend to comply with European PSD2 banking regulations during checkout.
- [ ] Architect the Rust backend to calculate dynamic Tax/VAT rates based on the user's billing address before finalizing the Stripe subscription payload.
- [ ] Implement a dynamic banner system that queries the Rust backend for active promotional campaigns or discounts and renders them above the pricing matrix.
- [ ] Build the "Compare Editions" printable PDF generator in Rust, providing enterprise buyers a clean, formatted takeaway document.

### Tier 5: Headless CMS, Blog & Changelog (Rust)
- [ ] Architect a headless CMS pipeline entirely in Rust: store Markdown-formatted Blog and Changelog posts in a dedicated Postgres `content_posts` table.
- [ ] Build the `/changelog` Angular application view, rendering chronological feature updates with category badges and relative timestamps.
- [ ] Implement the `marked.js` rendering pipeline specifically for marketing content, applying custom Angular directives to highlight inline code and blockquotes.
- [ ] Develop the `/blog` index page with category filtering, author attribution, and pagination utilizing URL query parameters mapped to the Rust API.
- [ ] Implement an automated RSS feed generator in Rust (`/blog/feed.xml` and `/changelog/feed.xml`) strictly adhering to the RSS 2.0 specification.
- [ ] Build the internal Angular Admin Dashboard specifically for the Marketing team to author, preview, and schedule the publication of future posts.
- [ ] Implement frontmatter parsing in the Rust backend to extract and validate SEO metadata, canonical links, and author associations from Markdown uploads.
- [ ] Develop an asynchronous image optimization pipeline in Rust utilizing `image` crate to convert blog post image uploads to WebP/AVIF formats natively.
- [ ] Implement a "Related Posts" algorithm in Postgres utilizing `pg_trgm` to recommend semantically similar blog content at the bottom of articles.
- [ ] Build the specific "Author Profile" pages (`/blog/author/:slug`), aggregating all posts written by a specific platform engineer or developer advocate.

### Tier 6: Enterprise Sales & Lead Generation
- [ ] Define the `sales_leads` Postgres table to capture inbound enterprise inquiries (Name, Corporate Email, Company Size, Phone, Region).
- [ ] Build the `/enterprise/contact` Angular form utilizing reactive forms with strict corporate email validation (rejecting `@gmail.com`/`@yahoo.com` addresses).
- [ ] Implement a Rust background worker to dispatch lead payloads directly to Salesforce via the Salesforce REST API.
- [ ] Connect the lead ingestion pipeline to Marketo or Hubspot webhooks to automatically enroll enterprise prospects into email nurturing campaigns.
- [ ] Build the "Request a Demo" calendar integration: securely embed calendar scheduling iframes or connect to the Calendly API for immediate SDR routing.
- [ ] Implement rate-limiting and robust CAPTCHA (hCaptcha) verification on all lead generation forms to prevent automated CRM spam attacks.
- [ ] Build the "Enterprise Whitepaper Download" gate: require an email submission to unlock signed S3 URLs for high-value compliance PDFs (SOC2, ISO27001).

### Tier 7: Global Analytics, Privacy & A/B Testing
- [ ] Integrate `@angular/localize` to compile language-specific builds of the marketing site (e.g., `es.domain.com`, `ja.domain.com`) for Tier-1 languages.
- [ ] Implement a localized URL router in Angular that detects the user's `Accept-Language` header and dynamically redirects to the correct locale directory.
- [ ] Architect the A/B Testing Engine in Rust: assign anonymous UUID cookies at the edge proxy and deterministically bucket users into marketing variants (e.g., Variant A vs Variant B Hero Copy).
- [ ] Emit A/B testing cohort assignments as telemetry events to an internal Kafka pipeline for downstream conversion rate analysis.
- [ ] Build the GDPR/CCPA compliant Cookie Consent banner, strictly pausing all telemetry and external script injections until explicit user opt-in is recorded.
- [ ] Implement an internal Analytics Sink endpoint in Rust to securely ingest anonymized page views, CTA clicks, and bounce rates without relying on third-party tracking scripts.
- [ ] Write a Rust statistical significance calculator (p-value, confidence intervals) accessible via an internal admin UI to evaluate A/B test completion states.
- [ ] Implement UTM parameter tracking across the entire marketing funnel, persisting origin campaign source data into the browser's `sessionStorage` until account creation.

### Tier 8: Ecosystem & Community Portals
- [ ] Build the `/sponsors` marketing page, showcasing top open-source maintainers and highlighting corporate matching funds.
- [ ] Implement the `/explore` page index, rendering dynamic collections of repositories categorized by topic (e.g., "Machine Learning", "Web3").
- [ ] Develop the trending repositories ranking algorithm in Postgres, heavily weighting recent stars, forks, and active PRs to generate the "Trending Today" lists.
- [ ] Build the "GitHub Stars" community program portal, rendering a searchable directory of recognized community leaders.
- [ ] Implement the Open Source Student Pack landing page, featuring specialized student verification flows integrated with the SheerID API.
- [ ] Expose an interactive dependency graph visualization on the ecosystem page, showing how major OSS projects intersect using D3.js.
