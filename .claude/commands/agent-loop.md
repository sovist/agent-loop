---
description: Run the multi-role Agent Loop on a ticket — /agent-loop <TICKET> [full|fast|solo|review]
argument-hint: <TICKET> [full|fast|solo|review]
---

Read `.claude/agent-loop/orchestrator.md` and run the Agent Loop.

- **Ticket**: the first argument (`<TICKET>`). In `review` mode it may be omitted, in
  which case the orchestrator reads it from the branch name with the Ticket pattern in
  `.claude/agent-loop/conventions.md`; if the branch carries no ticket it asks you for one.
  A single argument that is a mode name is the mode, not a ticket.
- **Mode**: the remaining argument, or `full` if absent. Valid values are `full`, `fast`,
  `solo` and `review`; anything else — stop and ask rather than guessing. `review` skips
  the Dev and reviews the branch as it already stands, so it neither creates a branch nor
  edits any source.
- **Resume**: if `current-tasks/<TICKET>/task.md` already exists, the ticket is resumed,
  not bootstrapped again, in the mode its `task.md` records — see **Resuming** in the
  orchestrator.

Arguments as given: `$ARGUMENTS`

Follow the orchestrator's algorithm exactly, including the mode gating in **Modes** and
the five rules under **Review mode**. You are the orchestrator: bootstrap, dispatch,
branch on verdicts, report. Never take on a role yourself and never edit source.
