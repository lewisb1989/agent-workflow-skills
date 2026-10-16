# Agent workflow skills

Rules and skills that give a coding agent one disciplined way to work in a repository: classify
the request before editing, build to a development checklist, keep context small, never deploy or
post without approval, and run a pull request end to end — open it, review it, re-review it, and
answer its review comments.

Everything is plain Markdown. The agent reads one index, [AGENTS.md](AGENTS.md), and loads a rule
or skill only when its load hint matches what it is about to do. It works with any agent that
reads `AGENTS.md` or `CLAUDE.md`, such as Claude Code or Codex.

## What's inside

| Path | Holds |
| --- | --- |
| [AGENTS.md](AGENTS.md) | The index: every rule and skill with a one-line summary and a load hint. |
| [CUSTOMISE.md](CUSTOMISE.md) | Every project-specific setting, with defaults and examples. |
| [rules/](rules/) | 13 short rules. Each opens with when to load it, states one `**Rule**`, and ends with why. |
| [skills/](skills/) | 6 skills, each a `SKILL.md` plus any `references/` and `templates/`. |

The skills:

| Skill | Does |
| --- | --- |
| [agent-create-pr](skills/agent-create-pr/SKILL.md) | Opens a PR: checks the ticket for open blockers, enforces draft and ready prerequisites, and writes a `[scope] Title` and an end-state body that becomes the squashed commit. |
| [agent-code-review](skills/agent-code-review/SKILL.md) | Reviews a branch or PR against the development checklist and writes a local report. Optional multi-agent mode runs independent Claude and Codex reviewers that cross-validate each other. |
| [agent-review-pr](skills/agent-review-pr/SKILL.md) | Runs `agent-code-review` on a GitHub PR, filters findings to the PR's scope and to what the discussion has not already said, and stages them as a pending review only after you say yes. Never submits. |
| [agent-address-pr-comments](skills/agent-address-pr-comments/SKILL.md) | Verifies each open review thread against the code, agrees the triage with you, fixes what is real and in scope, and posts your approved replies as one review. |
| [agent-create-worktree](skills/agent-create-worktree/SKILL.md) | Creates a git worktree in a predictable sibling directory, named after the branch, or a detached tree for reviewing a PR. |
| [ask-agent](skills/ask-agent/SKILL.md) | Runs a fresh or resumed headless Claude or Codex session with a fixed model roster and explicit session IDs. |

## How the pieces fit

- `think-before-coding` gates every edit; `development-checklist` is the single list of what a
  change must not miss, applied while designing, writing, self-reviewing, and reviewing.
- `agent-code-review` walks that checklist; `agent-review-pr` posts its findings;
  `agent-address-pr-comments` answers findings on your own PRs. All three share one severity guide.
- Every outward action waits for approval: commits, pushes, PR comments, ticket changes, and
  deploys (`no-deploy`), unless the user has already given that instruction.

## Adopt it

1. Copy the files into your repository, for example under `agents/`:

   ```bash
   git clone <this-repo-url> /tmp/agent-workflow-skills
   mkdir -p agents
   cp -R /tmp/agent-workflow-skills/{AGENTS.md,CUSTOMISE.md,rules,skills} agents/
   ```

2. Point your agent at the index. Its links are relative to `agents/`, so don't symlink it to the
   repository root; add one line to your existing root `AGENTS.md` / `CLAUDE.md` that sends the
   agent to `agents/AGENTS.md`.

3. For Claude Code skill discovery, link the skills directory:
   `mkdir -p .claude && ln -s ../agents/skills .claude/skills`.

4. Fill in [CUSTOMISE.md](CUSTOMISE.md): default branch, ticket pattern, issue tracker, CI checks,
   reviewer models, environments, and the rest.

5. Gitignore the review run directory (`REVIEW_RUN_DIR`, `.agent-runs/` by default).

6. Fill in the `## Structure` section of `AGENTS.md` with your repository's top-level layout.

All links inside the files are relative, so the set works from any directory as long as
`AGENTS.md`, `CUSTOMISE.md`, `rules/`, and `skills/` stay together.

## Customise it

- **Settings.** Names in `UPPER_SNAKE_CASE` resolve through `CUSTOMISE.md`. Keep the names and edit
  the values there — upgrades stay a plain copy — or replace each name with its value in the files
  and delete its row, which saves the agent a lookup.
- **Language and tool skills.** Add `skills/<language>-implementation/SKILL.md` for each language
  or tool you use, list it in `AGENTS.md` and in `IMPLEMENTATION_SKILLS`. The checklist and the
  review skills load them automatically.
- **Checklist sections.** Each section of `rules/development-checklist.md` has a **When** line, so
  domain sections cost nothing when a change does not touch them.
  Delete the ones your project will never need, or add your own in the same shape.
- **New rules.** One file per rule: a load hint, one `**Rule**:` line, the bullets, and a `Why:`.
  Index it in `AGENTS.md` with its hint. `keep-agents-index-in-sync` keeps the index honest.
- **Reviewers.** Change `REVIEW_ROSTER` to the models you have access to. To use other providers,
  adapt the commands in `skills/ask-agent/SKILL.md`; the multi-agent review only needs two
  independent model families.

## Requirements

`git`, `jq`, and the GitHub CLI (`gh`) authenticated as you. Multi-agent review also needs the
`claude` and `codex` CLIs.

## License

MIT — see [LICENSE](LICENSE).
