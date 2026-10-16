---
name: ask-agent
description: Run a fresh or resumed Claude or Codex headless session at the configured effort, persist its response, and record its session ID for a later follow-up.
---

Load on request when a task needs an independent Claude or Codex agent, or a
follow-up to a known agent session. Callers select a provider, output path, and
prompt; the model defaults below apply unless the caller explicitly overrides
them.

Settings in `UPPER_SNAKE` come from [CUSTOMISE.md](../../CUSTOMISE.md).

## Models

All calls use the `AGENT_EFFORT` effort (for example `xhigh`). The `SUPPORTED_MODELS` and
`REVIEW_ROSTER` settings are the single source of model IDs; other skills reference this section
instead of restating them.

### Supported models

`SUPPORTED_MODELS` lists the Claude and Codex model IDs a run may use. Use each provider's
published model catalog as the model-ID source. CLI release and account access determine
availability; validate an ID with a fresh session before dispatching it.

### Review-level roster

Review levels give review skills a consistent roster. `REVIEW_ROSTER` maps each level — `low`,
`medium` (default), and `high` — to exactly one Claude model and one Codex model, and records the
minimum CLI version each model needs.

A model newer than the installed CLI may need a CLI update. A CLI can log an unrecognised-model
diagnostic while still passing the ID to the API; use the response JSON and command exit status to
determine whether a run succeeded.

Judge a Codex run the same way: by exit status and output. A "Model metadata … not found" warning
means the CLI predates the model and the run may be degraded; update the CLI. A 400
`invalid_request_error` saying the model is not supported for the account is an entitlement
failure that no CLI update fixes.

When a roster model is unavailable, stop and report it; run another model only when the user
names it.

For a single call, use the `medium` model for its provider unless the caller
names another model. A review launches only the Claude and Codex pair listed
for every requested level.

## Fresh session

Use one fresh session for each independent task. Put only the task's required
scope in the prompt; do not pass the caller's conversation or another agent's
findings.

### Claude

```bash
prompt_file='<prompt-file>'
agent_prompt="$(< "$prompt_file")"
claude -p --model <model> --effort <effort> --output-format json "$agent_prompt" \
  < /dev/null > <response-json> 2> <stderr>
test $? -eq 0 || exit 1
jq -r -e '.result | select(type == "string" and length > 0)' <response-json> \
  > <output-file> || exit 1
test -s <output-file> || exit 1
jq -r -e '.session_id | select(type == "string" and length > 0)' <response-json> \
  > <session-id-file> || exit 1
test -s <session-id-file> || exit 1
```

### Codex

```bash
prompt_file='<prompt-file>'
agent_prompt="$(< "$prompt_file")"
codex exec -m <model> -c 'model_reasoning_effort="<effort>"' \
  --skip-git-repo-check --json -o <output-file> "$agent_prompt" \
  < /dev/null > <events-jsonl> 2> <stderr>
test $? -eq 0 || exit 1
test -s <output-file> || exit 1
jq -r -e 'select(.type == "thread.started") | .thread_id' <events-jsonl> \
  > <session-id-file> || exit 1
test -s <session-id-file> || exit 1
```

`<effort>` is `AGENT_EFFORT`. `<output-file>` is the agent's final message. Keep the
provider-specific JSON or JSONL only as run bookkeeping; the caller reads the Markdown output file.

## Continue a session

Resume only the exact ID saved by the fresh call. Never use a most-recent or
implicit-session option: another concurrent agent could select the wrong run.
Keep the same model and effort as the fresh call.

### Claude

```bash
prompt_file='<prompt-file>'
agent_prompt="$(< "$prompt_file")"
claude -p --resume "$(cat <session-id-file>)" --model <model> --effort <effort> \
  --output-format json "$agent_prompt" \
  < /dev/null > <response-json> 2> <stderr>
test $? -eq 0 || exit 1
jq -r -e '.result | select(type == "string" and length > 0)' <response-json> \
  > <output-file> || exit 1
test -s <output-file> || exit 1
```

### Codex

```bash
prompt_file='<prompt-file>'
agent_prompt="$(< "$prompt_file")"
codex exec resume -m <model> -c 'model_reasoning_effort="<effort>"' \
  --skip-git-repo-check --json -o <output-file> "$(cat <session-id-file>)" \
  "$agent_prompt" < /dev/null > <events-jsonl> 2> <stderr>
test $? -eq 0 || exit 1
test -s <output-file> || exit 1
```

## Rules

- Redirect stdin from `/dev/null` whenever the prompt is an argument. This
  prevents a headless session from waiting for additional input.
- Save the prompt in `<prompt-file>` and pass the loaded variable as one quoted
  argument. Never interpolate prompt text into shell source.
- Store each agent's output, session ID, stderr, and structured output under
  the caller's gitignored run directory.
- Treat a failed command, unparsable response, or empty output or session ID as
  an incomplete run. Do not resume, cross-validate, or consolidate it.
- A resumed session is appropriate for a follow-up that needs its own prior
  reasoning. Start a new session when independence matters.
- Do not use sandbox-bypass flags. The caller's prompt must say which files it
  may read or edit and what it must return.
