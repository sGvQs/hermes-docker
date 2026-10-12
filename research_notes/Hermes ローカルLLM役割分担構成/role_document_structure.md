# Document structure for a request -> implementer -> reviewer -> result pipeline on Hermes Agent with local LLMs

Source-type legend: [V]=vendor/official docs, [P]=paper/preprint, [B]=blog/anecdote/secondary. Several 2026 arXiv items were only seen via abstracts/search summaries; verify before relying on exact numbers.

## Q1. How do frameworks recommend implementer/reviewer separation and handoff contracts; what evidence do architect->editor / generator->critic loops help small models; what pitfalls?

### Takeaway
Every major framework converges on: separate fresh-context roles, a declared output contract (schema / expected_output / guardrail), and a hard cap on iterations. Evidence that two-stage pipelines help weak models exists (Aider: o1-mini and gpt-4o-mini gained) but is 2024 data on hosted models; evidence for pure self-critique is negative, so the reviewer needs an external signal (tests/tools) or a different model.

### Cited Findings
- Aider architect/editor: architect describes the solution freely, editor produces edit-format output; separation lets each focus on one demand. Results: o1-mini+deepseek (whole) 71.4% vs o1-mini solo 61.1%; gpt-4o-mini+gpt-4o-mini 60.2% vs 55.6% solo; Sonnet/GPT-4o also gained when paired with themselves. [V/benchmark by Aider author, 2024-09] — [Aider blog](https://aider.chat/2024/09/26/architect.html)
- A later Aider result (o3 architect + GPT-4.1 editor, 83% polyglot, lower cost than o3 alone) is reported only by a third-party write-up; official leaderboard not directly confirmed. [B] — [DataLearner/search summary](https://www.datalearner.com/en/blog/1051755633012011)
- Anthropic "Building effective agents": evaluator-optimizer fits when clear evaluation criteria exist and feedback demonstrably improves output; include stopping conditions such as max iterations; programmatic gates between chained steps; start simple, frameworks can hide prompts. [V] — [Anthropic](https://www.anthropic.com/engineering/building-effective-agents)
- Anthropic multi-agent research system: each subagent needs objective, output format, tool guidance, task boundaries; vague delegation caused duplicated/misread work; embed effort-scaling rules in orchestrator prompt; subagents may write artifacts to external storage and return lightweight references; multi-agent uses ~15x chat tokens; poor fit for tasks needing shared context / many dependencies. [V] — [Anthropic](https://www.anthropic.com/engineering/multi-agent-research-system)
- Claude Code subagents: markdown file + YAML frontmatter, body is the system prompt; fresh context window, does not see conversation history, so constraints must be restated in the delegation message; only final summary returns; tool allowlist per role (e.g. reviewer read-only); maxTurns limit; keep descriptions short, detail in body. [V] — [Claude Code docs](https://code.claude.com/docs/en/sub-agents)
- OpenAI Agents SDK: handoff transfers control, receiving agent by default sees entire prior history; `input_filter` controls what the next agent sees; `input_type` gives a validated schema for handoff args; `Agent.as_tool` gets structured results from a specialist without transferring the conversation. [V] — [OpenAI Agents SDK](https://openai.github.io/openai-agents-python/handoffs/)
- CrewAI: per-task `expected_output`, `output_pydantic/json`, `guardrail(s)` validated before the next task, `guardrail_max_retries` default 3. [V] — [CrewAI tasks](https://docs.crewai.com/en/concepts/tasks)
- MetaGPT: SOP-encoded roles exchange structured artifacts (requirements, design, interface specs) rather than chat; paper claims this reduces cascading hallucination and raises code success (85.9% HumanEval Pass@1); hallucination benefit asserted, not isolated. [P] — [MetaGPT arXiv](https://arxiv.org/pdf/2308.00352v6)
- Intrinsic self-correction without external feedback is weak and can degrade reasoning. [P] — [Huang et al.](https://arxiv.org/pdf/2310.01798)
- Self-refine amplifies self-bias; external accurate feedback reduces it. [P] — [Pride and Prejudice](https://arxiv.org/html/2402.11436v2)
- Harmful self-preference persists where the model is a weak generator; long CoT before verdict mitigates it. [P] — [arXiv 2504.03846](https://arxiv.org/pdf/2504.03846v3)
- Cross-family judging is recommended by practitioner sources; verbosity and other biases remain. [B] — [towardsdatascience](https://towardsdatascience.com/the-llm-judge-that-kept-agreeing-with-itself), [memx](https://memx.app/blog/llm-as-a-judge-biases-and-fixes/)
- MAST taxonomy: 14 failure modes in 3 categories (specification issues, inter-agent misalignment, task verification), from 1600+ traces over 7 MAS frameworks. Exact verification-mode names not confirmed in my search. [P] — [Cemri et al.](https://arxiv.org/abs/2503.13657v2)

### Inferences
- The reviewer's value on a small model comes mostly from (a) a different model/prompt context and (b) objective checks (run tests, diff vs acceptance criteria), not from "re-reading and judging".
- Rubber-stamping risk is highest when reviewer shares model + context with implementer; mitigate with different model, fresh context, a checklist forcing per-item evidence, and default-to-FAIL wording. (Mitigations are my synthesis from the self-preference papers; no direct measured test of this on local models found.)
- Non-convergence: cap rounds (suggest 2 revise cycles), and on cap escalate to human/stop with last verdict rather than continue.
- Aider-style gains were measured on whole-file/diff edit benchmarks with hosted models; transfer to 7-32B local models is plausible but unverified.

### Gaps
- No found benchmark of generator->critic loops specifically on small local models (7-32B) in 2025-2026.
- Could not retrieve LangGraph and AutoGen docs in this pass (no cited guidance from them); LangGraph's conditional-edge + recursion_limit and AutoGen's max_turns/termination conditions are known from general knowledge only and not sourced here.
- Official Aider leaderboard not directly read.

## Q2. Which document holds role definition, acceptance criteria, rubric, escalation/stop, environment facts; how to give the reviewer fresh context?

### Takeaway
Hermes gives four natural homes: SOUL.md (per-HERMES_HOME identity/role), AGENTS.md (project-wide shared facts and protocol, auto-loaded, also into delegate children), skills (on-demand procedures/rubrics), and the delegate_task `goal`/`context` (per-job acceptance criteria and artifacts). Fresh-context review is native to delegate_task and to separate profiles.

### Cited Findings
- Hermes auto-loads context files; SOUL.md is the identity slot #1, read only from HERMES_HOME (not the working dir), per profile; an empty file adds nothing; a default is seeded. [V] — [Hermes context files](https://hermes-agent.nousresearch.com/docs/user-guide/features/context-files)
- Project context: only ONE type loads, first match wins: .hermes.md -> AGENTS.override.md -> AGENTS.md -> CLAUDE.md -> .cursorrules; nested AGENTS.md chain from git root to cwd; startup cap scales with context window (20k-500k chars) or `context_file_max_chars`; truncation keeps 70% head / 20% tail; all files prompt-injection scanned. [V] — same page
- Skills: SKILL.md with frontmatter (description recommended <=60 chars), body sections "When to Use / Procedure / Pitfalls / Verification"; 3 levels (list ~3k tokens always in system prompt; skill_view full body; references/ files on demand); warn above ~24k chars body; `/skill-name` slash invocation, up to 5 stacked; cron and `-s` can preload. [V] — [Hermes skills](https://hermes-agent.nousresearch.com/docs/user-guide/features/skills)
- delegate_task: child gets fresh conversation, only `goal` + `context`; but project context files (AGENTS.md etc.) are embedded in the child's system prompt; only final summary returns; optional per-task `output_schema` with `schema_valid`/`schema_errors` and one correction turn; default flat (max_spawn_depth 1); children inherit parent toolsets, cannot be given a different toolset or model per call (`delegation.model` applies to all children); default 250 max iterations per child (global config); 10 concurrent default. [V] — [Hermes delegation](https://hermes-agent.nousresearch.com/docs/user-guide/features/delegation)
- Profiles: separate Hermes home each with own config/model, .env, SOUL.md, memory, sessions, skills, cron, gateway; `--clone` copies config/SOUL/skills; not a sandbox (can reach files outside); never point two agents at one profile. [V] — [Hermes profiles](https://hermes-agent.nousresearch.com/docs/user-guide/profiles)
- Claude Code analog: reviewer subagent as read-only tool allowlist; content in the delegation message is the only task input. [V] — [Claude Code docs](https://code.claude.com/docs/en/sub-agents)
- OpenAI SDK: `input_filter` is the explicit mechanism for hiding history from the next agent. [V] — [OpenAI Agents SDK](https://openai.github.io/openai-agents-python/handoffs/)

### Inferences
Suggested allocation:
| Content | Home |
|---|---|
| Role identity, voice, hard dos/don'ts (<~300-500 tokens) | SOUL.md of each role profile (b), or role block in delegate `context` (a) |
| Environment facts (paths, ports, commands, how to run tests), shared pipeline contract & handoff schema | Project AGENTS.md (auto-loaded for all roles and delegate children) |
| Step procedure, review rubric/checklist, escalation rules | One skill (e.g. `/review-pipeline`) with references/ files; loaded only by the role that needs it |
| Per-request acceptance criteria | Written by the requester/orchestrator into a task file or `context`; both implementer and reviewer receive the same criteria |
| Verdict/stop rules | Rubric file + orchestrator code (cap enforced outside the model where possible) |
- Fresh-context review: hand reviewer only (task, acceptance criteria, final artifact/diff, test output) - never implementer's transcript or self-assessment. In (a) this is automatic via delegate_task; in (b)/(d) pass via files or fields in a handoff block.
- Caveat for (a): children share the parent's model and toolsets in Hermes, so implementer/reviewer cannot be different models or have read-only tools within one delegate call; this weakens (a) for cross-model review. Profiles or external orchestration give model diversity and per-role toolsets.
- Caveat: AGENTS.md is injected into every role, so role-specific rubric text there wastes tokens and can confuse roles.

### Gaps
- Not confirmed whether delegation `output_schema` and profile-specific toolsets interact; how per-profile API ports are assigned (profiles page does not document; API page says each profile can run its own API server on a separate port).
- No published Hermes guidance specifically for implementer/reviewer pipelines was found.

## Q3. Which of (a)-(d) is the right structure?

### Takeaway
For small local models, the best-supported default is (d)-lite or (b): deterministic orchestration (script/cron) that sequences calls to role-specific Hermes profiles/ports, with files as handoff medium and verdict tokens parsed by code; (a) is simplest but cannot vary model or tools per child; (c) alone is the weakest for review independence.

### Cited Findings
- Anthropic recommends simplest workable pattern, programmatic gates between chain steps, iteration caps; orchestrator-workers when subtasks are unpredictable. [V] — [Building effective agents](https://www.anthropic.com/engineering/building-effective-agents)
- Hermes API server: OpenAI-compatible on 127.0.0.1:8642, API_SERVER_KEY required, `/v1/chat/completions` stateless, `/v1/responses` stateful; client `system` message layers on top of Hermes core prompt (agent keeps tools/memory/skills); `model` without `provider` ignored unless `direct_model_requests` enabled; each profile can run its own API server on separate port, profile name = model ID, `/p/<profile>/` prefix routing with per-profile key. [V] — [Hermes API server](https://hermes-agent.nousresearch.com/docs/user-guide/features/api-server)
- Tool access: API server gives full toolset incl. terminal. [V] — same page
- Delegation children cannot get different model per call. [V] — [Hermes delegation](https://hermes-agent.nousresearch.com/docs/user-guide/features/delegation)
- Multi-agent costs ~15x chat tokens; poor fit with shared context. [V] — [Anthropic](https://www.anthropic.com/engineering/multi-agent-research-system)

### Inferences
| Option | Pros | Cons for local models |
|---|---|---|
| (a) orchestrator+delegate_task | no extra infra; fresh context free; schema validation (`output_schema`) | orchestrator must itself follow a multi-step protocol (small models drift); same model/toolset for all children; orchestrator context grows |
| (b) profiles w/ SOUL.md | clean role identity, per-role model, memory/skills isolation | still needs something to sequence; N processes/ports; memory/skills can drift per profile; clone maintenance |
| (c) single skill | one file, portable | one model plays both roles in one context -> self-review bias; long body loaded permanently in context |
| (d) external script + API | deterministic loop, round cap, verdict parsing in code, different models/ports, reviewer context truly fresh, loggable | requires writing/maintaining script; each call a full Hermes agent turn unless stateless chat |
- Recommended: (d) calling (b) profiles for implementer and reviewer; keep SOUL.md tiny, shared facts in AGENTS.md, rubric in a skill or passed in the prompt. Use (a) as a fallback for single-model setups.

### Gaps
- No empirical comparison of these four patterns on Hermes was found.

## Q4. Output formatting for reliable parsing by small models, and token budgets

### Takeaway
Use short labelled plain-text/markdown sections plus a single-line verdict token that code parses; avoid forcing strict JSON or constrained decoding before reasoning has occurred, since format constraints degrade small-model accuracy. Keep each role prompt short and the number of simultaneous instructions low.

### Cited Findings
- Strict format restrictions reduce reasoning performance (10-30% per a follow-up's summary of Tam et al.). [P] — [Let Me Speak Freely?](https://arxiv.org/html/2408.02442v1), [arXiv 2603.13351](https://arxiv.org/pdf/2603.13351)
- For sub-3B models, hard schema decoding raised validity 61.5%->100% but accuracy fell 19.7%->11.0%; recommends free reasoning first, constrain late. [P, preprint, abstract only] — [Constraint Tax](https://arxiv.org/pdf/2605.26128)
- Format-requesting instructions themselves cause much of the loss; recommend decoupling reasoning from formatting (free pass then reformat). [P, preprint, summary only] — [arXiv 2604.03616](https://arxiv.org/pdf/2604.03616)
- Markdown vs JSON output "format tax". [B] — [tianpan.co](https://tianpan.co/blog/2026-04-23-markdown-beats-json-output-format-tax)
- Anthropic ACI advice: choose formats close to natural text, avoid overhead like counting lines or escaping; poka-yoke arguments. [V] — [Building effective agents](https://www.anthropic.com/engineering/building-effective-agents)
- Instruction following: prompt-level accuracy declines as instruction count rises while per-instruction accuracy stays stable; bias toward earlier instructions (later ones dropped); even frontier models only 68% at 500 instructions (not a small-model study). [P] — [IFScale](https://arxiv.org/html/2507.11538v1), [When Instructions Multiply](https://arxiv.org/pdf/2509.21051)
- Hermes skill list costs ~3k tokens in the system prompt; SKILL.md warn threshold ~24k chars; context file caps scale with window. [V] — [Skills](https://hermes-agent.nousresearch.com/docs/user-guide/features/skills), [Context files](https://hermes-agent.nousresearch.com/docs/user-guide/features/context-files)
- CrewAI guardrail retries default 3; Hermes delegate `output_schema` gives one correction turn. [V] — links above.

### Inferences (no source gives exact budgets for local models; these are my engineering suggestions)
- Budget guess: in 8k context, role prompt (SOUL + AGENTS + skill index) <= ~1.0-1.5k tokens total of fixed overhead you control, task+artifact gets the rest; at 32k, <=~2-3k; at 64k, <=~4k. Note the Hermes base system prompt, tool schemas and ~3k skill index also consume context, so on 8k models disable unused toolsets/skills.
- Put the 5-8 most critical rules first (position bias) and restate output format last.
- Handoff format (implementer): `## SUMMARY` / `## CHANGES` (file list) / `## HOW_TO_VERIFY` / `## OPEN_ISSUES`, with the artifact in a file, not inline.
- Reviewer format: per-criterion lines `C1: PASS|FAIL - <evidence>` then final line `VERDICT: PASS` or `VERDICT: FAIL` plus `FIXES:` numbered list (max N items). Parse with regex on last `VERDICT:` line; treat missing/ambiguous as FAIL (fail-closed). Ask evidence before verdict (CoT before verdict mitigates bias per arXiv 2504.03846).
- Where JSON is required, use free-form pass then a reformat pass or Hermes `output_schema`, and validate in code.
- Escalation: round cap 2-3, then output `STATUS: ESCALATE` with unresolved criteria list; also stop on repeated identical FAIL list (no-progress detection).

### Gaps
- No source found giving token budgets for role prompts specific to 8k-64k local models; no measurements of Hermes base prompt size.
- Reviewer rubber-stamping rates for small local reviewers not found.
