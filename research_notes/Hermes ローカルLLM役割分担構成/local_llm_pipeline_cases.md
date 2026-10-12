# Hermes Agent / local-LLM role-split pipelines: cases, configs, lessons (as of 2026-10-10)

Scope note: ~12 tool calls. Reddit, HN, YouTube, Zenn/Qiita/note were not reachable via search (no direct hits); GitHub issues, vendor docs and blogs were. Labels: [DOC]=official documentation, [ISSUE]=GitHub issue report (user claim), [BLOG]=third-party write-up, [ANECDOTE]=single user report.

## Q1. Which models work/fail with Hermes tool calling on Ollama; parser/template problems; recommended settings

### Takeaway
Official Hermes docs list only gemma4:31b as reliable for local tool calling; Qwen3.5/3.6 on Ollama has a long-running, still-open tool-call renderer/parser problem; the biggest Ollama-specific trap is num_ctx being silently capped (default 4096 via /v1) while Hermes needs >=64K.

### Cited Findings
- [DOC] Hermes local Ollama guide: only gemma4:31b (~20GB, 24+GB RAM) marked as having tool calling in its table; gemma2:27b/9b and llama3.2:3b marked "No" for tools. Hermes requires at least 64,000 tokens context for agentic work; Ollama default 2048. Fix via Modelfile `PARAMETER num_ctx 64000` + `ollama create`. Raw JSON in reply = server not parsing tool calls (llama.cpp needs `--jinja`; vLLM needs `--enable-auto-tool-choice --tool-call-parser hermes`). Hermes auto-repairs malformed tool calls and falls back to a cloud provider after 3 local failures (`fallback_providers`). `HERMES_API_TIMEOUT=1800` in ~/.hermes/.env; `OLLAMA_KEEP_ALIVE=24h` to avoid 5-min unload. No temperature or qwen guidance on that page. — [Hermes docs](https://hermes-agent.nousresearch.com/docs/guides/local-ollama-setup)
- [ISSUE] #43900 (opened 2026-06-11, still open, P2): Hermes reads context length from GGUF but Ollama's OpenAI-compatible /v1 ignores num_ctx, so models run at 4096; responses become finish_reason="length"/garbled. Reported with Ollama 0.30.7, Hermes 0.16.0 + gemma4:12b, Hermes 0.17.x + gpt-oss:120b. Workarounds confirmed by several users: set `OLLAMA_CONTEXT_LENGTH` server-side, or bake num_ctx into a Modelfile alias; set `max_tokens` in config.yaml for thinking models eating budget. Native /api/chat adapter PR #73285 awaiting maintainer decision (as of 2026-08-19). — [hermes-agent#43900](https://github.com/NousResearch/hermes-agent/issues/43900)
- [BLOG/third-party pkg] `hermes-ollama-native` routes Hermes to native /api/chat as a stopgap; claims /v1 can corrupt streamed tool-call output. — [PyPI](https://pypi.org/project/hermes-ollama-native/)
- [ISSUE] ollama#14493 (opened 2026-02-27, open): Qwen3.5 uses the Hermes-style JSON renderer/parser although trained on Qwen3-Coder XML format; unclosed `</think>` in history corrupts prompts; tool call inside a thinking block not recognized; stalls reported on 0.17.6, 0.18.2 (122B), persisting as of Aug-Sep 2026. One user (35B MLX, 2026-06-12) got better tool-call success switching to qwen3-coder renderer/parser. Maintainer rick-github disputed "completely non-functional". — [ollama#14493](https://github.com/ollama/ollama/issues/14493)
- [ISSUE] ZeroClaw (another harness) with qwen3.5:9b and 35b-a3b: reasoning text but no tool execution; Ollama 500 "expected element type <function> but have <parameter>"; direct API call worked. — [zeroclaw#3079](https://github.com/zeroclaw-labs/zeroclaw/issues/3079)
- [BLOG] Qwen3.6:35b (A3B MoE, Q4_K_M, ~24GB) via Ollama /v1 on 32GB+ Apple Silicon, dated 2026-04-18; no runtime problems or num_ctx reported (setup guide only, not a validation). — [codenote.net](https://codenote.net/ja/posts/hermes-agent-quick-setup-macos-ollama-qwen3-6/)
- [BLOG] Sampling values for Qwen: thinking mode temp ~1.0/top_p 0.95/top_k 20; instruct temp ~0.7/top_p 0.8/top_k 20/presence 1.5; unsourced in article. 2026-08-17. — [MindStudio](https://www.mindstudio.ai/blog/hermes-agent-qwen-local-stack/)
- [BLOG] Third-party: 7-14B Q4_K_M expected to miss tool calls and claim actions never run; models under ~30B may claim a memory save that didn't happen (attributed to Hermes docs, not verified here). — [bestllmfor](https://bestllmfor.com/guides/hermes-agent-local-models/)
- [ANECDOTE/repo] KV cache q8_0 saves ~25% VRAM; q4_0 reported ~92% throughput loss at 64K; `OLLAMA_FLASH_ATTENTION=1` for hangs; `num_predict -1` to stop truncation; Hermes needs aliases with custom Modelfiles. — [HermesAgent-MultiModel](https://github.com/InfiniteWhispers/HermesAgent-MultiModel)
- [DOC] Ollama's own Hermes integration page lists qwen3.6 (~24GB VRAM) as a local option. — [Ollama docs](https://docs.ollama.com/integrations/hermes)

### Inferences
- Fully reliable Qwen3.x tool calling on Ollama is not established; gemma4 (docs) and gpt-oss (repo + #43900 usage) are the better-evidenced picks. Always test with a tool-call smoke test per model/Ollama version.
- Set num_ctx server-side (OLLAMA_CONTEXT_LENGTH or Modelfile) and verify with `ollama ps`; do not trust Hermes' context_length setting.

### Gaps
- No temperature recommendation from official Hermes/Ollama docs. No Reddit/HN threads retrieved. Not verified whether #43900/#73285 resolved after Sept 2026.

## Q2. Minimum viable model size for reviewer vs implementer

### Takeaway
No source gives an empirical reviewer-vs-implementer size threshold. Only generic guidance: <~30B is unreliable for tool use in Hermes; 7-14B Q4 misses tool calls.

### Cited Findings
- [BLOG] Under ~30B or unreliable tool calling, verify memory writes manually; use stronger model for setup. — [bestllmfor](https://bestllmfor.com/guides/hermes-agent-local-models/)
- [BLOG] 8-12B models (Gemma 4 8B, Qwen3.5 27B on 16-32GB Mac) suited to classification, simple edits, routing; cloud for deep multi-step reasoning. — [search summary of haimaker/related blog, unverified]
- [ANECDOTE] Repo uses qwen25-coder-7b only for "fast single-file scripts", ornith-9b for multi-step coding, qwen3-14b-think for planning, gpt-oss-20b as tool-calling default/aggregator. — [HermesAgent-MultiModel](https://github.com/InfiniteWhispers/HermesAgent-MultiModel)
- [BLOG] Same-model self-review shares author's blind spots; acceptable for small factual diffs/style, not for risky or previously failing code. — [Cross-model adversarial review](https://codex.danielvaughan.com/2026/03/28/cross-model-adversarial-review/)

### Inferences
- A reviewer that only reads diffs + test output may be smaller than the implementer, but no evidence; treat as hypothesis. Tool-less reviewer sidesteps tool-call reliability.

### Gaps
- No case found measuring reviewer rubber-stamping with local models.

## Q3. Role-split / model-swap cases and risk-reduction patterns

### Takeaway
Few concrete local planner/implementer/reviewer cases exist; one repo (MoA, 16GB GPU) and Aider's architect/editor split. Hermes delegate_task per-model override has conflicting bug reports.

### Cited Findings
- [ANECDOTE] HermesAgent-MultiModel: Hermes + Ollama on RTX 4080/5080 16GB, task-type routing; MoA = qwen3-14b-think + ornith-9b references, gpt-oss-20b aggregator; do not co-load gpt-oss-20b and qwen3-vl-8b (24.5GB > 16GB); run one at a time (swap). — [repo](https://github.com/InfiniteWhispers/HermesAgent-MultiModel) via [hermesatlas](https://hermesatlas.com/projects/InfiniteWhispers/HermesAgent-MultiModel)
- [DOC] Hermes delegation: `model` param per subagent, defaults to parent; subagents have no parent history, max 3 parallel; only final summary returned. — [docs](https://hermes-agent.nousresearch.com/docs/user-guide/features/delegation)
- [ISSUE] Reports that delegate_task ignores per-task model / always inherits parent (version-specific, 2026): [#46880](https://github.com/NousResearch/hermes-agent/issues/46880), [#49332](https://github.com/NousResearch/hermes-agent/issues/49332), [#12440](https://github.com/NousResearch/hermes-agent/issues/12440), [#17685](https://github.com/NousResearch/hermes-agent/issues/17685), [#10995](https://github.com/NousResearch/hermes-agent/issues/10995) (not individually fetched).
- [DOC/third-party] Aider architect mode: architect model plans, editor model applies; runs linter/tests and re-prompts on failure (tests as objective check). `--architect --editor-model`. — [search results, e.g. computingforgeeks](https://computingforgeeks.com/?p=167356)
- [ANECDOTE] PoC: QwQ-32B architect + Qwen2.5-Coder editor on one GPU with Ollama. — GitHub PoC found via search (URL not captured).
- [DOC] Latency: prefill of system prompt + tool schemas can take minutes on CPU/low VRAM; mitigate with `hermes prompt-size`, disabling toolsets, keep_alive, `/compress`. Swapping between models reloads weights; keep_alive 24h only helps if memory fits. — [Hermes docs](https://hermes-agent.nousresearch.com/docs/guides/local-ollama-setup)

### Inferences
- Single-machine swapping: serialize roles (one model resident), set keep_alive per role, and prefer sequential subagent runs; run tests as the gate (deterministic), cap rounds, human approval before merge. These are supported by general principle above, not by a local-Hermes case.

### Gaps
- No found case: Hermes + Ollama + Telegram gateway with role-split pipeline (only generic Telegram gateway setup guides). No Zenn/Qiita/note article on role split found. No maintained repo of local pipeline role prompts found. No reported data on read-only reviewer toolsets, hallucinated-success rates (beyond the <30B claim), or max-rounds settings.
