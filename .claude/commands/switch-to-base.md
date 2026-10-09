---
description: Switch to the base branch and fast-forward it to the remote — stops on uncommitted changes, a base branch checked out in another worktree, or a diverged base branch
allowed-tools: Bash(git status:*), Bash(git worktree list:*), Bash(git rev-parse:*), Bash(git rev-list:*), Bash(git log:*)
---

Switch this checkout to the base branch and fast-forward it to the remote.

`<base>` and `<remote>` below are the **Base branch** and **Remote** settings in
`.claude/agent-loop/conventions.md`, section `## Loop settings`. Either one missing or
still a placeholder → STOP.

These steps are the only copy of this procedure. The Agent Loop bootstrap
(`.claude/agent-loop/orchestrator.md`, step 3) runs them before creating a branch. Every
**STOP** below stops that caller too: report what you found and ask the user. Never work
around a stop.

1. **Note where you start.** `git rev-parse --abbrev-ref HEAD` — the branch you are
   leaving, for the report.

2. **Uncommitted changes.** Run `git status --porcelain`. Any output → STOP.
   Do not stash: the stash is shared by every worktree of this repository, so another
   session can pop it. Do not commit the changes or carry them onto `<base>` either.

3. **Where `<base>` is checked out.**
   - This checkout is already on `<base>` → skip to step 5.
   - Otherwise run `git worktree list --porcelain`. A `branch refs/heads/<base>` line
     means another worktree holds it → STOP. Git allows a branch in only one worktree at
     a time. Say which path holds it, and offer the two ways forward: run this command in
     that checkout, or — when the goal is a new branch — branch straight from the remote
     with `git fetch <remote> <base>` and `git switch -c <branch> <remote>/<base>`, which
     leaves the local `<base>` alone. Do not detach HEAD and do not force.

4. **Switch.** `git switch <base>`.

5. **Fast-forward.** Note `git rev-parse HEAD`, then run `git pull --ff-only <remote> <base>`.
   A failed fast-forward means local `<base>` has diverged from `<remote>` → STOP. Do not
   reset, merge, or rebase it.

6. **Report** in at most three lines:
   - the branch you left, and how many of its commits `<remote>` does not have
     (`git rev-list --count <branch>@{upstream}..<branch>`; "never pushed" when it has no
     upstream). Skip this line when you started on `<base>`;
   - how many commits the pull brought in (`git rev-list --count <noted sha>..HEAD`);
   - the new tip: `git log --oneline -1`.
