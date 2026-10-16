---
name: agent-create-pr
description: >
  Use when opening a pull request — "create a PR", "open a PR for this branch", "raise a PR for
  this work". Covers the preconditions for a draft and for a ready PR, the issue-tracker blocker
  check, the `[scopes] Title` and the body in squashed-commit format ending with `Closes ABC-123`
  or `Part of ABC-123`, and watching the checks after it opens. Does not review, approve, or merge.
---

# agent-create-pr

Open one pull request for a feature or fix branch, in a shape a reviewer can act on: a title that
threads the branch slug and a body that says *why* before *what*. The diff already shows *what*
changed — the body exists to say *why*. Branches, commits, local checks, and pushes follow
[git-workflow](../../rules/git-workflow.md).

Settings in `UPPER_SNAKE` come from [CUSTOMISE.md](../../CUSTOMISE.md).

## Before opening any PR

- **Branch on a fresh base and pushed**, per git-workflow.
- **Committed** with approved, signature-free messages, per git-workflow.
- **No open blockers.** For each ticket in the branch name, read its blockers in `ISSUE_TRACKER`
  the way `TRACKER_BLOCKERS` describes. A blocker that is neither done nor cancelled is open. For
  any open blocker, stop and ask the user whether to open the PR anyway, naming each blocker and
  its state. Only the user's explicit yes for this PR proceeds; a standing instruction to work
  without approval does not cover it. Read the tracker through the workspace's integration for it
  or its API with the user's credential, never printing the credential. If the tracker cannot be
  read, say so and ask the same question.

## Before a PR is ready for review

A draft may skip these and say in its overview what is unfinished; a ready PR, or a draft being
marked ready, may not.

- **Work finished.** No "fix it in review" leftovers.
- **Feature complete.** A feature PR delivers its ticket's whole acceptance list, and the
  end-to-end or scripted check the ticket names for it passes.
- **Self-reviewed.** The change was built to
  [development-checklist](../../rules/development-checklist.md); walk every applicable section
  against the final diff once more and fix what it finds.
- **Local checks** for the affected packages passed, per git-workflow; the full CI runs
  on the PR.

## Title

`[<scopes>] Description` — the commit subject format from git-workflow.

- Match the branch slug — one kebab-case idea threads slug → title → commit.
- No PR number and no `[WIP]` (use Draft).

## Body

The PR squashes into one commit, so its title and body become that commit message. Shape: a short
overview, then `-` bullets, then the ticket line — nothing else. No test plan, no review
checklist, no restatement of the diff.

- **Overview** — a few sentences on *why*: the cause and the constraint that shapes the design,
  whatever the diff cannot say for itself. Drop it when the bullets already carry the whole story.
- **Bullets** — logical changes, one line each, with the why where it is not obvious. Anything the
  diff cannot show gets its own bullet: a required env or secret change, an ordering constraint
  against another PR or a deploy, a manual step.
- **End with the ticket** when the PR is tied to one, one line per ticket in the branch name, in
  the `TICKET_LINES` format: `Closes ABC-123` when the merge finishes the ticket, `Part of ABC-123`
  when it does not.
- **End state only.** Describe what the branch does, never how it got there: no review history,
  alternatives considered, "X instead of Y", "no longer", or notes addressed to reviewers. Context
  a reviewer needs goes in a PR comment. The text follows
  [writing-style](../../rules/writing-style.md).

## Opening it

- Use `gh`, authenticated as the user, and target `DEFAULT_BRANCH`.
- Open as **Draft** for work in progress or for a reference PR that needs no review — say which in
  the overview; the title stays clean.
- When the project gates a check behind a PR label (`E2E_CHECK`, `E2E_LABEL`), decide whether the
  change touches behaviour that check covers. If it does, ask the user and add the label on a yes
  (`gh pr edit "$PR" --add-label "<E2E_LABEL>"`); otherwise say in the check report that it did not
  run.
- Watch the checks afterwards. A red PR is not ready yet: read the first real failure (the rest are
  usually fail-fast cancellations), fix, push, re-check, then ask for review. It is ready only when
  every check in `REQUIRED_CHECKS` passes.

## Rules

- Operate under [`AGENTS.md`](../../AGENTS.md); load the rules its index selects for this action.
- A feature never spans several PRs or stacked PRs. One PR may carry several related features or
  tickets that are reviewed and shipped together.
- Never open from an unpushed or stale branch.
- Never open a PR whose ticket has an open blocker without the user's explicit yes.
- The issue tracker is read-only here: never create, edit, or move a ticket without the user's
  explicit approval.
- No AI signatures anywhere — title, body, or commits.
- Do not merge, approve, or dismiss reviews here.
