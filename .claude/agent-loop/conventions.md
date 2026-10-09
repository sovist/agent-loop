# Project conventions — Agent Loop

The only project-specific file in the loop. Every other file is generic: a private copy
changes this file and nothing else, so upstream updates merge cleanly.

- The orchestrator reads `## Loop settings` at bootstrap.
- Every role reads this whole file before its turn, then every file `## Read first` lists.
- A rule written here outranks the generic practice a role already knows.
- Fill in each `<…>`. "None" is a valid answer; delete example rows that do not apply.
- `{…}` is not a placeholder: the loop fills it in at run time — see **Branch tokens**.

---

## Loop settings

| Setting | Value | Used for |
|---|---|---|
| Base branch | `main` | branching off, `/switch-to-base`, the **Base** sha the reviewers diff from |
| Remote | `origin` | fast-forwarding the base branch |
| Ticket source | `none` | how bootstrap fetches the ticket — see **Ticket source** |
| Ticket URL | `—` | the link in `task.md`, the PR description and the final summary, e.g. `https://<site>.atlassian.net/browse/{TICKET}` |
| Ticket pattern | `[A-Z][A-Z0-9]+-[0-9]+` | review mode reads the ticket from the branch name with this regex |
| Branch pattern | `{prefix}/{TICKET}_{slug}` | the branch bootstrap creates — see **Branch tokens** |
| Build command | `<command>` | the Dev builds with it; reviewers verify with it |
| Test command | `<command>` | the Dev runs it before reporting; the Architect before approving |

### Ticket source
- `none` — bootstrap asks you for the title, type, description and acceptance criteria.
- `jira` — the Atlassian MCP server's `getJiraIssue` tool.
- Anything else — write the instruction in the cell. It must yield a title, a type, a
  description and acceptance criteria. E.g. for GitHub issues:
  `gh issue view {TICKET} --json title,body,labels`, the type taken from a label, and
  Ticket pattern `[0-9]+`.

### Ticket type → `{prefix}`

| Ticket type | `{prefix}` |
|---|---|
| Story, Feature, Task, Improvement | `features` |
| Bug | `bugs` |

A type this table does not list → bootstrap stops and asks (review mode creates no branch
and does not ask).

### Branch tokens
- `{TICKET}` — the ticket id, as given.
- `{prefix}` — from the type table above.
- `{slug}` — the ticket title: spaces → `-`, original word casing kept, characters git
  rejects in a ref name dropped. `<Extra slug rules, e.g. "drop a leading [...] component tag", or None>`
- `{surname}` — the developer's: the last word of `${GIT_AUTHOR_NAME:-$(git config user.name)}`,
  lowercased. Use it for a per-developer namespace, e.g. `{prefix}/{surname}/{TICKET}_{slug}`.

---

## Read first

Each role reads these before its turn. They outrank the generic patterns a role knows.

- `CLAUDE.md` (root), if the repo has one
- `<docs index>`, then the readme of the application the ticket touches
- `<test guide>`
- `<coding style guide>`

## Stack

The roles' stack expertise: the Architect plans in it, the Dev writes it, the reviewers
judge its idioms.

- Language and runtime: `<…>`
- Frameworks: `<…>`
- Data access: `<…>`
- Tests: `<test framework, assertion library>`

## Established patterns

The Dev reuses these; both reviewers check them. Deviation from one is a finding;
deviation from a textbook pattern this project does not use is not.

- Dependency injection: `<…>`
- Locking and concurrency: `<…>`
- Project layout: `<…>`
- Error handling: `<…>`

## Design — one owner, one copy

These checks apply to the code a change adds or modifies. They are not a reason to
refactor code the change does not otherwise touch: list existing violations as follow-ups.

### One owner per behavior (single responsibility)

Every member has one owner: the layer that already owns that kind of work.

- `<layer>` owns `<kind of work>` — e.g. services orchestrate, domain types own rules and invariants
- `<layer>` owns `<kind of work>`

**Check:** state the member's responsibility in one line. If the line needs "and", it is
two members.

### One copy of each piece of knowledge (don't repeat yourself)

Before adding logic, search for an existing implementation and reuse it. The duplication
that actually happens is a second implementation of something the repo already has.
Repetition inside a single diff is the rarer case.

**Check:** for any new helper, name the closest existing one and say why it does not fit.

### Limits
- DRY is about knowledge, not code that looks alike. Two blocks that change for different
  reasons stay separate; merging them couples unrelated changes.
- In tests, DRY covers setup: factories, base methods, shared assertions. Each test still
  reads as its own story.
- A new interface needs a consumer that requires it: a contract across a boundary, a
  second implementation, or a test that substitutes it. Not one per class by default.

## Contracts

The kinds of contract a plan lists when they change. "API" means all of these, not only
HTTP endpoints.

- Service interfaces and DTOs
- Domain methods other code calls
- Entities and tables
- Cache keys, message and job payloads, configuration settings
- `<project-specific kind, or None>`

Plans write signatures and shapes in `<language>`, in a fenced code block. No method bodies.

## Knock-on changes

What else must change when a contract does. The Architect lists every applicable row in
the plan; both reviewers check them.

| When the change touches | Also change |
|---|---|
| An entity or table | a migration, and what existing rows get |
| A public API type or endpoint | `<API snapshot or hash test>`, `<generated client>` |
| A shared service interface | `<every host that registers it>` |

**Public API consumers:** `<who depends on it — customers, an SDK, other services>`. A
breaking change to it is always `[HUMAN_REVIEW_NEEDED]`.

## Data flow

Questions every plan's `## Data Flow` answers, beyond the lock held, the save point and
the order of side effects:

- `<e.g. which copy of shared state is read and which is written>`

## Tests

- Naming: `<pattern, e.g. Method_ShouldResult_When_Condition>`
- Location: `<where the test for a given source file goes>`
- Deterministic only: no waiting on real time.
- `<fixtures, base classes, test substrate, skip attributes — or None>`

## Asking

Roles stop with `[HUMAN_REVIEW_NEEDED]` when a choice is genuinely the user's: two viable
approaches with meaningfully different trade-offs, a change that crosses application
boundaries, or ambiguous scope. This project also asks when:

- `<…, or None>`
