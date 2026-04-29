# Plan for gh-clone-models

### Tier 1: Model Registry, Pricing & Telemetry Schema
- [ ] Define the `ai_models` Postgres table: `id`, `provider` (openai, anthropic, meta, local), `model_name`, `context_window`, `max_output_tokens`, `status` (active, deprecated).
- [ ] Define the `ai_pricing_tiers` table: map specific models to input cost (`cost_per_1k_input`), output cost (`cost_per_1k_output`), and image processing cost (`cost_per_image`).
- [ ] Define the `ai_inference_logs` Postgres table to track usage for billing: `id`, `user_id`, `org_id`, `model_id`, `prompt_tokens`, `completion_tokens`, `total_cost`, `total_latency_ms`, `timestamp`.
- [ ] Implement a highly efficient caching layer in Redis to store active model configurations, routing rules, and deprecation notices for sub-millisecond retrieval.
- [ ] Expose an unauthenticated GraphQL endpoint `models { name, family, contextWindow, parameters }` to instantly populate public marketing and registry pages.
- [ ] Implement the `ai_fine_tunes` table: track enterprise-specific model adaptations (LoRA weights), their base models, and specific access control lists (ACLs).
- [ ] Implement strict daily and monthly budget caps at the Organization and User level, tracked atomically via Redis counters to prevent API abuse and bill shock.
- [ ] Build a background Rust aggregator that rolls up raw token usage logs into the primary billing engine daily for precise invoice generation.
- [ ] Expose an endpoint to generate CSV exports of token consumption by department/team to assist enterprise administrators in chargeback accounting.

### Tier 2: Universal LLM Gateway & Load Balancing (Rust)
- [ ] Build the `models-gateway` microservice natively in Rust using `axum`, `tokio`, and `reqwest`.
- [ ] Implement a normalized JSON payload parser: accept standard OpenAI-compatible `ChatCompletion` requests and dynamically translate them into Anthropic `Messages` API or Azure OpenAI schemas.
- [ ] Implement Server-Sent Events (SSE) streaming proxy logic: multiplex outbound LLM stream chunks from upstream providers back to the client immediately without buffering the full response in memory.
- [ ] Build a robust error translation layer: normalize provider-specific HTTP errors (e.g., Anthropic `overloaded_error`, OpenAI `rate_limit_exceeded`) into a standard internal `503 Service Unavailable` or `429 Too Many Requests` format.
- [ ] Implement Request Hedging: dispatch the same request to two identical provider endpoints simultaneously, returning the first successful chunked response and cancelling the slower request to guarantee lowest latency.
- [ ] Implement Fallback Routing: if the primary provider (e.g., Azure OpenAI) times out after 2000ms, automatically retry the request against a secondary provider (e.g., direct OpenAI API) without dropping the client connection.
- [ ] Implement Prompt Caching logic: specifically map repeating system prompts to Anthropic's `ephemeral` caching headers or internal Redis vector caches to dramatically lower costs on repetitive tasks.
- [ ] Implement Context Window Truncation: if an incoming prompt exceeds the model's `context_window`, utilize a fast Rust tokenizer (`tiktoken-rs`) to surgically truncate the middle of the prompt, preserving the system instruction and the most recent user query.
- [ ] Implement API Key / PAT validation middleware specifically for the Gateway, extracting `Authorization: Bearer` and executing a constant-time check against the Postgres `access_tokens` database.
- [ ] Enforce specific `read:models` and `write:models` scope validation on the Personal Access Token before routing the inference request.
- [ ] Implement geographic routing: route enterprise requests to specific Azure endpoints (e.g., `europe-west1`) strictly based on the enterprise's configured Data Residency requirements.

### Tier 3: Advanced Inference Features (Tools, Vision, Embeddings)
- [ ] Implement Tool Calling (Function Calling) passthrough: parse `tools` and `tool_choice` schemas from the OpenAI format and map them accurately to Anthropic's XML-based or native JSON tool-use definitions.
- [ ] Implement Vision API normalization: parse incoming `image_url` payloads (including base64 encoded strings) and correctly format them for GPT-4o Vision or Claude 3.5 Sonnet payload requirements.
- [ ] Enforce Image size restrictions: block image payloads exceeding 20MB at the gateway layer to prevent backend memory exhaustion and excessive provider costs.
- [ ] Implement the `/v1/embeddings` REST endpoint: route embedding array generation requests to specialized models (e.g., `text-embedding-3-small`), batching requests where necessary to respect upstream rate limits.
- [ ] Implement audio and transcription stubs (`/v1/audio/transcriptions`): route these strictly to Whisper backends, returning standard OpenAI compatible JSON responses.
- [ ] Build a local fallback embedding engine utilizing `candle` or `ort` in Rust to generate embeddings on the gateway nodes directly for free-tier users, bypassing OpenAI completely.

### Tier 4: Angular Interactive Playground & Chat UI
- [ ] Scaffold the Models Playground UI utilizing Angular 21 with Zoneless Change Detection (`provideExperimentalZonelessChangeDetection`) for maximum rendering speed.
- [ ] Implement a complex 3-pane split-view layout using `@angular/cdk/drag-drop` (System Prompt on the left, Chat Timeline in the middle, Parameter Tuning on the right).
- [ ] Integrate the Monaco Editor into the System Prompt and User Message text areas to provide syntax highlighting, auto-completion, and multi-line formatting.
- [ ] Build the Parameter Tuning controls: Angular reactive forms mapping visual sliders and numeric inputs to `Temperature`, `Top P`, `Max Tokens`, `Presence Penalty`, and `Frequency Penalty`.
- [ ] Implement the SSE Consumer service in Angular, utilizing the native browser `fetch` API to read stream chunks, decode UTF-8 `TextDecoder`, and append tokens to the UI signals in real-time.
- [ ] Render the Chat Timeline using GitHub Flavored Markdown (`marked.js`), specifically optimizing the DOM to handle real-time syntax highlighting of code blocks as they stream in character-by-character.
- [ ] Build the "Compare Models" split-pane feature: allow users to select two different models (e.g., Llama 3 70B vs GPT-4o), execute the exact same prompt, and stream the responses side-by-side.
- [ ] Implement Session History: save active chats into the local browser `IndexedDB` to allow users to refresh the page without losing their Playground state.
- [ ] Build the "Export Chat" functionality: allow users to instantly export their conversation as a `.md` file, or generate a private GitHub Gist directly from the Playground UI.
- [ ] Implement real-time token counting in the Angular UI, warning the user via a progress bar as their prompt approaches the selected model's maximum context limit.
- [ ] Ensure perfect Dark Mode / Light Mode CSS synchronization matching the overarching Primer design system.

### Tier 5: Serverless Inference SDK Compatibility (REST API)
- [ ] Implement the exact `POST /v1/chat/completions` REST endpoint to ensure 100% plug-and-play compatibility with standard OpenAI SDKs (Python, Node.js, Rust).
- [ ] Implement the `/v1/models` endpoint returning the exact JSON array format expected by the OpenAI specification so external tools can dynamically discover available models.
- [ ] Implement CORS (Cross-Origin Resource Sharing) policies explicitly allowing wildcard `*` origins and preflight `OPTIONS` requests for all `/v1/` endpoints to support client-side SDK web usage.
- [ ] Inject specific rate-limit response headers expected by standard LLM SDKs: `x-ratelimit-limit-requests`, `x-ratelimit-limit-tokens`, `x-ratelimit-remaining-tokens`, `x-ratelimit-reset-requests`.
- [ ] Synthesize internal `Usage` blocks dynamically on stream completion, appending the final `{ prompt_tokens: X, completion_tokens: Y }` JSON payload to the final `[DONE]` SSE chunk to ensure SDKs accurately track costs.

### Tier 6: Prompt Engineering, Context Injection & RAG
- [ ] Build the "Save Preset" feature in Angular, allowing users to save their specific combination of Model, System Prompt, and Parameters into the Postgres `ai_presets` table for quick recall.
- [ ] Implement contextual repository injections: provide an Angular UI toggle to "Include Repository Context", commanding the Rust backend to fetch the `README.md` and top-level directory structure, automatically prepending them to the system prompt.
- [ ] Expose an Angular component to "Generate Code": implement UI features where generated markdown code blocks have a one-click "Create Pull Request" or "Copy to Clipboard" overlay button.
- [ ] Build an AST parser in Rust to intercept gateway requests and securely inject dynamic variables (like the user's `@handle`, the current UTC date, and active repository stats) into the system prompt implicitly.
- [ ] Integrate with the Blackbird Search indexer: implement a mini-RAG (Retrieval-Augmented Generation) pipeline where `@search "query"` in the prompt dynamically queries Elasticsearch and injects the top 5 code snippets into the context window before forwarding to the LLM.

### Tier 7: Enterprise Provisioning, Budgets & Cost Routing
- [ ] Define Enterprise Policies for AI: allow Organization Owners to explicitly disable specific foundational models (e.g., blocking Anthropic due to compliance, strictly forcing Azure OpenAI).
- [ ] Implement Provisioned Throughput (PTU) routing: map specific high-paying enterprise tenants directly to dedicated GPU instances or reserved capacity endpoints to guarantee zero latency degradation during peak hours.
- [ ] Build the Billing Threshold Alarms: automatically dispatch an email and webhook notification to the Enterprise Admin when their organization consumes 80% and 100% of their monthly AI token budget.
- [ ] Enforce specific Single Sign-On (SSO) requirements for model inference: immediately reject API requests to `/v1/chat/completions` with a `403 Forbidden` if the user's PAT has not been authorized via the Enterprise SAML IdP.

### Tier 8: Trust, Safety, Redaction & Content Moderation
- [ ] Implement a pre-flight prompt moderation pipeline: evaluate incoming user inputs against an internal, fast, local toxicity model (e.g., ONNX-compiled DistilBERT) to instantly drop hateful, self-harm, or explicitly malicious prompts with a `400 Bad Request`.
- [ ] Implement post-flight completion scanning: parse the streaming output for PII leakage (Social Security Numbers, Credit Cards) and actively redact the stream bytes if a violation threshold is breached.
- [ ] Enforce "Zero Retention" enterprise policies: ensure the `models-gateway` operates entirely in RAM and explicitly bypasses the `ai_inference_logs` prompt/response body persistence for organizations requiring strict SOC2/HIPAA data privacy.
- [ ] Implement prompt injection / jailbreak detection heuristics: automatically quarantine and suspend users executing repetitive, high-volume automated prompt injections targeting model safeguards.
- [ ] Build the Data Sharing toggle in the Angular UI: ensure the "Allow my data to be used to train models" option defaults to false, and strictly enforce this flag when routing prompts to external providers (e.g., appending specific opt-out headers).