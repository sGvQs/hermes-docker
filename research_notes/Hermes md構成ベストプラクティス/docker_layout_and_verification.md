# Hermes Agent Docker layout, seeding, profiles and verification (as of 2026-10)

Legend: [docs] = official docs page text, [source] = raw repo file read, [issue] = GitHub issue, [community] = third-party, [inference] = my reasoning.
Note: WebFetch summaries were produced by a small model over each page; quotes are paraphrase-level unless in quotation marks. Two long pages (CLI reference, Configuration) were only partially read.

## 1. What the official Docker docs recommend (persistence / customization)

### Takeaway
All state lives in one host dir mounted at /opt/data (HERMES_HOME). The image (/opt/hermes) is immutable and root-owned. Docs seed only .env, config.yaml, SOUL.md on first boot; AGENTS.md is NOT mentioned or seeded by the Docker docs or stage2 script. The docs recommend no git/read-only-mount/backup procedure; those are inference.

### Cited Findings
- Volume: `-v ~/.hermes:/opt/data`; contents: `.env`, `config.yaml`, `SOUL.md`, `sessions/`, `memories/`, `skills/`, `home/`, `cron/`, `hooks/`, `logs/`, `skins/`, `profiles/<name>/`, `installs/` — [Docker docs](https://hermes-agent.nousresearch.com/docs/user-guide/docker)
- First boot: stage2 hook seeds `.env`, `config.yaml`, `SOUL.md` and syncs bundled skills; config migrations run on boot with timestamped backups of config.yaml/.env; `HERMES_SKIP_CONFIG_MIGRATION=1` skips — [Docker docs](https://hermes-agent.nousresearch.com/docs/user-guide/docker)
- Seed logic [source]: `if [ ! -f "$HERMES_HOME/$dest" ] && [ -f "$INSTALL_DIR/$src" ]` ; `.env`<-`.env.example`, `config.yaml`<-`cli-config.yaml.example`, `SOUL.md`<-`docker/SOUL.md`; refuses symlinked destinations; script never references AGENTS.md — [stage2-hook.sh](https://raw.githubusercontent.com/NousResearch/hermes-agent/main/docker/stage2-hook.sh)
- stage2 modes [source]: `.env` 600 (re-applied each boot), `config.yaml` 640, `auth.json` 600; creates dirs backups, cron, sessions, logs, logs/gateways, hooks, memories, skills, skins, plans, workspace, home, pairing — same stage2-hook.sh
- Recursive chown only on allowlist (cron, sessions, logs, hooks, memories, skills, skins, plans, workspace, home, profiles, pairing); top-level HERMES_HOME chowned non-recursively; /opt/hermes never chowned — same stage2-hook.sh. Implication: a user-created file/dir at top level of /opt/data (e.g. AGENTS.md) is not auto-chowned.
- UID/GID [source]: `HERMES_UID`/`HERMES_GID`, fallback `PUID`/`PGID`, numeric 1..65534, runs usermod/groupmod only if different; arbitrary `--user` UID errors exit 1. Docs example: `-e PUID=1000 -e PGID=10 -v /volume1/docker/hermes:/opt/data` — stage2-hook.sh; [Docker docs]
- Docs: "Do not make the whole data tree world-readable. It contains credentials."; outside containers HERMES_HOME forced 0700, inside containers modes left alone, `HERMES_HOME_MODE=0755` forces; `docker exec` drops to hermes user (`HERMES_DOCKER_EXEC_AS_ROOT=1` keeps root); gateway as root refused unless `HERMES_ALLOW_ROOT_GATEWAY=1` — [Docker docs]
- Secrets: `-e ANTHROPIC_API_KEY=...` overrides .env values (CI/secret managers) — [Docker docs]
- Never run two gateway containers against the same data dir; for separate containers per profile give each its own `~/.hermes-<name>` volume — [Docker docs]
- Compose example in docs: image `nousresearch/hermes-agent:latest`, `command: gateway run`, ports 8642/9119, `volumes: ~/.hermes:/opt/data`, `HERMES_DASHBOARD=1`; `docker compose up -d`, `docker compose logs -f` — [Docker docs]
- Pitfalls [docs]: `--shm-size=1g` for browser tools; overriding `entrypoint:` makes Hermes PID 1 (zombies) -> keep default or `init: true`; images before late Aug 2026 had /opt/hermes 0700 (`docker exec -u root hermes chmod 0755 /opt/hermes`); `localhost` inside container is Hermes itself; `docker rm` discards writable-layer installs (use derived image); state.db WAL can corrupt on virtiofs/9p (Docker Desktop macOS, OrbStack): new DBs fall back to DELETE journal; for existing: `docker exec hermes python3 -c "import sqlite3; print(sqlite3.connect('/opt/data/state.db').execute('PRAGMA journal_mode=DELETE').fetchone()[0])"` then `database.journal_mode: delete` in config.yaml; or use a named volume — [Docker docs]
- Logs: `docker logs` and `${HERMES_HOME}/logs/gateways/<profile>/current` — [Docker docs]
- Community entrypoint (archived 2026-08-02) also seeds SOUL.md/.env/config.yaml only if absent; says HERMES_HOME must be same absolute path host/container if using docker terminal backend with sibling sandboxes; non-root requires host dir writable by UID:GID — [mehalter/hermes-agent-docker](https://code.mehalter.com/mehalter/hermes-agent-docker) [community, not the official image]
- hermes-webui compose notes: `HERMES_HOME_MODE` meaning differs per image; a 0640 dir has no owner-x so agent can't traverse; use 0750/0701 for multi-container; `sudo docker compose` may resolve `${HOME}` to /root; don't run agent+dashboard containers on same volume (stale gateway_state.json) — [hermes-webui docker.md](https://github.com/nesquena/hermes-webui/blob/master/docs/docker.md) [community]

### Inferences
- Reproducible seeding: because seed only happens if file is absent, a git-tracked config repo should be applied by copying/mounting AFTER or INSTEAD of the seed (pre-populate host ~/.hermes before first `up`, or bind-mount single files). Single-file bind mounts over /opt/data/SOUL.md work with seed-if-missing (file exists), but note `.env` chmod 600 / config.yaml chmod 640 / chown by stage2 would act on the host file through the bind (and migrations rewrite config.yaml; a `:ro` mount of config.yaml may make migrations/backups fail — untested). Safer: keep config in a repo and copy into the volume with an init step.
- Bind-mounting a non-existent host file creates a directory (general Docker behavior); stage2's `[ -f ]` check would then see a directory and (SOUL.md) treat it as missing/skip; create the file on host first. Not documented by Hermes.
- Backups: stop container (or use the DELETE journal mode) and tar the volume; exclude/encrypt `.env` and `auth.json`. Not documented officially (docs page has no backup section).
- Suggested split: image = /opt/hermes (never edit); data volume = runtime state (sessions, state.db, memories, skills, cron, logs); repo = SOUL.md, config.yaml (secrets via `${VAR}` and `.env`), AGENTS.md, compose.yml, `.env.example`; `.env` and auth.json gitignored.

### Gaps
- No official statement on mounting individual files read-only, init-script patterns, or git-tracking the config.
- Hermes-specific official backup guidance not found (`hermes backup` CLI command may exist; not verified).

## 2. Profiles (HERMES_PROFILE, multi-profile)

### Takeaway
A profile is a separate HERMES_HOME at `~/.hermes/profiles/<name>/` (in Docker: `/opt/data/profiles/<name>/`), default profile is the root HERMES_HOME. Selecting a profile is `hermes -p <name>`; I could NOT confirm an env var named HERMES_PROFILE in docs.

### Cited Findings
- Each profile is `~/.hermes/profiles/<name>/`; default is `~/.hermes`; a dir counts as a profile only if it has one of `config.yaml`, `.env`, `SOUL.md`, `profile.yaml`, `auth.json`, `state.db`; HERMES_HOME is "the profile boundary"; selectors: `hermes -p <name>`, `--profile=<name>`, `hermes profile use <name>` (sticky) — [Profiles docs](https://hermes-agent.nousresearch.com/docs/user-guide/profiles)
- Per profile: config.yaml, .env, SOUL.md, memories/ (MEMORY.md, USER.md), sessions/state.db, skills, cron, plugins, logs, gateway PID — same page
- Docker: `docker exec hermes hermes profile create coder` registers s6 slot `/run/service/gateway-coder/`; `docker exec hermes hermes -p coder gateway start|stop|restart|status`; profiles restore on container restart if last state was running; each profile gets its own port via its own `.env` (`API_SERVER_PORT=8643`), not the container-wide `environment:`; per-profile logs `logs/gateways/<profile>/current` — [Docker docs]
- Same bot token on two profiles is blocked by token locks (Telegram, Discord, Slack, WhatsApp, Signal); multiplexed gateway exists (`gateway.multiplex_profiles`, `profile_routes`), details not read; under multiplex `${VAR}` in config.yaml resolves against that profile's own `.env` — Profiles docs; [Configuration docs](https://hermes-agent.nousresearch.com/docs/user-guide/configuration)
- Profiles isolate Hermes state, not the filesystem; set absolute `terminal.cwd` — Profiles docs
- stage2 chowns `profiles/` recursively every boot; each `profiles/*/.env` gets the Nous routing-override sync — [source]

### Inferences
- A per-profile SOUL.md goes at `/opt/data/profiles/<name>/SOUL.md`; AGENTS.md is NOT per-profile by location (it comes from terminal.cwd), so per-profile AGENTS.md is achieved by setting a different `terminal.cwd` in each profile's config.yaml.
- Whether `HERMES_PROFILE` env works: not found in docs/entrypoint; treat as unverified, prefer `hermes -p`, or set `HERMES_HOME=/opt/data/profiles/<name>`.

### Gaps
- HERMES_PROFILE env var: no confirmation (GitHub code search requires login).

## 3. AGENTS.md / SOUL.md loading rules (critical for layout)

### Takeaway
SOUL.md only from HERMES_HOME/SOUL.md (never cwd). AGENTS.md from the working directory (for gateway: configured `terminal.cwd`), NOT from HERMES_HOME. So putting AGENTS.md in /opt/data is ineffective unless terminal.cwd points there.

### Cited Findings
- SOUL.md loaded only from `HERMES_HOME/SOUL.md`, not searched in project; starter file auto-created, never overwrites — [Context files docs](https://hermes-agent.nousresearch.com/docs/user-guide/features/context-files); [Personality docs](https://hermes-agent.nousresearch.com/docs/user-guide/features/personality)
- Project context priority, first match wins: `.hermes.md`/`HERMES.md` > `AGENTS.override.md` > `AGENTS.md` > `CLAUDE.md` > `.cursorrules`; in a git repo AGENTS.md files merge from git root down to cwd; outside git only cwd — Context files docs
- Subdirectory hints: when tools touch paths, checks dir + up to 5 parents for AGENTS.md/CLAUDE.md/.cursorrules, appended to tool results not system prompt, 32,000 char cap — Context files docs
- Size: `context_file_max_chars` (default scales, floor 20,000, ceiling 500,000); head 70%/tail 20% truncation; `context_file_read_timeout` 5s — Context files docs
- Maintainer statement (closed issue #37480, Jul 2026): "SOUL.md is sourced from HERMES_HOME, while AGENTS.md is sourced from the configured working directory"; gateway cwd = `terminal.cwd` bridged to `TERMINAL_CWD`; shipped in v2026.7.1; reporter on v0.15.1 saw default identity on Telegram — [Issue #37480](https://github.com/NousResearch/hermes-agent/issues/37480). Reporter's resolution was never confirmed; closed "not planned".
- Config docs: `terminal.cwd` is "Gateway/cron working directory (CLI always uses launch dir)" — Configuration docs
- Docker image has `/opt/data/workspace` dir created by stage2 [source] — plausible place for a workspace repo/AGENTS.md (inference).

### Inferences
- Recommended Docker layout: mount project/workspace repo at e.g. `/opt/data/workspace` (or `/workspace`), set `terminal.cwd: /opt/data/workspace` in config.yaml, keep AGENTS.md in that repo root (git root so merge rules work); keep SOUL.md in the volume root (or mounted file).
- A `.hermes.md` in the same dir would override AGENTS.md entirely (only one type loads).

### Gaps
- Default value of terminal.cwd in the Docker gateway not found.

## 4. Verifying what was actually loaded

### Takeaway
Use `/context` (lists context files and loaded/truncated/skipped status) and `hermes prompt-size` (note: the command is `prompt-size`, not `prompt`). No documented command dumps the full rendered system prompt.

### Cited Findings
- `/context [all]` (alias `/ctx`): per-category token estimate; lists .hermes.md, AGENTS.md, CLAUDE.md, .cursorrules, SOUL.md with loaded/truncated/skipped; read-only, local; works on messaging (gauge) — [Slash commands](https://hermes-agent.nousresearch.com/docs/reference/slash-commands)
- `hermes prompt-size [--platform <name>] [--json]`: byte breakdown of system prompt and tool schemas, offline — [CLI reference](https://hermes-agent.nousresearch.com/docs/reference/cli-commands)
- `hermes config path`, `hermes config env-path`, `hermes config show`, `hermes config get <key>`, `hermes config check`, `hermes status [--full] [--deep]`, `hermes doctor [--fix]`, `hermes dump [--show-keys]`, `hermes logs [name] [-n N] [-f] [--since 1h] [--level L]`, `hermes logs list`, `hermes debug share --local`, `hermes chat -v` — CLI reference
- In issue #37480, `prompt-size` going from 0 B to ~16 KB after TERMINAL_CWD was set proved AGENTS.md was discovered — Issue #37480
- Fallback signal: if SOUL.md is empty/unreadable, identity falls back to "You are Hermes Agent, built by Nous Research." — Personality docs
- Blocked/truncated markers: `[BLOCKED: ...]`, `[...truncated AGENTS.md: kept 14000+4000 of 25000 chars...` — Context files docs
- Docker exec form: `docker exec hermes hermes prompt-size --platform telegram`, `docker exec hermes hermes -p coder prompt-size` (inference from docs forms)

### Gaps
- Not verified: whether `hermes prompt-size` run via CLI reflects gateway's terminal.cwd (pass `--platform telegram`; cwd for CLI is launch dir). Use `/context` from Telegram itself for the true gateway view.

## 5. What takes effect when

### Takeaway
Docs give no general rule. Documented points only.

### Cited Findings
- `/reload`: re-reads `.env` into running session; `/reload-mcp`: MCP servers from config.yaml; `/reload-skills`: skills dir — [Slash commands]
- `model.context_length` and `compression.*` on a running gateway apply next message — [Configuration docs]
- Gateway sessions don't auto-reset; `/new` or `/reset` starts new session — [Messaging docs](https://hermes-agent.nousresearch.com/docs/user-guide/messaging)
- Restart gateway after: `gateway.trust_env`, `hermes auth add/reset` (`hermes gateway restart` drains in-flight turns); `gateway.systemd_watchdog_seconds` needs `hermes gateway install --force` (not relevant to s6) — Messaging docs
- In Docker: `docker exec hermes hermes gateway restart` / `-p <name> gateway restart` (s6) or `docker compose restart hermes` — [Docker docs]
- Docs say system prompt stays stable across turns for prompt-cache preservation — Context files docs

### Inferences
- SOUL.md/AGENTS.md edits: very likely built into the system prompt at session start and cached, so effective after `/new` (new session) — possibly requiring gateway restart if the gateway caches the prompt per session/agent; not documented. Safest verification: edit, `/new`, then `/context`.
- Non-hot config keys: assume gateway restart; MCP via /reload-mcp.

### Gaps
- No explicit doc statement for SOUL.md/AGENTS.md reload timing; not tested.

## 6. Real-world examples and lessons (limited)

- Search hits (titles only, not all fetched): [fast.io compose guide](https://fast.io/resources/hermes-agent-docker-compose/), [fast.io production guide](https://fast.io/resources/hermes-agent-docker-deployment-production/), [Railway+Telegram gotchas (tessakriesel.com)](https://tessakriesel.com/how-i-deployed-hermes-agent-on-railway-with-telegram-and-every-gotcha-i-hit-along-the-way/), [benoitpetit/hermes-agent-docker](https://github.com/benoitpetit/hermes-agent-docker), [xmbshwll/hermes-agent-docker](https://github.com/xmbshwll/hermes-agent-docker), [agent37 blog](https://www.agent37.com/blog/hermes-docker), [betterclaw](https://www.betterclaw.io/blog/hermes-agent-docker-install). Only search-snippet level was read: UID/GID 1000 matching is the commonest lesson; one guide notes Telegram file attachments are sent by the gateway so container paths like /workspace must exist where the gateway can read them.
- No Zenn/Qiita/Reddit/Discord sources reviewed (not searched in this pass). No real-world "git-tracked SOUL.md/AGENTS.md workspace repo" example verified.

### Gaps
- Real-world custom SOUL/AGENTS compose examples and Japanese-language sources remain unresearched.
