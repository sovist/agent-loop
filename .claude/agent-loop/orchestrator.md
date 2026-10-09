# Agent Loop Orchestrator
## Max Iterations: 10 (`full`) — see **Modes**

---

## How to Start
Open your project in Claude Code and run:
```
/agent-loop <TICKET> [full|fast|solo|review]
```

Replace `<TICKET>` with the ticket id. The mode is optional and defaults to `full` — see
**Modes** below. In `review` mode the ticket is optional too, so `/agent-loop review`
reviews the current branch as it stands. The command lives in
`.claude/commands/agent-loop.md` and does nothing but point at this file, so the longhand
still works if you prefer it:
`Read .claude/agent-loop/orchestrator.md and start the agent loop for ticket <TICKET>`.

Everything project-specific — the base branch, the ticket source, the branch pattern, the
build and test commands, and the conventions the roles plan and review against — lives in
`.claude/agent-loop/conventions.md`. This file and the three role prompts are generic.

---

## Why each role gets its own context

In a single-context loop all three roles share one context window. The model reads a role
file and declares "I am now the Architect", but it is still the same context that just
wrote the code — carrying every dead end, every shortcut it talked itself into, and every
"I'll fix that later". A review from that context is not an independent review, and by
iteration 4 or 5 the accumulated context is also being compacted out from under it.

Here every role runs as a **subagent** with its own isolated context window:

| | One shared context | Subagent per role |
|---|---|---|
| Role switch | Read a role file, keep the same context | `Agent` call → fresh context window |
| Architect sees | Everything the Dev thought | `latest-changes.md` + the real diff |
| Orchestrator holds | Dev output + reviews + all source read | Loop state and short verdict lines |
| Long loops | Degrade as context fills | Flat — each turn starts clean |

This works because the file protocol is context-independent: every handoff goes through
`current-tasks/[TICKET]/`, and the subagents read those same files.

**You (the orchestrator) never take on a role and never edit source yourself.** You
bootstrap, dispatch, branch on verdicts, and report. That is the whole job.

---

## Plan before code

The Architect's first turn comes before any code exists. It writes `plan.md`: the
contracts the change touches, the data flow, and the test cases that set the bar for
approval. Every Dev then builds inside that plan.

Without it, the first design review comes after a full Dev turn. A wrong approach costs
that whole turn — a full build-and-test pass — and the next Dev starts cold. The
iteration 1 Dev would also write code without knowing the Approval Criteria, because the
Architect would only derive them in its first review.

The plan makes the loop follow the flow a careful developer does — investigate, plan, get
approval, implement. In `full` mode the loop pauses after the plan for the user's
approval, at the point where changing course is cheapest.

Two rules keep the plan from becoming a spec nobody questions:

- **The plan owns the contracts; the Dev owns the method bodies.** A Dev that changes a
  planned contract records it under `## Deviations from Plan`. Choices inside the
  contracts are not deviations.
- **The ticket is the spec, not the plan.** The reviewing Architect checks the code
  against the plan and the plan against `task.md`. `MAJOR REWORK` means the plan itself
  was wrong, and the loop re-plans before the next Dev.

What a plan covers, and its format, is defined under **Planning turn** in
`.claude/agents/architect.md`.

---

## The three subagents

Defined in `.claude/agents/` at the repo root — the body of each file *is* that role's
system prompt, loaded by the platform before the agent's first turn. Those three files are
the single source of truth for what each role does.

| `subagent_type` | Tools | Continuity | Writes |
|---|---|---|---|
| `senior-dev` | all | Fresh — new agent every iteration | source + `latest-changes.md` |
| `architect` | read-only + `Write` | Fresh — new agent every turn: the plan, then each review | `plan.md` (planning turn) or `latest-review.md` (review turn) |
| `tech-lead` | read-only + `Write` | Fresh — once per Architect approval | `pr-review.md` only |

**Every role is dispatched fresh.** There is no agent that survives from one iteration to
the next, and no continuation mechanism — each turn is an `Agent` call into a clean
context window. For the reviewers that is the point: a fresh Architect cannot rationalize
code it has never seen. Planning and reviewing are separate turns of the same role, so
the Architect reviewing iteration 1 did not write the plan either — it reads `plan.md`
like any other input.

For the Dev it is a constraint, and it shapes what `latest-changes.md` is for. The Dev
that reads it next iteration is not the one that wrote it, so that file is the *only*
thing carrying forward why a call was made. Its `## Architecture Decisions` and
`## Known Gaps / Questions for Architect` sections are not paperwork — they are the Dev's
handoff to its own successor, and the reason a cold respawn does not re-litigate a
decision the Architect already accepted.

The Architect's and Tech Lead's "read-only" is a **soft** guardrail: `Edit` is withheld,
but `Write` is needed for their review file and `Bash` is needed to build and diff, so a
determined agent could still touch source. The real enforcement is the rule in their
prompts plus the before/after tree comparison in Step 3.

---

## Modes

The mode decides **what the run is allowed to skip**. It changes nothing about the file
protocol, the stop conditions or the fingerprint checks — only which steps execute and
how many iterations are allowed.

| `MODE` | Chain | `max_iterations` | Skips | Output |
|---|---|---|---|---|
| `full` (default) | Plan → you approve → Dev → Architect ×N → Tech Lead | 10 | nothing | `pr-description.md` |
| `fast` | Plan → Dev → Architect ×N | 2 | the plan pause (P7) and Steps 14-20 (Tech Lead) | `pr-description.md` |
| `solo` | Dev | 1 | the plan, Step 2 (Architect) and Steps 14-20 | `pr-description.md` |
| `review` | Architect → Tech Lead | 1 | the plan, Step 1 (Dev) and bootstrap 2-3 | `review-report.md` |

Record the mode in `task.md` so a resumed run picks up the same one, and name it in the
bootstrap announcement. An unrecognised mode → STOP, ask the user; do not fall back to a
default silently.

**What each mode costs you, stated plainly:**

- `fast` drops the Tech Lead. That is a real gate, not a formality — but it is a *second*
  full review of a change the Architect already approved, and it can only ever stop the
  run, never improve the code. Dropping it is defensible on a ticket the Architect
  approved on iteration 1. The 2-iteration cap is the other half: if two Dev turns have
  not satisfied the Architect, the ticket is not a fast ticket, and the loop should hand
  back rather than grind.
  The plan still runs in `fast`, without the pause: it is what gives a 2-iteration cap a
  fair chance. A plan that needs the user's call still stops the run, because
  `[HUMAN_REVIEW_NEEDED]` is never skipped. A `MAJOR REWORK` stops it too: a wrong plan
  means this was not a fast ticket.
- `solo` drops the plan and both reviewers, so **you** are the Architect. It exists for
  mechanical tickets — a rename, a package bump, a doc-only change — where the review
  chain costs more than the work it reviews. Do not use it on anything touching auth,
  concurrency, or shared domain logic.
- `review` drops the Dev, and with it the plan — there is nothing to build. It reviews
  work that already exists on the current branch, whoever wrote it. It is the only mode
  that does not create a branch and the only one whose deliverable is findings rather
  than a PR description — see **Review mode** below.

In `fast` and `solo` the loop still writes `pr-description.md`, but it must say which
reviews were skipped — and, in `fast`, that nobody approved the plan. A description that
implies a sign-off nobody gave is the one failure mode these modes can introduce.

### Review mode

`review` inverts the loop: there is nothing to build, so the reviews *are* the product.
Five things follow from that, and none of them are optional.

1. **No branch is created.** Skip bootstrap steps 2-3 entirely. You are reviewing the
   branch that is already checked out; creating one off the base branch would move the
   user off the work they asked about. Record the current branch in `task.md` as-is.
2. **The orchestrator writes `latest-changes.md` itself.** Both reviewers read that file
   and no Dev ran, so synthesise it from git at bootstrap — changed-file list, the Base
   sha, and an explicit line saying it is generated and carries no Dev reasoning. Do not
   leave the placeholder text; an Architect that reads "_Empty — awaiting Iteration 1_"
   will conclude the iteration never happened. Never invent an `## Architecture
   Decisions` section: its absence is a fact about the review, not a gap to fill.
3. **The work may already be committed.** In the other modes the Dev never commits; here
   the work may be committed, uncommitted, or both. **The change under review** covers all
   three, so the reviewers read it exactly as in any other mode — but the Base must come
   from the remote base branch (bootstrap step 4), because the local one may be stale.
4. **The Tech Lead runs regardless of the Architect's verdict.** There is no Dev to send
   a `NEEDS CHANGES` back to, and stopping the chain on it would throw away the second
   opinion the mode exists to collect. Both reviewers run, then both verdicts are
   reported. The Architect's verdict is information here, not a gate.
5. **The output is `review-report.md`, never `pr-description.md`.** Writing a PR
   description would assert a readiness nobody established — the very failure the Tech
   Lead gate exists to prevent in `full`. Consolidate both reviews into the report and
   leave the judgement to the user.

The ticket may be omitted on the command line, but the run still needs one: take it from
the current branch name, which carries it by convention (the **Branch pattern** in the
conventions), by matching the **Ticket pattern**. Everything downstream is unchanged —
same `current-tasks/[TICKET]/` folder, same `task.md`, and the Architect derives its
Approval Criteria from the acceptance criteria as usual. If the branch name has no match,
STOP and ask for the ticket. Do not invent a task-folder name for the branch: an
off-convention branch is a question for the user, not a new layout.

The worktree fingerprint matters *more* here, not less: in the other modes a reviewer
that edits source stomps a subagent's work, and in this one it stomps the user's.

**Model choice is an orthogonal knob, not part of the mode.** See below.

---

## Dispatch rules

- **Always pass `run_in_background: false`.** Every step blocks the next one — there is
  nothing useful to do while a role runs.
- **`Agent` takes a `model` override.** It beats the `model:` frontmatter in
  `.claude/agents/<role>.md`, so a single run can be cheapened without editing the
  team-shared role files. Use it when the user asks for a cheaper or faster run; do not
  apply it on your own initiative, and do not tie it to the mode — a `fast` run on a
  subtle ticket still wants the strongest model reviewing it.
- **Pass only the ticket, the iteration number and — for the Architect — the turn
  (planning or review) in the prompt.** The subagent reads the task files itself. Do not
  paste `task.md` or the review into the prompt; that recreates the context bloat the
  subagents exist to remove.
- **The subagent's report is not shown to the user.** Relay the verdict line yourself.
- **Every dispatch is a fresh `Agent` call.** Do not try to continue a previous agent.
  The loop deliberately depends on nothing but the files in `current-tasks/[TICKET]/`,
  which is what keeps it portable across Claude Code surfaces.

---

## Worktree fingerprint

Run this before and after every Architect and Tech Lead turn, planning turns included.
One line of output; compare the two.

```bash
{ git status --porcelain               -- . ":(exclude)current-tasks"
  git diff HEAD                        -- . ":(exclude)current-tasks"
  git ls-files -z -o --exclude-standard -- . ":(exclude)current-tasks" | sort -z | xargs -0 -r sha256sum
} | sha256sum
```

It hashes three things, and it needs all three:

| Line | Catches |
|---|---|
| `git status --porcelain` | a path appearing or disappearing, or changing status code |
| `git diff HEAD` | **content** edits to tracked files |
| `git ls-files -o` + `sha256sum` | content edits to *untracked* files |

The status line alone is not enough, and this is the failure that matters most: if the
Architect edits a file the Dev already modified — by far the likeliest way a reviewer
"helps" — the porcelain line stays ` M src/app.py` before and after, and a path-only
comparison sees nothing. The content hashes are what actually close the hole. The third
line exists for the same reason on new files: a test file the Dev just created is
untracked, so `git diff HEAD` does not cover it. Its `-z` / `-0` keep a file name with
spaces in one piece; without them the file silently drops out of the hash.

---

## The change under review

Every mode reviews the same thing, and both reviewer prompts define it the same way:

- `git diff [Base]` — every edit to a tracked file since the Base sha, committed or not.
  Never `[Base]..HEAD`: the Dev does not commit, so that range is empty.
- every new file — `git ls-files -o --exclude-standard -- . ":(exclude)current-tasks"`.
  New files are untracked, and `git diff` does not show them.

The Base is the commit the branch starts from (bootstrap step 4). A Base that is too
early makes the reviewers read other people's commits as part of the change.

---

## Loop Algorithm

```
── BOOTSTRAP ────────────────────────────────────────────────────

INPUT: ticket id (e.g. ABC-123) — may be omitted in review mode,
       where it is read from the branch name instead
       MODE — full | fast | solo | review. Absent → full.
              Unrecognised → STOP, ask user.

0. Read `## Loop settings` in .claude/agent-loop/conventions.md.
   The file is missing, or a setting this run uses is still a `<…>`
   placeholder → STOP, ask the user to fill it in. `{…}` tokens are not
   placeholders: you fill them in at step 2.

   In review mode with no ticket given, take it from the current branch name
   with the conventions' Ticket pattern.
   No match → STOP, ask the user for the ticket.
   Every mode runs against current-tasks/[TICKET]/; there is no other layout.

   current-tasks/[TICKET]/task.md already exists → this ticket has run
   before. Do NOT bootstrap — no branch, no new files, nothing overwritten:
   task.md holds the user's ## Decisions. Go to RESUME.

   Resolve MODE and set the caps from the Modes table:
     full   → max_iterations = 10, plan = yes, plan_pause = yes, dev = yes, architect = yes, tech_lead = yes
     fast   → max_iterations = 2,  plan = yes, plan_pause = no,  dev = yes, architect = yes, tech_lead = no
     solo   → max_iterations = 1,  plan = no,  plan_pause = no,  dev = yes, architect = no,  tech_lead = no
     review → max_iterations = 1,  plan = no,  plan_pause = no,  dev = no,  architect = yes, tech_lead = yes

1. Fetch the ticket from the conventions' Ticket source:
   - jira    → the Atlassian MCP server's getJiraIssue tool
   - none    → ask the user for the title, type, description and
               acceptance criteria
   - anything else → follow the instruction written in the setting
   - Extract: title, description, type, acceptance criteria
   - Map the type to {prefix} with the conventions' type table.
     A type the table does not list → STOP, ask the user.
     (review mode: skip the mapping — no branch is created. Record the
      type in task.md as given.)

2. Derive branch name:            (review mode: SKIP — see step 3)
   - Fill the conventions' Branch pattern, resolving every token as the
     conventions define it.
   - The branch belongs to the developer, never the agent: no segment is
     `claude` or any tool's name. A token that resolves empty → STOP, ask
     the user; do not guess, and do not fall back to the OS account name.
   - Example: pattern {prefix}/{surname}/{TICKET}_{slug}
              → bugs/doe/ABC-123_Login-fails-after-password-reset
   - Confirm the final name with the user before creating it.

3. Create the branch locally off an up-to-date base branch. Do not push it.
   a. Run the steps in .claude/commands/switch-to-base.md — the same steps
      as /switch-to-base. Any STOP there → STOP, ask the user.
   b. git switch -c [branch]

   REVIEW MODE: skip steps 2 and 3. Do not create, switch, or rename a branch —
   you are reviewing the branch already checked out, and branching off the
   base branch would move the user off the work they asked about. Record the
   current branch (git rev-parse --abbrev-ref HEAD) in task.md as-is.

4. Record the Base — the commit the reviewers diff from (see
   **The change under review**):
   - full/fast/solo: git rev-parse HEAD. The branch was just cut, so this is
     exactly where it starts, whichever way step 3 cut it. Not a merge-base
     with the local base branch: when step 3 had to branch from
     [remote]/[base branch], the local one is stale and its merge-base is
     too early.
   - review: git fetch [remote] [base branch], then
     git merge-base [remote]/[base branch] HEAD — for the same reason.

5. Create task folder:
   - Path: current-tasks/[TICKET]/
   - Create subfolder: [task_folder]/history/

6. Write [task_folder]/task.md from the ticket, with the Base from step 4
   (see Task Format section)

7. Initialize empty placeholder files:
   - [task_folder]/plan.md            → "_Empty — awaiting Architect plan_"
     (solo/review mode: "_Skipped — [MODE] mode, no plan_")
   - [task_folder]/latest-changes.md  → "_Empty — awaiting Iteration 1_"
     (solo mode: unchanged — the Dev overwrites it)
     (review mode: NOT a placeholder. Generate it from git — see step 7a)
   - [task_folder]/latest-review.md   → "_Empty — awaiting Architect review_"
     (solo mode: "_Skipped — solo mode, no Architect review_")
   - [task_folder]/pr-review.md       → "_Empty — awaiting Tech Lead review_"
     (fast/solo: "_Skipped — [MODE] mode, no Tech Lead review_")

7a. REVIEW MODE ONLY — stand in for the Dev's report. Both reviewers read
    latest-changes.md and no Dev ran, so write it yourself:

      ## Iteration: 1
      ## Agent: Orchestrator (generated — review mode, no Dev ran)
      ## Status: READY FOR REVIEW

      ## Summary of Changes
      Pre-existing work on [branch]. Author unknown to this loop.

      ## Code Changes
      [one line per file from: git diff --stat [Base]
       then one line per new file from:
       git ls-files -o --exclude-standard -- . ":(exclude)current-tasks",
       marked (new)]

      Reviewers: `git diff [Base]` — covers committed and uncommitted work —
      and read the (new) files in full.

      ## Architecture Decisions
      None recorded. No Dev ran in this mode, so no reasoning is available —
      review the code on its own terms and do not assume intent.

      ## Tests Run
      None by this loop. Build and run the affected tests yourself.

    Leaving the placeholder text here is a real failure: an Architect that
    reads "_Empty — awaiting Iteration 1_" concludes the iteration never ran.

8. Announce bootstrap complete:
   "## ✅ Bootstrap complete
    Ticket   : [TICKET] — [Title]        (or "none — reviewing [branch]")
    Branch   : [branch name]             (review mode: "existing, not created")
    Folder   : [task_folder]/
    Mode     : [MODE] — [chain, from the Modes table]
    Starting loop..."

── RESUME (instead of steps 1-8, when task.md already exists) ───

R1. Read task.md: Mode, Branch, Base and ## Decisions. The mode comes from
    task.md. A different mode on the command line → ask before switching;
    on yes, update the Mode line first. Set the caps for that mode.
R2. The current branch must be task.md's Branch. Not → STOP, ask the user.
    Do not switch for them: the tree may hold uncommitted work.
R3. Find where the run stopped. The user said where ("from iteration N")
    → there. Otherwise compare the "## Iteration:" lines in the task files
    and take the furthest step of the highest iteration:
      pr-review.md has a verdict that         → Step 19, with that verdict
        ## Decisions does not record as sent back
      ## Decisions records the Tech Lead      → STEP 1 of iteration N+1,
        review of iteration N as sent back,     with the Tech Lead line
        and latest-changes.md is still
        iteration N
      latest-review.md has a verdict          → Step 3.10 of that iteration
      latest-changes.md READY FOR REVIEW      → STEP 2 of that iteration
      latest-changes.md IN PROGRESS           → STEP 1 of that iteration,
                                                with the retry line
      plan.md PLAN READY                      → P7 (full: approved only if
                                                ## Decisions says so)
      none of these                           → P1 (or MAIN LOOP, iteration
                                                1, when plan == no)
    Set iteration and approved to match.
R4. Announce: "## ⏯️ Resuming [TICKET] — [MODE], [where]" and continue there.

── PLAN (iteration 0 — before any code) ─────────────────────────

IF plan == no (solo, review):
    Announce: "## ⏭️ Plan skipped — [MODE] mode"
    Go to MAIN LOOP.

Steps P1-P7 run again as a RE-PLAN when a review returns MAJOR REWORK
(Step 0 of the main loop). [N] is then the iteration about to be built,
not 0.

P1. Announce: "## 📐 Plan | [TICKET]"
    (re-plan: "## 📐 Re-plan before iteration [N] | [TICKET]")
P2. fingerprint_before = <the recipe under "Worktree fingerprint">
P3. A fresh agent, like every other dispatch:
      Agent({
        subagent_type:     "architect",
        description:       "Architect plan",
        run_in_background: false,
        prompt: "Agent Loop, ticket [TICKET]. Planning turn, iteration 0.
                 Task folder: [task_folder]/
                 No code has been written yet. Write plan.md.
                 [Re-plan, use INSTEAD of the first and last lines:
                             'Agent Loop, ticket [TICKET]. Planning turn —
                              re-plan before iteration [N].
                              The iteration [N-1] review in latest-review.md
                              found the plan wrong; see its ## Plan Problems.
                              Revise plan.md.'
                  User-requested re-plan, use INSTEAD of the review line:
                             'The user asked for a re-plan; the reason is
                              under ## Decisions in task.md.'
                  After the user answered a plan pause, add:
                             'The user answered the last plan; see
                              ## Decisions in task.md.']"
      })
P4. Re-fingerprint and compare, exactly as in Step 3.9. The tree is clean
    before a first plan, but the rule is the same: STOP, show the diff,
    revert nothing.
P5. Relay the returned summary to the user.
    Archive: copy plan.md → history/iteration-[N]-plan.md (first plan: N = 0).
    An archive is never overwritten: a plan revised before the same
    iteration gets -2, -3, … (see File Map).
P6. Compare tokens, stripping the emoji from the file's "## Plan Status:"
    line, exactly as in Step 3.10:
      file_token != returned_token → STOP, ask user
      [HUMAN_REVIEW_NEEDED]        → STOP. Show the user the plan's
                                     ## Open Questions. Record the answer
                                     under ## Decisions in task.md, then
                                     re-run P1-P7.
      PLAN READY                   → continue to P7
    Anything else is an unrecognised token → STOP, ask user.
P7. IF plan_pause == yes (full):
      PAUSE for the user's approval. Show the plan's Approach, Contracts,
      Knock-on Changes and Test Cases — the parts that are expensive to
      change once code exists. The user may also edit plan.md directly;
      their edits stand.
        Approved          → record "Plan approved" under ## Decisions in
                            task.md (add "with edits" if they edited
                            plan.md) → MAIN LOOP
        Changes requested → record them under ## Decisions in task.md
                            → re-run P1-P7
    ELSE (fast):
      Continue to MAIN LOOP without pausing.
    A re-plan returns to Step 1 of the iteration it planned.

── MAIN LOOP ────────────────────────────────────────────────────

SET iteration   = 1
SET approved    = false
SET replan      = false
SET task_folder = current-tasks/[TICKET]

WHILE iteration <= max_iterations AND approved == false:

  ── STEP 0: RE-PLAN (only after a MAJOR REWORK) ────────────────
  IF replan == true:
      Run PLAN steps P1-P7 as a re-plan before iteration [N].
      replan = false

  ── STEP 1: SENIOR DEVELOPER TURN ──────────────────────────────
  IF dev == no (review mode):
      Announce: "## ⏭️ Senior Developer skipped — review mode"
      The work under review already exists and latest-changes.md was
      generated at bootstrap step 7a. Go straight to Step 2.

  1. Announce: "## 🧑‍💻 Iteration [N] — Senior Developer | [TICKET]"

  2. ALWAYS a fresh agent. No Dev survives from an earlier turn — a resumed run
     and iteration 5 of an unbroken run reach this step identically, because
     everything the Dev needs is on disk:

       Agent({
         subagent_type:     "senior-dev",
         description:       "Dev iteration [N]",
         run_in_background: false,
         prompt: "Agent Loop, ticket [TICKET], iteration [N].
                  Task folder: current-tasks/[TICKET]/
                  [If N == 1:  'First iteration — there is no review yet.
                                The plan is in plan.md.'
                   Else:       'The last Architect review is in
                                latest-review.md. Read it, plus the previous
                                iteration in history/, and address every
                                Required Change.'
                   If Step 0 re-planned this iteration, use INSTEAD of
                   the review line:
                               'The iteration [N-1] review found the plan
                                wrong, and plan.md has been revised — see
                                its ## Revisions. Read latest-review.md and
                                the revised plan, and rework the code to
                                the new plan.'
                   If the user asked for that re-plan, use instead:
                               'plan.md was revised at the user's request —
                                see its ## Revisions and ## Decisions in
                                task.md. Rework the code to the new plan.'
                   If retrying an unfinished iteration, add:
                               'The previous attempt at THIS iteration is in
                                latest-changes.md and reported IN PROGRESS.
                                Read it first and continue from there rather
                                than starting over.'
                   If sent back by the Tech Lead (Step 20), use INSTEAD of the
                   review line:
                               'The Architect approved this change; the Tech
                                Lead did not. The PR review is in
                                pr-review.md — read it and address every
                                Changes Requested item. Do not re-open
                                anything the Architect already signed off
                                on.']"
       })

  3. Relay the returned status line to the user.

  3a. Branch on the Dev's status token BEFORE dispatching the Architect:
        [HUMAN_REVIEW_NEEDED] → STOP, ask user. Do not review a blocked
                                iteration.
        IN PROGRESS           → the Dev is not done. Re-dispatch a fresh Dev
                                for the SAME iteration with the retry line
                                above; do not advance the iteration counter.
                                After 2 consecutive IN PROGRESS returns →
                                STOP, ask user.
        READY FOR REVIEW      → continue to Step 2.
      Anything else is an unrecognised token → STOP, ask user.

  4. Archive: copy latest-changes.md → history/iteration-[N]-dev.md
     (never overwrite an archive — see File Map)

  ── STEP 2: ARCHITECT TURN ─────────────────────────────────────
  IF architect == no (solo mode):
      Announce: "## ⏭️ Architect review skipped — solo mode"
      approved = false, reviewed = false → EXIT loop and go straight to
      PR DESCRIPTION. Do NOT run the Tech Lead and do NOT claim approval
      anywhere; the change is unreviewed and the description must say so.

  5. Announce: "## 🏛️ Iteration [N] — Architect Review | [TICKET]"

  5a. Take a WORKTREE FINGERPRINT so you can tell the Architect's writes
      from the Dev's. The tree is ALREADY dirty here — the Dev's source
      edits are uncommitted by design — so a bare `git status` after the
      review proves nothing.
        fingerprint_before = <the recipe under "Worktree fingerprint">

  6. A fresh agent, like every other dispatch:
       Agent({
         subagent_type:     "architect",
         description:       "Architect review [N]",
         run_in_background: false,
         prompt: "Agent Loop, ticket [TICKET], iteration [N]. Review turn.
                  Task folder: [task_folder]/
                  Review the change against the Base recorded in task.md —
                  git diff [Base] plus the new, untracked files — and the
                  plan in plan.md.
                  [Review mode, add: 'Review mode — no Dev ran, and the work
                   may be committed, uncommitted or both. latest-changes.md
                   is generated and records no decisions; judge the code on
                   its own terms.']"
       })

  7. Relay the returned verdict line to the user.
  8. Archive: copy latest-review.md → history/iteration-[N]-architect.md
     (never overwrite an archive — see File Map)

  ── STEP 3: CHECK STOP CONDITIONS ──────────────────────────────
  9.  Re-fingerprint and compare — never test for a dirty tree:
        fingerprint_after = <same recipe>
      IF fingerprint_after != fingerprint_before → the Architect wrote
      outside current-tasks/. Treat the review as untrusted → STOP, ask user.
      DO NOT revert anything: the Dev's work is uncommitted, so reverting a
      path the Dev also touched destroys it with no way back. Show the user
      `git diff HEAD -- . ":(exclude)current-tasks"` and let them decide.
      current-tasks/ is excluded because latest-review.md is the Architect's
      own deliverable and is expected to change.
  10. Compare verdicts by TOKEN, stripping the trailing status emoji from the
      file. "## Review Status: APPROVED ✅" carries the token APPROVED.
      IF file_token == returned_token == APPROVED → approved = true → EXIT loop
  11. IF either says "[HUMAN_REVIEW_NEEDED]" → STOP, ask user
  12. IF file_token != returned_token → STOP, ask user
  12a. IF token == MAJOR REWORK — the review says the plan itself is wrong:
         full   → replan = true → iteration++ → continue. Step 0 re-plans
                  before the next Dev, and pauses for approval like the
                  first plan did: the contracts the user approved changed.
         fast   → STOP, ask user. A wrong plan means this was not a fast
                  ticket; offer to continue in full mode, which re-plans
                  first.
         review → nothing to re-plan and no Dev to act on it. Go to 13.
  13. IF token == NEEDS CHANGES → iteration++ → continue
      (review mode, for NEEDS CHANGES and MAJOR REWORK alike: no Dev can act
       on the verdict, so do not iterate. EXIT loop and continue to the Tech
       Lead step.)
  13a. Any other token is unrecognised → STOP, ask user.

END WHILE

── ITERATION CAP REACHED ────────────────────────────────────────

IF approved == false AND architect == yes AND dev == yes:
    The cap ran out without an APPROVED review. STOP and ask the user.
    Do NOT run the Tech Lead and do NOT write pr-description.md.
    In fast mode say so explicitly: two Dev turns did not satisfy the
    Architect, so this was not a fast ticket — offer to continue in full
    mode from the current iteration rather than starting over.
    If the last verdict was MAJOR REWORK, the re-plan never ran — say so,
    and offer it as the first step of any continuation.

    Review mode is excluded: an unapproved verdict there is the finding,
    not a failed run, and the Tech Lead pass still has to happen.

── TECH LEAD STEP (after each Architect approval) ───────────────

IF tech_lead == no (fast or solo mode):
    Announce: "## ⏭️ Tech Lead review skipped — [MODE] mode"
    Skip to PR DESCRIPTION.

14. Announce: "## 👔 Tech Lead / PR Review | [TICKET]"
15. fingerprint_before = <the recipe under "Worktree fingerprint">
16. Agent({
      subagent_type:     "tech-lead",
      description:       "Tech Lead sign-off",
      run_in_background: false,
      prompt: "Agent Loop, ticket [TICKET]. Final PR review.
               Task folder: [task_folder]/
               [Review mode, add: 'Review mode — no Dev ran and the
                Architect verdict was [TOKEN], which may not be APPROVED.
                Review independently; do not treat the Architect verdict as
                a precondition. The work may be committed, uncommitted or
                both.']"
    })
17. Re-fingerprint and compare as in Step 3.9. A Tech Lead that wrote outside
    current-tasks/ → pause and ask user, again without reverting anything.
18. Relay the verdict. Archive: copy pr-review.md →
    history/iteration-[N]-tech-lead.md (never overwrite — see File Map)
19. Compare verdicts by TOKEN, exactly as in Step 3.10 — strip the trailing
    emoji from the file's "## PR Status:" line. "## PR Status: LGTM ✅"
    carries the token LGTM.
      IF file_token != returned_token       → STOP, ask user
      IF either says "[HUMAN_REVIEW_NEEDED]" → STOP, ask user
20. Branch on the verdict. The Tech Lead is a GATE, not a formality:
      LGTM              → continue to PR DESCRIPTION
      CHANGES REQUESTED → STOP. Do NOT write pr-description.md — the change is
                          not sign-off ready and a generated description would
                          say it is. Show the user the Changes Requested items
                          from pr-review.md and ask: send them back to the Dev,
                          or accept and proceed anyway?
                            Accept    → record it under ## Decisions in task.md
                                        → PR DESCRIPTION
                            Send back → record it under ## Decisions, with
                                        the iteration it reviewed, then
                                        approved = false, iteration++, and
                                        re-enter the MAIN LOOP at STEP 1 with
                                        the Tech Lead line in the Dev prompt.
                                        Keep pr-review.md for that Dev. The
                                        Architect reviews the iteration as
                                        usual, and on APPROVED a fresh Tech
                                        Lead runs again. The iteration cap
                                        still applies.
    Anything else is an unrecognised token → STOP, ask user.

    REVIEW MODE: there is no Dev to send anything back to, and no PR
    description to withhold. Neither verdict is a gate — carry both into the
    report and let the user decide.

── REVIEW REPORT (review mode only) ─────────────────────────────

21r. Write `[task_folder]/review-report.md`: both verdict lines, the Required
     Changes from latest-review.md, the Tech Lead's items from pr-review.md,
     and which of the two flagged each thing. Where they disagree, say so
     rather than merging them — that disagreement is the most useful output
     two independent reviewers produce.

     Do NOT write pr-description.md in this mode. Nobody built anything and
     nobody signed anything off; a PR description would assert a readiness
     this run never established.

     Then skip to FINAL SUMMARY.

── PR DESCRIPTION (full, fast, solo) ────────────────────────────
   (full mode: only after the Tech Lead returned LGTM)

21. Write `[task_folder]/pr-description.md` (see PR Description Format section)

    Its "## Reviews" section states what actually happened to each review.
    In fast or solo mode it MUST name the reviews that did not run — "Plan:
    not approved by the user (fast mode) / Architect: approved, iteration 2 /
    Tech Lead: skipped (fast mode)". In solo mode the plan line reads "Plan:
    skipped (solo mode)". If the user accepted a CHANGES REQUESTED verdict,
    the Tech Lead line says so. A description that reads as fully signed off
    when it is not is the one harm these modes can do.

── FINAL SUMMARY ────────────────────────────────────────────────

Print:
  "✅ Pipeline complete for [TICKET]
   📋 Ticket    : [the conventions' Ticket URL, or the id when there is none]
   🌿 Branch    : [branch name]
   📁 Folder    : [task_folder]/
   ⚙️  Mode      : [MODE]
   🔁 Iterations: [N] of [max_iterations]
   📐 Plan      : [approved by you | not reviewed by you (fast) | skipped]
                  [+ re-plans: K, if any]
   ✅ Reviews   : [Architect: approved iter N | NEEDS CHANGES | skipped]
                  [Tech Lead: LGTM | CHANGES REQUESTED | skipped]
   📄 Output    : [task_folder]/pr-description.md
                  (review mode: review-report.md)"
```

Nothing is committed or pushed at any point unless the user explicitly asks.

---

## task.md Format (auto-generated from the ticket)

```markdown
# [TICKET] — [Title]

## Ticket
- **URL**: [the conventions' Ticket URL, or "—"]
- **Type**: [Feature / Bug]
- **Branch**: `[full branch name]`
- **Base**: `[merge-base sha]`  ← reviewers diff against this
- **Mode**: `[full | fast | solo | review]`  ← a resumed run reads the mode from here

## Description
[Ticket description, cleaned up as plain markdown]

## Acceptance Criteria
- [ ] **AC-1** Criterion 1
- [ ] **AC-2** Criterion 2

(Numbered so plan.md's Test Cases can trace to them.)

## Tech Stack
[Copied from ## Stack in the conventions]

## Out of Scope
[Explicit exclusions from the ticket, or "Not specified"]

## Notes
[Additional context from ticket comments or labels, or "None"]

## Decisions
[The user's answers during the run — plan approval, plan change requests, answers to a
[HUMAN_REVIEW_NEEDED] pause — one line each, newest last. "None yet" at bootstrap.
Every role reads this section, and it outranks plan.md.]
```

`## Decisions` is the only place a user's answer survives. Every agent is fresh, so an
answer that lives only in this conversation is lost to the next one. Record it there
before the next dispatch.

---

## pr-description.md Format (auto-generated)

```markdown
## [TICKET] — [Title]

**Branch**: `[branch name]`
**Ticket**: [the conventions' Ticket URL, or the id]
**Type**: [Feature / Bug Fix]

---

## What Changed
[2-3 sentence summary from latest-changes.md]

## How to Test
[Derived from acceptance criteria in task.md]
1. Step 1
2. Step 2

## Checklist
- [ ] Acceptance criteria met (see the ticket)
- [ ] No regressions in existing behavior

## Reviews
- Plan: [approved by the user | not approved by the user (fast mode) | skipped (solo mode)]
- Architect: [approved, iteration N | skipped (solo mode)]
- Tech Lead: [LGTM | changes requested, accepted by the user | skipped ([MODE] mode)]

## Notes for Reviewer
[Contract changes — plan.md's Contracts as adjusted by the Dev's Deviations from Plan, or
"None". Then key decisions or trade-offs flagged by Architect or Tech Lead]
```

---

## Role Switching Protocol

There is no role switching — there is dispatch. You stay the orchestrator for the whole
run. Announce each dispatch so the user can follow the pipeline:

- `"## 📐 Plan | [TICKET]"` — before the planning `architect` call
  (re-plan: `"## 📐 Re-plan before iteration [N] | [TICKET]"`)
- `"## 🧑‍💻 Iteration [N] — Senior Developer | [TICKET]"` — before the `senior-dev` call
- `"## 🏛️ Iteration [N] — Architect Review | [TICKET]"` — before the review `architect` call
- `"## 👔 Tech Lead / PR Review | [TICKET]"` — before the `tech-lead` call

Never write `"I am now acting as the Architect"`. If you find yourself about to review
code in your own context, you have fallen back to a single-context loop — dispatch
instead.

---

## Stop Conditions

| Condition | Trigger | Action |
|---|---|---|
| 📐 Plan ready | Returned token `PLAN READY` **and** `## Plan Status: PLAN READY 📐` in plan.md | `full`: pause for the user's approval. `fast`: proceed to the Dev |
| 📐 Plan was wrong | Review token `MAJOR REWORK` | `full`: re-plan before the next Dev, then pause for approval. `fast`: stop, offer full mode. `review`: a finding, not a stop |
| ✅ Architect approved | Returned token `APPROVED` **and** `## Review Status: APPROVED ✅` in latest-review.md | Proceed to Tech Lead |
| 👤 Human needed | `[HUMAN_REVIEW_NEEDED]` in a return value or any file | Pause, ask user, record the answer under `## Decisions` in task.md |
| ⚠️ Verdict mismatch | Returned token ≠ the **token** on the file's `## Plan Status` (planning turn), `## Review Status` (Architect) or `## PR Status` (Tech Lead) line | Pause, ask user |
| ⚠️ Reviewer touched source | Post-turn worktree fingerprint ≠ pre-turn fingerprint for any Architect or Tech Lead turn, excluding `current-tasks/` | Pause, ask user — **do not revert** |
| ⚠️ Dev blocked | Dev returned `[HUMAN_REVIEW_NEEDED]` | Pause before the Architect turn |
| ⚠️ Dev not done | Dev returned `IN PROGRESS` twice in a row | Pause, ask user |
| ⚠️ Tech Lead requested changes | Returned token `CHANGES REQUESTED` | Stop — **do not** write `pr-description.md`; ask whether to send the items back to the Dev (a new iteration, then the Architect and a fresh Tech Lead again) or accept them. Review mode: not a stop — it goes in the report |
| ⚠️ Unrecognised token | A returned token outside the role's list | Pause, ask user |
| 🔁 Max iterations hit | Counter reaches `max_iterations` (10 full, 2 fast) | Stop, print warning; in fast mode offer to continue in full mode. Not reachable in review mode |

The Tech Lead is the last gate, not a formality. A `CHANGES REQUESTED` verdict must never
fall through to the PR description step — generating one would hand the user a
ready-to-paste description for a change its own final reviewer declined to sign off.

The `⚠️ Verdict mismatch` and `⚠️ Reviewer touched source` rows compare a **before/after
pair**, not an absolute state — and that is the whole point of them:

- The working tree is dirty throughout a run. The Dev's edits are uncommitted by design,
  so "dirty" is the normal condition and only a *change* across the review turn is a
  signal. `current-tasks/` is excluded because the reviewer's own review file lives there.
- That comparison must be over **content**, not the path list — see
  **Worktree fingerprint**. A reviewer editing a file the Dev already modified leaves the
  `git status` line untouched.
- Never repair a violation by reverting. Nothing in this loop is ever committed, so
  `git checkout -- <path>` on a file the Dev also touched throws their work away
  permanently. Show the user the diff and stop.
- `## Review Status` values carry a trailing emoji (`APPROVED ✅`, `NEEDS CHANGES 🔄`,
  `MAJOR REWORK 🚨`), and so do `## PR Status` values (`LGTM ✅`,
  `CHANGES REQUESTED 🔄`) and `## Plan Status` values (`PLAN READY 📐`), while the
  returned token never does. Strip the emoji and compare tokens, or the mismatch rule
  fires on every successful approval and every clean sign-off.

---

## Cost

Every role starts cold each iteration — the Dev re-reads `task.md`, the review and the
conventions, the Architect re-reads the diff and the surrounding source — so a subagent
loop costs more tokens per iteration than a single-context one. What you buy is a review
that is actually independent and a loop whose quality does not decay as the iteration
count climbs. On a two- or three-iteration ticket the difference is small; on a long one,
a single context's later reviews are not worth much anyway.

The cold Dev is the part you can most easily make worse: if `latest-changes.md` is thin,
the next Dev re-derives everything from the source and may undo a deliberate choice. Keep
its decision sections substantive and the re-read is cheap.

The plan adds one Architect turn per ticket, plus one per `MAJOR REWORK`. It is a
read-only turn with no build, so it pays for itself the first time it saves a Dev turn
built on the wrong approach.

If a run is burning more than expected there are three levers, in order of how much they
cost you: the **mode** (skip the reviews you do not need), the **`model` override on the
dispatch** (one run, no shared-file edit), and the **`model` and `effort` fields in
`.claude/agents/*.md`** (every run, for the whole team). Re-merging the roles back into
one context is not on the list — that is what the isolation exists to undo.

Two per-iteration costs are already removed and should stay removed: the Dev does not
paste file bodies into `latest-changes.md` (the reviewers read the real diff), and the
Architect does not re-run a build the Dev already ran unless it is about to approve or
the diff reaches code the Dev did not test.

---

## Useful Commands

| Goal | Claude Code prompt |
|---|---|
| Start a new ticket | `/agent-loop <TICKET>` |
| Start in fast mode | `/agent-loop <TICKET> fast` |
| Start in solo mode | `/agent-loop <TICKET> solo` |
| Review existing work on this branch | `/agent-loop review` |
| Review existing work, with the ticket | `/agent-loop <TICKET> review` |
| Start a new ticket (longhand) | `"Read .claude/agent-loop/orchestrator.md and start the agent loop for ticket <TICKET>"` |
| Resume a paused ticket | `/agent-loop <TICKET>` — an existing task folder resumes (see **RESUME**) |
| Resume from a given iteration | `"Resume agent loop for <TICKET> from iteration [N]"` |
| Re-plan before the next iteration | `"Re-plan <TICKET>: <why>"` — record the why under `## Decisions`, then run P1-P7 |
| Re-run the Architect only | `"Re-run Architect review for <TICKET> iteration [N]"` |
| Re-run Tech Lead only | `"Run Tech Lead review for <TICKET>"` |
| Regenerate PR description | `"Regenerate pr-description.md for <TICKET>"` |

The quoted prompts assume this file is loaded. In a new session, start them with
`Read .claude/agent-loop/orchestrator.md, then …`, or run `/agent-loop <TICKET>` first.

Resuming after a pause needs no special handling: every Dev turn is a fresh dispatch that
re-reads `task.md` and `latest-review.md`, so a resumed iteration 5 and an uninterrupted
iteration 5 are the same call. Nothing is held in memory between turns, which is why the
loop can be paused, resumed in a new session, or moved to another machine mid-ticket.
**RESUME** in the algorithm reads where the run stopped from the task files.

A resume reads the mode from `task.md`, not from the command line — so a ticket
bootstrapped in `fast` stays fast. To change it, say so explicitly ("resume <TICKET> in
full mode"); the Mode line in `task.md` is updated before anything is dispatched.

---

## File Map per Task
```
current-tasks/
└── <TICKET>/
    ├── task.md                   ← Auto-populated from the ticket; user answers under ## Decisions
    ├── plan.md                   ← Architect plan: contracts, data flow, test cases
    ├── latest-changes.md         ← Current Senior Dev output
    ├── latest-review.md          ← Current Architect review
    ├── pr-review.md              ← Current Tech Lead review
    ├── pr-description.md         ← Ready-to-paste PR description
    ├── review-report.md          ← review mode only, instead of pr-description
    └── history/
        ├── iteration-0-plan.md
        ├── iteration-1-dev.md
        ├── iteration-1-architect.md
        ├── iteration-2-dev.md
        ├── iteration-2-architect.md
        └── iteration-2-tech-lead.md
```

A re-plan before iteration N archives as `history/iteration-[N]-plan.md`, next to that
iteration's Dev and Architect files. The Tech Lead's review archives under the iteration
it signed off.

An archive is never overwritten. When the name is already taken — a plan revised twice
before the same iteration, a re-run Architect review — append `-2`, `-3`, …
(`iteration-0-plan-2.md`).

In review mode `latest-changes.md` is written by the orchestrator, not a Dev, and
`pr-description.md` is never written. In solo and review mode `plan.md` is a "Skipped"
placeholder. The folder layout is the same in every mode.

## Loop Files
```
.claude/
├── agents/              ← role prompts; edit these, nothing loads them from elsewhere
│   ├── senior-dev.md
│   ├── architect.md
│   └── tech-lead.md
├── commands/
│   ├── agent-loop.md    ← /agent-loop
│   └── switch-to-base.md← /switch-to-base (also bootstrap step 3)
└── agent-loop/
    ├── orchestrator.md  ← this file
    └── conventions.md   ← the only project-specific file
```
