---
description: Resume an interrupted deep-research run from its on-disk checkpoint
argument-hint: "[run-dir or slug — optional; defaults to most recent incomplete run]"
---

Resume a deep-research run that was interrupted (typically by a Claude
subscription usage-limit window). Act as the **Lead Orchestrator** and follow
the "Resuming an interrupted run" section of `CLAUDE.md`.

Target run: $ARGUMENTS

1. **Locate the run.** If a run dir or slug was given above, use it. Otherwise
   list `research/*/state.md` and pick the most recent run whose `phase` is not
   `done`. If nothing incomplete exists, say so and stop.
2. **Rebuild true status.** Read the target's `state.md`, then scan its
   `findings/` directory. A subtopic is genuinely done only if its
   `findings/agent-<n>.md` ends with `<!-- STATUS: COMPLETE -->`. Trust the
   files over a stale `state.md`, and update `state.md` to match.
3. **Report briefly** what you found: which subtopics are done, which are
   pending, and the phase you're resuming from.
4. **Continue the protocol.** Re-spawn `research-subagent` workers IN PARALLEL
   for pending subtopics only; persist each returned findings to
   `findings/agent-<n>.md` (workers return inline — the lead writes the file);
   then synthesize into `report.md`; then run the mandatory `citation-agent`
   pass. Do not redo completed work.
5. **Deliver** the concise answer in chat and point to the cited `report.md`.
