---
description: Run a multi-agent deep research operation on a topic or question
argument-hint: <research question or topic>
---

Run a **deep research operation** as the Lead Orchestrator, following the
protocol in `CLAUDE.md`.

Research question:

$ARGUMENTS

Steps:
1. **Plan** with extended thinking — restate the real question, decide the query
   type, and scale the worker count to its complexity (see the table in
   CLAUDE.md §2). If it's a trivial lookup, just answer it — don't spawn agents.
2. **Decompose** into non-overlapping subtopics. Pick a run directory
   `research/<YYYY-MM-DD>-<slug>/` and write `state.md` so the run is resumable
   if a usage-limit window interrupts it (see CLAUDE.md → "Resuming an
   interrupted run"). If the window resets mid-run, `/research-resume` continues.
3. **Delegate** by spawning `research-subagent` workers **in parallel** (all
   Agent calls in one message). Give each the five required pieces: objective,
   output format, source guidance, boundaries, and budget + agent number.
   (Workers return findings inline — they do not write files.)
4. **Persist + evaluate**: write each worker's returned findings to
   `research/<run-dir>/findings/agent-<n>.md`, confirm the `<!-- STATUS: COMPLETE -->`
   marker, then spawn a second wave only for real gaps.
5. **Synthesize** the report yourself to `research/<run-dir>/report.md`.
6. **Cite** by spawning `citation-agent` once, last — this is mandatory.
7. **Deliver** a concise answer in chat and point to the full cited report.

If the question is ambiguous or could be scoped several ways, ask one
clarifying question before spawning workers.
