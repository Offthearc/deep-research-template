# Deep Research for Claude Code

A drop-in [Claude Code](https://claude.com/claude-code) template that turns your
main session into a **Lead Orchestrator** which plans a research question,
fans it out to parallel worker agents, then synthesizes and cites a full report.

It's the multi-agent research pattern — a planner that delegates to many
narrowly-scoped workers — packaged as agents, slash commands, and an operating
manual you can clone into any project.

---

## How it works

```
                ┌──────────────────────────┐
   you ──/deep-research──▶ Lead Orchestrator │  (main session, Opus)
                │  plan → decompose → spawn  │
                └────────────┬───────────────┘
            ┌────────────────┼────────────────┐
            ▼                ▼                ▼
     research-subagent  research-subagent  research-subagent   (parallel, Sonnet)
     one subtopic       one subtopic       one subtopic
     search → think → return findings inline
            └────────────────┼────────────────┘
                             ▼
                  Lead persists findings,
                  evaluates gaps, may spawn
                  a second wave, then writes
                  the synthesized report
                             ▼
                     citation-agent          (final pass, Opus)
                  attributes every claim,
                  appends a References list
                             ▼
                  research/<date>-<slug>/report.md
```

The orchestrator never does the bulk searching itself — it **decomposes** the
question into non-overlapping subtopics, spawns a wave of workers, and
**synthesizes** their findings. Workers return findings inline; the lead owns
all file persistence (subagent file-writes are unreliable across environments).
A mandatory citation pass attributes every claim to a real source before
delivery.

## Roles

| Role | Who | Model | Job |
|------|-----|-------|-----|
| **Lead Orchestrator** | your main session | Opus | Plan, decompose, delegate, evaluate, synthesize. |
| **`research-subagent`** | spawned worker | Sonnet | Investigate ONE subtopic. Search → think → refine → return findings inline. |
| **`citation-agent`** | spawned worker | Opus | Final pass. Attribute every claim to a real source; append References. |

Fan-out happens only from the main session — workers never spawn their own
workers.

## Quick start

1. **Clone this template into your project** (or use it as the repo itself):

   ```bash
   git clone git@github.com:Offthearc/deep-research-template.git
   cd deep-research-template
   ```

   To add it to an existing project, copy the `.claude/` directory and
   `CLAUDE.md` into your project root.

2. **Open Claude Code** in the directory.

3. **Run a research operation:**

   ```
   /deep-research What are the tradeoffs between the major vector databases for RAG in 2026?
   ```

   The lead will plan, spawn workers in parallel, synthesize, cite, and drop the
   full report at `research/<date>-<slug>/report.md`.

> **Requirements:** Claude Code with web search/fetch enabled. The workers rely
> on `WebSearch` and `WebFetch`; `.claude/settings.local.json` carries an
> allowlist of fetch domains you can extend.

## Commands

| Command | What it does |
|---------|--------------|
| `/deep-research <question>` | Run a full research operation as the Lead Orchestrator. |
| `/research-resume [run-dir]` | Resume an interrupted run from its on-disk checkpoint. Defaults to the most recent incomplete run. |

You can also just ask in plain language — "research X", "deep-dive Y",
"investigate Z" — and the manual in `CLAUDE.md` kicks in.

## Scaling effort to the question

The lead matches fan-out to complexity rather than over-spawning:

| Query type | Workers | Tool calls each |
|------------|---------|-----------------|
| Simple fact / single lookup | 0 (answer directly) or 1 | 3–10 |
| Comparison / 2–4 distinct angles | 2–4 | 10–15 |
| Broad, open-ended, multi-part | 5–10+ | 10–20 |

## Run directory layout

Every run is self-contained and resumable:

```
research/<YYYY-MM-DD>-<slug>/
  state.md             # checkpoint: phase + per-subtopic status
  report.md            # final cited report (finalized by citation-agent)
  findings/
    agent-1.md          # one per worker, written by the lead from the
    agent-2.md          #   worker's inline return; ends with a STATUS marker
    ...
```

## Resuming an interrupted run

A Claude subscription enforces usage-limit windows, so a long run can be cut off
mid-flight. There's no magic pause/resume — instead **the on-disk state in the
run directory _is_ the checkpoint.** `state.md` tracks the phase
(`planning → workers → synthesis → citation → done`) and each subtopic's status,
and a subtopic only counts as done when its `findings/agent-<n>.md` ends with
`<!-- STATUS: COMPLETE -->`.

When your window resets, run `/research-resume`. The lead rebuilds the true
status from the files (trusting them over a stale `state.md`), re-spawns workers
only for pending subtopics, and continues through synthesis and citation —
redoing none of the completed work.

## Cost discipline

Workers run on Sonnet and dominate the bill because they run in parallel. Keep
worker count and tool budgets proportional to the question. The Opus lead and
citation passes are cheap by comparison — that's where the reasoning spend goes.

## Customizing

- **`CLAUDE.md`** — the full Lead Orchestrator operating manual. The source of
  truth for the protocol; edit it to change how research runs are conducted.
- **`.claude/agents/research-subagent.md`** — the worker prompt: search
  discipline (OODA loop), source-quality rules, and the inline findings format.
- **`.claude/agents/citation-agent.md`** — the citation pass: attribution rules
  and the "never invent a source" guardrails.
- **`.claude/commands/`** — the `/deep-research` and `/research-resume` slash
  commands.

If a run goes sideways (workers duplicate work or miss the point), the root
cause is almost always a vague worker prompt — tighten the delegation in
`CLAUDE.md` or the agent files.

## Repository layout

```
.
├── CLAUDE.md                          # Lead Orchestrator operating manual
├── README.md
└── .claude/
    ├── agents/
    │   ├── research-subagent.md       # parallel worker (Sonnet)
    │   └── citation-agent.md          # final attribution pass (Opus)
    ├── commands/
    │   ├── deep-research.md            # /deep-research
    │   └── research-resume.md          # /research-resume
    └── settings.local.json            # web-fetch domain allowlist
```
