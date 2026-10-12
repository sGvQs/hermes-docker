# Ollama model swapping for a sequential implementer -> reviewer workflow (Apple Silicon Mac mini)

## 1. Which settings make Ollama unload the implementer before loading the reviewer (or keep both)?

### Takeaway
Ollama loads a second model alongside the first only if it fits in memory; otherwise the request queues and an idle model is evicted. Control it with OLLAMA_MAX_LOADED_MODELS, OLLAMA_KEEP_ALIVE and per-request keep_alive (native API). Use keep_alive:0 on the implementer's last call for deterministic unload, or leave both resident if the memory budget allows.

### Cited Findings
- OLLAMA_MAX_LOADED_MODELS = max models loaded concurrently "provided they fit in available memory"; default 3x GPUs (3 for CPU). — [Ollama FAQ](https://raw.githubusercontent.com/ollama/ollama/main/docs/faq.mdx)
- OLLAMA_NUM_PARALLEL = parallel requests per model, default 1; memory needs scale with parallel x context length. — [Ollama FAQ](https://raw.githubusercontent.com/ollama/ollama/main/docs/faq.mdx)
- If a new model fits it loads alongside others; if not, new requests are queued until an idle model is unloaded to make room. Queue limit OLLAMA_MAX_QUEUE default 512, beyond which 503. — [Ollama FAQ](https://raw.githubusercontent.com/ollama/ollama/main/docs/faq.mdx)
- Default keep-alive is 5 minutes. OLLAMA_KEEP_ALIVE sets it globally; the per-request keep_alive on /api/generate and /api/chat overrides it. Values: duration string, seconds, negative = forever, 0 = unload immediately after response. — [Ollama FAQ](https://raw.githubusercontent.com/ollama/ollama/main/docs/faq.mdx)
- `ollama stop <model>` unloads immediately; `ollama ps` shows loaded models, processor (GPU/CPU split) and an UNTIL column ("Forever" with -1). — [Ollama FAQ](https://raw.githubusercontent.com/ollama/ollama/main/docs/faq.mdx), [DEV Community guide](https://dev.to/jovan_chan_9500711396d4e6/ollama-keeps-reloading-the-model-fix-vram-unloading-cold-starts-and-model-swapping-2026-25b7)
- Preload/pin: send an empty request (empty prompt) to load without generating; with keep_alive -1 it stays; empty prompt + keep_alive:0 unloads. CLI: `ollama run <model> ""`. — [Ollama FAQ](https://raw.githubusercontent.com/ollama/ollama/main/docs/faq.mdx), [DEV Community guide](https://dev.to/jovan_chan_9500711396d4e6/ollama-keeps-reloading-the-model-fix-vram-unloading-cold-starts-and-model-swapping-2026-25b7)
- Env vars must be set on the Ollama service, not the shell; on macOS use `launchctl setenv OLLAMA_KEEP_ALIVE "-1"` and restart; it does not persist across reboots without a launch agent, and after reboot the model is not loaded until the first request (a preload job is suggested). — [DEV Community guide](https://dev.to/jovan_chan_9500711396d4e6/ollama-keeps-reloading-the-model-fix-vram-unloading-cold-starts-and-model-swapping-2026-25b7), [search result summary of guides](https://oneuptime.com/blog/post/2026-01-27-ollama-local-llm-inference/markdown)

### Inferences
- Swap-forcing recipe: OLLAMA_MAX_LOADED_MODELS=1 (hard guarantee of one resident model; unverified in sources beyond the FAQ semantics), OLLAMA_NUM_PARALLEL=1, and keep_alive "5m"-ish. Alternatively keep MAX_LOADED=2 and rely on fit-check; but on a tight budget eviction is implicit and less predictable than MAX_LOADED=1.
- Keep-both recipe: MAX_LOADED_MODELS=2, KEEP_ALIVE=-1 (or 24h), preload both at startup, only if weights + KV cache of both fit with headroom for macOS.
- Note: because the Docker clients reach the host Ollama, these are host-side settings; Hermes cannot change them per-call over /v1 (see Q3).

### Gaps
- No official statement that MAX_LOADED_MODELS=1 evicts proactively vs. only when needed (FAQ describes eviction when memory does not fit). Test with `ollama ps`.
- Whether Apple Silicon's recommended GPU working-set limit (about 2/3-3/4 of unified memory by default) affects the fit-check was not sourced here.

## 2. Swap latency for 20-40 GB models on M-series, and mitigation

### Takeaway
I found no reliable, directly applicable Apple Silicon benchmark for 20-40 GB models. Only generic figures exist (7B cold start 3-10 s; 70B ~18 s NVMe / ~74 s SATA on non-Mac hardware). Measure on the actual Mac mini.

### Cited Findings
- 7B cold start roughly 3-10 s; 70B model ~74 s from SATA SSD and ~18 s from NVMe (attributed to Markaicode benchmarks, not verified, not Apple Silicon). — [DEV Community guide](https://dev.to/jovan_chan_9500711396d4e6/ollama-keeps-reloading-the-model-fix-vram-unloading-cold-starts-and-model-swapping-2026-25b7)
- A vendor README (commercial interest, unverified) reports Ollama cold start 20-30 s (load + first token) for a 1.5B model on an M3 MacBook Pro 16 GB. — [Squish README via search](https://gitblind.noratr.app/konjoai/squish)
- A Gemma 4 guide says that on 32GB+ unified memory a 26B model plus an embedding model can be resident together comfortably (anecdotal). — [ClawSouls guide](https://docs.clawsouls.ai/docs/guides/local-llm-gemma4)
- Memory scaling: KV cache grows with context length and with NUM_PARALLEL (parallel requests multiply context). — [Ollama FAQ](https://raw.githubusercontent.com/ollama/ollama/main/docs/faq.mdx)

### Inferences
- Mac mini internal SSD read speed (several GB/s) suggests a 20-40 GB cold load on the order of 5-30 s plus first-token prompt processing; this is an estimate, not sourced. Page-cache effects may make repeat swaps faster if free RAM remains.
- Mitigations: batch work (do all implementer calls, then all reviewer calls) to minimize swaps; use a smaller reviewer that co-resides; same model with different system prompt (zero swap); keep_alive long enough to bridge a task's implement-review cycle.
- MoE (e.g. 35B-A3B at 4-bit, ~20 GB weights) has low active parameters so generation is fast, but all weights must still be resident, so load time/memory follow total size, not active size (general MoE property, not sourced here).

### Gaps
- No sourced measured load times for 20-40 GB GGUF on M-series; no sourced KV-cache sizing numbers for the specific 35B-A3B model.

## 3. Does the OpenAI-compatible /v1 endpoint honor num_ctx / keep_alive?

### Takeaway
No. /v1 cannot set num_ctx (official docs) and the official OpenAI-compat docs list no keep_alive. Set context via OLLAMA_CONTEXT_LENGTH, a Modelfile with PARAMETER num_ctx (call the derived model name via /v1), or use native /api/chat.

### Cited Findings
- Official: "The OpenAI API does not have a way of setting the context size for a model"; workaround is a Modelfile with `FROM` + `PARAMETER num_ctx`, `ollama create mymodel`, then use `mymodel`. — [Ollama OpenAI compat docs](https://docs.ollama.com/api/openai-compatibility)
- Supported chat fields include model, messages, stream, stop, seed, max_tokens, temperature, top_p, penalties, response_format, tools, reasoning_effort; unsupported: tool_choice, logit_bias, user, n, logprobs. keep_alive is not listed. — [Ollama OpenAI compat docs](https://docs.ollama.com/api/openai-compatibility)
- Secondary: num_ctx passed in options or top-level on /v1 is silently dropped (tracked as Ollama issue #11409 per a blog; I did not verify the issue directly). — [particula.tech](https://particula.tech/blog/ollama-num-ctx-silent-prompt-truncation), [BetterClaw](https://www.betterclaw.io/blog/ollama-context-window-truncates-agent-config)
- Secondary: default context was 4096 and, per one source, since a Feb 2026 change is chosen from VRAM (4k / 32k / 256k tiers). Silent prompt truncation is the main risk for agents. — [runaihome](https://runaihome.com/blog/ollama-truncating-input-prompt-num-ctx-fix-2026/), [Ollama FAQ](https://raw.githubusercontent.com/ollama/ollama/main/docs/faq.mdx) (default 4096, OLLAMA_CONTEXT_LENGTH)
- OpenClaw works around this by injecting options.num_ctx in its native-mode integration. — [OpenClaw docs](https://docs.openclaw.ai/providers/ollama/advanced)

### Inferences
- For Hermes over /v1: set OLLAMA_CONTEXT_LENGTH globally on the host and/or create per-role Modelfile aliases (e.g. `impl-ctx32k`, `review-ctx16k`) with num_ctx and (optionally) other params; keep_alive must be host env (OLLAMA_KEEP_ALIVE) since /v1 has no per-request knob. Whether Ollama honors a `keep_alive` extra body field on /v1 was not confirmed.
- Different num_ctx between requests to the same model name generally forces a reload (common knowledge/anecdote; not confirmed in fetched sources) so keep a single num_ctx per model alias.

### Gaps
- Not verified: Ollama issue #11409 current status; whether /v1 accepts keep_alive; exact reload trigger on num_ctx change.

## 4. Same model with different role vs different models for review quality

### Takeaway
Evidence favors an external or different reviewer over intrinsic self-review, but no source tests small local models for cross-model review specifically.

### Cited Findings
- Intrinsic self-correction (no external feedback) often does not improve and can degrade reasoning; reported gains frequently relied on oracle labels; multi-agent debate did not beat self-consistency at equal compute. — [Huang et al., ICLR 2024](https://arxiv.org/pdf/2310.01798)
- Models can fix errors when told where they are; failure is mainly in detecting them. — [summary of Huang et al.](https://liner.com/review/large-language-models-cannot-selfcorrect-reasoning-yet)
- LLM evaluators recognize their own outputs and favor them (self-preference correlates linearly with self-recognition). — [Panickssery et al. 2024](https://arxiv.org/pdf/2404.13076)
- Follow-up: much self-preference is legitimate for strong evaluators; harmful self-preference occurs mainly where the evaluator generates poorly; chain-of-thought before judging reduces it. — [arXiv 2504.03846](https://arxiv.org/html/2504.03846v1)

### Inferences
- A different model family as reviewer reduces self-preference/shared blind spots; a same-model reviewer is cheaper (no swap) and works best when grounded in external signals (tests, lint, diffs, specs) rather than free-form self-critique.
- Practical hybrid: reviewer = smaller/different model that can co-reside, with execution feedback (tests) in the loop.

### Gaps
- No found study on cross-model review with small (<40B) local models; no direct comparison of same-model-different-prompt vs different-model reviewers on code.

## 5. Concurrency: Telegram gateway, cron jobs, subagents hitting one Ollama

### Takeaway
Ollama queues requests FIFO per the FAQ; with MAX_LOADED=1 and mixed model names, concurrent clients can thrash (each model switch evicts the other). Serialize or give each role a fixed model.

### Cited Findings
- Requests for a model that does not fit wait in a queue until it can load; queued in order; limit 512 (503 beyond). — [Ollama FAQ](https://raw.githubusercontent.com/ollama/ollama/main/docs/faq.mdx)
- Same-model concurrent requests are processed in parallel up to NUM_PARALLEL, each multiplying KV cache. — [Ollama FAQ](https://raw.githubusercontent.com/ollama/ollama/main/docs/faq.mdx)
- If the new model needs eviction of a loaded model, a cold start occurs; coexisting models avoid this. — [DEV Community guide](https://dev.to/jovan_chan_9500711396d4e6/ollama-keeps-reloading-the-model-fix-vram-unloading-cold-starts-and-model-swapping-2026-25b7)

### Inferences
- Interleaved Telegram chat (reviewer or chat model) + cron (implementer) can ping-pong models. Mitigations: single role-to-model map used by all clients, one worker/lock for pipeline stages, keep chat model small enough to co-reside, and client timeouts longer than load time.
- Ollama does not appear to unload a model mid-request that is actively serving; unclear if a pending request for another model preempts. Not sourced.

### Gaps
- No source on priority/preemption behavior or fairness among clients.

## 6. Alternatives on Apple Silicon (llama-swap, LM Studio, MLX, vllm-mlx)

### Takeaway
llama-swap is a purpose-built role/model-swapping proxy over OpenAI-compatible servers and is a good fit for explicit role-based swapping; the other options were not researched in depth.

### Cited Findings
- llama-swap picks/starts the upstream server by the request's `model` field and replaces a running different one; works with llama.cpp, vLLM, others; per-model `cmd`, `ttl` auto-unload, `unloadTimeout`; `matrix` DSL lets multiple models run simultaneously; OpenAI (`/v1/chat/completions`, embeddings, responses, models) and Anthropic `/v1/messages` endpoints; macOS install via Homebrew or prebuilt binaries. — [llama-swap README](https://raw.githubusercontent.com/mostlygeek/llama-swap/main/README.md)
- Config example: `models: model1: cmd: llama-server --port ${PORT} --model /path/to.gguf`. — [llama-swap README](https://raw.githubusercontent.com/mostlygeek/llama-swap/main/README.md)

### Inferences
- llama-swap + llama-server lets per-role ctx size (`-c`), parallel slots and TTL be configured in one YAML, and is addressed via the same /v1 interface Hermes already uses; it removes the /v1 num_ctx limitation. Cost: leaves Ollama's model library; runs on host reachable via host.docker.internal.
- MLX/mlx-lm server and vllm-mlx can be wrapped by llama-swap too (it supports arbitrary `cmd` servers per README), which would suit MLX-format models.

### Gaps
- Not researched/sourced: LM Studio's JIT loading/TTL docs, mlx-lm server behavior, vllm-mlx maturity, and Apple Silicon throughput comparisons (MLX vs llama.cpp). Search budget used for the core Ollama questions.
