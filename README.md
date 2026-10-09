# Agent Loop

A plan-gated, multi-role development loop for [Claude Code](https://claude.com/claude-code).

An Architect plans the change before any code exists. A Senior Dev builds inside the
plan. The Architect reviews the real diff, and that repeats until it approves — then a
Tech Lead does the final PR review. Every role is a subagent spawned fresh for each turn,
and every handoff is a file.

```
/agent-loop ABC-123 [full|fast|solo|review]
```

## The loop

```
ticket
    │
    ▼ Bootstrap (orchestrator — your session)
current-tasks/<TICKET>/task.md      branch cut from the base branch, never pushed
    │
    ▼ Plan (iteration 0 — before any code)
📐 architect   planning turn → plan.md
    │
    └─ you approve (full mode) → continue
    │
    ▼ Loop (max 10 iterations)
🧑‍💻 senior-dev  fresh every iteration → code + latest-changes.md
🏛️  architect   fresh every review    → latest-review.md
    │
    └─ NEEDS CHANGES → loop back (a new Dev)
    └─ MAJOR REWORK  → re-plan, you approve, then a new Dev
    └─ APPROVED ✅   → continue
    │
    ▼ After approval
👔 tech-lead    fresh → pr-review.md
    │
    └─ CHANGES REQUESTED → you choose: back to a new Dev, or accept
    └─ LGTM ✅          → continue
    │
    ▼ Written by the orchestrator
📄 pr-description.md  ← paste this into your PR
```

Nothing is committed or pushed unless you ask.

## What makes it work

- **Plan before code.** The Architect's first turn writes `plan.md`: the contracts as
  signatures and shapes — never method bodies — plus knock-on changes, owners, reuse,
  data flow, and test cases traced to acceptance criteria. Contracts are what gets
  expensive to change once code builds on them.
- **The plan owns the contracts; the Dev owns the bodies.** A Dev that changes a planned
  contract records it under `## Deviations from Plan`. The Architect treats an unrecorded
  one as a Required Change.
- **The ticket is the spec, not the plan.** Code that faithfully follows a wrong plan is a
  finding. `MAJOR REWORK` means the plan was wrong, and the loop re-plans before the next
  Dev.
- **Bugs start with a failing test.** The Dev writes the plan's regression case first. It
  must fail the way the plan predicts — if it passes, or fails for another reason, the
  plan's root cause is wrong and the loop stops for you.
- **Every role is fresh, every turn.** A reviewer that never saw the reasoning cannot
  rationalize the code. The loop holds no live state, so it can be paused, resumed days
  later in a new session, or moved to another machine mid-ticket: `/agent-loop <TICKET>`
  picks it up where the task files say it stopped.
- **Reviewers read the change, not the report.** They read `git diff` from the **Base**
  sha in `task.md`, plus every new file. The Dev's `latest-changes.md` is a claim to
  verify.
- **Your answers outrank the plan.** Every answer you give at a pause goes under
  `## Decisions` in `task.md` — the only way a fresh agent learns it.
- **Exact tokens, checked twice.** Each reviewer returns a bare status token and writes
  the same token to its file. A mismatch pauses the loop.
- **Reviewers cannot quietly fix code.** The orchestrator hashes the working tree's
  content before and after every Architect and Tech Lead turn. A change stops the loop,
  and nothing is reverted — nothing is committed, so a revert would take the Dev's work
  with it.

The full algorithm, stop conditions and file formats are in
[`.claude/agent-loop/orchestrator.md`](.claude/agent-loop/orchestrator.md).

## How it differs

Fresh subagents per role, read-only reviewers, an orchestrator that only judges, file
handoffs and plans that stop at signatures are common ground by now:
[Superpowers](https://github.com/obra/superpowers) (`writing-plans` and
`subagent-driven-development`), [squad](https://github.com/danielfarnose/claude-code-dev-loop)
and [subagent-loop](https://github.com/MustafaMese/subagent-loop) do most of them. Going by
their docs as of October 2026, three things here are in none of them:

- **A bug's root cause is falsifiable.** The plan states the root cause as a hypothesis
  and predicts which assertion the regression test fails, and why. The Dev runs that test
  before the fix. If it passes, or fails for another reason, the loop stops: the plan was
  wrong, and the Dev does not get to hunt for a different fix. Superpowers and squad write
  the test first; none of the three ask it to fail the predicted way.
- **Reviewer edits are detected, not just forbidden.** squad and subagent-loop keep
  reviewers read-only through their tool lists, and squad has them review a frozen commit. Here the work
  stays uncommitted in one tree, so the orchestrator hashes its content before and after
  every reviewer turn. A change — even to a file the Dev already modified — stops the loop.
- **Knock-on changes are planned.** The conventions file lists what else moves when a
  contract does — an API snapshot test, a generated client, a second host that registers
  the same services, a migration — and the plan names every one that applies before any
  code exists.

It also makes different choices:

- **The whole ticket per iteration.** The Dev builds the ticket and the reviewers judge the
  whole change, every iteration. Superpowers and subagent-loop split a plan into tasks,
  and subagent-loop runs them in parallel waves — the better fit for large plans.
- **Nothing is committed.** Every change stays in the working tree for you to commit;
  squad merges approved work itself.
- **Files you copy, not a plugin.** The roles and the orchestrator live in your repo, next
  to the conventions they read.

## The three subagents

| `subagent_type` | Model | Effort | Tools | Continuity |
|---|---|---|---|---|
| `senior-dev` | opus | high | all | Fresh every iteration |
| `architect` | opus | xhigh | read-only + `Write` | Fresh every turn — the plan, then each review |
| `tech-lead` | opus | xhigh | read-only + `Write` | Fresh, once per Architect approval |

A fresh Dev re-derives context the previous one had, so `latest-changes.md` has to carry
it: its `## Architecture Decisions` section is the Dev's handoff to its own successor.

Read-only is soft. `Edit` is withheld from the reviewers, but they need `Write` for their
review file and `Bash` to build and diff, so the guardrail is their prompt plus the
worktree fingerprint — not tool permissions alone.

To cheapen one run, ask for a `model` override on the dispatch. To change it for the
whole team, edit the `model:` and `effort:` frontmatter in `.claude/agents/<role>.md`.

## Modes

A mode decides **what the run is allowed to skip**. Nothing else changes — same file
protocol, same stop conditions, same worktree checks.

| Mode | Chain | Max iterations | Skips | Output |
|---|---|---|---|---|
| `full` (default) | Plan → you approve → Dev → Architect ×N → Tech Lead | 10 | nothing | `pr-description.md` |
| `fast` | Plan → Dev → Architect ×N | 2 | the plan pause and the Tech Lead | `pr-description.md` |
| `solo` | Dev | 1 | the plan, Architect and Tech Lead | `pr-description.md` |
| `review` | Architect → Tech Lead | 1 | the plan and the Dev | `review-report.md` |

- **`fast`** drops the Tech Lead — a second full review of a change the Architect already
  approved. If two Dev turns have not satisfied the Architect, this was not a fast ticket:
  the loop hands back to you and offers to continue in `full`. The plan still runs,
  without waiting for your approval.
- **`solo`** drops the plan and both reviewers, which makes **you** the Architect. For
  mechanical work — a rename, a package bump, a doc-only change. Not for auth,
  concurrency, or shared domain logic.
- **`review`** drops the Dev and points both reviewers at work already on the current
  branch, whoever wrote it. It never creates or switches a branch, the Tech Lead runs even
  on `NEEDS CHANGES`, and the output is `review-report.md` — never a PR description,
  because nothing was built or signed off.

In `fast` and `solo` the PR description names the reviews that did not run. The mode is
recorded in `task.md`, so a resumed ticket keeps it.

## Install

1. Copy these files into your repository, at the same paths:

   ```
   .claude/agents/architect.md
   .claude/agents/senior-dev.md
   .claude/agents/tech-lead.md
   .claude/commands/agent-loop.md
   .claude/commands/switch-to-base.md
   .claude/agent-loop/orchestrator.md
   .claude/agent-loop/conventions.md
   ```

2. Fill in `.claude/agent-loop/conventions.md` — see the next section.
3. Ignore `current-tasks/`: add it to `.gitignore`, or to `.git/info/exclude` for a
   personal setup. This is required: `/switch-to-base` stops on any untracked file, so an
   earlier ticket's folder would block the next one.
4. Commit `.claude/agents/`, `.claude/commands/` and `.claude/agent-loop/`, so the whole
   team runs the same roles.

Optional: `/switch-to-base` pre-approves only read-only git commands. To skip the prompt
for the switch and the pull, allow them in `.claude/settings.json`, with your own base
branch and remote:

```json
{
  "permissions": {
    "allow": ["Bash(git switch main)", "Bash(git pull --ff-only origin main)"]
  }
}
```

To namespace the commands (`/acme:agent-loop`), move both into `.claude/commands/acme/`
and update every mention of them: search the copied files for `agent-loop.md` and
`switch-to-base`.

**Requirements**
- Claude Code with subagents (`.claude/agents/`)
- git, and a shell with `sha256sum` and GNU-style `xargs -r` for the worktree
  fingerprint — Linux, or Git Bash on Windows. On macOS, install coreutils or swap in
  `shasum -a 256`.
- Optional: an Atlassian MCP server, to fetch tickets from Jira

## Fill in the conventions

`.claude/agent-loop/conventions.md` is the only project-specific file. A private copy
changes it and nothing else, so upstream updates merge cleanly.

| Section | Read by | What goes there |
|---|---|---|
| `## Loop settings` | orchestrator | base branch, remote, ticket source and URL, ticket and branch patterns, build and test commands |
| `## Read first` | every role | your existing docs: `CLAUDE.md`, architecture notes, the test guide |
| `## Stack` | every role | language, frameworks, data access, test framework |
| `## Established patterns` | Dev, reviewers | the patterns a change must reuse: DI, locking, project layout |
| `## Design` | every role | which layer owns which kind of work; the SRP and DRY checks come pre-filled |
| `## Contracts` | Architect | the kinds of contract a plan lists |
| `## Knock-on changes` | Architect, reviewers | what else moves when a contract does |
| `## Data flow` | Architect | the project's own data-flow questions |
| `## Tests` | Architect, Dev | test naming, location, determinism |
| `## Asking` | Architect, Dev | extra triggers for `[HUMAN_REVIEW_NEEDED]` |

- **Point, don't copy.** If a rule already lives in `CLAUDE.md` or your docs, list that
  file under `## Read first` and leave the rule out. Two copies drift apart.
- **Write rules a reviewer can check.** "An entity change ships with a migration" is
  checkable; "follow best practices" is not.
- **Spend the most care on knock-on changes.** They are the edits a careful developer
  forgets — an API snapshot test, a generated client, a second host that registers the
  same services — and the plan lists them before any code exists.
- **Ticket source is optional.** `none` makes bootstrap ask you for the ticket. `jira`
  uses the Atlassian MCP server. Any other value is an instruction the orchestrator
  follows, e.g. `gh issue view {TICKET} --json title,body,labels`.
- **`<…>` is yours to fill in; `{…}` is not.** Tokens such as `{TICKET}` and `{slug}` are
  filled in by the loop at run time.

## Usage

| Goal | Prompt |
|---|---|
| Start a ticket | `/agent-loop <TICKET>` |
| Start, skipping the Tech Lead | `/agent-loop <TICKET> fast` |
| Start, no reviewers | `/agent-loop <TICKET> solo` |
| Review this branch as it stands | `/agent-loop review` |
| Resume a paused ticket | `/agent-loop <TICKET>` — an existing task folder resumes |
| Resume from a given iteration | `"Resume agent loop for <TICKET> from iteration N"` |
| Re-plan before the next iteration | `"Re-plan <TICKET>: <why>"` |
| Re-run the Architect only | `"Re-run Architect review for <TICKET> iteration N"` |
| Re-run the Tech Lead | `"Run Tech Lead review for <TICKET>"` |
| Regenerate the PR description | `"Regenerate pr-description.md for <TICKET>"` |
| Back to an up-to-date base branch | `/switch-to-base` |

The quoted prompts work in the session where the loop ran. In a new session, start them
with `Read .claude/agent-loop/orchestrator.md, then …`.

In `full` mode the loop always pauses once, after the plan. Approve it, ask for changes,
or edit `plan.md` yourself — your edits stand. It also pauses when a role returns
`[HUMAN_REVIEW_NEEDED]`, when a returned token disagrees with the file, when a reviewer
touched anything outside `current-tasks/`, when the Dev reports `IN PROGRESS` twice
running, and when the Tech Lead returns `CHANGES REQUESTED`.

## Files per ticket

```
current-tasks/
└── <TICKET>/
    ├── task.md              ← the ticket; your answers under ## Decisions
    ├── plan.md              ← Architect plan: contracts, data flow, test cases
    ├── latest-changes.md    ← current Senior Dev report
    ├── latest-review.md     ← current Architect review
    ├── pr-review.md         ← Tech Lead sign-off
    ├── pr-description.md    ← ready-to-paste PR description
    ├── review-report.md     ← review mode only, instead of pr-description.md
    └── history/
        ├── iteration-0-plan.md
        ├── iteration-1-dev.md
        ├── iteration-1-architect.md
        └── iteration-1-tech-lead.md
```

Archives are never overwritten: a second file for the same step gets `-2`, `-3`, …

## Cost

Every role starts cold, so the loop spends more tokens per iteration than a single
context playing all three roles. What that buys is a review that is actually independent,
and a loop whose quality does not decay as iterations climb. The levers, cheapest first:
the mode, a `model` override for one run, and the `model` / `effort` frontmatter for
everyone.

## License

MIT — see [LICENSE](LICENSE).
