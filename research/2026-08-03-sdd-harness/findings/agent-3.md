# Agent 3 — Spec-Driven Development Tooling Landscape & Claude Code Subagent Mechanics (mid-2026)

## (a) OpenSpec

**What it is:** An open-source CLI/skill framework ("Spec-driven development (SDD) for AI coding assistants") maintained at [Fission-AI/OpenSpec](https://github.com/Fission-AI/OpenSpec) (note: several unrelated forks/clones with the same name exist on GitHub — Fission-AI is the canonical one referenced by third-party writeups). No API keys required; works across multiple AI coding tools (Claude Code, Cursor, Devin) via slash commands or context files.

**Structure:**
```
openspec/
├── specs/     # current source of truth (requirements + scenarios, Markdown)
├── changes/[feature-name]/
│   ├── proposal.md   # why/what's changing
│   ├── specs/        # delta requirements/scenarios
│   ├── design.md      # technical approach
│   └── tasks.md        # implementation checklist
└── archive/[date-feature-name]/
```
Workflow verbs (the `opsx` command set): `explore → propose → apply → archive`, plus an "expanded" profile adding `new`, `continue`, `ff` (fast-forward), `verify`, `bulk-archive`, `onboard`. — [github.com/Fission-AI/OpenSpec](https://github.com/Fission-AI/OpenSpec), [docs/opsx.md](https://github.com/Fission-AI/OpenSpec/blob/main/docs/opsx.md)

**Context bloat — documented root causes:**
1. **Auto-generated skills.** OpenSpec installs ~10 dedicated `SKILL.md` files into `.claude/skills/openspec-*/` at setup. Because Claude Code's skill mechanism surfaces skill descriptions/metadata into context up front regardless of whether they're used in a given session, a GitHub issue explicitly asks "why bloat context with 10 skills?" and questions whether they're needed at all for users following a manual (non-autonomous) flow — closed with no maintainer rebuttal. — [Fission-AI/OpenSpec issue #611](https://github.com/Fission-AI/OpenSpec/issues/611)
2. **Context injection is proactive, not on-demand.** OpenSpec wraps up to 50KB of project convention/stack context in `<context>` tags and *prepends it to every artifact instruction* — a static, always-loaded blob rather than retrieved just-in-time. — [docs/opsx.md](https://github.com/Fission-AI/OpenSpec/blob/main/docs/opsx.md)
3. **Verbose artifact chain.** The proposal → design → tasks chain intentionally externalizes reasoning into files so it survives across sessions ("context hygiene," clearing context before implementation is explicitly recommended in the README), but this trades chat-context bloat for filesystem/read bloat — every apply step re-reads proposal+specs+design+tasks. — [github.com/Fission-AI/OpenSpec](https://github.com/Fission-AI/OpenSpec), [jamasoftware.com blog](https://www.jamasoftware.com/blog/openspec-guide/)
4. Independent corroboration: an HN commenter on an OpenSpec discussion notes spec drift causes "duplication and contradictions across specs" over time, and a schema/format tension between rigid YAML (better context control) vs. freeform Markdown (worse). — [HN thread](https://news.ycombinator.com/item?id=47994433)

**Runaway autonomy — documented root cause:**
`/opsx:apply` is explicitly designed to be **"autonomous within scope"** — "the agent handles task implementation without requiring permission between steps," iterating through the entire `tasks.md` checklist in one shot. OpenSpec does offer "delivery modes" from fully-guided to autonomous, but the default apply behavior is multi-task, unattended execution — which is the mechanism that produces large, hard-to-review diffs and agent drift within a single invocation. The one-artifact-at-a-time design was meant as a scope guard, but a comparison piece notes this very constraint felt "sluggish," prompting the maintainers to add a fast-forward (`ff`) escape hatch — which itself reintroduces the batching/autonomy risk it was meant to prevent. — [docs/opsx.md](https://github.com/Fission-AI/OpenSpec/blob/main/docs/opsx.md), [Ovidiu Eftimie comparison](https://ovidiueftimie.substack.com/p/openspec-vs-spec-kit)

**Verdict:** Copy — the delta-spec/archive model (specs = truth, changes = proposed diffs) as a durable, version-controlled memory substitute for chat history. Avoid — auto-installing many always-on skill files per project, and defaulting `apply` to unattended multi-task execution without a per-task confirmation gate.

## (b) GitHub spec-kit

**What it is:** Official GitHub toolkit ([github/spec-kit](https://github.com/github/spec-kit)) implementing a strict, ordered slash-command pipeline for agents like Copilot, Claude Code, Codex.

**Pipeline:** `/constitution` (writes non-negotiable project principles to `.specify/memory/constitution`) → `/specify` (what/why, no implementation detail) → `/plan` (technical blueprint) → `/tasks` (breaks plan+spec into LLM-sized tasks) → `/implement`. A left-to-right hard dependency order is enforced — you cannot plan before specifying or implement before tasking. An `/analyze` command acts as a consistency/quality gate checking spec, plan, and tasks against the constitution. — [zread.ai command reference](https://zread.ai/github/spec-kit/5-core-commands-constitution-specify-plan-tasks-and-implement), [github/spec-kit](https://github.com/github/spec-kit)

**Documented criticisms:**
- **Heavy artifact overhead, weak enforcement.** A Scott Logic review (cited across secondary sources) reported the plan phase alone generating 2,000+ lines of Markdown including a 406-line research doc it judged duplicative; the reviewer estimated they were "around ten times faster" with plain iterative prompting than the full pipeline. Governance is by convention (the constitution is just a doc the agent is told to check), not enforced mechanically.
- **Spec-to-bug gap / rework cost.** [Issue #1092 "High Level Design Concerns"](https://github.com/github/spec-kit/issues/1092): after heavy spec investment, implementation quality can still be poor; no defined workflow for going from a bug back into the spec, so users end up rewriting specs repeatedly; author states directly: "the $ cost of time writing/rewriting specs is costing me more than the $ cost of writing code" — risk of "great specs, no MVP."
- **Context doesn't persist between chat sessions** despite great specs — one user reported moving away from spec-kit because "AI forgets everything between sessions... every new chat starts from zero," i.e., the artifacts exist but aren't automatically re-loaded/re-grounded. — [LogRocket](https://blog.logrocket.com/github-spec-kit/), [den.dev](https://den.dev/blog/github-spec-kit/)
- **Maintenance/stability concerns and breaking changes.** Community discussion questioned ongoing maintenance after a key contributor left Microsoft; v0.10.0 removed the `--ai` flag family, breaking pre-June-2026 tutorials/scripts. — [Discussion #1482](https://github.com/github/spec-kit/discussions/1482)

**Verdict:** Copy — the `constitution` idea (a standing, checkable set of project principles distinct from any one feature spec) and the hard-ordered command dependency chain. Avoid — no built-in loop from bug/failure back into spec revision, and no cost/time governor on spec generation itself (it can balloon documentation for trivial changes).

## (c) AWS Kiro & EARS

**What it is:** AWS's agentic IDE ([kiro.dev](https://kiro.dev/docs/specs/)), spec-driven by design — code generation is gated on structured specs.

**Structure:** Three files per feature — `requirements.md` (user stories + acceptance criteria in **EARS notation**: "WHEN [condition/event] THE SYSTEM SHALL [expected behavior]," chosen for testability and traceability), `design.md` (architecture, sequence diagrams), `tasks.md` (discrete, dependency-sequenced, trackable tasks). Two entry paths — **requirements-first** (behavior → design → tasks) or **design-first** (architecture/pseudocode → requirements → tasks) — both converge on the same task structure. A "Quick Spec" mode runs all three phases with no approval gates for simple features. — [kiro.dev/docs/specs/feature-specs/](https://kiro.dev/docs/specs/feature-specs/)

**Note:** EARS itself is a pre-existing formal requirements syntax (not Kiro-invented) that Kiro adopted to make requirements machine-checkable/testable rather than prose. A secondary comparison source (CodeMySpec) frames Kiro's EARS notation against BDD-style specs; EARS's origin/history not independently verified — out of scope for this pass.

**Verdict:** Copy — EARS-style structured requirement statements are a cheap, high-value addition for making "requirements" machine-checkable rather than free prose, and the two-entry-path (requirements-first vs design-first) flexibility. Avoid — nothing disqualifying found; Kiro is IDE-proprietary so its command/approval-gate mechanics aren't directly portable to a Claude Code harness.

## (d) Claude Code subagents — official mechanics (mid-2026)

**Definition & frontmatter** ([code.claude.com/docs/en/sub-agents](https://code.claude.com/docs/en/sub-agents)): a subagent is a Markdown file with YAML frontmatter (`name`, `description` — used by Claude to decide when to delegate, `tools` — comma-separated allowlist; **omit to inherit all tools**, `model` — lets you route to a cheaper/faster model like Haiku for cost control) followed by the system prompt as the Markdown body. Project-level agents live in `.claude/agents/` and are committed to the repo so the whole team shares the same specialist definitions; user-level agents are personal.

**Context isolation:** Each subagent runs in its **own context window** with its own tool access and permissions. The stated purpose is explicit: "the subagent does that work in its own context and returns only the summary" — used when a side task would otherwise flood the main conversation with search results, logs, or file contents that won't be referenced again.

**Nesting (cannot spawn subagents) — this has been in flux:**
- For roughly two years the rule was absolute: a subagent could not invoke the Task tool to spawn another subagent.
- v2.1.172 (~June 9, 2026) quietly broke that rule.
- July 21, 2026 (v2.1.217): nesting disabled outright, concurrency capped at 20 simultaneous subagents (`CLAUDE_CODE_MAX_CONCURRENT_SUBAGENTS`), and a pre-existing bug where `--max-budget-usd` didn't count background-subagent spend was fixed.
- July 24, 2026 (v2.1.219): nesting reinstated at a default depth limit of **3** (`CLAUDE_CODE_MAX_SUBAGENT_SPAWN_DEPTH`; set to 1 to fully disable).
- A separate per-session cap of 200 total subagents exists (v2.1.212, `CLAUDE_CODE_MAX_SUBAGENTS_PER_SESSION`, cannot be fully disabled).
- **Caveat:** official docs in places still describe the old "cannot nest" behavior, lagging the actual shipped behavior — treat this as an area of doc/behavior drift; verify against the live CLI version rather than assuming either extreme. — [digitalapplied.com](https://www.digitalapplied.com/blog/claude-code-subagent-depth-limits-budget-caps-2026), [anthropics/claude-code issue #61993](https://github.com/anthropics/claude-code/issues/61993), [issue #4182](https://github.com/anthropics/claude-code/issues/4182)
- The deep-research harness's own operating manual assumes the **strict no-nesting model** ("you do not nest... all fan-out happens from the main session") — that is the safe, conservative design choice and matches the 2-year default even though the CLI now technically permits shallow nesting.

**Slash commands & hooks:** Custom slash commands = Markdown files in `.claude/commands/`, invoked by filename, with `$ARGUMENTS` substitution. Hooks ([code.claude.com/docs/en/hooks](https://code.claude.com/docs/en/hooks)) are user-defined shell commands / HTTP endpoints / LLM prompts firing at lifecycle events, communicating via stdin/JSON and exit codes/stdout/stderr — useful for enforcing policy (e.g., blocking a commit, validating an artifact) outside the model's own judgment.

**Anthropic's multi-agent research system** ([anthropic.com/engineering/multi-agent-research-system](https://www.anthropic.com/engineering/multi-agent-research-system), [effective-context-engineering-for-ai-agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)): orchestrator-worker pattern — a LeadResearcher plans, writes the plan to persistent memory, then spawns parallel subagents each independently searching and reasoning, returning only a **condensed 1,000–2,000 token summary** back to the lead even though each subagent may burn tens of thousands of tokens internally. Opus-lead + Sonnet-subagents beat single-agent baselines by >90% on their internal eval. Explicit trade-off documented: **multi-agent fan-out is a poor fit for tasks needing shared context or with heavy inter-agent dependencies — most coding tasks have fewer truly parallelizable subtasks than research does**, i.e., this architecture was validated for *research*, not general coding, and should be applied to coding with that caveat in mind.

**Verdict:** Copy — strict tool/model scoping per subagent via frontmatter, "return condensed summary not raw transcript" as the core context-isolation contract, and hooks for hard policy enforcement instead of relying on the model to self-police. Avoid — relying on subagent nesting (behavior is unstable/version-dependent and officially documented behavior lags shipped behavior); avoid assuming the research-system's high-parallelism pattern transfers cleanly to coding tasks, which Anthropic itself flags as having fewer independent/parallelizable threads and more cross-agent dependencies.

## Source quality notes
- Fission-AI/OpenSpec repo and its `docs/` files are primary and current; issue #611 is a single unresolved user complaint (n=1, no maintainer response) — treat as illustrative, not conclusive, though it directly corroborates the user's stated experience.
- spec-kit criticisms are second-hand via aggregator/summary sources (LogRocket, den.dev) referencing "Scott Logic's review," which could not be fetched directly — flagged as a gap.
- code.claude.com/docs pages are authoritative/primary and current as of the fetch.
- The subagent-nesting timeline (digitalapplied.com) is a secondary blog synthesizing multiple GitHub issues/PRs; the underlying GitHub issues corroborate the *existence* of the flip-flopping but exact dates/version numbers were not independently double-verified.
- Anthropic engineering blog posts are primary/highly authoritative.
- Kiro docs (kiro.dev) are primary/official.

## Open questions (outside boundary)
- EARS notation's origin/history and adoption elsewhere — not investigated.
- Scott Logic's original spec-kit review post was never located/fetched directly — worth a targeted follow-up if spec-kit detail needs firming up.
- BMAD-METHOD was mentioned as a heavier per-persona alternative to OpenSpec (jamasoftware blog) — not covered here.
- The precise current (Aug 2026) nesting-depth default should be re-verified against the live docs at harness-design time.

## Ranked sources
1. [code.claude.com/docs/en/sub-agents](https://code.claude.com/docs/en/sub-agents) — official Anthropic docs, primary.
2. [anthropic.com/engineering/multi-agent-research-system](https://www.anthropic.com/engineering/multi-agent-research-system) — Anthropic engineering blog, primary.
3. [anthropic.com/engineering/effective-context-engineering-for-ai-agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) — Anthropic engineering blog, primary.
4. [github.com/Fission-AI/OpenSpec](https://github.com/Fission-AI/OpenSpec) and [docs/opsx.md](https://github.com/Fission-AI/OpenSpec/blob/main/docs/opsx.md) — primary repo/docs.
5. [github.com/github/spec-kit](https://github.com/github/spec-kit) — primary repo.
6. [kiro.dev/docs/specs/feature-specs/](https://kiro.dev/docs/specs/feature-specs/) — official AWS Kiro docs.
7. [Fission-AI/OpenSpec issue #611](https://github.com/Fission-AI/OpenSpec/issues/611) — primary user complaint, direct corroboration of context-bloat claim.
8. [github/spec-kit issue #1092](https://github.com/github/spec-kit/issues/1092) — primary user complaint on spec ROI/quality gap.
9. [digitalapplied.com subagent depth limits](https://www.digitalapplied.com/blog/claude-code-subagent-depth-limits-budget-caps-2026) — secondary synthesis, useful but dates/versions not independently double-checked.
10. [Ovidiu Eftimie: OpenSpec vs Spec-Kit](https://ovidiueftimie.substack.com/p/openspec-vs-spec-kit) — practitioner comparison, reasonably balanced, second-hand.
11. [jamasoftware.com/blog/openspec-guide](https://www.jamasoftware.com/blog/openspec-guide/) — vendor content-marketing but accurately describes mechanism; light on downsides.

<!-- STATUS: COMPLETE -->
