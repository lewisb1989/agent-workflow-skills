# Git workflow

Load on request before creating a branch, committing, pushing, or opening or updating a PR.

**Rule**: Branch, commit, verify, and push the same way every time.

Settings in `UPPER_SNAKE` come from [CUSTOMISE.md](../CUSTOMISE.md).

## Branches

- GitHub flow: feature and fix branches, named for their tickets → protected `DEFAULT_BRANCH`.
  Never commit directly to `DEFAULT_BRANCH`.
- One feature never spans several pushed branches or stacked PRs. Local branches that merge into
  the feature branch, such as one per subagent worktree, are fine.
- Rebase a branch on the latest `DEFAULT_BRANCH` before opening its PR; the PR lands as one
  squashed commit.
- Name new branches by `BRANCH_PATTERN`, by default `<user>/<slug>-<ticket>[-<ticket>...]`:
  - `<user>` — the operator's GitHub login, from `gh api user --jq .login`.
  - `<slug>` — 2–5 kebab-case words, max 40 chars, `a-z 0-9 -`, no leading or trailing hyphen.
    Drop filler words.
  - `<ticket>` — the issue-tracker key, matching `TICKET_PATTERN` (e.g. `ABC-123`), required
    while `TICKET_REQUIRED` is `yes`. When it is unknown, ask for it; omit it only when the user
    confirms there is no ticket. Several tickets are appended in order.
- An existing branch, including a PR's head branch, keeps its original name.

```
<user>/lint-ci-tidy-ABC-123
<user>/lint-ci-tidy-ABC-123-ABC-456
```

## Commits

- Commit only finished work, after the user approves the exact message — unless they already told
  you to commit. One commit per coherent change.
- Message: `[<scopes>] Subject`, optionally followed by a blank line and `-` bullets of logical
  changes. The squash builds the `DEFAULT_BRANCH` commit from the PR title and body, so branch
  commits may stay one line; the detail belongs in the PR body.
  - Scopes are the areas touched, comma-separated, drawn from `COMMIT_SCOPES` and as used in
    `DEFAULT_BRANCH` history: `[api]`, `[infra]`, `[docs]`, `[api,web]`.
  - Subject: imperative, specific, ≤ 72 chars including the tag, no trailing period.
  - Bullets use `-`, never `*`, and start with a verb such as `Add`, `Update`, `Fix`, `Remove`, or
    `Refactor`; logical changes, not file-by-file detail.
  - State what the commit does; no narration of discussion, design history, or review.
- No AI signatures anywhere — commits, PR titles and bodies, or GitHub comments: no
  `Co-Authored-By`, `Generated with`, or tool badges.

## Local checks before push

Before a push that opens or updates a PR, check only what the change affects. CI runs the full
suite on the PR.

- Run `LOCAL_CHECKS` for the changed paths: by default, the validation commands of the `CI_CONFIG`
  jobs the changed paths trigger — format check, build or lint, and the tests covering the
  change — scoped to the affected packages, each exiting zero. Run formatters as checkers, never
  as writers.
- A push to a draft PR may carry failing or skipped checks when the PR says which; every check in
  `REQUIRED_CHECKS` passes before the PR is marked ready.
- When the change touches a shared or public surface, add its direct consumers, since a check
  scoped to one package never builds them.
- Leave the full workspace build, the whole test suite, fuzzing, benchmarks, and e2e to CI.
- Never run a publishing or mutating step: registry logins, image builds that push, releases,
  deploys, cache or artifact uploads, or repository writes.
- [context-discipline](context-discipline.md) counts a request to open or update the PR as
  approval for these checks; keep their output out of the prompt as it says.
- After the push, the PR's CI is the full check: read its result before asking for review.

## Push

- Push only on a request for that push; a request to make or commit changes is not one. A request
  to open or update a PR is.
- Read every check result before pushing. Never chain checks and the push in one command: a
  pipeline or a grep over a log can exit zero on a failed run.
- Force-push only your own unreviewed history, and only with `--force-with-lease`.
