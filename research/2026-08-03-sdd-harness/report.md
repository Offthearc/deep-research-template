# SDD Harness for Claude Code — Design Report

**Question:** Design a lightweight, practical Spec-Driven Development harness for Claude Code that combines a Unified-Process-style project layer with a feature-based spike→spec→implement pipeline, avoids OpenSpec-style context bloat, and reuses what already works in the user's research + coding harnesses.

**TL;DR:** Keep the research harness's *contracts* (lead-orchestrator, 5-field delegation, state checkpointing, completion markers, persist-by-owner), drop its *topology* (flat parallel fan-out) — coding is a relay race, not a fan-out. Run a one-time project layer (use cases + module map + constitution) that feeds a per-feature pipeline with exactly two human gates. Use arc42 only as a vocabulary (its canvas as the 1-page architecture format), borrow OKF's frontmatter *pattern* but not the spec, and adopt AIUP's "Harness layer" (constitution) and change-routing rule while rejecting its per-feature ceremony. Roster: 5 agents + you-as-Leader; cut Architect, Architecture Reviewer, Refactoring Agent, and Test Designer as standing agents.

---

## 1. Research Findings

### Arc42 (findings/agent-1.md)
- Free, open template (2005, Starke/Hruschka), 12 sections, all explicitly optional [1]; the project itself ships a lean single-page variant family — the **Software Architecture Canvas** (Architecture Communication Canvas = "zip version" of full arc42, Architecture Inception Canvas for greenfield) [2]. Sources: [1], [2], [3], [4].
- Practitioner criticism targets *misuse* (filling all 12 sections regardless of project size), not the template itself [5]; arc42's own FAQ frames it as anti-bloat and agile-compatible [3][4].
- **Verdict:** full arc42 is too heavyweight to load per-feature inside a ~15k budget (Constraints, Deployment View, Glossary, Quality Tree are static across features). But it *earns a place as vocabulary*: use the canvas as the shape of the 1-page project architecture doc, and section 9 (Architectural Decisions) as the ADR slot [1]. Detail in §7.

### Google OKF (findings/agent-1.md)
- **You were not misremembering — it's real and very new.** Google Cloud announced the **Open Knowledge Format** on **June 12, 2026** (7 weeks old at research time) [6][7]: a vendor-neutral spec for packaging AI-agent context as a directory of markdown files with YAML frontmatter. Spec v0.2 at `GoogleCloudPlatform/knowledge-catalog/okf/SPEC.md` [8]. (Distinct from the unrelated Open Knowledge Foundation.)
- Only mandatory field: `type`. Reserved filenames `index.md` (progressive disclosure) and `log.md` (change history). Optional provenance/trust/lifecycle field families, actor-identity conventions, version negotiation. [8]
- Independent critique (Marc Bara): OKF standardizes the *container*, not semantics — `type` values are unregistered free text, so conformant bundles can be mutually unintelligible. Reference tooling tilts Google-ecosystem despite vendor-neutral framing. [9]
- **Verdict:** the *pattern* (markdown + minimal frontmatter + index files) is exactly right and is what this harness should do anyway; the *spec* is machinery for a multi-producer knowledge-exchange problem you don't have. Borrow 3 frontmatter fields, skip the rest. Detail in §7.

### Unified Process — unifiedprocess.ai (findings/agent-2.md)
- **AIUP ("AI Unified Process")**: "Specs at the center. AI handles the rest." [11] Use-case specs are the authoritative source; code is regeneratable. Four RUP-named phases (Inception/Elaboration/Construction/Transition), two human roles (Requirements Engineer, Software Engineer) [10], explicit Greenfield and Brownfield workflows (brownfield = reverse-engineer entity model + use cases from code, then baseline the spec) [10][13]. Ships as Claude Code plugins (`/requirements`, `/entity-model`, `/use-case-spec`, `/implement`) [12].
- Lineage is the *lightweight* branch of UP — Jacobson's Use-Case 2.0/3.0 [14][15] — not document-heavy RUP.
- Its best idea for you is the **"What / Harness / How" three-layer model**: What = use cases/domain artifacts; **Harness = written engineering guardrails (conventions, test strategy, allowed libraries — the tacit senior-dev knowledge)**; How = generated code [16]. The Harness layer is precisely your project-level constitution.
- **Genuinely useful:** spec-as-single-routing-point for all change ("spec changes first"); feature/change/bug treated uniformly as use-case modification (bug = code disagrees with spec; enhancement = spec must change) [10][14]; test-protected regeneration; the Harness layer; brownfield reverse-engineering flow.
- **Conflicts with anti-bloat:** its Elaboration artifact set (entity model + BPMN + architecture doc + mockups + OpenAPI, all before code) is per-project ceremony that must NOT be reenacted per feature [10]; its HITL is continuous-but-advisory ("last chance to course-correct cheaply") [17] — *weaker* than your two hard gates, so adopt its gate *positions* but your gate *enforcement*. AIUP has **no spike concept** and **no token-budget concept** — both are your additions on top of it.

### SDD tooling landscape (findings/agent-3.md)
- **OpenSpec bloat root causes confirmed:** (1) installs ~10 always-surfaced `SKILL.md` files per project (issue #611, unanswered) [18]; (2) prepends up to **50KB** of project context to *every* artifact instruction, statically rather than on demand [19]; (3) `apply` is "autonomous within scope" by default — iterates the whole `tasks.md` unattended, producing exactly the runaway-autonomy you hit; the `ff` fast-forward escape hatch reintroduces the batching it was meant to prevent [19][20]. **Copy** its delta-spec model (specs/ = truth, changes/ = proposed diffs, archive/) [21]; **avoid** always-on context injection and unattended multi-task apply.
- **spec-kit:** copy the **constitution** (standing, checkable project principles) and hard-ordered command chain [22]; avoid its failure modes — a review reported 2,000+ lines of markdown from one plan phase and "10× faster without it" [23][24]; issue #1092: no bug→spec revision loop, "cost of rewriting specs exceeds cost of writing code." [25]
- **Kiro:** copy **EARS-notation acceptance criteria** ("WHEN … THE SYSTEM SHALL …") — machine-checkable requirements at near-zero token cost — and the requirements-first/design-first dual entry. [26]
- **Claude Code mechanics (constrains everything below):** subagents = `.claude/agents/*.md` with `name`/`description`/`tools`/`model` frontmatter; each runs in its own context window and returns a summary [27]; hooks give *mechanical* (non-model) policy enforcement [28]; subagent nesting has flip-flopped in 2026 (currently shallow-nesting allowed, docs lag) → **design for no nesting** [29]; Anthropic's own multi-agent research write-up explicitly warns the parallel pattern was validated for research, and that coding has fewer parallelizable subtasks — direct confirmation of your intuition [30].

### Ground truth — your harnesses (findings/agent-4.md, agent-5.md)
- **What's working:** hook-enforced verification (PostToolUse pytest/commit hooks, Stop→init.sh, Bash allowlist — the harness, not the prompt, prevents skipping tests) [31]; reviewer barred from approving on red [32]; scope-proportionality rule (PoC ≈ 50 lines of docs / MVP ≈ 100 / Production ≈ 200) [33]; react's design-token pipeline [34]; deep-research's state.md + `<!-- STATUS: COMPLETE -->` + persist-by-lead contract [35].
- **What's broken (conflicts flagged, per your instruction):**
  - `AGENTS.md` truncated identically in both repos (dies at "## 4. How to choose a task") — a shared broken template. [36][37]
  - harness-python `docs/verification.md` has two empty code fences — the "prove it works" doc is itself unfilled. [38]
  - React agents were forked by copy/paste from Python: `implementer.md` still says `uv run pytest` [39], `reviewer.md`'s example still references `src/cli.py` [40]. **This is the strongest argument for a shared-core + language-overlay implementer design.**
  - `architect.md` is ~1,900 words in both repos, 3× any other prompt [33][41] — your own documented bloat case, and the main input to the "cut the standing Architect" decision below.
  - No `model:` frontmatter in any coding agent [33][41] — contradicts your research harness's own cost discipline [35].
  - **Design-conflict flag:** your PROMPT says the leader "maintains context, assigns tasks"; your harness-python `leader.md` says *"Never accept a subagent result that arrives as raw text in chat"* (file-reference-only returns) [42], while deep-research says the opposite (*inline returns, lead persists*) [35]. These are opposite persistence contracts. Resolution in §2: coding workers own their artifact files (code must be written to disk anyway); the *status* comes back inline and tiny.

---

## 2. Overall Architecture

### What transfers from the research harness, and what doesn't

| Research-harness mechanism | Transfers? | Why |
|---|---|---|
| Lead orchestrator in the main session; no nesting | ✅ Yes | Same role; Claude Code's nesting behavior is unstable in 2026 [29] — keep all fan-out at the top. |
| 5-field delegation contract (objective / output format / source guidance / boundaries / budget+identity) | ✅ Yes, re-skinned | "Source guidance" becomes "authoritative context": which files define truth (spec, constitution, module interface docs) and which are out of scope. |
| `state.md` checkpoint + phase machine + resume | ✅ Yes | Works for any interruptible multi-step process; becomes per-feature `state.md`. |
| Completion marker trusted over state file | ✅ Yes, upgraded | For coding, the marker is *the tests passing* + a status block — artifacts over assertions. |
| Persist-by-lead (workers never write files) | ⚠️ Modified | Code is file-shaped; implementers must write `src/` and `tests/`. Replace with **single-writer-per-artifact-class**: each artifact has exactly one owning agent (spec → Spec Agent, code+tests → Implementer, review verdict → Reviewer, roadmap/state → Leader). Workers return a ≤300-token status summary inline; the lead never ingests diffs. |
| Effort-scaling table (don't over-spawn) | ✅ Yes | Trivial change ⇒ skip spike, collapse pipeline (see §4 sizing tiers). |
| Flat parallel wave of homogeneous workers | ❌ No | Coding subtasks are dependent (spec→tests→impl) and mutate shared state. Anthropic's own write-up flags this [30]. Parallelism survives only where independence is real: research fan-out, brownfield Explore sweep, and features touching **disjoint modules**. |
| Mechanical synthesis (staple N findings together) | ❌ No | Merging code requires compile/test gates, not concatenation. The "synthesis" of a feature is the green test suite + review verdict. |
| Single end-of-run quality pass (citation-agent) | ⚠️ Partial | Final review stays, but correctness must be checked incrementally (TDD + hooks), not only at the end. |

**The topology, in one line:** research is a *fan-out with mechanical merge*; coding is a *relay race with two human checkpoints* — same lead, same contracts, different shape:

```
PROJECT LAYER (once, or on onboarding)          FEATURE LAYER (many small passes)
/project-init | /project-onboard                /feature <id>
  vision → use cases → module map                 [research] → [spike] → spec →
  → constitution → roadmap                        spec review → GATE 1 →
  (HITL conversation, not a gate)                 TDD implement → review → GATE 2 → merge
        └────────── populates roadmap.md ──────────┘   └── updates roadmap.md ──┘
```

### Modularity assumption — confirmed, with a correction
Your assumption is right and is the *load-bearing* context-economy mechanism: modular boundaries are what let an Implementer hold one module's internals + neighbors' interfaces only, keeping tasks under ~15k tokens. Interface-only knowledge of neighbors is also exactly what makes your interface-only testing rule (Philosophy #2) coherent — the test surface and the context surface are the same surface.

**The correction:** the failure mode is real — every module costs a doc, a boundary to police, and a line in every agent's context. Heuristic for where boundaries fall:

> **A module boundary is earned when the interface is meaningfully smaller than the implementation** (interface compression), **and the module is independently testable through that interface alone** (you can write its tests without importing anything internal to it or to its neighbors). If a candidate module's interface would just mirror its implementation, or its tests would need a neighbor's internals, it's not a module — it's a folder.

Corollaries: file size and "feels big" are not reasons to split; a module doc that can only restate the code means merge the module back; target the count where `architecture/modules/` stays readable in one sitting (roughly 3–9 modules for the project sizes in your reference repos).

### Arc42's place (explicit, as requested)
Not as a template. Yes as (a) the **Architecture Communication Canvas** shape for `architecture/overview.md` (1 page, greenfield uses the Inception Canvas variant) [2], and (b) the section-9-style ADR convention [1]. Full 12-section arc42 is rejected for this harness — wrong grain (whole-system, stakeholder-oriented) for per-feature work and unaffordable per-task.

---

## 3. Final Agent Roster

**Evaluations of your proposed additions:** Architect — **cut** as standing agent (your own 1,900-word architect.md is the bloat exhibit [33]; project-level architecture is produced in `/project-init` with you in the loop; feature-level architecture is a spec section, promotable per §4). Architecture Reviewer — **merge** into Reviewer (spec-review mode reviews architecture; separate agent = separate context load for the same judgment). Use Case Generator — **resurrected but merged**: it's the project-layer *mode* of the Specification Agent (`/project-init` invokes it), not a standing agent; keeping it separate would duplicate the Spec Agent's requirements skills. Researcher — **keep**: reuse your existing `research-subagent` unchanged. Spike/Prototype Agent — **keep as dedicated agent** (see rationale below). Refactoring Agent — **cut**: a refactor is a feature pass whose spec is a refactoring charter (behavior-preserving, tests-as-contract); no new agent needed. Test Designer — **cut as agent, keep as artifact**: the Test Plan is a mandatory spec section (Spec Agent writes it, human approves it at Gate 1, Implementer executes it red-first, Reviewer verifies tests match plan). A separate Test Designer would add a fourth handoff carrying the same spec context.

**Spike agent rationale (your open question):** dedicated agent, not a Spec Agent phase — for a *context* reason, not a role-purity reason. Spike work generates the noisiest context in the pipeline (dead ends, stack traces, abandoned attempts). Run it in a subagent and that noise dies with the subagent's context window; the Spec Agent reads only the distilled `findings.md` + reference excerpt. Folding the spike into the Spec Agent would make the spec's context window carry the whole exploration transcript — the exact bloat you're avoiding. Anti-bloat caps: spike writes only under `features/<id>/spike/`; returns findings ≤1 page + a reference implementation ≤150 lines; tool budget ~20 calls; hook forbids `src/` writes and any import from `spike/` in production code.

| Agent | Scope | Reuse | Model | Tools (frontmatter) | NOT for |
|---|---|---|---|---|---|
| **Project Leader** | You + a thin `CLAUDE.md` role in the **main session** (not a subagent — matches harness-python). Routes pipeline, spawns agents, owns `roadmap.md` + `state.md`, enforces gates. Never edits `src/`, `tests/`, or specs. | Project-specific (only the pipeline table + module map pointer vary) | main session | `Read, Glob, Grep, Bash, Agent` | Writing any deliverable artifact itself. |
| **Specification Agent** | Feature mode: `spec.md` (problem, non-goals, assumptions, reference-impl excerpt, architecture delta, EARS test plan). Project mode (= Use Case Generator): vision, `UC-NNN` use cases, roadmap seed. Never writes code. | General | opus | `Read, Write, Edit, Glob, Grep, AskUserQuestion` | Spiking, implementing; investigating the web (delegate to Researcher via Leader). |
| **Spike Agent** | Throwaway validation of a feature's open questions; distills to `spike/findings.md` + `spike/ref/`. | General | sonnet | `Read, Write, Edit, Glob, Grep, Bash` | Features with no open questions (skip); anything meant to be merged. |
| **Implementer-python** (template below; -rust/-ts later) | One approved spec → code + tests, TDD, in one module's write scope. | General core + per-language overlay | sonnet | `Read, Write, Edit, Glob, Grep, Bash` | Unapproved/absent spec; changing the spec (must report `SPEC-CONFLICT` and stop); touching neighbor-module internals. |
| **Reviewer** | Spec mode (pre-Gate-1): completeness, testability, architecture fit, budget. Code mode (pre-Gate-2): reruns full verification suite, spec-adherence, test-level rules. Read-only + run. | General | opus | `Read, Glob, Grep, Bash` | Fixing anything (verdict + required-changes list only). |
| **research-subagent** | Unchanged from deep-research repo. | General (already exists) | sonnet | `WebSearch, WebFetch, Read, Glob, Grep` | Trivial lookups (Leader handles). |

Note the fix to a ground-truth defect: **every agent now pins `model:`** (your coding repos omitted it [33][41]).

### Python Implementer — reference instantiation

Shared core (identical across languages) = everything except the `<!-- LANGUAGE OVERLAY -->` block. Generate `implementer-rust.md` / `implementer-ts.md` by swapping the overlay — never by copy-editing the whole file (that's how `uv run pytest` leaked into your React repo [39]).

```markdown
---
name: implementer-python
description: Implements ONE approved feature spec in Python, test-first. Use only
  after Gate 1 approval. Not for spikes, spec edits, or multi-feature batches.
tools: Read, Write, Edit, Glob, Grep, Bash
model: sonnet
---

You implement exactly one approved feature spec. You are not the designer:
if the spec is ambiguous or wrong, STOP and return `SPEC-CONFLICT: <one line>`.

## Context you load (nothing else)
1. `features/<id>/spec.md` — the contract you implement
2. `docs/constitution.md` — project conventions (short; obey all of it)
3. `docs/architecture/modules/<your-module>.md` + the *interface* docs of
   modules the spec names as dependencies. Do NOT read neighbor internals.

## Protocol (TDD, strict order)
1. Read the spec's Test Plan. Write the tests it defines — failing (red).
   Run them; confirm they fail for the right reason.
2. Implement the minimum that satisfies the spec. Run tests until green.
3. Run the full verification suite; all must pass:
   `uv run pytest -q` · `uv run ruff check` · `uv run ruff format --check` · `uv run ty check`
4. Return inline (≤300 tokens): status, files touched, test counts,
   deviations (should be none), ending with `<!-- STATUS: COMPLETE -->`.

## Hard rules (all languages)
- Scope: write only inside your module's paths + its `tests/`. Never edit the
  spec, roadmap, or other modules.
- Test ONLY the public interface (below). Never test internals. No mocks for
  code you can execute for real (tmp dirs, real components, in-memory fakes at
  system edges only).
- No TODOs, no dead code, no commented-out code, no speculative generality.

<!-- LANGUAGE OVERLAY: python -->
## Public interface convention (authoritative, review-enforced)
- A module's public surface is EXACTLY the names in `__all__` of its package
  `__init__.py`. No `__all__` ⇒ no public surface ⇒ not directly testable.
- Anything underscore-prefixed is private and UNTESTABLE, even if importable.
- Tests import only `from <package> import <name>` where <name> ∈ `__all__`.
  `from <package>._x import ...` or `import <package>.internal_module` in a
  test file is a review-blocking violation (mechanically checked by
  `scripts/check_test_imports.py`).
## Style
- Python 3.13+, uv-managed. Types everywhere; make illegal states unrepresentable
  (dataclasses/enums/NewType) before writing defensive checks.
- Functional-first: pure functions + immutable data for internal logic.
  EXCEPTION — library code consumed by other people/teams: expose the idiomatic
  Python class-based API (this is what your users expect); keep the class a thin
  shell over functional internals.
- Application vs library: application ⇒ single entry point, config at the edge,
  exceptions may terminate; library ⇒ `__all__`-curated API, no I/O side effects
  at import, errors as typed exceptions documented in the interface doc.
- Comments: only for non-obvious *why*. Default is none.
<!-- /LANGUAGE OVERLAY -->
```

**Why `__all__` (and not underscore-only or "documented entry points"):** it's the only Python convention that is *declared in code* (a greppable, single-source-of-truth list) rather than inferred, which makes it enforceable by a ~20-line script run as a PostToolUse/pre-review hook — your "enforceable in review, not aspirational" bar. Underscore-prefix still applies as the belt-and-suspenders private marker; "documented entry points" fails the bar because docs drift.

---

## 4. Optimized Workflow

### Layer connection: project-based UP → feature-based pipeline

The UP layer runs **once per project** (or once per major epic/brownfield onboarding), produces the fixed artifacts every feature pass reuses, and never runs per-feature. Its HITL is **conversational, not gated** — you're in the room while it happens (AskUserQuestion-driven), so adding formal gates there would be ceremony; your two hard gates stay where they are, in the feature layer. The bridge between layers is the roadmap (§5): the use-case pass populates it; the feature pipeline consumes and updates it.

**Greenfield entry — `/project-init`** (≈ AIUP Inception+Elaboration [10], compressed to 4 artifacts):
1. Leader + Spec Agent (project mode) interview you → `docs/project.md` (vision, 1 page) and `docs/use-cases/UC-NNN-*.md` (Jacobson-lite [15]: actor, goal, main flow, key extensions, ~½ page each — no BPMN, no entity-model ceremony unless the domain is data-heavy, in which case one `entity-model.md` page is allowed).
2. Architecture conversation (you + Leader, Spec Agent drafting) → `architecture/overview.md` (Inception-Canvas shape: module map + dependency direction) + one interface 1-pager per module.
3. `docs/constitution.md` (AIUP's Harness layer [16] + spec-kit's constitution [22]): conventions, test strategy, public-interface rule, allowed libraries, budgets. ≤2 pages, hook-checked (§7).
4. Use cases → roadmap features (one UC ⇒ 1–3 features, sized to one pipeline pass each).

**Brownfield entry — `/project-onboard`** (≈ AIUP brownfield reverse-engineering [10][13] — this is where a parallel wave *does* transfer, because reading is independent even when writing isn't): Leader fans out 2–5 read-only Explore/research agents over the codebase → same four artifact types, but `constitution.md` records conventions *as found* (descriptive first, prescriptive after you bless it) and `roadmap.md` seeds from your backlog instead of fresh use cases. New use cases are written only for the parts you're about to change — do not reverse-engineer UC docs for stable code you'll never touch. Both entries converge on identical artifacts, so the feature pipeline is entry-agnostic — one harness, two doors, as you wanted.

### The feature pipeline

Sizing tiers first (the research harness's "scale effort" rule, transferred):

| Tier | Signals | Pipeline |
|---|---|---|
| **Trivial** (bugfix where spec already right, 1 module, no interface change) | no open questions | Leader → Implementer (spec = the failing acceptance criterion) → Reviewer → Gate 2 only |
| **Standard** | interface changes or multi-file | full pipeline, spike optional |
| **Uncertain** | open questions, ambiguous behavior, new tech | full pipeline, **spike mandatory** — per your Philosophy #5, the Leader may not skip it just because it's "busywork"; skipping requires you to say so at feature kickoff |

Full pipeline (statuses in → brackets are what the Leader writes to the roadmap):

```
 1. Select feature from roadmap (you + Leader)                      [→ in-progress]
 2. Research (optional; research-subagent)
 3. Spike (per tier)                                                [→ spiked]
    Spike Agent → spike/findings.md + spike/ref/ (throwaway, capped)
 4. Specification                                                   [→ spec'd]
    Spec Agent → spec.md {Problem, Non-goals, Assumptions (spike-informed),
    Reference Implementation (excerpt/pointer into spike/ref, ≤150 lines),
    Architecture Delta, Test Plan (EARS acceptance criteria [26])}
 5. Spec + Architecture Review (Reviewer, spec mode) → review-spec.md
 6. ── GATE 1: you approve spec ──
 7. Implementation (Implementer-<lang>, TDD: red → green → suite)
 8. Code Review (Reviewer, code mode; reruns suite) → review-code.md
 9. ── GATE 2: you approve merge ──
10. Merge; Leader updates roadmap, extracts ADRs if the spec's
    Architecture Delta changed module boundaries                    [→ merged]
```

**Architecture: standalone doc or spec section?** Split the question by grain — that resolves your hesitation. *Project-grain* architecture (module map, dependency direction) is always standalone (`architecture/overview.md` + module 1-pagers) and long-lived. *Feature-grain* architecture is a spec **section** (Architecture Delta) by default — your instinct that one section is "too thin" is right exactly when the delta **changes a module's public interface or adds/removes/re-couples a module**; that condition mechanically triggers promotion: the delta is applied to the standing module docs at merge and the decision captured as an ADR. So no feature ever produces a third floating "architecture doc" — it either fits in the spec or it amends the standing architecture. Reviewer (spec mode) checks the trigger.

### Gate rejection — explicit revision loops

Rejection is a **routing decision the Leader makes by classifying the objection**, never an automatic "go back to step 3":

| Gate | Objection class | Route | Cost |
|---|---|---|---|
| 1 | Wording/scope trim, criteria edits | **Revision pass**: Spec Agent patches spec.md; Reviewer re-reviews *the diff only*; back to Gate 1 | minutes |
| 1 | Wrong assumptions / "we don't actually know how X behaves" | **Back to spike** with the specific question; then spec revision (assumptions section rewritten — not just the test plan) | one spike |
| 1 | Wrong problem / conflicts with architecture | **Full re-spec** (rare); if module map itself is wrong → project-layer amendment first | full step 4 |
| 2 | Code defect, missing tests, convention violation | **Fix loop**: same Implementer, Reviewer's required-changes list as input; re-review; ≤2 cycles then escalate to you | cheap |
| 2 | Implementation revealed the spec was wrong | **Spec amendment**: Spec Agent patches spec + assumptions; **mini-Gate-1 on the amendment diff** (you approve the delta, not a re-read); then targeted re-implementation | medium |
| 2 | Feature shouldn't merge at all | Roadmap → `parked` with a note; branch preserved | — |

Two invariants: (a) per your instruction, rejection may rewrite *any* upstream artifact including spike-surfaced assumptions — the test plan is never the only thing revised; (b) code is never patched to diverge from the spec silently (AIUP's rule: the spec changes first, the code follows [10]) — that's what keeps specs from rotting into fiction, the spec-kit failure documented in issue #1092 [25].

---

## 5. Progress Tracking

**Artifact:** `roadmap.md` at repo root (not buried in `docs/`) — one table, one line per feature. It is the bridge artifact you predicted: the project layer writes its initial rows (from use cases), the feature layer advances them.

```markdown
# Roadmap
<!-- Owned by Project Leader. Humans edit priorities; agents edit status. -->
| ID  | Feature              | UC     | Module(s)   | Status      | Feature dir      |
|-----|----------------------|--------|-------------|-------------|------------------|
| 007 | Tag filtering        | UC-003 | notes, cli  | in-review   | features/007-... |
| 008 | Export to md         | UC-004 | export      | approved    | features/008-... |
| 009 | Sync backend         | UC-005 | sync        | proposed    | —                |
```

- **Status enum** (matches your requested lifecycle): `proposed → spiked → spec'd → approved → in-progress → in-review → merged`, plus `parked`.
- **Owner: the Project Leader, confirmed** — it's the only agent that sees every transition, and single-writer ownership (§2) means nobody else races it. You edit rows/priorities as a human whenever you like; the Leader treats your edits as authoritative.
- **Update points:** end of `/project-init`/`/project-onboard` (seed rows), and every pipeline transition in §4's bracketed statuses. A cheap `Stop` hook can validate the table (every `in-*` row has a feature dir; every feature dir has a roadmap row) so drift is caught mechanically, like your existing `detect_done.mjs` pattern [43].
- **Bridging the layers concretely:** the UC column is the traceability thread — AIUP's "every line traces to a requirement" [10] reduced to one table cell (its full audit machinery rejected as bloat). Roadmap rows point down to `features/<id>/` (feature layer) and back to `docs/use-cases/` (project layer). Per-feature `state.md` (pipeline phase, gate outcomes, rejection notes) is the *fine-grained* checkpoint enabling `/feature-resume` — the roadmap deliberately stays coarse so it never becomes a context sink: budget one line per feature, ~1 page per 40 features.

---

## 6. Scalable Folder Structure

```
project/
├── CLAUDE.md                     # Leader role + startup protocol. THIN (<500 words):
│                                 #   points at roadmap, constitution, pipeline. No lore.
├── roadmap.md                    # §5. THE project-level view.
├── .claude/
│   ├── settings.json             # hooks (verification, gate guards, budget checks)
│   │                             #   + Bash allowlist (pattern from harness-python [31])
│   ├── agents/                   # reusable across projects — sync from a template repo
│   │   ├── spec-agent.md
│   │   ├── spike-agent.md
│   │   ├── implementer-python.md #  core + <!-- LANGUAGE OVERLAY --> (§3)
│   │   ├── reviewer.md
│   │   └── research-subagent.md  #  unchanged from deep-research
│   ├── commands/
│   │   ├── project-init.md       # greenfield entry
│   │   ├── project-onboard.md    # brownfield entry
│   │   ├── feature.md            # /feature <id> — run the pipeline
│   │   ├── feature-resume.md     # resume from features/<id>/state.md
│   │   └── deep-research.md      # existing research harness, reused
│   └── scripts/
│       ├── check_test_imports.py # enforces __all__ interface rule (§3)
│       ├── check_doc_budget.sh   # word-count ceilings on constitution/specs (§7)
│       └── verify.sh             # single entry: test+lint+types (hook target)
├── docs/
│   ├── project.md                # vision, 1 page (project layer)
│   ├── constitution.md           # conventions/test strategy/budgets, ≤2 pages
│   ├── use-cases/UC-001-*.md     # ~½ page each, Jacobson-lite
│   ├── architecture/
│   │   ├── overview.md           # arc42-canvas module map, 1 page
│   │   └── modules/<name>.md     # 1-pager per module: PUBLIC INTERFACE + invariants
│   └── adr/NNNN-*.md             # MADR-lite, ≤½ page, append-only
├── features/<NNN-slug>/          # one dir per feature pass (the OpenSpec idea [21], kept)
│   ├── state.md                  # pipeline checkpoint (research-harness pattern)
│   ├── spike/findings.md, ref/   # throwaway; excluded from implementer scope
│   ├── spec.md                   # the Gate-1 artifact
│   └── review-spec.md, review-code.md
├── research/                     # deep-research runs (existing convention)
├── src/                          # one top-level package per module
└── tests/                        # mirrors modules; imports only public surfaces
```

Merged features' dirs move to `features/archive/` (OpenSpec's archive move [21] — cheap and keeps the active namespace small). Agent definitions live in a **template repo** and are synced, not copy-forked, into projects — the mechanical fix for the python→react fork rot [39][40].

---

## 7. Knowledge Management Strategy

Evaluated together (per Deliverable 7): arc42 answers *what a doc should say*, OKF answers *how docs should be packaged*; both are adopted only as shapes, neither as a standard. The lightest convention that still does the job:

1. **Packaging (from OKF, 3 fields only):** every doc under `docs/` and `features/` carries frontmatter `type` / `status` / `refs` (e.g. `type: spec, status: approved, refs: [UC-003, module:notes]`). That's the slice of OKF [8] that lets a script or agent filter/route docs without reading bodies; provenance/trust/lifecycle families, reserved `index.md`/`log.md` semantics, actor identities, and version negotiation are rejected — they solve multi-producer knowledge exchange, and (per the Bara critique [9]) don't buy semantic consistency anyway. `roadmap.md` already serves as the index; git already serves as the log.
2. **Architecture content (from arc42):** canvas-shaped `overview.md` [2]; module 1-pagers whose only mandatory sections are **Public interface** and **Invariants/decisions** — interface-first because that's the only part neighbor agents ever load; ADRs in MADR-lite (Context/Decision/Consequences, ≤½ page) filling arc42's section-9 role [1].
3. **Requirements content (from Kiro/EARS [26] + Jacobson [15]):** use cases ~½ page; acceptance criteria in EARS form ("WHEN … THE SYSTEM SHALL …") so the Test Plan maps 1:1 onto test names.
4. **Budgets, enforced not aspirational** (`check_doc_budget.sh` as a hook, warning at 80%): constitution ≤2 pages · spec ≤2 pages (the 2-minute-read bar) · module doc ≤1 page · ADR ≤½ page · use case ≤½ page · spike findings ≤1 page · worker inline return ≤300 tokens.
5. **Anti-OpenSpec loading rule:** context is **pulled per stage, never pushed globally** — each agent's prompt names the exact files it loads (§3 table); nothing outside `CLAUDE.md` (kept <500 words) is auto-injected. This is the direct countermeasure to OpenSpec's 50KB-prepend [19] and 10-skills [18] patterns.

**Budget flag (Philosophy #4 honesty):** worst-case Implementer load = constitution (~1.2k tokens) + spec (~1.5k) + own module doc + 2 neighbor interfaces (~1.5k) + the code it edits. Docs ≈ 4–5k, leaving ~10k for code — fine for well-modularized projects, but a feature spanning 3+ modules or a monolith brownfield **will** exceed 15k. The design response is to treat the budget as a *decomposition signal*, not a hard stop: the Leader splits the feature per module, or narrows the spike. The onboarding pass's Explore agents also legitimately exceed 15k; that's a bounded one-time cost, accepted rather than hidden.

---

## 8. Failure Modes (specific to this architecture)

1. **Spike gravity — the throwaway becomes the implementation.** Because the spec embeds reference-implementation excerpts, an Implementer can rationalize importing or transplanting `spike/` code wholesale, inheriting its shortcuts (no error handling, hardcoded paths) with Gate-1 legitimacy. *Mitigations:* spike dir outside Implementer read scope (only the spec's excerpt travels); hook rejects any `spike/` import or path reference in `src/`; spec template labels the excerpt "illustrative — reimplement under constitution rules"; Reviewer diffs suspicious similarity.
2. **Amendment-drift at Gate 2 — specs rot into fiction.** Under rejection pressure, patching code without the spec amendment (skipping the mini-Gate-1) is always locally faster; three features later the specs no longer describe the system — precisely the spec-kit #1092 [25] / OpenSpec spec-drift [44] failure, now with *approved* stale specs, which is worse. *Mitigations:* the fix-loop cap (≤2 cycles then human); Reviewer code-mode checks the diff against the *current* spec text, failing on undocumented behavior; merge hook requires `spec.md` mtime ≥ last `src/` change when review-code.md contains "deviation".
3. **Fixed-context creep — the constitution becomes OpenSpec.** Every gate lesson tempts a new constitution rule; every module adds a doc; the "small fixed context" grows monotonically until each agent task starts 8k tokens deep — the same slope OpenSpec slid down [18][19], arriving via governance instead of tooling. *Mitigations:* hard hook-enforced ceilings (§7) so adding a rule forces deleting one; recurring lessons graduate into *hooks/scripts* (mechanical enforcement is ~0 tokens at runtime) rather than prose; quarterly constitution prune as a roadmap chore.
4. **Roster fork rot** (observed, not hypothetical — the `uv run pytest` leftover in your React implementer [39]): per-language overlays drift after core-prompt fixes. *Mitigation:* template-repo sync (§6) plus a CI grep for cross-language tokens (`pytest` in a TS overlay, `vitest` in a Python one).

---

## Open Questions / Caveats

- **Subagent nesting** is version-unstable in mid-2026 (flip-flopped twice in July) [29]; this design assumes none — re-verify against live docs [27] before ever leaning on it.
- **OKF is 7 weeks old** [6]; re-check the spec (v0.3+?) [8] before borrowing more than the 3-field pattern.
- **Parallel features across disjoint modules** (two Implementers, two features, zero shared files) is deliberately left out of v1 — add only after the sequential pipeline is boring and reliable.
- Your reference repos' small defects (truncated `AGENTS.md` in both [36][37], empty verification fences in python [38], missing `model:` frontmatter [33][41]) are worth fixing in the template repo before it becomes the sync source.
- The spec-kit "2,000 lines from one plan phase" criticism comes via secondary sources [23][24] (the original Scott Logic review wasn't directly fetched); the direction is corroborated by primary issue #1092 [25] but treat the specific numbers as reported.

---

## References

[1] arc42 — Overview (template, 12 sections, section 9 = Architectural Decisions) — https://arc42.org/overview/
[2] arc42 Software Architecture Canvas (Communication / Inception / Tech Stack canvases) — https://canvas.arc42.org/
[3] arc42 FAQ E-1 — how much architecture documentation is enough — https://faq.arc42.org/questions/E-1/
[4] arc42 FAQ E-3 — arc42 and agile/lean approaches — https://faq.arc42.org/questions/E-3/
[5] Hompus — arc42 practical series (practitioner usage/tailoring) — https://blog.hompus.nl/2026/02/01/arc42-practical-series/
[6] Google Cloud Blog — How the Open Knowledge Format can improve data sharing (OKF announcement, June 12 2026) — https://cloud.google.com/blog/products/data-analytics/how-the-open-knowledge-format-can-improve-data-sharing
[7] MarkTechPost — Google Cloud Introduces Open Knowledge Format (OKF), June 16 2026 — https://www.marktechpost.com/2026/06/16/google-cloud-introduces-open-knowledge-format-okf-a-vendor-neutral-markdown-spec-for-giving-ai-agents-curated-context/
[8] Open Knowledge Format specification v0.2 (GoogleCloudPlatform/knowledge-catalog, okf/SPEC.md) — https://github.com/GoogleCloudPlatform/knowledge-catalog/blob/main/okf/SPEC.md
[9] Marc Bara — Google's New Format for Agent Context: A Standard or Just a Folder? — https://medium.com/@marc.bara.iniesta/googles-new-format-for-agent-context-a-standard-or-just-a-folder-82fb21d92041
[10] AI Unified Process — Methodology (phases, roles, artifacts, change model, principles) — https://unifiedprocess.ai/methodology.html
[11] AI Unified Process — site overview ("Specs at the center. AI handles the rest.") — https://unifiedprocess.ai/
[12] AI Unified Process — Tools (Claude Code plugin marketplace, `/requirements`, `/entity-model`, `/use-case-spec`, `/implement`) — https://unifiedprocess.ai/tools.html
[13] AI Unified Process — Enterprise (governance, brownfield reverse-engineering workflow) — https://unifiedprocess.ai/enterprise.html
[14] AI Unified Process — Articles index (Use-Case 2.0/3.0 lineage, "Bug or Enhancement?") — https://unifiedprocess.ai/articles.html
[15] Ivar Jacobson International — Use-Case 3.0: The Definitive Guide, Refreshed — https://www.ivarjacobson.com/files/use-case_3.0_v1.0.pdf
[16] Martinelli — What / Harness / How: The Three Layers of AIUP — https://martinelli.ch/what-harness-how-the-three-layers-of-aiup/
[17] AI Unified Process — Tutorial ("this is your last chance to course-correct cheaply") — https://unifiedprocess.ai/tutorial.html
[18] Fission-AI/OpenSpec issue #611 — "why bloat context with 10 skills?" — https://github.com/Fission-AI/OpenSpec/issues/611
[19] OpenSpec — docs/opsx.md (context injection up to 50KB; `apply` "autonomous within scope"; `ff`) — https://github.com/Fission-AI/OpenSpec/blob/main/docs/opsx.md
[20] Ovidiu Eftimie — OpenSpec vs Spec-Kit — https://ovidiueftimie.substack.com/p/openspec-vs-spec-kit
[21] Fission-AI/OpenSpec — repository (specs/ changes/ archive/ delta-spec model) — https://github.com/Fission-AI/OpenSpec
[22] github/spec-kit — repository (`/constitution` → `/specify` → `/plan` → `/tasks` → `/implement`) — https://github.com/github/spec-kit
[23] LogRocket — GitHub Spec Kit review — https://blog.logrocket.com/github-spec-kit/
[24] den.dev — GitHub Spec Kit — https://den.dev/blog/github-spec-kit/
[25] github/spec-kit issue #1092 — "High Level Design Concerns" (no bug→spec loop; rewrite cost) — https://github.com/github/spec-kit/issues/1092
[26] AWS Kiro docs — Feature specs (EARS acceptance criteria; requirements-first vs design-first) — https://kiro.dev/docs/specs/feature-specs/
[27] Claude Code docs — Subagents (frontmatter `name`/`description`/`tools`/`model`; own context window) — https://code.claude.com/docs/en/sub-agents
[28] Claude Code docs — Hooks — https://code.claude.com/docs/en/hooks
[29] Digital Applied — Claude Code subagent depth limits & budget caps (2026 nesting timeline) — https://www.digitalapplied.com/blog/claude-code-subagent-depth-limits-budget-caps-2026
[30] Anthropic Engineering — How we built our multi-agent research system — https://www.anthropic.com/engineering/multi-agent-research-system
[31] harness-python — hook + Bash-allowlist configuration — /home/dcp/Projects/harness-python/.claude/settings.json
[32] harness-python — reviewer agent prompt (no approval on red) — /home/dcp/Projects/harness-python/.claude/agents/reviewer.md
[33] harness-python — architect agent prompt (~1,900 words; PoC/MVP/Production doc proportionality; no `model:`) — /home/dcp/Projects/harness-python/.claude/agents/architect.md
[34] harness-react — designer agent prompt (DESIGN.md → design_check.mjs → tokens.css pipeline) — /home/dcp/Projects/harness-react/.claude/agents/designer.md
[35] deep-research — operating manual (state.md, `<!-- STATUS: COMPLETE -->`, persist-by-lead, model/cost discipline) — /home/dcp/Projects/deep-research/CLAUDE.md
[36] harness-python — truncated AGENTS.md — /home/dcp/Projects/harness-python/AGENTS.md
[37] harness-react — truncated AGENTS.md (identical break) — /home/dcp/Projects/harness-react/AGENTS.md
[38] harness-python — docs/verification.md (two empty code fences) — /home/dcp/Projects/harness-python/docs/verification.md
[39] harness-react — implementer agent prompt (`uv run pytest` leftover) — /home/dcp/Projects/harness-react/.claude/agents/implementer.md
[40] harness-react — reviewer agent prompt (`src/cli.py` example leftover) — /home/dcp/Projects/harness-react/.claude/agents/reviewer.md
[41] harness-react — architect agent prompt (~1,950 words; no `model:`) — /home/dcp/Projects/harness-react/.claude/agents/architect.md
[42] harness-python — leader agent prompt ("Never accept a subagent result that arrives as raw text in chat") — /home/dcp/Projects/harness-python/.claude/agents/leader.md
[43] harness-react — detect_done.mjs mechanical check — /home/dcp/Projects/harness-react/scripts/detect_done.mjs
[44] Hacker News — OpenSpec discussion (spec drift: "duplication and contradictions across specs") — https://news.ycombinator.com/item?id=47994433
