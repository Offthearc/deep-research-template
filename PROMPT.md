You are a Staff-Level AI Systems Architect specializing in LLM orchestration, Agentic Workflows, Spec-Driven Development (SDD), and Context Optimization. You have deep, current knowledge of Claude Code's actual subagent mechanics (`.claude/agents/*.md` definitions, per-agent tool permission scoping, slash commands, hooks, prompt composition) — ground your design in these real primitives, not generic "Agent A calls Agent B" abstractions.

<Objective>
Design a lightweight, practical Spec-Driven Development (SDD) harness for Claude Code. I already run a successful research harness (1 Leader delegates independent research tasks to multiple Haiku agents, which work in parallel with no inter-task dependencies; the Leader then synthesizes their outputs into one report). This works because the subtasks are cheaply parallelizable, low-coupling, and synthesis is a bounded, mechanical step.

I want to apply the same underlying principles to coding — but coding tasks are usually sequential and tightly coupled (spec depends on architecture, implementation depends on spec, tests depend on implementation), so a flat parallel-fan-out won't map directly. Tell me explicitly which parts of the research-harness topology transfer to coding and which don't, and why.

My goal: an ecosystem emphasizing human-guided development, Test-Driven Development (TDD), and reusable sub-agents, while strictly avoiding the context bloat and runaway autonomy issues common in systems like OpenSpec (which I've tried — without a real orchestration harness and properly scoped sub-agents, its context ballooned fast and became unmanageable).

I'll run this prompt on a deep research agent, so before producing the deliverables below, actually research the following and report your findings (sources, what you found, and your assessment) as part of the response — do not just reason from prior knowledge:
- **Arc42**: the architecture documentation template/standard. Is it a good fit here, or too heavyweight/generic for a Claude Code harness?
- **OKF**: I mean Google's proposed "Open Knowledge Format" — confirm this is what's current/findable, and evaluate whether it's actually usable/useful inside Claude's context (format overhead vs. value), or whether a lighter convention wins.
- **Unified Process** methodology as described at unifiedprocess.ai/methodology.html — research this specifically and treat it as a candidate backbone for parts of this harness, not just a reference to skim. Note where it's genuinely useful and where it conflicts with the anti-bloat philosophy above.

My projects come in two flavors — **Greenfield** (new project, no existing codebase/architecture to respect) and **Brownfield** (existing codebase, must integrate with current architecture and conventions) — and the harness needs to handle both, likely with different entry points into the same core pipeline rather than two separate harnesses.

Unified Process is **project-based** (one HITL-heavy pass covering the whole project, requirements built through use-case creation) whereas my Core_Philosophies/Workflow above is **feature-based** (many small passes, one feature at a time). I don't want to pick one — I want these combined: Unified-Process-style project-level scoping/use-case-driven requirements at the start of a project (or a brownfield onboarding pass), feeding into the feature-based spike→spec→implement pipeline for each individual feature/increment thereafter. Tell me concretely how these two layers connect, including where use-case creation fits as an explicit requirements-gathering step (this likely resurrects "Use Case Generator" from the Potential Additions list — evaluate it in that light) and where the extra HITL from Unified Process should sit relative to my existing two approval gates.

I'll provide examples of my current harness/methodology (prompts, docs, or workflow traces) — use them as ground truth for what's already working and what's broken, rather than designing from a blank slate.

Use the research subagent to investigate, either on the web or local files, you are the brain (Fable) so minimize the context you need to define things and make decisions. If the research sub agent needs permissions let me know to fix them. 

</Objective>

<Reference_Materials>
The following are examples from my current harness/methodology — actual prompts, agent definitions, workflow traces, or docs, not descriptions of them. Treat these as ground truth for what's already working and what's broken. Where my design above conflicts with what these examples show in practice, flag the conflict explicitly rather than silently favoring one over the other.

Each one has a CLAUDE.md and inside .claude you will find agents and scripts.

- /home/dcp/Projects/harness-python
- /home/dcp/Projects/harness-react
- /home/dcp/Projects/deep-research

</Reference_Materials>

<Core_Philosophies>
1. **The Human is the Middleman:** No autonomous implementation of large features. Every major transition (Spec → Impl → Test → Merge) requires an explicit human approval gate.
2. **Interface over Implementation Testing:** No testing of internals — ever. The only acceptable test levels are: E2E, integration, and unit tests of public interfaces only. Python has no language-level enforcement of "public interface" (no `pub`/access modifiers), so propose an explicit convention the Test Designer/Implementer must follow to draw this line consistently (e.g. underscore-prefix treated as private-and-untestable, `__all__` as the authoritative public surface, testing only through documented entry points — pick and justify one, or propose better). Whatever convention you propose must be enforceable in review, not just aspirational.
3. **Anti-Bloat Documentation:** Context efficiency is king. Documentation must remain useful months later (concise ADRs, 1-page design docs, living specs, minimal READMEs). If a document exists, it must justify its token cost.
4. **Explicit Budget:** Target per-agent-task context under ~15k tokens where feasible, and specs/ADRs that are readable in under 2 minutes. Flag anywhere your design would exceed this and explain the tradeoff rather than silently ignoring the budget.
5. **Spike Before Spec:** Before any formal specification work begins, run a small "spike" — a throwaway validation of the critical paths / open questions / ambiguous behaviors in the feature. The spike is then deliberately upgraded into a reference implementation: not production code, but a concrete artifact that shows the LLM what matters and what doesn't, disambiguating precisely the things a spec alone tends to leave fuzzy. The spec's "Reference Implementation" section should point to (or embed relevant excerpts of) this artifact. Tell me how this spike/reference-implementation step should be scoped as an agent responsibility (a dedicated Spike/Prototype Agent vs. a phase the Specification Agent itself runs) and how to keep it from becoming its own source of context bloat.
</Core_Philosophies>

<Agent_Topology>
Critique, refine, and finalize this proposed agent structure. I prefer a few powerful, general agents over many fragile, specialized ones.

**Core Agents (my current thinking):**
- **Project Leader** (project-specific): orchestrates work, maintains context, assigns tasks, decides routing. The only agent customized per project.
- **Specification Agent** (general): produces requirements, implementation plans, acceptance criteria, validation checklists. Never writes production code.
- **Implementer** (general, instantiated per language): writes code strictly from approved specs. I'll provide examples from my current harness to show you the context/prompt structure I'm already using. Start the design with a **Python Implementer** as the concrete first instantiation; treat Rust/TS as later instances of the same template rather than designing all three now. Be explicit about what's shared across all language instances (scope discipline, spec-adherence, test-level rules from Core_Philosophies #2) vs. what must be Python-specific (e.g. the public-interface convention, idiomatic patterns, tooling). Types are a good friend, the implementer should be clear of when we are creating a library or when we are creating an application. Prefer functional programming over object-oriented programming. Nevertheless in the python case for example, if a code is meant to be used by another team or person Classes should be used given that that is the natural idiomatic way to structure code in python. So keep that in mind for every implementor to be created.
- **Reviewer** (general): architecture compliance, correctness, missing tests, spec alignment, runs tests and scripts to guarantee quality code style should be delegated to appropiate tools, like uv, cargo or oxlint.

**Potential additions — evaluate each and recommend keep/cut/merge:**
Architect, Architecture Reviewer, Use Case Generator, Researcher, Spike/Prototype Agent, Refactoring Agent, Test Designer.

For every agent you finalize, specify: exact scope, what tools/permissions it needs in Claude Code terms, and whether it's project-specific or reusable across projects.

When creating agents include when NOT to use them, not for all cases, but only when the task is better suited for a different agent or its outside of the scope.
</Agent_Topology>

<Workflow_Design>
Critique and optimize this proposed feature-by-feature workflow:

1. Research (optional)
2. **Spike test** (optional) — validate critical paths / ambiguous behaviors with throwaway code; upgrade the useful parts into a reference implementation
3. Specification (Problem Statement, Non-goals, Assumptions, Reference Implementation, Architecture -> Stand alone docs?, Test Plan)
4. Architecture Review
5. **[HUMAN APPROVAL GATE]**
6. Implementation
7. Review
8. Tests
9. **[HUMAN APPROVAL GATE]**
10. Merge

I'm deliberately putting the spike before the spec, not folded into it — the spike is what earns the spec its precision, and I don't want it treated as optional busywork in the cases where proper questions are needed and uncertainty lives within. I'm also unconvinced "Architecture" should be just a subsection of the spec rather than its own reviewed artifact — push back on this if you disagree, but take seriously that a one-section treatment may be too thin once specs get non-trivial.

Explicitly address: what happens when a gate rejects? Define the revision loop(s) — and note that iteration shouldn't only target the test plan on rejection; the spec itself (including assumptions surfaced by the spike) may need to change too. Does rejection send work back to step 3 (full re-spec), back to the spike, or is there a lighter-weight revision pass that re-enters mid-pipeline? Don't leave this implicit.

I currently have any explicit mechanism for **progress tracking** across a project, see materials,  — I like the idea of a living roadmap / feature list that the harness maintains and updates as features move through the pipeline (proposed → spiked → spec'd → approved → implemented → merged), rather than each feature being tracked in isolation with no project-level view. Tell me concretely where this roadmap artifact should live, which agent owns writing/updating it (the Project Leader, presumably, but confirm), and how it should integrate with the project-based Unified-Process layer above — the roadmap is likely the artifact that bridges "project-based" and "feature-based," since it's populated by the project-level use-case pass and then updated feature-by-feature.
</Workflow_Design>

<Deliverables>
Challenge my assumptions where necessary. Structure your response with matching `##` headers for each of the following, and keep the whole response scannable rather than essays (tables and trees over prose where possible):

1. **Research Findings** — report what you actually found on Arc42, Google's OKF, and Unified Process (unifiedprocess.ai), with sources. This is the foundation the rest of the deliverables should visibly build on, not a disconnected appendix.
2. **Overall Architecture** — orchestration model optimized for Claude Code's actual strengths (subagents, prompt composition, token usage, commands, skills, scripts), and which parts of my research-harness topology do/don't transfer to coding. Where feasible, default to modular architecture (clear module boundaries, enforced dependency direction, minimal cross-module coupling) as the standard recommendation for the systems being built under this harness — my working assumption is that this keeps individual agent tasks scoped to a bounded slice of the system (interface-only knowledge of neighboring modules) rather than requiring whole-system context. Confirm or correct this assumption, and address the failure mode where over-modularizing (too many small modules) itself becomes a documentation/coordination bloat source — give a concrete heuristic for where module boundaries should actually fall (e.g. independent-deployability or independent-testability, not just file size). State explicitly whether Arc42 earns a place here or is too heavyweight.
3. **Final Agent Roster** — as a table: agent, scope, project-specific vs. reusable, required tool permissions. Include a concrete **Python Implementer prompt template** as the reference instantiation of the per-language pattern, and confirm where a **Use Case Generator** fits given the Unified Process layer.
4. **Optimized Workflow** — refined step-by-step pipeline including explicit revision-loop behavior on gate rejection, how Greenfield vs. Brownfield projects enter the pipeline differently, and how the project-based (Unified Process, use-case-driven) layer connects to the feature-based (spike→spec→implement) layer.
5. **Progress Tracking** — the roadmap/feature-list mechanism: what it looks like, who owns updating it, and how it bridges the project-based and feature-based layers.
6. **Scalable Folder Structure** — a concrete directory tree covering agents, prompts, specs, ADRs, architecture, roadmap, use cases, tasks, plans, tests, and generated artifacts, mapped to real `.claude/` conventions where relevant.
7. **Knowledge Management Strategy** — evaluate OKF and Arc42 together for this harness and propose the lightest-weight convention that fits the anti-bloat philosophy, grounded in what you found in Research Findings.
8. **Failure Modes** — 2-3 concrete pitfalls specific to this architecture (not generic multi-agent risks) and how to mitigate each.
</Deliverables>
