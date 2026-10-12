# Best practices for agent instruction files (AGENTS.md / CLAUDE.md / SOUL.md / rules) applied to small local-model agents (Hermes Agent + Ollama)

## Q1. What do vendors/tools recommend about structure, size, layering (global vs project vs directory)?

### Takeaway
All major tools converge on: short, specific, concrete, split by scope, layered global -> project -> directory, with later/closer files winning. Size targets differ (Claude Code under ~200 lines; Cursor under 500 lines; Codex 32 KiB combined cap; Hermes 20,000-char floor cap per file), but every vendor says shorter is better and that stale/vague rules hurt.

### Cited Findings
- Claude Code: "target under 200 lines per CLAUDE.md file. Longer files consume more context and reduce adherence." Imports (@path) organize but do not reduce context cost because imported files load at launch. — [Claude Code memory docs](https://code.claude.com/docs/en/memory)
- Claude Code: write verifiable, specific rules ("Use 2-space indentation" not "Format code properly"; "API handlers live in `src/api/handlers/`" not "Keep files organized"); group under markdown headers/bullets; contradictory instructions may be resolved arbitrarily; review periodically for outdated/conflicting rules. — [Claude Code memory docs](https://code.claude.com/docs/en/memory)
- Claude Code: CLAUDE.md is "context, not enforced configuration"; use hooks/permissions for hard guarantees. Load order is broad -> specific (user, project, local), so project instructions appear later in context. Conflicting user/project rules: "Claude may follow either one." — [Claude Code memory docs](https://code.claude.com/docs/en/memory)
- Claude Code: block-level HTML comments in CLAUDE.md are stripped before injection (maintainer notes cost no tokens). — [Claude Code memory docs](https://code.claude.com/docs/en/memory)
- Claude Code best practices: include bash commands Claude can't guess, env quirks, non-default style rules, gotchas; exclude what can be inferred from code, standard conventions, long docs (link instead), frequently-changing info, file-by-file descriptions. Test per line: "Would removing this cause Claude to make mistakes? If not, cut it." "Bloated CLAUDE.md files cause Claude to ignore your actual instructions." Use emphasis ("IMPORTANT") on a single line only; if many lines are emphasized none stands out. — [Claude Code best practices](https://code.claude.com/docs/en/best-practices)
- Codex: AGENTS.md chain built once per run: global (`~/.codex`) then from project root down to cwd, at most one file per directory, concatenated root-down so closer files override; `AGENTS.override.md` replaces the regular file; combined cap `project_doc_max_bytes` default 32 KiB; keep review rules short and in the AGENTS.md nearest the code. — [OpenAI Codex AGENTS.md docs](https://learn.chatgpt.com/docs/agent-configuration/agents-md)
- Cursor: "Keep rules under 500 lines"; split large rules; focused, actionable, scoped; keep always-on rules to a minimum; reference files instead of copying contents (prevents staleness); avoid copying style guides (use linter), listing every command, rare edge cases; add a rule only after seeing the agent repeat a mistake. — [Cursor rules docs](https://cursor.com/docs/context/rules)
- Cline: rules consume context every request; "Keep rules concise and link to external documentation"; one concern per file; be specific; include the why; point to example code; "Outdated rules confuse Cline and waste context." — [Cline rules docs](https://docs.cline.bot/customization/cline-rules)
- Anthropic context engineering: system prompts should be at the "right altitude" (not brittle hardcoded logic, not vague); find "the smallest set of high-signal tokens"; minimal does not mean short; start minimal and add instructions based on observed failures; prefer a few canonical examples over long edge-case lists. — [Anthropic: Effective context engineering](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
- Hermes Agent: one project context file loads per session, first match wins (`.hermes.md`, `AGENTS.override.md`, `AGENTS.md`, `CLAUDE.md`, `.cursorrules`); SOUL.md loads separately; nested AGENTS.md discovered when the agent enters subdirectories; edits take effect only on a new session. — [Hermes context files](https://hermes-agent.nousresearch.com/docs/user-guide/features/context-files), [Hermes Which File Does What](https://hermes-agent.nousresearch.com/docs/user-guide/which-file-does-what)
- Hermes tips: keep file short ("the agent reads it every turn"), use `##` headers, concrete examples, state what not to do, list key paths/ports, "stale context is worse than no context." — [Hermes context files](https://hermes-agent.nousresearch.com/docs/user-guide/features/context-files)

### Inferences
- A sensible budget for a 64k-context local model: always-loaded instruction material (SOUL + AGENTS + memory) well under ~2-3k tokens; the tool definitions and chat history need the rest. This is my inference, not a vendor number.
- Because Hermes loads only ONE project file (first match), do not split environment facts across AGENTS.md and CLAUDE.md expecting both to load; and `AGENTS.override.md` silently replaces AGENTS.md, so a stale override can mask edits.
- Directory-level nested files are the right place for sub-project specifics so they load only when the agent enters that directory.

### Gaps
- The agents.md site itself (agents.md) could not be fetched (DNS failure); the open convention's own recommendations (suggested sections, "no required fields") were not verified first-hand. Claude Code and Codex docs both confirm AGENTS.md is read as plain markdown.
- Windsurf rules docs not fetched; no Windsurf-specific figures (commonly cited 6,000-char per-rule limit is unverified here).

## Q2. How do small/local models follow instructions; does bloat degrade performance?

### Takeaway
Evidence is strong that instruction-following degrades with more instructions and longer context, and that models use start/end of context better than the middle; evidence for small local models specifically is thin and mostly indirect. Context files help most when they contain non-obvious, tool/command-specific facts and hurt when they contain generic overviews or unnecessary requirements.

### Cited Findings
- ETH Zurich/LogicStar study (AGENTbench, 138 tasks): context files generally did not improve task success and raised inference cost by over 20%; agents followed instructions well, but repo overviews were not helpful; tooling instructions (e.g. uv, repo-specific commands) were used far more often; conclusion: human-written files "should describe only minimal requirements." — [arXiv 2602.11988](https://arxiv.org/abs/2602.11988), [HTML version](https://arxiv.org/html/2602.11988v1)
- Same study numbers: developer files ~+4% average; LLM-generated files about -2 to -3% (paper figures differ between sections); extra steps ~2.5-3.9 per task. — [arXiv HTML](https://arxiv.org/html/2602.11988v1). Note: tested coding agents on mostly frontier models, not small local models; commentary notes it compared generic files vs none, not well-written vs poorly-written — [secondary commentary](https://codex.danielvaughan.com/2026/04/13/what-eth-zurich-paper-gets-wrong-right-about-agents-md/)
- "Lost in the middle": performance is highest when relevant info is at the beginning or end of the context and drops when it is in the middle. — [Liu et al., arXiv 2307.03172](https://arxiv.org/abs/2307.03172)
- Chroma "Context Rot" (July 2025): performance drops as input length grows even on simple tasks; position and related distractors matter. — [Chroma research](https://trychroma.com/research/context-rot) (read via search summary only)
- Instruction count: IFScale found even best frontier models reach only ~68% at 500 instructions, with a bias toward earlier instructions; an "instruction compiler" study found rewriting rule sets helps weaker models most. — [IFScale/related, arXiv 2510.14842](https://arxiv.org/pdf/2510.14842), [arXiv 2608.02639](https://arxiv.org/pdf/2608.02639) (from search snippets; not fully read)
- Anthropic: context is "a finite resource with diminishing marginal returns"; recall degrades as context grows; performance degrades as the window fills. — [Anthropic context engineering](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents), [Claude Code best practices](https://code.claude.com/docs/en/best-practices)
- Negative instructions: practitioner write-ups argue "don't" rules are less reliable and can prime the banned term; Anthropic-style advice is to say what to do instead. Anecdotal/blog level. — [16x Eval on negative instructions](https://eval.16x.engineer/blog/the-pink-elephant-negative-instructions-llms-effectiveness-analysis), [Hackernoon](https://HackerNoon.com/llms-dont-understand-negation)
- Ollama context: multiple guides report a 4096-token default that silently truncates the front of the prompt (the system prompt) with no warning; OpenAI-compatible `/v1` endpoint may ignore `num_ctx`; set `OLLAMA_CONTEXT_LENGTH` server-side. A Hermes issue reports empty/truncated outputs once the prompt outgrows the window. — [fast.io guide](https://fast.io/resources/ollama-context-window/), [Hermes issue #43900](https://github.com/NousResearch/hermes-agent/issues/43900). Vendor blogs/GitHub issue only; Ollama's official docs not checked, and defaults may differ by version.

### Inferences
- For a ~64k local model, small-model weaknesses make these habits worth following (my synthesis, not directly tested in sources): few rules (aim ~10-20 total); one imperative sentence each; positive phrasing ("Use `docker exec` ... is unavailable, use X"); put the 3-5 must-not-violate rules at the very top and/or repeat as a one-line reminder at the end; concrete literals (paths, commands, ports) over prose; one worked example for the trickiest behavior; avoid emphasis inflation.
- Hard constraints should be enforced outside the prompt (tool allowlists, container permissions, missing binaries) because instruction files are advisory.
- Verify the effective num_ctx actually reaches Ollama (e.g. check `ollama ps` / server logs) before tuning prompts; a truncated system prompt looks like "model ignores instructions."

### Gaps
- No rigorous benchmark found on qwen3.5-35B-A3B/gemma instruction-file adherence specifically; MoE-with-3B-active behavior unstudied in sources found.
- Harada et al. "Curse of Instructions" primary paper not retrieved; the p^N compliance formula and its numbers are secondary and unverified.
- No controlled study found on repeating instructions or negative-phrasing effects for small models.

## Q3. Division of content: persona vs project instructions vs environment facts vs memory

### Takeaway
Hermes defines the split explicitly: SOUL.md = identity/tone (global); AGENTS.md = project instructions (commands, ports, paths, workflows); MEMORY.md/USER.md = small agent-written notes (hard-capped); skills = on-demand procedures. Never duplicate across layers.

### Cited Findings
- SOUL.md: personality, tone, communication style; lives in `~/.hermes/` (HERMES_HOME), loaded as slot #1 of the system prompt; global to the instance; seeded but never overwritten; skipped for delegated sub-agents. — [Which File Does What](https://hermes-agent.nousresearch.com/docs/user-guide/which-file-does-what), [prompt assembly (zh)](https://hermes-agent.nousresearch.com/docs/zh-Hans/developer-guide/prompt-assembly)
- AGENTS.md: "project instructions: commands, ports, paths, and workflows," written by you. MEMORY.md: agent's own notes (environment facts, conventions, tool quirks). USER.md: user profile. — [Which File Does What](https://hermes-agent.nousresearch.com/docs/user-guide/which-file-does-what)
- Memory limits: MEMORY.md 2,200 chars (~800 tokens), USER.md 1,375 chars (~500 tokens); injected as a frozen snapshot at session start; skip content already in SOUL.md/AGENTS.md, temporary details, large raw data; use a skill instead of memory for location details needed on recurring tasks; consolidate >80% usage. — [Hermes memory docs](https://hermes-agent.nousresearch.com/docs/user-guide/features/memory)
- Context file cap: `context_file_max_chars` (default scales with model window, 20,000-char floor); oversize files keep 70% head + 20% tail with a truncation marker. — [Hermes context files](https://hermes-agent.nousresearch.com/docs/user-guide/features/context-files)
- Claude Code analog: CLAUDE.md for always-on facts; multi-step or part-of-codebase-only procedures go to skills or path-scoped rules; auto memory for agent-learned notes (first 200 lines/25 KB loaded). — [Claude Code memory docs](https://code.claude.com/docs/en/memory)
- Anthropic: just-in-time context via lightweight identifiers (paths, links) and structured note-taking to NOTES.md-style files outside the window. — [Anthropic context engineering](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)

### Inferences
- Suggested layout for this repo: SOUL.md (~10-20 lines: role, language, tone, response style, honesty about limits; no paths); AGENTS.md (~40-80 lines: runtime facts, path map, allowed/unavailable tools, workflow rules); MEMORY.md left to the agent (do not hand-seed with facts that belong in AGENTS.md); longer how-tos as skills or files the AGENTS.md points to.
- A language/persona preference (e.g. reply in Japanese) belongs in SOUL.md or USER.md, not duplicated in AGENTS.md.

### Gaps
- Exact tokens-per-char for Japanese text under Qwen/Gemma tokenizers not found; Japanese content typically costs more tokens per character than English, so char caps (2,200 / 1,375) are tighter in tokens for Japanese than the "~800 tokens" figure implies (inference).

## Q4. How to state runtime-environment facts so weaker models follow them, and keep them current

### Takeaway
State environment as a short, literal "facts + consequences" block (what exists, what doesn't, exact paths/commands to use instead), generated from or checked against the compose file rather than hand-maintained prose. Sources give general guidance; none tests container-constraint phrasing on small models specifically.

### Cited Findings
- Include "developer environment quirks" and "bash commands Claude can't guess" in always-on files; exclude info that changes frequently. — [Claude Code best practices](https://code.claude.com/docs/en/best-practices)
- Reference files instead of copying their contents "prevents them from going stale." — [Cursor rules](https://cursor.com/docs/context/rules)
- State the why; point to examples; remove rules that no longer apply. — [Cline rules](https://docs.cline.bot/customization/cline-rules)
- Hermes: list key paths and ports; state what not to do; "stale context is worse than no context." — [Hermes context files](https://hermes-agent.nousresearch.com/docs/user-guide/features/context-files)
- Claude Code `/doctor prompt-audit` checks for references to files/commands that don't exist and contradictions (tool-specific, but the audit idea is portable). — [Claude Code memory docs](https://code.claude.com/docs/en/memory)
- Hard enforcement should not rely on prose: instructions are advisory; use hooks/permissions. — [Claude Code memory docs](https://code.claude.com/docs/en/memory)

### Inferences
Proposed template (my recommendation):
```
## Runtime (verified YYYY-MM-DD against docker-compose.yml)
- You run inside a Docker container (Linux). The host is macOS; you cannot see it.
- The `docker` CLI is NOT installed. Do not try to run docker; ask the user to run it on the host.
- Writable workspace: /workspace  (mounted from host ./workspace). Notes/outputs go here.
- Read-only: /<path>. Persistent state: /opt/data (= HERMES_HOME).
- Model server: Ollama at http://host.docker.internal:11434 (not localhost).
- If a task needs something unavailable, say so and give the exact command for the user.
```
- Write each constraint as fact + replacement action; one line each; use literal paths. Keep to <= ~15 lines; put it near the top of AGENTS.md (primacy) and optionally one-line recap at the end.
- Single source of truth: docker-compose.yml owns paths/ports/mounts; AGENTS.md carries a minimal derived copy with a "verified against compose on DATE" line and a pointer ("see /workspace/docker-compose.yml") only if that file is actually mounted. Optionally generate the block via a small script/Makefile target so drift is detectable; use HTML comments for maintainer notes (stripped in Claude Code; Hermes behavior with comments not verified, and its scanner flags "hidden HTML comments", so avoid them in Hermes files).
- Enforce physically where possible (don't install docker CLI, don't mount docker.sock, read-only mounts) so a wrong guess fails loudly rather than relying on the prompt.
- Re-verify after every compose change; stale facts are the highest-risk item because the agent trusts them.

### Gaps
- No source tests how well small models obey "tool X is unavailable" statements; recommendations above are extrapolations.
- Whether Hermes strips HTML comments from AGENTS.md before injection is not documented in what I read; docs say the scanner flags hidden HTML comments as potential injection.

## Q5. Security hygiene and anti-patterns

### Takeaway
Never put secrets in instruction/memory files (they are injected into every prompt and may be echoed or exfiltrated); treat any file the agent can write that is later loaded as an injection vector; Hermes scans but explicitly says scanning is not a substitute for review.

### Cited Findings
- Hermes scans context files for prompt-injection patterns (instruction overrides, hidden HTML comments, credential exfiltration, secret file access, invisible characters), blocks flagged project files; scanner "is not a substitute for reviewing context files in shared repositories"; SOUL.md in HERMES_HOME only warns, does not block. — [Hermes context files](https://hermes-agent.nousresearch.com/docs/user-guide/features/context-files)
- Memory entries are scanned before acceptance because they are injected into the system prompt; invisible Unicode blocked. Hermes memory docs also note the model may claim a save without making the tool call; `write_approval: true` lets you review saves. — [Hermes memory docs](https://hermes-agent.nousresearch.com/docs/user-guide/features/memory)
- Claude Code shows an approval dialog for external imports "to protect you from files other people commit to a shared project." — [Claude Code memory docs](https://code.claude.com/docs/en/memory)
- Anti-patterns named by vendors: over-long/over-specified files, contradictory rules, vague rules ("write clean code"), codebase overviews/file-by-file descriptions, pasted style guides/API docs, emphasis everywhere, stale rules, rarely-used edge cases. — [Claude Code best practices](https://code.claude.com/docs/en/best-practices), [Cursor rules](https://cursor.com/docs/context/rules), [Cline rules](https://docs.cline.bot/customization/cline-rules)
- Frozen-snapshot behavior: Hermes SOUL/AGENTS/memory changes only apply in the next session; `/new` at task boundaries. — [Hermes memory docs](https://hermes-agent.nousresearch.com/docs/user-guide/features/memory)

### Inferences
- Credentials (API keys, tokens) belong in env vars / compose secrets used by tools, never in SOUL/AGENTS/MEMORY; do not commit `.env`. Check MEMORY.md/USER.md periodically since the agent writes them (a user might paste a token in chat and the agent might "remember" it).
- Make AGENTS.md read-only to the agent (read-only mount or file permissions) so it cannot rewrite its own rules from web content; let it write only to MEMORY.md and workspace. Any agent-writable file that gets re-read next session is persistent prompt-injection surface.
- Also avoid: duplicating the same fact in SOUL, AGENTS and memory (diverges, wastes budget); putting a chat-style persona essay in the context window of a 3B-active MoE; documenting host-only details (macOS paths) that the container cannot use, which confuses the model about where it is.

### Gaps
- No empirical rates found for prompt injection via agent-writable memory in local-model setups.
- Windsurf/Cursor security guidance on rules files not reviewed.
