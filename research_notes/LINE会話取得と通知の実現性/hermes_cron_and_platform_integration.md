# Hermes Agent: cron, platform adapters (LINE), webhook/API server, MCP and division of labour

Source tags: [docs] = official docs page, [source] = repo source file, [inference] = my reasoning, [issue] = GitHub issue (none consulted). Docs fetched were summarised by a fetch tool, so exact wording may be paraphrased; verify keys against the live docs before use. Docs version was not stated on the pages; "~v0.21" not independently confirmed.

## 1. Platforms, LINE adapter, generic webhook platform

### Takeaway
Hermes ships a bundled LINE platform plugin (`plugins/platforms/line/`) plus a separate generic `webhook` platform. LINE needs a public HTTPS URL for the Messaging API webhook. Once a group/room ID is allowlisted, the adapter processes every message in that group (no mention required), but it has no history-fetch API, so it only sees messages arriving after it is running.

### Cited Findings
- Supported platforms listed in Messaging docs: Telegram, Discord, Slack, Google Chat, WhatsApp, WhatsApp Cloud API, Signal, Mattermost, Matrix; Microsoft Teams, Feishu/Lark, DingTalk, WeCom, WeCom Callback, Weixin, QQ, Yuanbao; BlueBubbles, Photon (iMessage); Email, SMS, Home Assistant (plugin), LINE, ntfy, SimpleX, IRC, Buzz, Raft. Also "Webhooks" and "API Server" entries. — [docs](https://hermes-agent.nousresearch.com/docs/user-guide/messaging)
- LINE is a bundled platform plugin at `plugins/platforms/line/` (files: `__init__.py`, `adapter.py`, `plugin.yaml`); enable with `enabled: true` under `gateway.platforms.line` in `~/.hermes/config.yaml`. — [docs](https://hermes-agent.nousresearch.com/docs/user-guide/messaging/line); [source](https://github.com/NousResearch/hermes-agent/tree/main/plugins/platforms/line)
- LINE env vars (in `~/.hermes/.env`): `LINE_CHANNEL_ACCESS_TOKEN` (required), `LINE_CHANNEL_SECRET` (required, signature verification), `LINE_HOST`, `LINE_PORT` (default 8646), `LINE_PUBLIC_URL` (needed for image/voice/video sends), `LINE_ALLOWED_USERS` (U...), `LINE_ALLOWED_GROUPS` (C...), `LINE_ALLOWED_ROOMS` (R...), `LINE_ALLOW_ALL_USERS` (default false, dev only), `LINE_HOME_CHANNEL`, `LINE_SLOW_RESPONSE_THRESHOLD` (45s), `LINE_PENDING_TEXT`, `LINE_BUTTON_LABEL`, `LINE_DELIVERED_TEXT`, `LINE_INTERRUPTED_TEXT`, `LINE_EXPIRED_TEXT`. At least one allowlist is required. — [docs](https://hermes-agent.nousresearch.com/docs/user-guide/messaging/line)
- Webhook path is `/line/webhook` (full URL `https://<your-tunnel>/line/webhook`), health at `/line/webhook/health`. Docs say use a public HTTPS tunnel (Cloudflare Tunnel, ngrok, devtunnel); a fixed named tunnel is preferred. In LINE console: paste URL, Verify, enable "Use webhook". — [docs](https://hermes-agent.nousresearch.com/docs/user-guide/messaging/line)
- 1:1 chats get a response to every message; group (C) and room (R) chats get a response only if the ID is allowlisted. Typing indicator is 1:1 only. — [docs](https://hermes-agent.nousresearch.com/docs/user-guide/messaging/line)
- Adapter source: after the allowlist gate, `_dispatch_event` drops only redelivered events, the bot's own messages, and unauthorized sources. No mention requirement, no buffering, no `require_mention`-style key, no message-history API (only media content fetch by message ID). — [source](https://raw.githubusercontent.com/NousResearch/hermes-agent/main/plugins/platforms/line/adapter.py) (summarised by fetch tool)
- Outbound: reply token first (free, single-use, ~60 s), falls back to metered Push API. Text bubble max 5000 chars (split ~4500, up to 5 bubbles per call); no edit API; markdown stripped. — [docs](https://hermes-agent.nousresearch.com/docs/user-guide/messaging/line)
- Cron with `deliver: line` uses a standalone Push-only sender to `LINE_HOME_CHANNEL`. — [docs](https://hermes-agent.nousresearch.com/docs/user-guide/messaging/line)
- Generic webhook platform: default port 8644 (`WEBHOOK_PORT` or `platforms.webhook.extra.port`), POST `/webhooks/<route-name>`, `GET /health`. Config under `platforms.webhook.extra`: `port`, `host`, `secret`, `routes`, `rate_limit`, `max_body_bytes`; per route: `events`, `secret`, `profile`, `prompt`, `filters`, `script`, `skills`, `toolsets`, `deliver`, `deliver_extra`, `deliver_only`, `cron_job`, `coalesce`, `mirror_to_session`. — [docs](https://hermes-agent.nousresearch.com/docs/user-guide/messaging/webhooks)
- Webhook toolset `hermes-webhook` includes the full toolset including terminal. — [docs](https://hermes-agent.nousresearch.com/docs/user-guide/messaging)

### Inferences
- LINE Messaging API webhooks only deliver events for chats the bot account has joined; the bot must be invited to the group. A user's personal LINE account/other chats cannot be read (LINE platform property, not from these docs) [inference].
- Because there is no history API, messages sent while the gateway/tunnel is down are likely lost unless LINE redelivers; "hourly scan" must therefore be built on messages captured as they arrive (webhook), not pulled on demand [inference].
- Whether LINE-received messages in a group are stored as session history that cron could read is not documented; a safer pattern is an external receiver that appends to a JSONL file [inference].
- LINE adapter replies to every allowlisted group message by default: a passive "listen only" bot is not an obvious config option; the agent would answer in the group unless prompt/other settings suppress it [inference from "no require_mention key"].

### Gaps
- Not documented: a "read-only/no-reply" mode for LINE groups; LINE group message persistence for later querying.
- Did not check GitHub issues for LINE adapter bugs.
- Exact platform count/version for v0.21 not confirmed.

## 2. Cron / scheduled jobs

### Takeaway
Cron is executed by the gateway daemon (so inside the gateway container), ticks every 60 s, supports `every 1h` / `0 * * * *`, per-job model/provider/workdir/toolsets/deliver, a `[SILENT]` suppression marker, and a script-gated `wakeAgent:false` / `no_agent` mode that skips the LLM entirely when there is nothing new.

### Cited Findings
- CLI: `hermes cron <list|create|edit|pause|resume|run|remove|status|runs|incidents|doctor|tick>`; `create` has alias `add`, flags include `--skill` (repeatable), `--workdir`, `--provider`, `--model`, `--reasoning-effort`. Positional args of `create` are not shown in the docs; use `hermes cron create --help`. — [docs](https://hermes-agent.nousresearch.com/docs/reference/cli-commands); [docs](https://hermes-agent.nousresearch.com/docs/user-guide/features/cron)
- Schedule formats: `in 30m`; `30m`; `every 30m`; `every 2h`; `every hour`; `every day at 9am`; `weekdays at 9am`; cron expr e.g. `0 */6 * * *`, `0 9 * * 1-5`; ISO timestamp for one-shot. Intervals/cron repeat until removed. — [docs](https://hermes-agent.nousresearch.com/docs/user-guide/features/cron)
- Job fields: `action`, `schedule`, `prompt` (fresh session, must be self-contained), `name`, `skill`/`skills`, `workdir` (absolute existing dir; loads its AGENTS.md/CLAUDE.md/.cursorrules, file/terminal tools run there), `enabled_toolsets` (e.g. `["web","file"]`), `deliver`, `model`/`provider`, `pinned`, `no_agent`, `script` (inside `$HERMES_HOME/scripts/`), `context_from`, `continuity`, `attach_to_session`. Model resolution: per-job pin -> `cron.model` in config.yaml -> main agent model. — [docs](https://hermes-agent.nousresearch.com/docs/user-guide/features/cron)
- Delivery targets: `origin`, `local`, `telegram` (home channel via `TELEGRAM_HOME_CHANNEL`), `telegram:123456`, `telegram:-100123:17585` (chat:thread), `discord:#name`, other platform names for home channels, `bot-chat`, `all`, comma fan-out e.g. `telegram,discord`. — [docs](https://hermes-agent.nousresearch.com/docs/user-guide/features/cron)
- `[SILENT]`: if the final response contains `[SILENT]`, delivery is suppressed; output still saved to `~/.hermes/cron/output/`; failed runs always deliver. — [docs](https://hermes-agent.nousresearch.com/docs/user-guide/features/cron)
- `no_agent` jobs deliver script stdout verbatim; empty stdout = silent tick; non-zero exit/timeout sends error alert; last stdout line `{"wakeAgent": false}` skips the agent for that tick. — [docs](https://hermes-agent.nousresearch.com/docs/user-guide/features/cron)
- "Cron execution is handled by the gateway daemon"; checks due jobs every 60 s; jobs in `~/.hermes/cron/jobs.json`, output `~/.hermes/cron/output/{job_id}/{timestamp}.md`. Cron sessions cannot create jobs unless `cron.allow_agent_scheduling`. — [docs](https://hermes-agent.nousresearch.com/docs/user-guide/features/cron)
- Docker: image `nousresearch/hermes-agent`, mount `~/.hermes` to `/opt/data`, run `gateway run`; never run two gateways on the same data dir; unattended gateway/cron sessions enable tool-loop hard stops by default. — [docs](https://hermes-agent.nousresearch.com/docs/user-guide/docker)
- Docs caution: a gateway started without the shell's env may use different `auth.json`/`.env` (HOME/HERMES_HOME mismatch). Pre-run script timeout default 3600 s. — [docs](https://hermes-agent.nousresearch.com/docs/user-guide/features/cron)

### Inferences
- Hourly: `every 1h` or `0 * * * *` (the latter aligns to the hour; `every 1h` is relative to creation/last run) [inference].
- Best fit for the use case: `script` that reads the day's messages file, applies deterministic filters, prints `{"wakeAgent": false}` when nothing is new, otherwise prints data for the LLM to judge; LLM answers `[SILENT]` when nothing is worth notifying. Exactly how script stdout is injected into the prompt is not shown in the fetched text [inference].

### Gaps
- `hermes cron create` positional syntax not in docs fetched (use `--help`).
- Whether `[SILENT]` must be the entire response or just contained: docs say "contains".
- Reliability of `[SILENT]` from small local models not documented.

## 3. Webhook platform / API server / one-shot CLI (pushing data in, delivering to Telegram)

### Takeaway
Three entry points: webhook routes (HMAC-signed POST, can deliver to Telegram, optionally LLM-free via `deliver_only`), OpenAI-compatible API server on 8642 (Bearer key; also has `/api/jobs` and `/v1/runs`), and `hermes chat -Q -q` / `--query-file` for one-shot CLI.

### Cited Findings
- Webhook signatures: GitHub `X-Hub-Signature-256`; GitLab `X-Gitlab-Token`; Standard Webhooks; Generic V2 `X-Webhook-Signature-V2` + `X-Webhook-Timestamp` (HMAC-SHA256 of `<timestamp>.<body>`, ±300 s); V1 legacy. Every route needs a secret; `INSECURE_NO_AUTH` accepted only on loopback. — [docs](https://hermes-agent.nousresearch.com/docs/user-guide/messaging/webhooks)
- Route `deliver` options include `telegram` (home channel unless `deliver_extra.chat_id`), `log` default. Prompt templates use `{dot.notation}`; `{__raw__}` dumps payload truncated at 4000 chars. Defaults: 30 req/min per route, 1 MB body. — [docs](https://hermes-agent.nousresearch.com/docs/user-guide/messaging/webhooks)
- `deliver_only: true` skips the agent and sends the rendered prompt literally (200 ok / 502 on target reject); `deliver` must not be `log`; cannot combine with `cron_job`. — [docs](https://hermes-agent.nousresearch.com/docs/user-guide/messaging/webhooks)
- Route key `cron_job` exists (purpose not explained in fetched text). — [docs](https://hermes-agent.nousresearch.com/docs/user-guide/messaging/webhooks)
- API server: enable `API_SERVER_ENABLED=true`; port `API_SERVER_PORT` default 8642; host `API_SERVER_HOST` default 127.0.0.1; auth `API_SERVER_KEY` required (Bearer); config under `gateway.api_server:` (`enabled`, `port`, `host`, `key`, `cors_origins`, `model_name`, `max_concurrent_runs`, ...). Endpoints: `POST /v1/chat/completions`, `/v1/responses`, `/v1/runs` (+events SSE, stop, approval), `/api/jobs` CRUD with `/pause` `/resume` `/run`, `/api/sessions/*`. — [docs](https://hermes-agent.nousresearch.com/docs/user-guide/features/api-server)
- In Docker, to reach it externally set `API_SERVER_HOST=0.0.0.0`, key >= 8 chars, publish `-p 8642:8642`. — [docs](https://hermes-agent.nousresearch.com/docs/user-guide/docker)
- One-shot CLI: `hermes chat -Q -q "<prompt>"`, `hermes chat --oneshot -q`, `--query-file <PATH|->`, `--format stream-json` (JSONL). — [docs](https://hermes-agent.nousresearch.com/docs/reference/cli-commands)

### Inferences
- Simplest external push: external script POSTs a signed JSON to a webhook route with `deliver: telegram`; or script calls `/v1/chat/completions` and sends the answer itself to Telegram Bot API (bypassing Hermes delivery) [inference].
- If the external script already makes the notify decision deterministically, `deliver_only` or a direct Telegram Bot API call avoids the LLM entirely [inference].
- Webhook agent-run responses go to `deliver`; whether an agent can answer `[SILENT]` on webhook routes is not documented [gap].

### Gaps
- Whether `/api/jobs` payload supports the same fields (script, wakeAgent) as `hermes cron`: not verified.
- Webhook-route-triggered LLM runs and `[SILENT]` suppression: not documented in fetched text.

## 4. MCP

### Takeaway
Hermes is an MCP client configured under `mcp_servers` in `~/.hermes/config.yaml`; stdio and HTTP servers supported, so `line/line-bot-mcp-server` (stdio via npx) can in principle be attached, but it is a send/push-oriented server and cannot read group history.

### Cited Findings
- Stdio: `command`, `args`, `env`, `timeout`, `connect_timeout`; HTTP: `url`, `headers`; filtering `tools.include` / `tools.exclude` (include wins), `enabled: false`. Only explicitly configured `env` plus a safe baseline is passed to stdio servers. — [docs](https://hermes-agent.nousresearch.com/docs/user-guide/features/mcp)
- Example form:
```yaml
mcp_servers:
  my_tools:
    command: "uvx"
    args: ["some-mcp-package"]
    env: { API_BASE: "https://example.test" }
    tools: { include: [search_docs] }
```
— [docs](https://hermes-agent.nousresearch.com/docs/user-guide/features/mcp)
- Docker specifics for MCP not on that page. — [docs](https://hermes-agent.nousresearch.com/docs/user-guide/features/mcp)

### Inferences
- line-bot-mcp-server exposes Messaging API calls (push/broadcast/profile etc.); the LINE Messaging API has no endpoint to list past group messages, so the MCP cannot supply "scan messages" [inference; I did not fetch the LINE MCP repo].
- stdio MCP via npx requires Node in the Hermes image; unverified [inference].

### Gaps
- Did not verify line-bot-mcp-server tool list or whether the Hermes image ships Node/npx.

## 5. Persistence / dedup / "only today's messages"

### Takeaway
Docs recommend script-gated runs with your own state files (not Hermes' `state.db`) and `continuity=true` for prompt-level dedup; deterministic dedup in files is more reliable.

### Cited Findings
- Script can gate on file mtime vs a saved state file, readiness flag, or new rows in your own DB; do not query Hermes's internal `state.db` (schema changes). `continuity=true` feeds the job its previous output, suggested for "items not already covered". — [docs](https://hermes-agent.nousresearch.com/docs/user-guide/features/cron)
- Run outputs persisted at `~/.hermes/cron/output/{job_id}/`; `~/.hermes` is the volume at `/opt/data` in Docker. — [docs](https://hermes-agent.nousresearch.com/docs/user-guide/features/cron); [docs](https://hermes-agent.nousresearch.com/docs/user-guide/docker)

### Inferences
- Pattern: receiver appends JSONL (id, ts, text) to `/opt/data/...`; pre-run script filters by today's date (JST), drops IDs already in `notified.json`, writes IDs after successful judgement; LLM only classifies [inference].
- Hermes "memory" is not documented here as a dedup store; avoid for exactly-once logic [inference].

### Gaps
- Memory feature semantics not researched.

## 6. Local models and determinism

### Takeaway
The fetched docs contain no local-model tool-call reliability guidance; the recommendation to keep rules in code is inference, supported by Hermes's own design (script gate, `no_agent`, `deliver_only`).

### Cited Findings
- Hermes offers LLM-free paths: `no_agent` cron, `wakeAgent:false`, webhook `deliver_only`. — [docs](https://hermes-agent.nousresearch.com/docs/user-guide/features/cron); [docs](https://hermes-agent.nousresearch.com/docs/user-guide/messaging/webhooks)
- Cron page does not cover tool-loop limits or local model advice (fetch tool reported); Docker page says unattended sessions enable tool-loop hard stops by default. — [docs](https://hermes-agent.nousresearch.com/docs/user-guide/docker)

### Inferences
- Make the LLM a pure function: script passes pre-filtered, pre-numbered messages in the prompt (no tool use needed), require a fixed output format or `[SILENT]`; restrict `enabled_toolsets` to minimal to reduce tool-call failure surface; consider per-job `model`/`provider` pin [inference].
- Small local models may fail to emit exact `[SILENT]`; a deterministic post-check is not available in cron itself, so consider doing judgement via API/`hermes chat -Q` from an external script that parses JSON and sends Telegram itself [inference].

### Gaps
- No issues/docs found on local-model reliability.

## 7. Security

### Takeaway
Gateway is deny-by-default with per-platform allowlists; a public tunnel exposes only the LINE webhook port, protected by LINE signature verification, but the Hermes agent behind it has full tools, so allowlists and tight toolsets matter.

### Cited Findings
- Default: unknown users denied unless allowlisted or paired; env allowlists `TELEGRAM_ALLOWED_USERS`, `DISCORD_ALLOWED_USERS`, `SIGNAL_ALLOWED_USERS`, `GATEWAY_ALLOWED_USERS`; `GATEWAY_ALLOW_ALL_USERS=true` not recommended; pairing codes via `hermes pairing approve` expire after 1 h; config `platforms.<name>.extra.allow_from`. — [docs](https://hermes-agent.nousresearch.com/docs/user-guide/messaging)
- LINE: channel secret used for signature verification; allowlists `LINE_ALLOWED_USERS/GROUPS/ROOMS`. — [docs](https://hermes-agent.nousresearch.com/docs/user-guide/messaging/line)
- API server grants full toolset incl. terminal; binds 127.0.0.1 by default; key required. Webhook: signature proves sender not content safety; treat payload as untrusted (prompt injection), prefer narrow templates. — [docs](https://hermes-agent.nousresearch.com/docs/user-guide/features/api-server); [docs](https://hermes-agent.nousresearch.com/docs/user-guide/messaging/webhooks)
- Docs warn opening any port on an internet-facing machine is a security risk; dashboard non-loopback needs auth. — [docs](https://hermes-agent.nousresearch.com/docs/user-guide/docker)

### Inferences
- Use a named Cloudflare Tunnel mapped only to `/line/webhook` (path restriction) or a tiny external receiver that verifies `X-Line-Signature` and forwards locally; keep 8642/8644 unpublished (bind to localhost) [inference].
- Group chat text is attacker-controllable input to an agent with terminal tools: prompt injection risk; restrict `enabled_toolsets` for the judging job [inference].
- Quick-tunnel (ngrok free/trycloudflare) URLs change on restart, breaking the LINE webhook setting [docs-implied: "fixed named tunnel is better"].

### Gaps
- Cloudflare Tunnel/ngrok-specific risks not researched beyond the above.

## Recommended division of labour (inference)
1. Thin external receiver (or Hermes LINE adapter) captures LINE events to JSONL; verify signature.
2. Hermes cron hourly (`0 * * * *`), `script` for deterministic filter (today JST, undeduped, keyword rules) with `wakeAgent:false` when empty.
3. LLM (Hermes cron prompt, minimal toolsets) judges importance only; reply `[SILENT]` if none; `deliver: telegram`.
4. Script/state file records notified IDs. If local model proves flaky, move judgement into an external script calling the API server and posting to Telegram directly.
