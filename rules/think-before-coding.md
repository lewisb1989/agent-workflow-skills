# Think before coding

Classify request type and load required rules before coding; load on request at task start, before
the first edit.

**Rule**: Don't assume. Don't hide confusion. Surface tradeoffs.

- This rule is a hard gate. Short, one-line, or "obvious" requests are not exemptions. A branch /
  feature slug in the request (e.g. `(add-retry-budget)`) is a PR-mode signal — treat it as one.
- Before the first edit, output a one-line classification: request type, selected mode, rules and
  skills loaded, and assumptions. No edit until that line exists in the response.
- Feature / new feature / ticket / branch / PR-mode work → create the feature branch in the
  existing checkout before the first edit.
- Building a planned feature → check it is ready before the first edit: its ticket lists the
  acceptance criteria, and every behaviour decision it needs is recorded. If not, stop and ask.
- Existing-file narrow fix → load [minimal-change.md](minimal-change.md).
- Any code change → load [development-checklist.md](development-checklist.md) before the first
  edit and keep applying it: design against it, write code and tests to it, and self-review
  against it before calling the work done.
- Branch / commit / amend / PR / push → load [git-workflow.md](git-workflow.md) before acting.
- Live external state / CI / deploy / package state → load
  [no-guessing-current-state.md](no-guessing-current-state.md).
- Domain-specific implementation → load the relevant skill only after repo mode is decided.
- State assumptions explicitly; if uncertain, ask.
- Multiple interpretations exist → present them, don't pick silently.
- Simpler approach exists → say so. Push back when warranted.
- Unclear → stop, name what's confusing, ask.

Why: silent assumptions become silent bugs. A 30-second clarifying question saves an hour of wrong implementation.
