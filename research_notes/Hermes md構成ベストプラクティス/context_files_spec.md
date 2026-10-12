# Hermes Agent context-file / markdown system spec

Evidence labels: [docs] = hermes-agent.nousresearch.com/docs page text (fetched via summarizer, not verbatim); [src] = agent/prompt_builder.py excerpt (raw.githubusercontent main, first 100k of 114k chars). Source code was only partly visible; [inf] = inference.
Docs base: https://hermes-agent.nousresearch.com/docs/user-guide/features/{context-files,memory,personality,skills,cron}, /user-guide/docker, /user-guide/configuration, /developer-guide/prompt-assembly. Source: https://github.com/NousResearch/hermes-agent/blob/main/agent/prompt_builder.py

## 1. Project context files (AGENTS.md / CLAUDE.md / .hermes.md / .cursorrules)

### Takeaway
Exactly ONE project-context type loads per session (first match wins): `.hermes.md`/`HERMES.md` > `AGENTS.override.md` > `AGENTS.md` > `CLAUDE.md` > `.cursorrules`. Discovery is relative to the agent's cwd, not HERMES_HOME. SOUL.md is separate and global.

### Cited Findings
- Priority list: `.hermes.md`, `AGENTS.override.md`, `AGENTS.md`, `CLAUDE.md`, `.cursorrules`; first match wins [docs context-files]. prompt-assembly page lists `.hermes.md`/`HERMES.md` (walks to git root), `AGENTS.md`, `CLAUDE.md`, `.cursorrules` / `.cursor/rules/*.mdc` and omits AGENTS.override.md [docs prompt-assembly] -> docs inconsistent on override.
- `.hermes.md` is searched from cwd up to git root; with no git root only cwd is checked; `.hermes.md` checked before `HERMES.md` [src `_find_git_root`, `_find_hermes_md`]. YAML frontmatter stripped from `.hermes.md` [src `_strip_yaml_frontmatter`].
- AGENTS.md inside a git repo: git-root AGENTS.md first, then each intermediate dir down to cwd, deeper wins, provenance headers, dedup; outside git only cwd [docs context-files].
- Progressive subdirectory discovery: `SubdirectoryHintTracker` (agent/subdirectory_hints.py) injects AGENTS.md/CLAUDE.md/.cursorrules when tools (read_file, terminal, search_files) touch a subdir; checks dir + up to 5 parents; once per dir per session; appended to the tool result; 32,000-char cap per hint [docs].
- Assembled under `# Project Context` header in system prompt via `build_context_files_prompt()` [docs].
- Size: `context_file_max_chars` (config.yaml, default null) -> dynamic cap = 0.06 x model window, clamped 20,000..500,000 chars [src `_CONTEXT_FILE_WINDOW_FRACTION=0.06`, `CONTEXT_FILE_MAX_CHARS=20_000`, ceiling 500_000; docs configuration]. Truncation 70% head / 20% tail with marker [src ratios 0.7/0.2; docs]. Read timeout `context_file_read_timeout` default 5s, slow file skipped (iCloud/NFS) [src+docs]. Files read as utf-8-sig [src].
- Cron: context files NOT loaded by default; only when job `workdir` set, then AGENTS.md/CLAUDE.md/.cursorrules from that dir [docs cron].
- Subagents/delegation use `skip_context_files` [docs prompt-assembly].

### Inferences
- [inf] Putting CLAUDE.md and AGENTS.md in the same dir: AGENTS.md wins, CLAUDE.md ignored. A `.hermes.md` hides both.
- [inf] Context files are in the cached system prompt, so edits apply on the next session/rebuild, not mid-session.

### Gaps
- Whether `AGENTS.override.md` is actually implemented in the main-branch code (docs inconsistent; excerpt didn't show it).
- Exact CLAUDE.md handling in source (not in visible excerpt).

## 2. SOUL.md (persona)

### Takeaway
Global identity file at `$HERMES_HOME/SOUL.md` only (never cwd). Occupies slot #1, replaces `DEFAULT_AGENT_IDENTITY`. Intended home for persona/tone.

### Cited Findings
- Path `~/.hermes/SOUL.md` or `$HERMES_HOME/SOUL.md`; loaded only from HERMES_HOME; seeded if missing; existing never overwritten [docs personality].
- Empty/unreadable -> built-in default identity; injected verbatim without wrapper after scan + truncation; not repeated in context-files section [docs personality].
- Scanner hit in user-authored SOUL.md: warn and load (not block); profile-distribution-owned SOUL.md is still blocked [src `user_authored`; docs context-files].
- Skipped for subagents (`skip_context_files`) [docs prompt-assembly].
- `/personality <name>` is a session overlay stored in `display.personality`, last in prompt stack; `agent.personalities` (or top-level `personalities:`) define custom ones; `agent.system_prompt` is manual system prompt, applies only when no personality selected [docs personality].
- `ephemeral_system_prompt`, gateway session overlays are not persisted into cached prompt [docs prompt-assembly].

### Gaps
- When SOUL.md is re-read (docs silent). [inf] with prompt cached per session: new session (`/new`) or rebuild; gateway restart is the safe bet but not documented as required.

## 3. Memory (MEMORY.md / USER.md)

### Takeaway
Agent-written, small, frozen snapshot per session. MEMORY.md = agent notes/learned facts (2,200 chars); USER.md = user profile/preferences (1,375 chars).

### Cited Findings
- Paths `~/.hermes/memories/MEMORY.md`, `~/.hermes/memories/USER.md`; profiles: `~/.hermes/profiles/<name>/memories/` [docs memory].
- Written via `memory` tool (add/replace/remove, no read); loaded at session start as frozen snapshot, writes hit disk immediately but show in prompt next session; no auto-compaction, over-limit write errors [docs memory].
- Config `memory:` keys: `memory_enabled` true, `user_profile_enabled` true, `memory_char_limit` 2200, `user_char_limit` 1375, `write_approval` false, `prefetch_spill_enabled` false [docs memory].
- Guidance: task procedures/preferences go to skills; memory only for facts relevant every session; declarative not imperative [src `build_memory_guidance`].
- Memory snapshots sit in volatile tier of prompt [docs prompt-assembly]. External providers (Honcho, Mem0, etc.) run alongside [docs memory].

## 4. Skills (SKILL.md)

### Cited Findings
- `~/.hermes/skills/` (category folders ok); SKILL.md with YAML frontmatter (`name`, `description`, `version`, `platforms`, `metadata.hermes`) [docs skills].
- Prompt gets compact `<available_skills>` index; full body via `skill_view(name)`; references via `skill_view(name,path)` [docs+src `build_skills_system_prompt`].
- `skills.external_dirs`; project skills in `<project>/.hermes/skills/` or `.agents/skills/`, only after `hermes skills trust`; precedence project > local > create_dir > external [docs; src].
- Agent writes via `skill_manage`; `skills.write_approval: true` stages; linter warns at ~24k chars body [docs].
- Index fixed per session for prompt-cache; mid-session changes sent as note; `/reload-skills` refreshes slash commands [docs]. Disk snapshot `.skills_prompt_snapshot.json` [src].

## 5. Cron / job prompts

### Cited Findings
- Jobs in `~/.hermes/cron/jobs.json`; output `~/.hermes/cron/output/{job_id}/{timestamp}.md`; prompt must be self-contained, fresh session [docs cron].
- Prompt scanned for injection/exfil at create/update [docs cron]. Skills attached via `--skill`. Scripts must live in `$HERMES_HOME/scripts/`. Cron cannot schedule more cron unless `cron.allow_agent_scheduling`.
- Default detached from any repo; `workdir` absolute path enables cwd + context files.
- Cron also gets delivery channel platform hint [docs prompt-assembly].

## 6. Prompt assembly order and caching

- Tiers: stable (identity=SOUL.md, tool/model guidance) -> context (caller system_message, project context files, git snapshot, operator instructions, platform hints) -> volatile (skills index, MEMORY/USER snapshots, external memory, timestamp/session/model/provider, `# Hermes runtime environment` block with host, home, cwd) [docs prompt-assembly; docs internally inconsistent on skills/platform_hints tier].
- Built once, cached; rebuilt at session start, compaction, explicit invalidation [docs].
- `platform_hints` in config.yaml with `append`/`replace` customizes per-platform guidance [docs] -- a config-level place for environment facts to the model per platform [inf].

## 7. cwd in gateway/Docker

### Cited Findings
- `terminal.cwd` default `"."`, comment "Gateway/cron working directory (CLI always uses launch dir)" [docs configuration].
- `terminal.docker_mount_cwd_to_workspace` default false: when true bind-mounts launch dir to `/workspace`; applies to the Docker terminal backend (sandbox), not the official image's own data dir [docs configuration].
- Official image: `/opt/data` is single source of truth (host `~/.hermes`), containing `.env`, `config.yaml`, `SOUL.md`, `memories/`, `skills/`, `sessions/`, `home/`, `cron/`, `hooks/`, `logs/`, `skins/`; profiles under `/opt/data/profiles/<name>/`; install tree `/opt/hermes` read-only [docs docker]. Docs never literally state HERMES_HOME=/opt/data; user stated it.
- Runtime env block reports cwd to the model [docs prompt-assembly].

### Inferences
- [inf] In gateway/Telegram the project-context lookup uses the resolved agent cwd (`resolve_agent_cwd()` [src]) = `terminal.cwd` (or MESSAGING_CWD env, not confirmed). If cwd is `.`/`/opt/data`-ish, AGENTS.md at `/opt/data/AGENTS.md` would be found only if that is the cwd. Safest: set `terminal.cwd` to a dir containing AGENTS.md (e.g. /opt/data/workspace) or put `.hermes.md` there; verify via `/context` or runtime-environment block.
- [inf] Environment facts ("runs in Docker, terminal backend local") fit best in a project-context file (AGENTS.md/.hermes.md in terminal.cwd) or `platform_hints`, or a USER/MEMORY entry; SOUL.md for persona only. Not an officially stated mapping.

### Gaps
- Gateway cwd resolution, MESSAGING_CWD, and restart/`/new` requirements are not documented in pages fetched (messaging page silent; configuration page only 0-200k of 263k chars read).
- Docker page documents no container cwd/workspace.

## 8. Security scanning

- `_scan_for_threats(content, scope="context")`; patterns in `tools/threat_patterns` (not seen) [src]. Categories per docs: instruction override ("ignore previous instructions"), deception ("do not tell the user"), system prompt override, hidden HTML comments, hidden display:none divs, credential exfil (curl with $API_KEY), secret file access (cat .env), invisible Unicode/bidi [docs context-files].
- Hit on .hermes.md/AGENTS.md/CLAUDE.md/.cursorrules -> replaced by `[BLOCKED: <file> contained potential prompt injection (prompt_injection). Content not loaded.]`; subdir hints scanned too [docs].
- Practical: avoid phrases like "ignore previous", "do not tell the user", `cat .env`, `curl ... $KEY`, HTML comments in AGENTS.md [inf].

## Quick placement table (per docs intent + inference)
(a) persona/tone: SOUL.md ($HERMES_HOME). (b) runtime/env facts: AGENTS.md or .hermes.md in terminal.cwd [inf]. (c) project conventions: AGENTS.md/.hermes.md. (d) user prefs: USER.md (agent-written; 1,375 chars). (e) learned facts: MEMORY.md (2,200) and skills for procedures.
