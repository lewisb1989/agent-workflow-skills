# Customise

Every project-specific value the rules and skills expect, in one place. A rule or skill names a
setting in `UPPER_SNAKE_CASE`; the agent looks it up here when a step needs it.

Edit the **Value** column for your project. A value in *italics* is an example to replace or
delete; anything else is a working default. To drop the lookup, you can instead replace each name
with its value throughout the files (search for the name) and delete its row — see the README.

## Repository and CI

| Setting | Value | Used by |
| --- | --- | --- |
| `DEFAULT_BRANCH` | `main` | git-workflow, no-guessing-current-state, agent-create-pr, agent-code-review |
| `GH_CLI` | `gh` | every GitHub command in the skills. Set a wrapper here if your machine routes GitHub accounts per repository. |
| `CI_CONFIG` | `.github/workflows/` | git-workflow (where to find the jobs a change triggers) |
| `LOCAL_CHECKS` | The validation commands of the `CI_CONFIG` jobs the changed paths trigger, scoped to the affected packages. *e.g. `make check PKG=<pkg>`, `cargo clippy -p <crate> && cargo test -p <crate>`, `pnpm --filter <pkg> lint test`* | git-workflow, and every skill that runs local checks through it |
| `REQUIRED_CHECKS` | Every check branch protection requires. *e.g. `lint`, `build`, `test`* | git-workflow, agent-create-pr |
| `E2E_CHECK` | *`e2e`* — a check that does not run on an ordinary PR push. Write `none` if every check runs on every push. | no-guessing-current-state, agent-create-pr, agent-review-pr |
| `E2E_LABEL` | *`e2e`* — the PR label that makes `E2E_CHECK` run on a PR | no-guessing-current-state, agent-create-pr |
| `PREAPPROVED_FORMATTERS` | `none`. *e.g. `cargo fmt`, `gofmt -w`, `prettier --write`, `ruff format`* — formatters an agent may run on its own changes without asking | context-discipline, AGENTS.md |

## Branches and commits

| Setting | Value | Used by |
| --- | --- | --- |
| `BRANCH_PATTERN` | `<user>/<slug>-<ticket>[-<ticket>...]`, where `<user>` is `gh api user --jq .login` | git-workflow, agent-create-worktree |
| `TICKET_PATTERN` | `ABC-123` — a project key, a hyphen, a number (`[A-Z][A-Z0-9]+-[0-9]+`) | git-workflow, agent-code-review |
| `TICKET_REQUIRED` | `yes` — a new branch needs a ticket unless the user confirms there is none | git-workflow |
| `COMMIT_SCOPES` | *`api`, `web`, `infra`, `docs`, `agents`* — the areas a `[scope]` tag may name; combine as `[api,web]` | git-workflow |

## Issue tracker

| Setting | Value | Used by |
| --- | --- | --- |
| `ISSUE_TRACKER` | *Linear*, *Jira*, or *GitHub Issues* — and how the agent reads it: an MCP or IDE integration, or the tracker's API with a token from the environment that is never printed | agent-create-pr, agent-code-review |
| `TRACKER_BLOCKERS` | How to list a ticket's blockers and tell open from closed — see below | agent-create-pr |
| `TICKET_LINES` | `Closes ABC-123` when the merge finishes the ticket, `Part of ABC-123` when it does not. Use the keywords your tracker's GitHub integration recognises. | agent-create-pr |
| `PLANS_LOCATION` | the issue tracker — where private plans and trackers live | memory |
| `DOCS_DIR` | `docs/` — public product and architecture docs | memory |

`TRACKER_BLOCKERS` examples:

- **Linear:** the issue's `inverseRelations` with `type: "blocks"`; a blocker is open while its
  `state.type` is neither `completed` nor `canceled`.

  ```graphql
  { issue(id: "ABC-123") { inverseRelations { nodes { type issue { identifier state { name type } } } } } }
  ```

- **Jira:** inward issue links of type "is blocked by"; a blocker is open while its status
  category is not Done.
- **GitHub Issues:** the issue's "blocked by" dependencies; a blocker is open while it is open.

## Agents and review

| Setting | Value | Used by |
| --- | --- | --- |
| `AGENT_EFFORT` | `xhigh` — reasoning effort for every headless run | ask-agent |
| `SUPPORTED_MODELS` | see below | ask-agent |
| `REVIEW_ROSTER` | see below | ask-agent, agent-code-review (multi-agent) |
| `DEFAULT_REVIEW_LEVEL` | `medium` | agent-code-review, agent-review-pr |
| `REVIEW_RUN_DIR` | `.agent-runs/reviews/` — must be gitignored | agent-code-review, agent-review-pr |
| `WORKTREE_ROOT` | `<parent of the repo>/<repo>-worktrees/` | agent-create-worktree |
| `WORKTREE_SUBMODULES` | `none` — which submodules each kind of work needs in a fresh worktree. *e.g. docs: none; any build: `vendor/core`; work on the plugin layer: also `third_party/plugins`* | agent-create-worktree |

`SUPPORTED_MODELS` — the model IDs a run may use. Check each against the provider's model catalog;
CLI release and account access decide availability.

- *Claude:* *`claude-sonnet-5`, `claude-sonnet-5-5`, `claude-opus-5`, `claude-opus-5-5`,
  `claude-fable-5-1`*
- *Codex:* *`gpt-5.5`, `gpt-5.6-luna`, `gpt-5.6-terra`, `gpt-5.6-sol`, `gpt-6-luna`, `gpt-6-sol`,
  `gpt-6.1-sol`, `gpt-6-astra`*

`REVIEW_ROSTER` — one Claude and one Codex model per review level. A review launches only the pair
for each requested level; a single `ask-agent` call uses the `medium` model for its provider.

| Review level | Claude | Codex | Minimum CLI |
| --- | --- | --- | --- |
| `low` | *`claude-sonnet-5-5`* | *`gpt-6-luna`* | *Claude Code 2.1.284; Codex CLI 0.156.1* |
| `medium` (default) | *`claude-opus-5-5`* | *`gpt-6.1-sol`* | *Claude Code 2.1.280; Codex CLI 0.159.1* |
| `high` | *`claude-fable-5-1`* | *`gpt-6-astra`* | *Claude Code 2.1.251; Codex CLI 0.153.4* |

Confirm minimum versions against each CLI's release notes. Newer model families can also need an
account with access to them. The mechanism needs two independent model families; to use other
providers, change the commands in [ask-agent](skills/ask-agent/SKILL.md) and the names here.

## Environments

| Setting | Value | Used by |
| --- | --- | --- |
| `ENVIRONMENTS` | *`dev`, `staging`, `prod`* | no-deploy |
| `ENV_POLICY` | `none` — every change to every environment needs approval per action. *e.g. agents may update `dev`; `prod` is operator-only, never changed by an agent and read only on explicit request* | no-deploy |
| `DEPLOY_COMMANDS` | `helm upgrade`, `terraform apply` / `tofu apply`, `kubectl` mutations, secret writes, image rollouts, *and your deploy scripts* | no-deploy |

## Code conventions

| Setting | Value | Used by |
| --- | --- | --- |
| `IMPLEMENTATION_SKILLS` | `none` — the project's `*-implementation` skills, one per language or tool, each also listed in AGENTS.md. *e.g. `python-implementation`, `terraform-implementation`* | development-checklist §13, code-comments, AGENTS.md |
| `CONCURRENCY_MODEL` | In-process state is owned by one task and changed only through messages sent to it; shared state behind a lock needs a stated reason | development-checklist §3 |
| `MIGRATION_PATH` | *the service's embedded migrations, run at startup* — how schema changes ship | development-checklist §5 |
| `OBSERVABILITY_GUIDE` | `none`. *e.g. `docs/observability.md`* — the project's logging, tracing, and alerting conventions | development-checklist §9 |

## Assumed, not configurable

- GitHub hosts the repository and `GH_CLI` is authenticated as the user.
- PRs land as one squashed commit, so the PR title and body become the commit message.
- `DEFAULT_BRANCH` is protected; nobody commits to it directly.
- `git`, `jq`, and, for multi-agent review, the `claude` and `codex` CLIs are installed.
