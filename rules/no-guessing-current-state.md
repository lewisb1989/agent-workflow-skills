# No guessing current state

Load on request when a task depends on state that may have changed since the last session.

**Rule**: Verify live state before acting or claiming.

- Check GitHub PRs, CI, reviews, branches, deploys, packages, model catalogs, and external docs from their current source.
- Use local git / repo files for local state; use GitHub / CI / cluster tools for remote state.
- Quote the command, source, PR, commit, run, or file that established the state.
- If current state cannot be checked, say what is unverified and avoid merge / deploy / release claims.
- Memory can suggest where to look; it is not proof of current state.

**A green check is not the same as "everything ran".** Before reporting CI green, know which jobs
actually ran on that commit:

- A gated check such as `E2E_CHECK` ([CUSTOMISE.md](../CUSTOMISE.md)) may be a **merge gate**:
  it runs in the merge queue, on `DEFAULT_BRANCH`, on a manual dispatch, or on a PR labelled
  `E2E_LABEL` — *not* on an ordinary PR push. A green PR without that label means it did not run.
  When your change touches behaviour it covers, ask the user and add the label on a yes
  (`gh pr edit <n> --add-label <E2E_LABEL>`), and say which lanes ran when you report status.
- A **cancelled** job is not a passed job. `fail-fast` cancels sibling matrix lanes as soon as one
  fails, so "only lane X failed" usually means the others never finished — check every lane's
  conclusion, not just the failures.
- Verify a push produced a new run: `git push` on an unchanged branch succeeds silently and
  triggers nothing, which looks identical to a real push. Match the run's `headSha` to the branch
  tip before believing a result belongs to your change.

Why: stale state causes false approvals, wrong deploy claims, duplicate fixes, and broken follow-up work.
