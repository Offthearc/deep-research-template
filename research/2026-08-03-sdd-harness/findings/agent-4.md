# Agent 4 — Coding Methodology Ground Truth from `harness-python` and `harness-react`

## Repo: harness-python

**File inventory** (approx sizes by line count):
- `CLAUDE.md` — 60 lines (~430 words) — role directive, loaded automatically
- `AGENTS.md` — 46 lines but **truncated**: ends mid-document at "## 4. How to choose a task" with no body (~330 words of actual content)
- `.claude/settings.json` — 51 lines — hooks + Bash permission allowlist
- `.claude/agents/leader.md` — 86 lines (~600 words)
- `.claude/agents/architect.md` — 283 lines (~1,900 words) — by far the largest
- `.claude/agents/implementer.md` — 81 lines (~650 words)
- `.claude/agents/reviewer.md` — 81 lines (~550 words)
- `docs/architecture.md` — 44 lines (project-specific: notes-cli, 3-layer design)
- `docs/conventions.md` — 58 lines
- `docs/verification.md` — 46 lines, **contains two empty code fences** (Level 2 "Integration test," Level 3 "Manual smoke test" — both are unfilled placeholder blocks)
- `CHECKPOINTS.md` — 47 lines, checkpoints C1–C5
- `feature_list.json` / `feature_list.example.json` — task list, Spanish descriptions
- `scripts/commit_on_done.sh`, `init.sh` (Spanish comments/output), `progress/current.md`

### Agent roster

| Agent | Role | Model | Tools | Prompt size (approx) |
|---|---|---|---|---|
| `leader` | Orchestrator; decompose/coordinate only | not specified | `Read, Glob, Grep, Bash, Agent` | ~600 words |
| `architect` | Requirements → `docs/architecture.md` + `docs/verification.md`, PoC/MVP/Production classification | not specified | `Read, Write, Edit, Glob, Grep, Bash, AskUserQuestion` | ~1,900 words |
| `implementer` | Codes exactly one feature + tests + self-verify | not specified | `Read, Write, Edit, Glob, Grep, Bash` | ~650 words |
| `reviewer` | Approve/reject only, never edits | not specified | `Read, Glob, Grep, Bash` | ~550 words |

Note: none of the four `.claude/agents/*.md` frontmatter blocks specify a `model:` field — model choice is left to whatever Claude Code defaults to, contradicting the general "Sonnet workers / Opus lead" cost-discipline pattern.

### Workflow pipeline
`architect → implementer → reviewer`, orchestrated by a mandatory `leader` role baked into `CLAUDE.md` itself ("In this repository you always act as the `leader` subagent"). Startup protocol: read `AGENTS.md` → read `feature_list.json` + `progress/current.md` → run `./init.sh` (stop if it fails) → check `docs/architecture.md` for placeholder → call architect if needed.

Escalation table from `leader.md`:
```
| Task complexity | Subagents |
| Trivial (1 file) | 1 implementer → 1 reviewer |
| Medium (2–3 files) | 1 implementer → 1 reviewer |
| Complex (refactor) | 2–3 Explore → 1 implementer → 1 reviewer |
| Very complex | Split into sub-tasks and apply this table again |
```

**TDD/verification discipline**: implementer must "write tests that validate every acceptance criterion" using `tempfile.TemporaryDirectory()` — no mocks — then run `uv run pytest`, `ruff check`, `ruff format --check`, `ty check`, `./init.sh` before marking done. Reviewer independently reruns the same suite and is barred from approving on any red signal.

**Hard rules verbatim** (`CLAUDE.md`):
> "❌ **Do not edit** files in `src/` or `tests/` directly (not with Edit, not with Write, not with Bash)."
> "❌ **Do not mark** features as `done` in `feature_list.json`."

**Anti-broken-telephone rule** (`leader.md`):
> "When launching any subagent, explicitly instruct it to **write results to a file** and return only the reference... Never accept a subagent result that arrives as raw text in chat without a file reference."

**Architect constraint on scope proportionality** (`architect.md`):
> "Keep docs proportional to the solution type (PoC ≈ 50 lines, MVP ≈ 100 lines, Production ≈ 200 lines)."

Enforcement is also mechanical, not just prompt-based: `.claude/settings.json` runs `uv run pytest tests -q` and `commit_on_done.sh` as a `PostToolUse` hook after every Edit/Write, and forces `./init.sh` on `Stop`. The permission allowlist restricts Bash to a fixed set of `uv`/`ruff`/`ty`/git commands — the harness, not just the prompt, prevents skipping verification.

### Conventions (`docs/conventions.md`)
Python 3.13+, PEP 8, 80-col max, `ruff` for lint/format, `ty` for types, f-strings only, `snake_case`/`PascalCase`/`UPPER_SNAKE` naming table, one test file per module (`tests/test_<module>.py`), one `unittest.TestCase` class per unit, domain exceptions in `src/exceptions.py`, and:
> "By default, **none** [comments] are written. They are only allowed when explaining a non-obvious *why*."

### Working vs. broken/bloated
- **Working**: the hook-enforced pipeline (settings.json) is a genuinely strong mechanism — it doesn't rely on the agent's goodwill to run tests/commit; the shell hooks do it regardless.
- **Broken**: `AGENTS.md` is truncated — "## 4. How to choose a task" has a header and nothing under it in either repo. Referenced as the "entry point... navigation map" yet missing content.
- **Broken/incomplete**: `docs/verification.md` Level 2 and Level 3 sections are empty code fences — a template never filled in for this project, despite the file's own golden rule being "the agent doesn't say 'it works', it proves it."
- **Bloated**: `architect.md` at ~1,900 words is nearly 3x the size of any other agent prompt, carrying a large embedded diagram-type table, a full document template (with example code), a whole self-review checklist, and phase-gated interview logic. Comprehensive but heavy for a per-session-loaded subagent prompt.
- **Inconsistency**: internal tooling artifacts (`init.sh` output, `feature_list.example.json` descriptions) are in Spanish while all prompt/doc files are in English — suggests an original Spanish-authored template partially translated.

---

## Repo: harness-react

**File inventory** (approx sizes):
- `CLAUDE.md` — 65 lines (~480 words)
- `AGENTS.md` — 46 lines, **same truncation bug** as python repo (dies at "## 4. How to choose a task")
- `.claude/settings.json` — 58 lines — adds a `design_check.mjs` PostToolUse hook alongside vitest + commit hook
- `.claude/agents/leader.md` — 93 lines (~650 words)
- `.claude/agents/architect.md` — 289 lines (~1,950 words)
- `.claude/agents/designer.md` — 125 lines (~850 words) — **react-only, no python equivalent**
- `.claude/agents/implementer.md` — 96 lines (~750 words)
- `.claude/agents/reviewer.md` — 101 lines (~680 words)
- `DESIGN.md` — 122 lines — YAML front-matter design-token schema + rationale body, following the `design.md` format (`github.com/google-labs-code/design.md`)
- `docs/architecture.md` — 57 lines (React layered: components/hooks/api)
- `docs/conventions.md` — 59 lines
- `docs/verification.md` — 60 lines — **fully filled in**, 4 levels (unit/component, static checks, design conformance, manual smoke), no empty placeholders
- `CHECKPOINTS.md` — 65 lines — C1–C6 (adds C6 for DESIGN.md conformance)
- `scripts/`: `commit_on_done.sh`, `detect_done.mjs`, `design_check.mjs`, `validate_features.mjs`
- `feature_list.json`, `init.sh`, `progress/current.md`, `src/`, `tests/`

### Agent roster

| Agent | Role | Model | Tools | Prompt size (approx) |
|---|---|---|---|---|
| `leader` | Orchestrator | not specified | `Read, Glob, Grep, Bash, Agent` | ~650 words |
| `architect` | Architecture docs (UI behavior, not visuals) | not specified | `Read, Write, Edit, Glob, Grep, Bash, AskUserQuestion` | ~1,950 words |
| `designer` | Owns `DESIGN.md` + generated theme tokens | not specified | `Read, Write, Edit, Glob, Grep, Bash, AskUserQuestion` | ~850 words |
| `implementer` | One feature, code + Vitest/RTL tests | not specified | `Read, Write, Edit, Glob, Grep, Bash` | ~750 words |
| `reviewer` | Approve/reject + design conformance | not specified | `Read, Glob, Grep, Bash` | ~680 words |

### Workflow pipeline
`architect → designer → implementer → reviewer`. The `designer` is inserted between architect and implementer specifically for UI: "After the architect, before the first implementer, when `DESIGN.md` is missing/placeholder or a new UI surface is added. Owns `DESIGN.md` + the theme tokens."

**Designer protocol** (`designer.md`): reads `feature_list.json`, `docs/architecture.md`, existing `DESIGN.md`; decides whether to preserve an existing project-specific theme or infer one from the description; writes `DESIGN.md` per the `design.md` YAML-front-matter + markdown-body format (colors, typography, rounded, spacing, components); then:
```bash
node scripts/design_check.mjs --write   # generate src/theme/tokens.css
node scripts/design_check.mjs           # must pass (tokens in sync)
```
> "❌ Do not write application components or logic in `src/`... ❌ Do not hand-edit `src/theme/tokens.css`."

**TDD/verification discipline**: implementer runs Vitest + React Testing Library ("Query by role/label/text the way a user would... render real components rather than mocking them"), then `npx vitest run`, `npm run lint`, `npx prettier --check .`, `npm run typecheck`, `npm run design:check`, `./init.sh`. Reviewer reruns the same suite plus a manual read of `DESIGN.md`'s Components section against the actual component code:
> "The script catches rogue values; **you** catch a component that is 'technically tokenized' but does not match its `DESIGN.md` spec."

**Hard rules verbatim** (`CLAUDE.md`):
> "❌ **Do not edit** files in `src/` or `tests/` directly... `DESIGN.md` and `src/theme/tokens.css` are owned by the `designer` — do not hand-edit them either."

### Conventions (`docs/conventions.md`)
TypeScript strict, React 19 function components only, Prettier (no semicolons, single quotes, 80-col), ESLint incl. `react-hooks` rules, PascalCase components/`useCamelCase` hooks naming table, one component per file with co-located CSS, and a design-token section:
> "**Never hardcode colors, spacing, radius, or font families.** Use the CSS custom properties generated from `DESIGN.md`... If a value you need is missing, it is a design change: ask the `designer` to add the token to `DESIGN.md`."

### Working vs. broken/bloated
- **Working**: `docs/verification.md` here is fully fleshed out (4 concrete levels, no empty fences) — noticeably better-maintained than the python counterpart.
- **Working**: the design-token pipeline (`DESIGN.md` → `design_check.mjs --write` → generated `tokens.css`) is a coherent mechanism for keeping "no hardcoded hex" enforceable via both a script gate and human review.
- **Bug/copy-paste leftover**: `implementer.md` step 7 still instructs writing "Output of the final `uv run pytest tests -v` run" into `progress/impl_<feature-name>.md` — a **Python-specific artifact left in the React file** (should say `npx vitest run`). Confirms the React agents were forked from the Python ones and not fully scrubbed.
- **Bug/copy-paste leftover**: `reviewer.md`'s "Required changes" example still reads "Remove `import requests` from `src/cli.py`" — a Python-specific example surviving in the React template verbatim.
- **Same `AGENTS.md` truncation bug** as python (content missing after "## 4. How to choose a task").
- **Bloated**: `architect.md` again the largest prompt at ~1,950 words, nearly identical structure/length to python's version — most content copy-pasted with only example diagrams/tables swapped for React-flavored ones.

---

## Cross-repo comparison

### Genuinely shared / could be a single general-purpose agent definition
- **`leader.md`** — 90%+ identical between repos; the only React-specific delta is the `designer` step insertion. Prime material for a single parameterized "leader" agent with a language-specific pipeline array injected.
- **`architect.md`** — structurally identical (same phases, same document template skeleton, same checklist, same diagram-type table verbatim). Only the Block-A tech-constraint question wording, the example diagrams, and one added sentence about the designer's ownership differ. The single most duplicated, most bloated file — best candidate for consolidation into a shared "architect" core + small language-specific appendix.
- **`reviewer.md`** — shares its whole skeleton (protocol steps 1–4, hard rules, verdict format) with only the verification-command list and (react) the design-conformance section as deltas.
- **`implementer.md`** — shares its whole skeleton with only the verification-command list and test-framework specifics as deltas — exactly where the Python leftovers leaked into the React copy; evidence the fork was literal copy/edit rather than templating.
- **`CLAUDE.md` / `AGENTS.md`** — near-verbatim; `AGENTS.md`'s truncation bug identical in both, confirming a shared broken template rather than two independent authoring mistakes.
- **`CHECKPOINTS.md` / `docs/verification.md` / `docs/conventions.md`** — same section skeleton with content-only deltas.
- **Hook/settings pattern** — same structure (`PostToolUse` test-run + commit hook, `Stop` → `init.sh` force-check, Bash allowlist) — the *mechanism* is shared even though concrete commands differ per toolchain.

### Genuinely language/stack-specific (should stay separate)
- **`designer.md` + `DESIGN.md` + `design_check.mjs`** — the entire design-token subsystem has no Python equivalent; UI-specific and correctly isolated to the React repo.
- Toolchain commands: `uv`/`ruff`/`ty`/`pytest` vs `npm`/`eslint`/`prettier`/`tsc`/`vitest`.
- Naming conventions: `snake_case` vs `camelCase`/`PascalCase`, module layout (`storage.py`/`notes.py`/`cli.py`) vs (`components/`/`hooks/`/`api/`).
- `init.sh` implementation details — same *shape*, different *content*, appropriately not shared verbatim.
- Test doctrine specifics: `tempfile.TemporaryDirectory()` no-mocks rule vs RTL "query by role, render real components" rule — same underlying philosophy ("don't mock what you can execute for real") expressed in stack-appropriate terms.

### Key overall pain point
The React repo was evidently produced by copying the Python repo's agent files and adapting them, but the adaptation was incomplete in at least two load-bearing spots (implementer's progress-file instructions, reviewer's example violation) and one navigation file (`AGENTS.md`) is broken identically in both — a shared defect from the common template. This strongly supports factoring `leader.md`, `architect.md`, `reviewer.md`, and `implementer.md` into a shared "core" prompt plus small per-stack override blocks, which would both shrink the ~1,900-word architect prompt and eliminate the risk of copy-paste leftovers recurring in a third (e.g., Go, Rust) variant.

## Source quality notes
All findings come directly from primary source files. Not opened (budget-limited): `scripts/commit_on_done.sh`, `scripts/design_check.mjs`, `scripts/validate_features.mjs`, `scripts/detect_done.mjs`, `progress/current.md`, `progress/history.md`, `README.md` in either repo — a full audit of the automation scripts is a gap if finer mechanical detail is later required.

## Open questions
- Neither repo's agent frontmatter specifies a `model:` field — worth flagging for model-cost allocation per subagent role.
- Did not verify whether a third/fourth sibling variant (e.g., Go) exists — outside the two-repo boundary.
- Did not read README.md in either repo.

## Ranked sources
1. `/home/dcp/Projects/harness-python/CLAUDE.md`, `/home/dcp/Projects/harness-react/CLAUDE.md` — authoritative, auto-loaded session instructions.
2. `/home/dcp/Projects/harness-{python,react}/.claude/agents/{leader,architect,implementer,reviewer,designer}.md` — the actual agent prompt definitions, primary and complete.
3. `/home/dcp/Projects/harness-{python,react}/.claude/settings.json` — ground truth for mechanically-enforced (not just prompted) behavior.
4. `/home/dcp/Projects/harness-{python,react}/docs/{architecture,conventions,verification}.md` and `CHECKPOINTS.md` — primary project-level convention docs.
5. `/home/dcp/Projects/harness-{python,react}/AGENTS.md`, `feature_list.json`/`feature_list.example.json`, `DESIGN.md`, `init.sh` — supporting artifacts confirming workflow claims.

<!-- STATUS: COMPLETE -->
