# Plan for gh-clone-copilot

### Tier 1: LLM Gateway & Rate Limiting Proxy
- [ ] Implement the `copilot-proxy` service in Rust, acting as a high-performance HTTP gateway between the IDE/Web frontend and the backend LLM providers (OpenAI, Azure, Anthropic).
- [ ] Build the Token Bucket rate-limiting algorithm explicitly tracking *LLM token counts* per user and organization to prevent billing exhaustion.
- [ ] Implement a semantic caching layer using Redis to cache responses for identical prompt hashes, reducing API costs and latency.
- [ ] Build the telemetry ingestion pipeline to capture anonymized acceptance rates, rejection rates, and edit distances for Copilot suggestions.
- [ ] Implement strict payload scrubbing: redact hardcoded secrets, PII, and API keys from prompts *before* they are transmitted to external LLM providers using the `hyperscan` regex engine.
- [ ] Design a streaming Server-Sent Events (SSE) router that multiplexes LLM response tokens directly to the requesting client without buffering the full response.
- [ ] Support multiplexed provider routing: dynamically route requests to different models (e.g., GPT-3.5 for autocomplete, GPT-4 for chat) based on latency SLAs and user tier.

### Tier 2: Copilot Chat & Angular Web UI Integration
- [ ] Build the `CopilotChatComponent` in Angular, implementing a draggable, resizable sliding panel available globally across the repository view.
- [ ] Implement highly optimized DOM rendering for the chat timeline using `@angular/cdk/scrolling` to handle extremely long conversation contexts.
- [ ] Support dynamic context injection: when a user highlights code in the Monaco Editor and opens Chat, automatically serialize the selection, file path, and language as system context.
- [ ] Implement streaming markdown rendering for chat responses, updating the UI token-by-token without triggering full Angular Change Detection cycles (using Signals).
- [ ] Build custom Angular components for chat "Tools" (e.g., rendering a specific Commit, Issue, or PR card dynamically when referenced by the LLM).
- [ ] Implement "Slash Commands" (`/explain`, `/tests`, `/fix`) utilizing the Angular Command Palette for fast prompt scaffolding.
- [ ] Inject repository-level context dynamically: utilize Blackbird code search API to fetch the 5 most relevant snippets based on the user's chat query to append to the system prompt (RAG).

### Tier 3: Inline Autocomplete & Monaco Integration
- [ ] Integrate the `inlineCompletions` Provider API into the Monaco Editor to support ghost-text autocomplete suggestions in the browser.
- [ ] Build the debounce and cancellation logic in RxJS: automatically cancel pending LLM HTTP requests if the user continues typing to prevent race conditions.
- [ ] Implement a suffix/prefix matching algorithm to ensure the LLM's suggested code perfectly aligns with the characters immediately preceding and following the cursor.
- [ ] Send local file context (imports, type definitions, nearby functions) explicitly constructed within the prompt to improve autocomplete relevance.
- [ ] Implement multi-file context tracking: maintain a rolling window of recently edited files in IndexedDB to provide cross-file autocomplete context to the LLM.
- [ ] Track precise keystroke telemetry around ghost text: log `Accept` (Tab), `Partial Accept` (Ctrl+Right), and `Reject` (Esc/Typing) events to evaluate model performance.

### Tier 4: Copilot Workspace & Issue-to-PR Automation
- [ ] Implement the "Generate Plan" Rust endpoint: pass an Issue body and repository structure to the LLM to generate a structured JSON plan of required file changes.
- [ ] Build the `CopilotWorkspaceComponent` in Angular, rendering the generated plan as an interactive, editable checklist before any code is generated.
- [ ] Implement the "Implement Plan" engine: orchestrate multiple parallel LLM calls (one per file) to generate the code diffs based on the approved plan.
- [ ] Build a server-side virtual Git merge engine to apply the LLM-generated diffs directly into a new, temporary Git branch without requiring a local clone.
- [ ] Render the generated diffs in the Angular frontend utilizing the split-diff view, allowing the user to make manual corrections before finalizing.
- [ ] Implement the "Create Pull Request" automated flow, generating the PR title, description, and linking the original issue completely via the LLM.
- [ ] Integrate CI/CD feedback loop: if the automated PR fails CI, parse the build logs and feed them back into the LLM for a self-healing attempt.

### Tier 5: Enterprise AI Policies & Governance
- [ ] Define the `copilot_policies` Postgres table to manage organization-level AI configurations.
- [ ] Implement the "Disable Telemetry" enforcement: instantly drop all telemetry payloads at the edge proxy if the organization policy forbids data collection.
- [ ] Implement the "Block Public Code Match" filter: query a secondary index of open-source code and block suggestions that match >150 characters of known public code without attribution.
- [ ] Provide an Audit Log UI specifically for Copilot, tracking exactly when and where Copilot was enabled, disabled, or when policies were modified.
- [ ] Implement IP allowlisting strictly for Copilot API endpoints, ensuring IDEs can only request completions from approved corporate networks.
- [ ] Build the "Seat Management" UI, allowing admins to provision or revoke Copilot licenses for individual developers or Teams dynamically.
### Tier 6: RAG Pipeline, Code Embeddings & Vector Search
- [ ] Deploy a distributed Vector Database (e.g., `pgvector` or `Qdrant`) tightly coupled with the repository storage nodes.
- [ ] Implement an asynchronous Rust worker that intercepts `git push` events and passes modified files through `tree-sitter` for semantic chunking (functions, classes, structs).
- [ ] Generate dense vector embeddings for all semantic chunks using a high-throughput inference pipeline (e.g., Rust `candle` or `ort` with a pre-trained embedding model).
- [ ] Build the Retrieval-Augmented Generation (RAG) query engine: translate a user's Chat query into an embedding, perform an ANN (Approximate Nearest Neighbor) search, and extract the top 10 relevant code snippets.
- [ ] Implement strict RBAC filtering on the vector search: ensure the embedding search only returns results from repositories the user explicitly has read access to.
- [ ] Build a background job to garbage-collect and recalculate vector embeddings when a file is deleted or heavily refactored.
- [ ] Implement metadata tagging on embeddings (Language, File Path, Commit SHA) to allow the LLM routing proxy to explicitly filter context (e.g., "Only look at Rust files").
- [ ] Parse `.github/copilot-instructions.md` within the repository root to dynamically inject repository-specific coding conventions (e.g., "Always use Axum extractors") into the system prompt.

### Tier 7: Security, Finetuning & Offline Capabilities
- [ ] Implement LLM Prompt Injection detection logic at the gateway layer, discarding requests that attempt to override system instructions or extract hidden context.
- [ ] Build a local Small Language Model (SLM) fallback runner: embed a quantized model via `llama.cpp` bindings in Rust to handle simple autocompletes with <20ms latency when external providers are slow.
- [ ] Implement the "Finetuning Data Extraction" pipeline: securely aggregate and anonymize merged Pull Request diffs to construct training corpora for enterprise-specific custom models.
- [ ] Configure differential privacy algorithms on the extracted telemetry to guarantee no individual developer's sensitive logic is memorized by the global model.
- [ ] Build the "Copilot Output Sanitizer" middleware: execute `cargo check` or `npm run lint` virtually on generated code blocks before presenting them to the user to guarantee syntactical correctness.
- [ ] Expose an Angular UI Dashboard for Copilot ROI (Return on Investment), rendering charts displaying "Lines of Code Accepted" vs "Lines Written Manually" across the enterprise.
- [ ] Implement "Context Providers" in the Angular frontend, allowing users to explicitly `#` reference specific issues, PRs, or external documentation links within their Chat prompt.
- [ ] Configure dynamic LLM timeout circuit breakers: if OpenAI/Azure response times exceed 5000ms, immediately degrade gracefully to the local fallback model.
- [ ] Enforce "Ephemeral Chat" enterprise policies ensuring chat logs and prompt histories are instantly purged from the Postgres database upon session termination.

### Tier 8: Language Server Protocol (LSP) & Copilot CLI
- [ ] Develop the standalone `replica-copilot-lsp` binary in Rust, strictly adhering to the Language Server Protocol specification for cross-IDE compatibility.
- [ ] Implement bidirectional JSON-RPC over `stdio` in the LSP server to communicate seamlessly with VS Code, JetBrains, and Neovim clients.
- [ ] Build the Copilot CLI integration: implement backend APIs specifically to support terminal-based interactions (e.g., `gh copilot suggest "how to undo a commit"`).
- [ ] Implement the shell-context extraction endpoints: parse OS environment variables, shell history, and terminal errors submitted by the CLI to formulate highly accurate bash/powershell suggestions.
- [ ] Develop the authentication bridge for the CLI and LSP: handle OAuth device-flow (`/login/device`) natively in Rust to provision tokens for local environments.

### Tier 9: Copilot Autofix (CodeQL Integration)
- [ ] Architect the Copilot Autofix engine: a specialized Rust pipeline that intercepts newly created CodeQL SAST vulnerabilities (SARIF uploads).
- [ ] Aggregate the vulnerable code snippet, the CodeQL taint-tracking data flow, and the abstract syntax tree into a specialized LLM prompt.
- [ ] Dispatch the specialized prompt to the `copilot-proxy` gateway, utilizing high-context models (e.g., GPT-4o) to generate a secure, syntactically correct code replacement.
- [ ] Apply the generated fix locally within a temporary virtual Git tree using `gitoxide`, running an automated syntax validation pass to ensure the fix compiles.
- [ ] Render the Autofix suggestion in the Angular Pull Request UI as an interactive "Suggested Change" diff block, allowing developers to review and commit the patch with one click.
- [ ] Track Telemetry on Autofix: monitor the acceptance rate of AI-generated security fixes versus manual remediations to constantly fine-tune the backend prompt compilation.

### Tier 10: Copilot Extensions & 3rd-Party Agent Dispatch
- [ ] Build the `Copilot Agent Dispatcher` in Rust, an API gateway specifically designed to route user prompts from Copilot Chat to registered third-party endpoints (e.g., `@docker`, `@sentry`).
- [ ] Implement the Extension Registration UI in Angular, allowing Organization Admins to install and authorize external Copilot Agents using standard OAuth 2.0 flows.
- [ ] Define the standard Agent Protocol JSON schema, ensuring third-party services receive the necessary context (active file, selection, thread history) uniformly and securely.
- [ ] Implement strict prompt scrubbing before dispatch: automatically detect and redact internal repository secrets or PII before sending the context payload to an external agent provider.
- [ ] Support streaming responses from external agents back through the gateway, multiplexing the SSE chunks directly to the Angular/VS Code client.
- [ ] Render specific UI cards in the Angular Copilot Chat interface (e.g., an interactive "Docker Container Status" widget) based on custom Markdown/HTML payloads returned by the third-party agent.
- [ ] Implement a strict timeout and fallback mechanism: if an external agent fails to respond within 5 seconds, terminate the connection and display a native error badge to the user.
