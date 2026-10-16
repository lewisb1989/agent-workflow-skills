# No unapproved deploys

Load on request before any command that changes a live environment.

**Rule**: Never deploy or otherwise mutate a live environment without approval for that action — except where a repo-local policy grants standing permission.

- Covers every command in `DEPLOY_COMMANDS` ([CUSTOMISE.md](../CUSTOMISE.md)): by default `helm upgrade`, `tofu` / `terraform apply`, secret writes, image rollouts, `kubectl` mutations, and the project's own deploy scripts.
- **Standing permission:** when the project sets an `ENV_POLICY` for its `ENVIRONMENTS`, that policy is authoritative — for example, **dev is updatable** by agents, and **prod is operator-only** (never updated by an agent; readable only on explicit request, read-only). Without one, every environment needs approval per action.
- When in doubt, propose the exact command and wait.
