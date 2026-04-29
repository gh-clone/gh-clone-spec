# Plan for gh-clone-pages

### Tier 1: Distributed DNS & Edge Routing Plane
- [ ] Deploy a distributed edge routing tier utilizing OpenResty (Nginx + Lua) or Envoy to intercept all traffic destined for `*.github.io`.
- [ ] Implement a highly optimized SNI (Server Name Indication) parser in the edge proxy to extract the requested host before TLS termination.
- [ ] Build a Redis-backed routing table that maps custom apex domains (e.g., `example.com`) to their corresponding internal `user/repo` storage bucket.
- [ ] Implement Wildcard DNS resolution logic natively handling the fallback routing from `sub.user.github.io` to `user.github.io`.
- [ ] Build strict validation for CNAME/ALIAS records to prevent Subdomain Takeover attacks (verifying domain ownership via TXT records).
- [ ] Implement HTTP/2 and HTTP/3 (QUIC) protocol support at the edge to accelerate static asset delivery.
- [ ] Configure dynamic Gzip and Brotli compression based on the `Accept-Encoding` header, pre-compressing known text mimetypes.
- [ ] Implement specific routing logic to handle `user.github.io/repository-name` subpath resolution transparently mapping to the correct deployment bucket.

### Tier 2: Dynamic TLS & ACME Certificate Management
- [ ] Integrate a background ACME client (e.g., `certmagic` or `lego`) to automatically request Let's Encrypt certificates for user-provided custom domains.
- [ ] Implement the ACME HTTP-01 challenge responder within the edge router, serving exact tokens at `/.well-known/acme-challenge/`.
- [ ] Implement the ACME DNS-01 challenge logic specifically to support wildcard certificate issuance where required.
- [ ] Design an AES-256-GCM encrypted Postgres table to securely store issued TLS private keys and full certificate chains.
- [ ] Build a daily cron job that scans the certificate database and automatically triggers renewals for domains expiring within 30 days.
- [ ] Implement OCSP Stapling in the TLS handshake to optimize browser connection speeds.
- [ ] Enforce "Enforce HTTPS" toggles, generating automatic 301 redirects from port 80 to 443 strictly at the routing layer.

### Tier 3: Jekyll Build Pipeline & GitHub Actions
- [ ] Author a native `actions/jekyll-build-pages` composite action that safely wraps the Jekyll Ruby executable.
- [ ] Implement strict plugin whitelisting within the Jekyll execution environment to prevent arbitrary remote code execution (RCE) on the runner.
- [ ] Author a native `actions/upload-pages-artifact` action that packages the generated `_site` directory into an optimized tarball.
- [ ] Author a native `actions/deploy-pages` action that securely transmits the artifact to the internal Pages storage backend using OIDC tokens.
- [ ] Implement OIDC subject validation (`sub` claim) to ensure that only a workflow originating from the target repository's `main` branch can deploy.
- [ ] Design the immutable deployment storage architecture: extract tarballs to hashed directories (e.g., `/deployments/{uuid}`) rather than overwriting existing files.
- [ ] Provide atomic swaps for deployments: update the Redis routing pointer to the new `{uuid}` directory to achieve zero-downtime rollouts.
- [ ] Implement automated deployment cleanup logic, permanently deleting deployment directories older than 90 days to conserve SSD space.

### Tier 4: CDN Caching & Header Manipulation
- [ ] Integrate with a global CDN (e.g., Fastly, Cloudflare) or deploy internal Varnish cache nodes in multiple geographic regions.
- [ ] Parse the custom `_headers` file (Cloudflare Pages/Netlify format) to allow users to specify custom `Content-Security-Policy` or `CORS` headers.
- [ ] Parse the custom `_redirects` file to implement 301/302 redirects natively within the edge router before hitting storage.
- [ ] Implement `Cache-Control` header overriding, setting long max-age values for static assets while keeping HTML expiration brief.
- [ ] Develop a webhook trigger that issues an instant `PURGE` request to the CDN invalidating the site's cache immediately upon a successful deployment.
- [ ] Implement specific fallback logic: if the exact file path is not found, dynamically check for `{path}.html` or `{path}/index.html`.
- [ ] Implement custom 404 handling: if all path resolutions fail, serve the repository's `404.html` (if present) returning a strict HTTP 404 status.
- [ ] Implement a strict bandwidth metering and rate-limiting system to automatically disable sites exceeding fair-use limits (e.g., serving 100GB of video files).

### Tier 5: Observability, Security & Edge Defense
- [ ] Expose Prometheus metrics at the OpenResty/Envoy edge tracking cache hit/miss ratios, 4xx/5xx errors, and TLS handshake latencies.
- [ ] Implement a Web Application Firewall (WAF) rule set specifically tuned to drop malicious payloads targeting static sites.
- [ ] Enforce strict mTLS (Mutual TLS) between the edge routing nodes and the internal artifact storage backend.
- [ ] Build an automated abuse detection heuristic to identify and suspend Pages sites hosting crypto-mining scripts or phishing kits.
- [ ] Implement distributed tracing (W3C trace context) propagation from the initial edge request down to the S3 bucket fetch.
