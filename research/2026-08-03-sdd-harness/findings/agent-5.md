# The deep-research harness at `/home/dcp/Projects/deep-research`

## File inventory
- `/home/dcp/Projects/deep-research/CLAUDE.md` — the operating manual (163 lines), source of truth for the whole protocol.
- `/home/dcp/Projects/deep-research/README.md` — user-facing description/quick-start, restates the same protocol for humans.
- `/home/dcp/Projects/deep-research/.claude/agents/research-subagent.md` — worker agent definition (frontmatter: `tools: WebSearch, WebFetch`, `model: sonnet`).
- `/home/dcp/Projects/deep-research/.claude/agents/citation-agent.md` — final-pass agent definition (frontmatter: `tools: Read, Edit, Write, Glob, Grep`, `model: opus`).
- `/home/dcp/Projects/deep-research/.claude/commands/deep-research.md` — `/deep-research <question>` slash command, a thin restatement of the CLAUDE.md steps as an execution checklist.
- `/home/dcp/Projects/deep-research/.claude/commands/research-resume.md` — `/research-resume [run-dir]` slash command for checkpoint recovery.
- `/home/dcp/Projects/deep-research/.claude/settings.local.json` — permission allowlist (`WebSearch` plus a fixed list of `WebFetch(domain:...)` entries: anthropic.com, aws.amazon.com, github.com, docs.claude.com, langchain sites, etc.). No scripts directory exists in this repo — only agents + commands + settings.

## Topology

```
you ──/deep-research──▶ Lead Orchestrator      (main session, Opus, never nests)
              │  plan → decompose → spawn
              └────────────┬───────────────
      ┌───────────────────┼───────────────────┐
      ▼                   ▼                   ▼
research-subagent   research-subagent   research-subagent   (parallel, Sonnet, 3–5/wave)
one subtopic each — search → think → return findings INLINE (no file writes)
      └───────────────────┼───────────────────┘
                           ▼
         Lead persists returns to findings/agent-<n>.md,
         evaluates gaps, optionally spawns a 2nd wave,
         then writes report.md itself
                           ▼
                  citation-agent            (single pass, Opus, last)
         attributes every claim, appends References
                           ▼
              research/<date>-<slug>/report.md
```

Key topological facts:
- **Single level of fan-out.** "The orchestrator is the top-level agent — you do not nest (a subagent never spawns its own subagents). All fan-out happens from the main session." (`CLAUDE.md` lines 16–17)
- Workers are **spawned in a single message per wave** so they run concurrently: "launch the wave — **all Agent calls in a single message** so they run concurrently. Spawn 3–5 at a time." (`CLAUDE.md` §4)
- Waves can repeat: a second wave is spawned only for genuine gaps found when evaluating wave-1 results (`CLAUDE.md` §5), never as a matter of course.
- Exactly one `citation-agent` pass, always last, never parallelized with anything.

## The delegation contract (verbatim)

Each worker prompt **MUST contain all five** of these (`CLAUDE.md` §3):
> 1. **Objective** — the specific question this worker answers, stated precisely... 2. **Output format**... 3. **Source guidance**... 4. **Boundaries** — what this worker should NOT touch (the slices owned by its siblings), so coverage tiles cleanly. 5. **Budget + identity** — a tool-call budget and the worker's agent number `<n>`.

Workers' return obligation (`research-subagent.md` lines 51–56):
> **You do not write files.** Return your findings as your final message; the lead persists them verbatim to `research/<run-dir>/findings/agent-<n>.md`. (This is deliberate: subagent file-writes are unreliable across environments, so the lead owns persistence.)

Findings must end with `<!-- STATUS: COMPLETE -->`, and this marker is the sole done/pending signal:
> "A subtopic counts as `done` only when its `findings/agent-<n>.md` exists **and ends with the marker** `<!-- STATUS: COMPLETE -->`. A findings file without that marker was interrupted mid-write — treat it as `pending` and re-spawn that worker." (`CLAUDE.md` §"Resuming...")

Checkpoint/resume mechanics:
- `state.md` is written **before** the first worker is spawned, containing: the question, run dir, current `phase` (`planning → workers → synthesis → citation → done`), and a subtopic table with `status` (`pending`/`done`) (`CLAUDE.md` §"Resuming an interrupted run").
- `state.md` is updated after each wave and on every phase change.
- On resume, the file system is authoritative over the checkpoint file: "Trust the files over a stale `state.md`" (`CLAUDE.md` line 124, restated in `research-resume.md` step 2).
- `/research-resume` re-spawns `research-subagent` **only** for subtopics still `pending`, then proceeds through synthesis and the mandatory citation pass — never redoing completed work.

## Budget / cost discipline

Worker fan-out and budget scale to query complexity (`CLAUDE.md` §2, repeated in README):

| Query type | Workers | Tool calls each |
|---|---|---|
| Simple fact / single lookup | 0 (answer directly) or 1 | 3–10 |
| Comparison / 2–4 distinct angles | 2–4 | 10–15 |
| Broad, open-ended, multi-part | 5–10+ | 10–20 |

Explicit anti-over-spawn rule: "If you can answer correctly without delegating, just do it. Spawning agents for a trivial lookup wastes time and tokens." (`CLAUDE.md` line 38)

Cost model stated plainly: "Workers run on Sonnet and dominate the bill because they run in parallel. Keep worker count and tool budgets proportional to the question. The Opus lead and citation passes are cheap by comparison — spend reasoning there." (`CLAUDE.md` §"Cost discipline")

## Why this works for research (structural properties enabling parallel fan-out)

1. **Independent subtasks with no inter-task dependencies.** Subtopics are deliberately decomposed to be "non-overlapping" with "clear boundaries" (`CLAUDE.md` §3) — one worker's output never feeds another worker's input. Research questions decompose into genuinely separable facts/angles (e.g., "cause A of chip shortage" vs "cause B"), so nothing blocks on anything else mid-wave.
2. **Mechanical, additive synthesis.** The lead's synthesis step (§6) is essentially reading N independent findings files and merging them into sections — no findings file needs to "compile" or "run" against another; conflicting claims are just reported as disagreements rather than resolved by rework.
3. **Inline returns + lead-owned persistence** removes a whole failure class (unreliable subagent file writes) without needing any coordination between workers — each worker's return is self-contained text, verified only by a trailing marker string, so the lead can persist it with a dumb, content-agnostic write.
4. **Homogeneous worker capability.** Every `research-subagent` has identical tools (`WebSearch`, `WebFetch`) and an identical procedure (OODA loop). There's no need to route different subtopics to differently-skilled workers, so the lead's decomposition problem is just "split the question," not "match subtask to capability."
5. **Idempotent, side-effect-free investigation.** A worker searching the web and reading pages doesn't mutate shared external state, so re-spawning a `pending` worker on resume is always safe — there's no risk of double-applying a change (unlike code edits).
6. **A single, cheap correctness gate at the end.** Because claims are just prose with URLs, one citation-agent pass at the very end can validate/attribute the entire report — there's no need for per-worker verification loops.

## Transferable vs. non-transferable to a sequential coding pipeline

| Mechanism | Transferable? | Reason |
|---|---|---|
| `state.md` checkpoint file (phase + per-subtask status table) | **Transferable** | Generic pattern: persist phase + status table before starting work, update after each unit completes. Works for any long-running, interruptible multi-step process regardless of whether the units run in parallel or sequence. |
| Completion marker (`<!-- STATUS: COMPLETE -->`) as the ground-truth done signal, trusted over the checkpoint file | **Transferable** | A cheap, mechanically-checkable "did this really finish" signal is useful anywhere subagent output could be truncated/interrupted — e.g., a coding step could end its diff/patch output with an analogous marker, and the lead trusts the artifact (patch applied, tests file exists) over a stale state file. |
| Persist-by-lead contract (workers return inline; only the lead writes to disk) | **Transferable, with a caveat** | The reliability argument (subagent file-writes are inconsistent) is environment-specific, but the general principle — the orchestrator is the single writer of shared state — reduces write-write races and partial-write corruption in any multi-agent setup, coding included. Caveat: in coding, the "artifact" workers produce often *is* a file diff/patch, so "inline return, lead persists" may need to mean "worker proposes a diff, lead applies it" rather than "worker never touches the filesystem" — since code changes are naturally file-shaped in a way research prose is not. |
| Scoped worker prompts with the five required fields (objective, output format, source guidance, boundaries, budget+identity) | **Transferable in structure, not in content** | The discipline of "explicit objective + explicit non-overlapping boundary + explicit budget" combats the same failure mode (vague delegation) in coding. But "source guidance" (primary vs. SEO-bait sources) has no coding analog; it'd be replaced by something like "which files/modules are authoritative, which tests define correctness." |
| Flat parallel wave (3–5 agents spawned in one message, all independent) | **Non-transferable as-is** | This depends on subtasks having no inter-task dependencies (property #1 above). Coding steps are typically sequential/dependent — e.g., "write failing test" must precede "implement," "implement" must precede "refactor," a schema change must land before code that depends on it compiles. Fanning out coding subtasks in one parallel wave risks merge conflicts, inconsistent shared state, and broken intermediate builds in a way that independent research findings never risk. |
| Mechanical/additive synthesis (lead merges N findings files into report.md) | **Non-transferable as-is** | Merging code changes is not just concatenation/summarization — it requires resolving edits against a single evolving codebase (compile, test, merge conflicts). A coding "lead" can't just read N diffs and staple them together the way it staples together N findings sections; it needs a build/test gate, which research synthesis never needed. |
| Single last-mile citation-agent as sole quality gate | **Partially transferable** | The idea of "one final agent pass that checks the whole artifact against ground truth" maps to a coding pipeline's final test/lint/review pass. But it can't be the *only* gate the way it is here — in coding, correctness should ideally be checked incrementally (tests run after each step), not solely at the very end, because later steps in a dependent pipeline may build on a broken earlier step. |

## Open questions (outside boundary, noted for lead)
- No `.claude/scripts/` or `.claude/hooks/` exist in this repo — orchestration is entirely prose (agent/command markdown files) plus permission settings; there is no programmatic state-machine or script enforcing the phase transitions. Worth confirming whether the sibling coding harnesses (`harness-python`, `harness-react`) rely on scripts/hooks for enforcement instead of prose-only convention — that would be a structural difference the lead may want to weigh when comparing topologies.
- `settings.local.json`'s domain allowlist is research-specific plumbing (web-fetch permissions) and not relevant to a coding-pipeline comparison; flagging only so the lead doesn't mistake it for a topology mechanism.

<!-- STATUS: COMPLETE -->
