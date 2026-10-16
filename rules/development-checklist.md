# Development checklist

Load at the start of every coding task, before the first edit, and keep it applied until the work
is done; load it for every code review.

**Rule**: It lists what a change must not miss, and it governs every stage — not a final gate.

Settings in `UPPER_SNAKE` come from [CUSTOMISE.md](../CUSTOMISE.md).

## How to use it

- **Before the first edit:** read the applicable sections; they shape the design and the plan.
  Design comes first.
- **While writing code:** apply each applicable item as you write. Concurrency, crash recovery,
  error handling, observability, and tests are built in from the start, never bolted on at the end.
- **Before calling the work done or opening the PR:** self-review the final diff against every
  applicable section and fix what it finds.
- **Reviewing someone else's change:** walk the same sections over their diff.
- A section applies when the change touches what its **When** line names; sections without one
  always apply.
- Each item is a question. An unmet item the code, config, or tests can settle is a defect: fix it
  while developing, report it while reviewing. An item they cannot settle is stated as unverified,
  never assumed.
- Weigh each item against what the code is for and where it runs. A dev-only mock, a test harness,
  or a service that runs as one replica by design does not owe production hardening: note an item
  that does not apply, with the reason, rather than adding machinery for it. Every guard you add
  names the concrete scenario in this deployment that needs it.

## 1. Intent and scope

- Does the change do what its ticket or task asks, and is anything it requires missing?
- For a feature: is it complete — every acceptance criterion met, the feature built to run end to
  end in a non-production environment, and the check its ticket names passing — and does any part
  of its design still wait on an open decision?
- Does the change carry edits unrelated to that intent?
- Is the solution sized to the problem — no speculative abstraction, configurability, or
  abstraction with one caller ([minimal-change](minimal-change.md))?

## 2. Design and data flow

When: calculations, persistence, service or module boundaries, new or updated dependencies.

- Which service or component owns the calculation and the authoritative data? Is there one
  source of truth, or can two derivations drift?
- Trace the data flow end to end: where it enters, where it is persisted, what is cached, and how
  the cache is invalidated.
- What new dependencies — services, packages, infrastructure — does it introduce, and what happens
  if each fails at the worst time?
- Is each new library needed — nothing in the workspace or standard library already does it — and
  within the dependency rules of the language's `*-implementation` skill, when there is one?
- Is each new dependency from its official registry or upstream repository, with its exact name and
  publisher verified against typosquatting?
- Does each dependency bump link its changelog or release notes and confirm that the resulting
  behavior is expected?
- How do scheduling, retries, persistence, and recovery interact?
- Does the responsibility sit in the right service or module, with the smallest public surface?
- In the code the change touches, does each layer do only its own job? Persistence stores and
  loads — queries, transactions, constraints — and makes no business decisions; business logic
  builds no SQL or wire formats and is testable without a database; API and message handlers
  translate and delegate.
- Is it consistent with similar implementations in the repository, and is any departure
  justified?

## 3. Concurrency

When: async tasks, threads, multiple replicas, shared state, writes from more than one place,
retries, queues.

- Does in-process state follow the project's convention, `CONCURRENCY_MODEL` — for example, owned
  by one task and changed only through messages sent to it? A departure, such as shared state
  behind a lock, needs a stated reason the convention does not fit.
- Can two tasks, threads, or replicas act on the same state at once? What serializes them — a
  single owner, a unique constraint, a transaction, a row lock, or an advisory lock?
- Is every read-modify-write atomic (compare-and-set, conditional `UPDATE`, `SELECT … FOR
  UPDATE`), and is a failed compare-and-set handled, never swallowed?
- Is every retried or redelivered operation idempotent — an idempotency key, a unique constraint,
  or an upsert?
- Does ordering matter, and is it guaranteed or enforced?
- Are locks held for the shortest scope — no await while holding one, no lock-order deadlock?
- Is async code cancellation-safe: can a dropped future or cancelled task (timeout, select,
  shutdown) leave state half-updated?
- Does it still hold with the service scaled to more replicas?

## 4. Crash recovery and restart

When: stateful services, background jobs, multi-step operations, in-memory state, external side
effects.

- If the process dies at any line — between two writes, or after an external call but before
  recording it — what state is left, and does the next start repair it?
- Is a crash logged, and does it stop the service? A background worker, task, or thread that dies
  must stop or restart the whole service — something awaits or supervises it — so the service
  never runs on without that loop while looking healthy.
- Does anything that must survive a restart live only in memory: counters, nonces, queues, or a
  cache that is the source of truth?
- On startup, does the service reconcile in-flight work from persisted state (and from any
  external system it acts on) instead of starting blank or repeating it?
- Is each side effect recorded before, or atomically with, its trigger, so a restart neither loses
  nor duplicates it?
- Can work stuck in an intermediate state be detected and resumed or failed, with a timeout?
- Does shutdown drain or checkpoint in-flight work on `SIGTERM`, and does the service come back
  from an out-of-memory kill or restart without an operator when the state it finds is consistent?
- When the state it finds is inconsistent — persisted data that contradicts itself or the external
  system of record — does it raise a Critical alert and freeze without changing anything, or fail
  closed where continuing could act on diverged state, for an operator to repair? It never guesses
  a repair.

## 5. Persistence, rollout, and compatibility

When: schema, migrations, stored formats, config, public APIs, message or wire formats, deploy.

- Are writes that must be atomic in one transaction, with constraints that enforce the invariant?
- Does a migration work while old and new code run side by side, and does it ship through
  `MIGRATION_PATH` rather than a manual step?
- Is a deploy ordering required, and is it stated in the PR?
- Is rollback safe, including for data the new version already wrote?
- Do public API, WebSocket, event, or stored-format changes stay compatible for existing
  consumers?
- Are new env vars, secrets, and config values set for every environment, with safe defaults?

## 6. Money and precision

When: amounts, prices, rates, fees, balances, share calculations.

- Are values decimal or integer units, never binary floats?
- Is every rounding mode and direction explicit and in the system's favor where it matters?
- Are overflow, underflow, and division by zero impossible or handled?
- Are units and decimal places converted consistently at every boundary?

## 7. Blockchain data

When: node RPC reads, events, receipts, transaction submission.

- Are reorgs and confirmation depth handled?
- When reads from more than one node or provider are compared, are they taken at the same block,
  and does an absence carry the answering node's height before it counts as "not there"?
- Are nonces, gas, and resubmission safe under retries, concurrent senders, and restarts?

## 8. Errors

- Is every error handled or propagated with context, never swallowed?
- Does every external call have a timeout, and a bounded retry?
- Is each new failure visible: a log at the right level, a metric, and an alert where an operator
  must act?

## 9. Observability

When: new or changed logging, units of work, task or service boundaries, alerts.

- Does it follow the project's observability guide, `OBSERVABILITY_GUIDE`, when there is one?
- Does each unit of work — a request, a message, a job run, an operation — get one correlation
  scope carrying its id, so every line beneath it carries that id? Long-running loops get none.
- Does the id follow the work through every boundary — into spawned tasks, through queues and
  messages, and into calls to other services? A unit of work started from another records the
  originating id in a role field, so the chain stays joinable.
- Is every id in its own field — the subject's under `id`, other entities under role names
  (`order_id`, `job_id`) — and never only inside a message string?
- Are levels right: `info` for a state transition of an identifiable thing, `debug` for per-step
  detail and "why nothing happened", `trace` for mechanical echoes with no decision?
- Can a failure be diagnosed from the logs alone? Following one id shows every step the unit took —
  its inputs, each decision and its reason, the external calls and their results.
- Is each alert fired inside the unit's scope, so it carries the same ids as the logs? Is its
  severity right — Critical only when the system is blocked and an operator must act now, Warning
  for anything that can wait — and does it fire once per episode rather than on every check?
- Are logs free of secrets and personal data?

## 10. Security and trust boundaries

When: user, API, external-system, or config input; auth; secrets; crypto; external calls.

- Is authorization checked where the action happens, not only upstream?
- Is input validated at the boundary it crosses, and when a check fails, does the code refuse to
  act — alert, log, change nothing — rather than "recover" on data it just proved wrong?
- Are secrets kept out of code, logs, state, and committed files?
- Is all crypto from vetted libraries, never home-rolled?

## 11. Performance and resources

When: hot paths, loops over external data, queries, caches, queues.

- Any unbounded loop, query, memory growth, or queue?
- Any N+1 query or missing index for a new query shape?
- Any blocking call on an async path?

## 12. Tests

- **They test what they claim.** A test's name matches its assertions, and each assertion would
  fail if the behavior broke. No tautologies, no asserting that a mock returned what it was told,
  no bare success or no-exception check where the value matters.
- **Mutation-checked when the tests run anyway.** For a new or changed test, make one temporary
  edit to the code under test that breaks exactly what the test's name claims, and run it: that
  test must fail and nothing unrelated should. Undo the edit by hand, never with `git checkout` or
  `git restore`, which also discard uncommitted work. A test that stays green is a defect. Without
  a test run, ask: if this line were inverted, which assertion would fail?
- **They cover the changed branches.** The success path, every new or changed error path,
  boundaries (zero, empty, maximum, off-by-one), missing or null inputs, retries and timeouts,
  and every failure mode sections 2–11 found — including concurrent callers and a restart from
  persisted state where the behavior is testable.
- **Right level.** Unit tests for logic; integration tests where the database, network, or
  external-system boundary is the behavior. End-to-end coverage is the operator's call.
- **Deterministic.** No sleeps for synchronization; time, randomness, and network are controlled;
  no dependence on test order or on shared global state.
- **Regression.** A bug fix carries a test that fails without the fix.
- **Not weakened.** A deleted, skipped, or ignored test, a loosened assertion, or a widened
  tolerance is a defect unless the PR justifies it.
- **Realistic data.** Amounts use production precision; fixtures match real shapes.
- **Actually runs.** The tests sit in a CI lane that runs for these paths.

## 13. Code quality and domain skills

- Does the change follow every `*-implementation` skill for the files it touches — each one listed
  in `IMPLEMENTATION_SKILLS` and in the [AGENTS.md](../AGENTS.md) index?
- Does it add process-global mutable state — statics, singletons, a library's default registry?
  That is a hidden dependency: it breaks test isolation and allows only one instance per process.
  The language's `*-implementation` skill lists the exceptions.
- Clean and idiomatic, with no dead code, unused dependencies, or leftover debugging the change
  created.
- Names come from the design or ticket vocabulary, spelled out (`pending_refunds`, not `items` or
  `data`). A constant or variable whose name no longer describes its use gets a new name, not a
  new use.
- Comments are terse and describe current behavior ([code-comments](code-comments.md)).

## 14. Docs and indexes

- Are the nearest `AGENTS.md`, README, and runbooks updated for changed commands, paths, env vars,
  or behavior ([keep-agents-index-in-sync](keep-agents-index-in-sync.md))?

## 15. Workspace rules

- Does the change follow every rule and skill [AGENTS.md](../AGENTS.md) selects for it?
