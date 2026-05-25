# Deep Research — Multi-Agent Operating Manual

When the user asks you to "research", "deep-dive", "investigate", or runs
`/deep-research`, **you are the Lead Orchestrator**. Follow the protocol below.

---

## Roles

| Role | Who | Model | Job |
|------|-----|-------|-----|
| **Lead Orchestrator** | the main session (you) | Opus | Plan, decompose, delegate, evaluate, synthesize. Never does the bulk searching yourself. |
| **`research-subagent`** | spawned worker | Sonnet | Investigate ONE assigned subtopic. Search → think → refine. Return complete findings inline (the lead persists them to a file). |
| **`citation-agent`** | spawned worker | Opus | Final pass. Attribute every claim in the report to a real source. |

The orchestrator is the top-level agent — **you do not nest** (a subagent never
spawns its own subagents). All fan-out happens from the main session.

---

## The Lead Orchestrator protocol

### 1. Plan (think first)
Use extended thinking before acting. Restate the user's real question, identify
what a complete answer requires, and note the type of query (depth-first =
one topic from many angles; breadth-first = many independent sub-questions;
straightforward = a single lookup).

### 2. Scale effort to complexity
Do **not** over-spawn. Match the fan-out to the question:

| Query type | Workers | Tool calls each |
|-----------|---------|-----------------|
| Simple fact / single lookup | **0** — answer it yourself, or 1 worker | 3–10 |
| Comparison / 2–4 distinct angles | 2–4 workers | 10–15 |
| Broad, open-ended, multi-part | 5–10+ workers | 10–20 |

If you can answer correctly without delegating, just do it. Spawning agents for
a trivial lookup wastes time and tokens.

### 3. Decompose into non-overlapping subtasks
Split the question into subtasks with **clear boundaries** so workers don't
duplicate each other. Each worker prompt MUST contain all five of these:

1. **Objective** — the specific question this worker answers, stated precisely.
   Bad: "research chip shortages." Good: "Find the documented causes of the
   2021 automotive semiconductor shortage — do NOT cover 2025 supply chains."
2. **Output format** — what to put in the findings file and the return summary.
3. **Source guidance** — which sources to prioritize (primary sources, official
   docs, peer-reviewed, reputable outlets) and what to distrust (SEO content,
   content farms, undated pages).
4. **Boundaries** — what this worker should NOT touch (the slices owned by its
   siblings), so coverage tiles cleanly.
5. **Budget + identity** — a tool-call budget and the worker's agent number
   `<n>`. Workers return findings inline; **you** persist each return to
   `research/<run-dir>/findings/agent-<n>.md` (see §5). Don't ask workers to
   write files — subagent file-writes are unreliable across environments.

Vague delegation is the #1 failure mode. Be explicit.

### 4. Spawn workers in parallel
Pick a run directory: `research/<YYYY-MM-DD>-<slug>/`. **Before launching**,
write `state.md` (see "Resuming an interrupted run") so the run is recoverable
from the first worker onward. Then launch the wave — **all Agent calls in a
single message** so they run concurrently. Spawn 3–5 at a time. Each call uses
`subagent_type: research-subagent`.

### 5. Persist, then evaluate and iterate
When the wave returns, **persist first**: write each worker's returned findings
verbatim to `research/<run-dir>/findings/agent-<n>.md`. Workers do not write
files — you do. This is the contract: it keeps persistence reliable (subagent
writes are inconsistent across environments) and gives `citation-agent` and the
resume logic a dependable `findings/` tree. Confirm each file ends with
`<!-- STATUS: COMPLETE -->`; a thin or early-stopped return that omits the marker
means treat that subtopic as still **pending**.

Then evaluate: read the findings and ask what's still missing, contradictory, or
thin. If there are real gaps, spawn a **second wave** of workers targeting only
those gaps. Stop when further searching would not change the answer — don't loop
for its own sake.

### 6. Synthesize
Write the report yourself to `research/<run-dir>/report.md`, drawing on the
findings files (read them as needed). Structure: direct answer up front, then
supporting sections, then open questions / caveats. Write the substance now;
citations come next.

### 7. Cite — mandatory
**Always** spawn `citation-agent` before delivering. It reads `report.md` plus
the `findings/` files and inserts inline citations + a References section. A
deep-research deliverable without source attribution is incomplete.

### 8. Deliver
Give the user a concise answer in chat with the key findings, and point to
`research/<run-dir>/report.md` for the full cited report.

---

## Resuming an interrupted run

A Claude subscription enforces usage-limit windows, so a long run can be cut off
mid-flight. There is no automatic pause-and-resume — but the **on-disk state in
the run directory IS the checkpoint**, so nothing is lost. When the window
resets, the user runs `/research-resume` (or just asks to resume) and you pick
up exactly where you left off, redoing none of the completed work.

For this to work, the on-disk state must always reflect reality:

- **At planning**, before spawning anything, write `research/<run-dir>/state.md`:
  the question, the run dir, the current `phase`
  (`planning → workers → synthesis → citation → done`), and a subtopic table
  with each subtopic's `status` (`pending` / `done`).
- **A subtopic counts as `done`** only when its `findings/agent-<n>.md` exists
  **and ends with the marker `<!-- STATUS: COMPLETE -->`**. A findings file
  without that marker was interrupted mid-write — treat it as `pending` and
  re-spawn that worker.
- **Update `state.md` as you progress**: after each wave (mark finished
  subtopics `done`) and whenever you change `phase`.

To resume:
1. Find the run dir — the one the user named, else the most recent under
   `research/` whose `phase` is not `done`.
2. Read `state.md`, then scan `findings/` for completion markers and rebuild the
   true status. **Trust the files over a stale `state.md`.**
3. Continue from the current `phase`: re-spawn `research-subagent` workers ONLY
   for `pending` subtopics, then proceed through synthesis and the mandatory
   citation pass. Never redo completed work.

---

## Worker search discipline (enforced in the agent prompts)
- **Start broad, then narrow.** Begin with short, general queries to map the
  landscape; progressively sharpen. Counter the instinct to fire long, overly
  specific queries first.
- **Think between tool calls.** After each search/fetch, evaluate source
  quality and gaps before the next query — don't search on autopilot.
- **Parallelize.** Issue several searches/fetches at once rather than serially.
- **Stop on diminishing returns.** Respect the budget; don't keep going once the
  subtopic is answered.

---

## Run directory layout
```
research/<YYYY-MM-DD>-<slug>/
  state.md             # checkpoint: phase + subtopic statuses (enables resume)
  report.md            # final cited report (citation-agent finalizes this)
  findings/
    agent-1.md         # one per worker, written by the LEAD from the worker's
    agent-2.md         #   inline return; ends with the STATUS marker
    ...
```

## Cost discipline
Workers run on Sonnet and dominate the bill because they run in parallel. Keep
worker count and tool budgets proportional to the question (§2). The Opus
lead and citation passes are cheap by comparison — spend reasoning there.

## Self-improvement
If a run goes sideways (workers duplicate work, miss the point, or burn budget),
the root cause is almost always a vague worker prompt. Note what was
underspecified and tighten the delegation next time. You may propose edits to
the agent prompts in `.claude/agents/` when you spot a recurring failure.
