# Hermes Agent Skills and Memory systems, and best practices

## SKILL.md format, layout, indexing, token cost

### Takeaway
Skills are directories under `~/.hermes/skills/` with a SKILL.md. Loaded by progressive disclosure: a compact index (~3k tokens) is in the prompt, full bodies loaded on demand via `skill_view`.

### Cited Findings
- Frontmatter: core `name`, `description`, `version`; `platforms` (macos/linux/windows); under `metadata.hermes`: `tags`, `category`, `fallback_for_toolsets/tools`, `requires_toolsets/tools`, `config`; top-level `required_environment_variables` — [Skills docs](https://hermes-agent.nousresearch.com/docs/user-guide/features/skills)
- Layout: `~/.hermes/skills/<category>/<skill>/SKILL.md`, optional `references/ templates/ scripts/ assets/ examples/`; hub state `.hub/lock.json`, `.hub/quarantine/`, `.hub/audit.log`; `.bundled_manifest`; `.no-bundled-skills` opt-out marker; project skills in `<project>/.hermes/skills/` and `<project>/.agents/skills/`; pending writes `~/.hermes/pending/skills/` — [Skills docs](https://hermes-agent.nousresearch.com/docs/user-guide/features/skills)
- Precedence: project -> local -> `skills.create_dir` -> `skills.external_dirs` — same
- Origins: bundled (copied on install/`hermes update`; user-edited copies skipped), hub-installed (tracked in lock.json with hash), agent-created (via `skill_manage`) — same
- Progressive disclosure: L0 `skills_list()` name/description/category (~3k tokens); L1 `skill_view(name)` full SKILL.md; L2 `skill_view(name, path)` one reference file — same
- Warnings: SKILL.md body > ~24k chars; > 60 files in `references/`; slash stacking up to 5 `/skill` tokens — same
- Config keys `skills.*`: `external_dirs`, `create_dir`, `write_approval` (default false; true stages writes for review), `trusted_project_dirs`, `project_discovery`, `tier1_advisory`, `guard_agent_created` — same
- Index snapshot: `get_hermes_home()/.skills_prompt_snapshot.json`, `_SKILLS_SNAPSHOT_VERSION = 4`, rebuilt if version or manifest mismatch; in-process LRU cache max 32; rendered in `## Skills` / `<available_skills>` block grouped by category; non-coding categories shown "names only" — [prompt_builder.py (source)](https://raw.githubusercontent.com/NousResearch/hermes-agent/main/agent/prompt_builder.py)
- `SKILLS_GUIDANCE`: "When you work out a non-trivial workflow, record it with skill_manage for future reuse." — same
- Sharing: sources official, skills-sh, well-known, url, github, clawhub, lobehub, browse-sh; trust levels builtin/official/trusted/community; taps in `~/.hermes/skills/.hub/taps.json`; `hermes skills check|update` (locally edited skipped unless `--force`) — [Skills docs](https://hermes-agent.nousresearch.com/docs/user-guide/features/skills)

### Inferences
- The agent "decides" to load a skill by matching task to index descriptions, so `description` quality is the main lever (the index only carries name/description/category).
- Each added skill costs index tokens every session; unused skills are pure overhead until archived.

### Gaps
- Exact per-skill token cost in index not stated. Docs say nothing on required vs optional frontmatter beyond name/description; the open agentskills.io standard relationship not verified.

## Memory: storage, caps, consolidation, Docker

### Takeaway
Built-in memory is two tiny always-injected files (MEMORY.md 2,200 chars, USER.md 1,375 chars) plus unlimited on-demand session search in SQLite; external providers run alongside.

### Cited Findings
- `MEMORY.md` (agent notes, 2,200 chars ~800 tok) and `USER.md` (user profile, 1,375 chars ~500 tok) in `~/.hermes/memories/`; per profile `~/.hermes/profiles/<name>/memories/` — [Memory docs](https://hermes-agent.nousresearch.com/docs/user-guide/features/memory)
- Tool actions `add`, `replace`, `remove` (substring `old_text` match); no read action; injected at session start — same
- No auto-compaction: over-limit write errors; agent must send one `operations` batch; consolidate past ~80% — same
- Frozen snapshot: mid-session edits hit disk immediately but enter system prompt only next session — [Tips](https://hermes-agent.nousresearch.com/docs/guides/tips)
- Session search: `~/.hermes/state.db` SQLite FTS5, `session_search` returns raw messages — [Memory docs](https://hermes-agent.nousresearch.com/docs/user-guide/features/memory)
- Config `memory:` keys: `memory_enabled`, `user_profile_enabled`, `memory_char_limit` (2200), `user_char_limit` (1375), `write_approval` (false), `provider`, `prefetch_spill_enabled` — same
- External providers: bundled Holographic, RetainDB, ByteRover; plugins Honcho, Hindsight, Supermemory, Mem0, OpenViking; `hermes memory setup|status` — same
- Save: preferences, env facts, corrections, conventions, explicit "remember"; skip: trivia, easily searchable facts, raw dumps, session ephemera, content already in SOUL.md/AGENTS.md — same
- Docker: all state in `/opt/data` (maps to host `~/.hermes`): `.env`, `config.yaml`, `SOUL.md`, `sessions/`, `memories/`, `state.db`, `skills/`, `plugins/`, `hooks/`, `cron/`, `logs/`, `profiles/`, `home/` — [Docker docs](https://hermes-agent.nousresearch.com/docs/user-guide/docker)
- Bind mounts on virtiofs/9p (Docker Desktop macOS, OrbStack) can corrupt `state.db`; use named volume or `database.journal_mode: delete`; never run two gateway containers on the same data dir; runtime user hermes UID 10000, `HERMES_UID/GID` — same

### Inferences
- On the user's Mac Docker setup with a bind mount, state.db corruption risk applies; check docker-compose.yml volume type.
- Memory is frozen per session, so a wrong entry persists in every future prompt until removed.

### Gaps
- Whether the agent writes memory fully automatically vs on nudges (periodic memory review) not confirmed from docs fetched.

## Curator, auto-creation, risks

### Takeaway
Curator (v0.12.0) archives/consolidates only agent-created skills; nothing is auto-deleted; consolidation is off by default. Docs are internally inconsistent on default day counts.

### Cited Findings
- Config under `curator:` `enabled` true, `interval_hours` 168, `min_idle_hours` 2, `stale_after_days` 14, `archive_after_days` 30, `consolidate` false, `prune_builtins` false, `archive_ttl_days` 0, `backup.enabled` true, `backup.keep` 2 — [Curator docs](https://hermes-agent.nousresearch.com/docs/user-guide/features/curator)
- Same page's lifecycle text cites 30/90 days; a search summary also said prune_builtins default true — conflicting; treat defaults as version-dependent — [Curator docs](https://hermes-agent.nousresearch.com/docs/user-guide/features/curator), [CodeCut](https://codecut.ai/hermes-autonomous-skills-curator/)
- Archive dir `~/.hermes/skills/.archive/`; snapshots `~/.hermes/skills/.curator_backups/` (tar.gz, consolidation only) — Curator docs
- Commands: `hermes curator status|run [--dry-run --consolidate --background]|pause|resume|pin|unpin|adopt|list-unmanaged|archive|restore|list-archived|prune|purge|backup|rollback|ledger` — Curator docs
- Managed scope: agent-created skills only; hand-written, `created_by: learn`, hub skills excluded unless adopted; pinned skills and skills used by cron protected — Curator docs
- Disable: `curator.enabled: false` or `hermes curator pause` — Curator docs
- Auto-creation example: 14-tool-call debugging task produced generic skill; hand-made near-duplicates got merge proposal; review needed — [CodeCut](https://codecut.ai/hermes-autonomous-skills-curator/)
- Guidance text: "Skills come first" for task learnings; memory is "the narrow exception for facts that apply to EVERY" session; "A fact stale within a week belongs in session history" — [prompt_builder.py](https://raw.githubusercontent.com/NousResearch/hermes-agent/main/agent/prompt_builder.py)
- Review control: `skills.write_approval: true` and `memory.write_approval: true` stage writes for approval — docs above

### Inferences
- A wrong environment fact ("I am on the host Mac") is likely when agent runs in container; remedy: put the environment truth in AGENTS.md/SOUL.md or MEMORY.md manually and enable `memory.write_approval`.
- Bloat mitigation: periodic `hermes curator run --consolidate --dry-run`, pin good skills, `adopt` hand-written ones if wanted managed.

### Gaps
- Exact auto-creation trigger threshold not documented. Reddit, Zenn/Qiita, GitHub Discussions content not retrieved (not searched deeply).

## What belongs where (skill vs memory vs SOUL.md vs context file)

### Takeaway
SOUL.md = persona/tone only; AGENTS.md = project rules; memory = small durable facts ("what"); skills = procedures ("how"); history = everything else.

### Cited Findings
- "Use memory for 'what,' skills for 'how.'" Create a skill when a task takes 5+ steps and will repeat — [Tips](https://hermes-agent.nousresearch.com/docs/guides/tips)
- SOUL.md at `$HERMES_HOME/SOUL.md` (global, HERMES_HOME only), personality/tone/style only; AGENTS.md for architecture, conventions, key paths/ports, keep concise — [Context files](https://hermes-agent.nousresearch.com/docs/user-guide/features/context-files)
- Only one project context file loads, first match: `.hermes.md` -> `AGENTS.override.md` -> `AGENTS.md` -> `CLAUDE.md` -> `.cursorrules`; SOUL.md always separately — same
- Context cap: `context_file_max_chars`, scaled by model window, floor 20,000 / ceiling 500,000; truncation 70% head / 20% tail — same
- "Every character counts against your token budget since they're injected into every single message"; keep system prompt stable for prompt-cache; `hermes prompt-size` shows fixed overhead — [Tips](https://hermes-agent.nousresearch.com/docs/guides/tips)
- Third-party: avoid dumping everything in SOUL.md (inconsistent behavior across projects) — [LumaDock via search](https://lumadock.com/tutorials/hermes-soul-agents-user-memory-files) (single tutorial, low confidence)

### Inferences
- Anti-patterns: long procedures in memory (2,200 cap); repo facts in SOUL.md; unreviewed auto skills; letting memory accumulate past 80%.
- Note the user's own CLAUDE.md files may be picked up by Hermes as project context if no AGENTS.md exists.

### Gaps
- Community (Reddit, Zenn, Qiita) best-practice posts not found in this pass.
