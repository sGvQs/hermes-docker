# Hermes Agent: mechanisms for using multiple models in one workflow (implementer -> reviewer with Ollama)

Source tags: [DOCS] official docs page, [ISSUE] GitHub issue, [3RD] third-party blog, [INF] inference. All docs fetched 2026-10-10 via a summarizing fetcher (not raw source); I did not read repo source files (tools/delegate*) directly.

## Q1. Which mechanism is the intended way to run implementer -> reviewer chains, and what are the exact config keys?

### Takeaway
There is no single "pipeline" feature. Two official building blocks exist: (a) `delegate_task` with a single global `delegation.model/provider/base_url` override (one child model for ALL children, no per-task model), and (b) Kanban, where each task is assigned to a profile and can carry its own `--model/--provider`; Kanban is what the docs point to for "a stronger model on a specific task". For a true implementer-then-reviewer chain with two different Ollama models, Kanban (parent/child task links, one profile per role) is the only mechanism with per-role models; delegation alone gives only one fixed child model.

### Cited Findings
- `delegation:` block in `~/.hermes/config.yaml` keys: `max_iterations` (250), `max_concurrent_children` (10), `independent_completions` (false), `worktree_isolation` (false), `max_spawn_depth` (1), `orchestrator_enabled` (true), `model`, `provider`, `base_url`, `api_key`, `api_mode` (`chat_completions`/`codex_responses`/`anthropic_messages`), `request_overrides`, `child_timeout_seconds` (0 = none), `surface_child_process_notifications`, `compression_threshold_tokens` (0=off, min 16000), `oneshot_max_children` (2) — [DOCS Delegation](https://hermes-agent.nousresearch.com/docs/user-guide/features/delegation)
- Local endpoint example for children:
  ```yaml
  delegation:
    model: "qwen2.5-coder"
    base_url: "http://localhost:1234/v1"
    api_key: "local-key"
  ```
  — [DOCS Delegation](https://hermes-agent.nousresearch.com/docs/user-guide/features/delegation)
- Resolution precedence: `base_url` > `provider` > inherit from parent; `model` always applies. There is no per-task model parameter, so one `delegation.model` applies to every child in a batch; docs point to the kanban board for per-task overrides — [DOCS Delegation](https://hermes-agent.nousresearch.com/docs/user-guide/features/delegation)
- `delegate_task` has no `toolsets` parameter; model cannot widen toolsets per call; children get `goal` and `context` only — [DOCS Delegation](https://hermes-agent.nousresearch.com/docs/user-guide/features/delegation)
- Kanban per-task override: `hermes kanban create "hard refactor" --assignee coder --model <m> --provider <p>`; `hermes kanban set-model <task_id> <model> --provider <p>`; `set-model <id> none` clears. `--provider` requires a model. Without override, worker uses its profile's configured model — [DOCS Kanban](https://hermes-agent.nousresearch.com/docs/user-guide/features/kanban)
- Kanban: durable board in `~/.hermes/kanban.db` shared across profiles; each task has one assignee profile, status, parent links; dispatcher (in gateway, `kanban.dispatch_in_gateway: true`, `dispatch_interval_seconds: 60`) promotes `todo` -> `ready` when parents are done and spawns `hermes -p <assignee>` with that profile's HERMES_HOME, each worker an OS process — [DOCS Kanban](https://hermes-agent.nousresearch.com/docs/user-guide/features/kanban)
- Other kanban keys: `failure_limit` (2), `max_in_progress`, `max_in_progress_per_profile`, `dispatch_profiles`, `auto_promote_children` (true), `default_workdir`, `auto_decompose` (true), `orchestrator_profile`, `default_assignee`; aux tasks `auxiliary.kanban_decomposer`, `auxiliary.triage_specifier`, `auxiliary.profile_describer` — [DOCS Kanban](https://hermes-agent.nousresearch.com/docs/user-guide/features/kanban)
- Kanban pattern doc'd: frontier model for orchestrator profile, cheap models in worker profile configs (`~/.hermes/profiles/coder/config.yaml` -> `model.default`); enable board tools with `hermes -p planner tools enable kanban` — [DOCS Kanban](https://hermes-agent.nousresearch.com/docs/user-guide/features/kanban)
- Ollama as provider: `model: {default: qwen2.5-coder:32b, provider: custom, base_url: http://localhost:11434/v1, context_length: 64000}`; named endpoints under top-level `providers:` (`api`, `key_env`, `transport`, `default_model`, `models.<id>.context_length`); legacy `custom_providers:` list migrated by `hermes update` — [DOCS Providers](https://hermes-agent.nousresearch.com/docs/integrations/providers)
- Mixture-of-Agents 2.0 (not a sequential pipeline): per iteration references run first, outputs are appended as private context, one aggregator writes reply/tool calls. Keys `moa.default_preset`, `moa.presets.<name>.reference_models[]` (provider, model), `moa.presets.<name>.aggregator`, `reference_temperature`, `aggregator_temperature`, `max_tokens`, `enabled`; commands `/moa`, `/moa <preset>`, `/model review --provider moa`; aggregator cannot be another MoA preset — [3RD](https://www.tonyreviewsthings.com/hermes-agent-mixture-of-agents-20/) (not verified against official docs)
- `auxiliary.review` pins a model for `/review` — [DOCS Delegation](https://hermes-agent.nousresearch.com/docs/user-guide/features/delegation)

### Inferences
- [INF] Implementer->reviewer with different Ollama models: (1) Kanban: profile `implementer` (model A), profile `reviewer` (model B), create impl task, then review task with parent link to impl task; reviewer worker reads the parent result via board tools. Or (2) set `delegation.model` = reviewer model and let the main (implementer) model call `delegate_task` to review its output: parent = implementer, child = reviewer. Option 2 is simpler and returns the result synchronously to the parent; this inverts "who is fixed" but fits the single-override constraint.
- [INF] Because `delegation.model` is global, a three-role chain (planner/impl/reviewer) via delegation alone is not expressible; use Kanban or profiles.
- [INF] MoA is for parallel opinions per turn, not role-sequenced work; poor fit.

### Gaps
- Did not read `tools/delegate_tool.py` source, so tool parameter schema (goal, context, tasks[], role) is only partially confirmed from the docs summary.
- Whether Kanban worker reviewer can automatically see the implementer's output/artifacts (kanban_* tools semantic: comments/handoff summaries) was not verified; the doc page was truncated at 100k of 135k chars.
- Official docs "multi-agent" / ACP / MCP-based multi-agent setups were not researched; no evidence found either way.

## Q2. Parallel vs sequential; what do subagents share/inherit?

### Takeaway
Delegation batches run in parallel threads; children start with a fresh conversation, inherit toolsets/credentials, and are blind to parent memory writes (memory tool blocked). Kanban workers are separate OS processes with their own profile (own SOUL.md, memory, skills); sequencing is via parent/child task links.

### Cited Findings
- Batches run via `ThreadPoolExecutor` sized to `max_concurrent_children`; concurrency and depth multiply (depth 3 x 3 children = 27 leaves) — [DOCS Delegation](https://hermes-agent.nousresearch.com/docs/user-guide/features/delegation)
- Children inherit parent's enabled toolsets, API key, provider config, credential pool; fresh conversation, only `goal` and `context`; project context files from workspace embedded in system prompt — [DOCS Delegation](https://hermes-agent.nousresearch.com/docs/user-guide/features/delegation)
- Leaf children cannot use `delegate_task`, `clarify`, `memory`, `send_message`, `cronjob`, `start_chat`; both roles keep `execute_code` — [DOCS Delegation](https://hermes-agent.nousresearch.com/docs/user-guide/features/delegation)
- `worktree_isolation: true` gives each child its own git worktree (git + local backend only) — [DOCS Delegation](https://hermes-agent.nousresearch.com/docs/user-guide/features/delegation)
- Profile isolates config.yaml, .env, SOUL.md, memories, sessions, skills, cron jobs, state DB, gateway state; not isolated: filesystem access, real HOME for tool subprocesses (unless `terminal.home_mode: profile`), cwd governed by `terminal.cwd`; "Never point two agent processes at the same profile" — [DOCS Profiles](https://hermes-agent.nousresearch.com/docs/user-guide/profiles)
- Kanban workspaces: `--workspace scratch|worktree|worktree:<path>|dir:<path>` — [DOCS Kanban](https://hermes-agent.nousresearch.com/docs/user-guide/features/kanban)

### Inferences
- [INF] Whether SOUL.md of the parent is inherited by delegated children is not stated; docs only say workspace context files (AGENTS.md etc.) are in child system prompt. Assume children do NOT get parent personality/memory.
- [INF] To make a "sequential" delegate chain, the parent must call `delegate_task` twice in different turns (second call includes first's output in `context`). With a single `delegation.model`, both children use the same model.

### Gaps
- Exact SOUL.md/skills inheritance for delegated children: not stated.
- Whether parent cwd (`terminal.cwd`) is inherited: not stated explicitly.

## Q3. Interaction of per-profile / per-job model settings with a single Telegram gateway

### Takeaway
One gateway process = one profile = one bot token. Multiple profiles need multiple bots/gateways (separate Telegram tokens), though the Kanban dispatcher spawns other profiles' workers from one gateway. Cron jobs can carry their own model; an in-session `/model` switches the active session.

### Cited Findings
- Each profile runs its own gateway process with own bot token in `~/.hermes/profiles/<name>/.env`; `<name> gateway start`; `coder gateway install` creates `hermes-gateway-coder`; token locks block a second gateway using the same token (Telegram, Discord, Slack, WhatsApp, Signal) — [DOCS Profiles](https://hermes-agent.nousresearch.com/docs/user-guide/profiles)
- Official Docker image: s6-overlay supervises per-profile gateways, `hermes -p <name> gateway start/stop/restart` — [DOCS Profiles](https://hermes-agent.nousresearch.com/docs/user-guide/profiles)
- Profile cloning strips channel credentials unless `--clone-channels` — [DOCS Profiles](https://hermes-agent.nousresearch.com/docs/user-guide/profiles)
- Kanban dispatcher runs inside the gateway and spawns `hermes -p <assignee>` workers; `kanban.dispatch_profiles` is a per-home claim allowlist — [DOCS Kanban](https://hermes-agent.nousresearch.com/docs/user-guide/features/kanban)
- Cron per-job: `hermes cron edit <id> --provider <p> --model <m>` or `~/.hermes/cron/jobs.json`; order: per-job pin, then `cron.model`/`cron.model_provider`, then main model; pinned jobs don't use fallback; `--pin` freezes the current main model; `--workdir` loads AGENTS.md/CLAUDE.md; `deliver` accepts `telegram`, `telegram:<chat_id>`, `origin`, `bot-chat:<profile>` — [DOCS Cron](https://hermes-agent.nousresearch.com/docs/user-guide/features/cron)
- `/model custom:qwen-2.5`, `/model custom:local:qwen-2.5` (named provider triple syntax); switches persist base URL/provider to config — [DOCS Providers](https://hermes-agent.nousresearch.com/docs/integrations/providers)
- `/personality [name]` exists in CLI — [DOCS README](https://github.com/NousResearch/hermes-agent)

### Inferences
- [INF] Because `/model` persists to config.yaml (per docs), a /model switch in Telegram likely changes the profile default for later sessions too; per-platform model override was not found.
- [INF] With one Telegram bot, single model per profile; role split must happen behind it (delegation or kanban), which is compatible.

### Gaps
- Per-platform or per-chat model override: no source found.
- "Multi-profile gateway" in a single process: docs describe separate processes only.
- Did not research ACP (`acp_adapter/`) or MCP-based multi-agent; only the directories were seen in the repo listing.

## Q4. Limits: depth, iteration caps, timeouts, token/cost accounting

### Takeaway
Defaults: child iterations 250, concurrency 10, depth 1, no child wall-clock timeout; API timeouts default 1800s. Slow local Ollama models may need timeouts raised. Cost accounting for children was not found.

### Cited Findings
- `max_iterations` 250 per child; exhausting returns `exit_reason: max_iterations`, `truncated: true`; `max_concurrent_children` 10 (env `DELEGATION_MAX_CONCURRENT_CHILDREN`); `max_spawn_depth` 1, no ceiling; `oneshot_max_children` 2 — [DOCS Delegation](https://hermes-agent.nousresearch.com/docs/user-guide/features/delegation)
- `child_timeout_seconds` 0 = none; positive = inactivity cap (30s floor); warning at ~80% idle; heartbeat monitor abandons children frozen 450s idle or 1200s in a tool — [DOCS Delegation](https://hermes-agent.nousresearch.com/docs/user-guide/features/delegation)
- Legacy env defaults: `HERMES_API_TIMEOUT` = 1800s, `HERMES_API_CALL_STALE_TIMEOUT` = 90s, Anthropic native 900s; per-provider `providers.<id>.request_timeout_seconds`, `stale_timeout_seconds`, `models.<model>.timeout_seconds` override env — [DOCS Configuration](https://hermes-agent.nousresearch.com/docs/user-guide/configuration)
- Auxiliary common keys: `provider` ("auto"), `model`, `base_url`, `api_key`, `timeout`, `fallback_chain`, `reasoning_effort`, `extra_body`; named tasks: `vision`, `approval`, `compression` (also `no_progress_timeout`), `title_generation`, `skills_hub`, `mcp`, `triage_specifier`, `tts_audio_tags`, plus mentioned `kanban_decomposer`, `profile_describer`, `voice_chat`, `background_review`, `curator`; legacy `compression.summary_model` auto-migrated — [DOCS Configuration](https://hermes-agent.nousresearch.com/docs/user-guide/configuration)
- Hermes requires >= 64K context; Ollama default num_ctx depends on VRAM (4096 under 24GB); set `OLLAMA_CONTEXT_LENGTH=64000` or Modelfile `PARAMETER num_ctx 64000`; `model.ollama_num_ctx` config key — [DOCS Providers](https://hermes-agent.nousresearch.com/docs/integrations/providers)
- Fallback: `fallback_providers: [{provider, model, base_url, key_env}]`, `hermes fallback add|list|remove|clear`; subagents inherit chain only if unpinned; `delegation.fallback_providers` overrides, `[]` disables; pinned subagent never falls back; fallback is turn-scoped; switching resets prompt cache — [DOCS Fallback](https://hermes-agent.nousresearch.com/docs/user-guide/features/fallback-providers)
- Cron script timeout 3600s (`HERMES_CRON_SCRIPT_TIMEOUT`) — [DOCS Cron](https://hermes-agent.nousresearch.com/docs/user-guide/features/cron)
- Community suggests `HERMES_STREAM_READ_TIMEOUT` for slow local providers — [3RD search snippet](https://hermes-agent.nousresearch.com/docs/reference/faq) (not confirmed from fetched page)

### Inferences
- [INF] Two different local Ollama models on one GPU: Ollama loads/unloads per request (model swap cost); the first call after a swap can be slow, so set generous `child_timeout_seconds`/provider timeouts and consider `OLLAMA_MAX_LOADED_MODELS`/`OLLAMA_KEEP_ALIVE` (Ollama settings, not Hermes; keep_alive not mentioned in Hermes docs).

### Gaps
- Token/cost accounting per child (usage rollup to parent, `/usage`/insights): not found.
- Precise Docker/env var for stream read timeout: unverified.

## Q5. Known issues with delegation + custom/Ollama providers

### Takeaway
Open bugs exist around delegating to Ollama Cloud (context-length misclassification, provider+base_url combination dropping the API key); the issue has low priority (P3) with an open fix PR.

### Cited Findings
- #44207 (open, P3, labels provider/ollama, tool/delegate): `delegation.model: qwen3-coder-next:cloud`, `provider: ollama-cloud`, `base_url: https://ollama.com/v1` fails with "Context length exceeded (1,584 tokens)". Reviewer analysis: error reachable only after a provider 400/overflow-pattern; suspect inherited parent `max_tokens`; reporter's later theory: provider+base_url forces "custom", dropping `OLLAMA_API_KEY`, yielding 401 misclassified as context overflow; workaround: remove `delegation.base_url`. Linked PR #89138 open; `delegation.context_length` is not an existing key — [ISSUE #44207](https://github.com/NousResearch/hermes-agent/issues/44207)
- #44214 closed as duplicate of #44207; PR #14303 (allow local providers like Ollama/LM Studio in delegation without API keys) closed as duplicate, merge status not shown — [GitHub issue search](https://github.com/NousResearch/hermes-agent/issues?q=delegat+ollama)
- Issue #17685 referenced: per-task `model` override ignored in delegation (described as a different bug) — [ISSUE #44207](https://github.com/NousResearch/hermes-agent/issues/44207)
- Third-party claim: delegation does not inherit fallback config unless set — consistent with official [DOCS Fallback](https://hermes-agent.nousresearch.com/docs/user-guide/features/fallback-providers)

### Inferences
- [INF] For local Ollama (`provider: custom`) use `delegation.base_url: http://host.docker.internal:11434/v1` (inside Docker) + `model` + any dummy `api_key`; avoid also setting a named built-in provider. Ensure child model has >=64K num_ctx or compression/overflow errors appear.
- [INF] In the user's docker-compose setup, the Ollama host address must be reachable from the Hermes container (not localhost).

### Gaps
- Only issues matching "delegat ollama" were checked; other related issues (profiles, kanban + Ollama) were not searched.
- Whether #44207 fix (PR #89138) has landed in the latest release: unknown.
