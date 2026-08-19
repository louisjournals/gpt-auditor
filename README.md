# gpt-auditor

`gpt-auditor` is a cross-model architecture and delivery workflow for high-risk coding and product changes.

It starts from an **existing architect plan**, challenges that plan across multiple rounds, obtains a locked implementation contract, executes through a selected coding backend, verifies the result, runs an independent skeptical audit, fixes blocking findings, regresses the final state, and reports the outcome with evidence.

## Default workflow

```text
Existing architect plan
        ↓
Round 0 — import / synchronize the plan
        ↓
GPT challenge #1 → Claude/Opus revision
        ↓
GPT challenge #2 → Claude/Opus revision
        ↓
GPT challenge #3 → Claude/Opus LOCK
        ↓
Selected executor
        ↓
Execute → Verify → Independent audit → Fix → Regression
        ↓
One safe run-owned commit
        ↓
Concise completion report
```

Default role profile:

- **Architect / lock owner:** Claude Opus 5 Max on claude.com
- **Independent challenger + orchestrator/auditor:** GPT-5.6
- **Execution backend:** chosen per run — GPT/Codex or Claude/Claude Code

The role architecture is intentionally separate from vendor names so future model combinations can be substituted without changing the workflow invariants.

## What it is for

Use it when a wrong decision would be expensive, for example:

- architecture or data-model changes
- migrations
- authentication / authorization
- public contracts or APIs
- irreversible or high-blast-radius changes
- unclear root cause
- product work where static checks alone cannot prove correctness

Do not use it automatically for routine copy edits, obvious one-file fixes, formatting, or trivial CSS changes.

## Requirements

The default profile expects:

1. A supplied architect plan. `gpt-auditor` does not replace the initial planning step.
2. A ChatGPT/GPT host capable of running the challenger/orchestrator role.
3. Browser automation that can control an exact Claude thread in the background, inspect message roles, fill/send prompts, and read completed replies without relying on the active tab.
4. An authenticated Claude session with **Opus 5 Max** available for the default architect role.
5. A verified execution backend for the target workspace:
   - GPT/Codex-compatible local execution, or
   - Claude Code.
6. Git when the target workspace is a repository.

The included transport contract is written around LocalOps-style browser and local execution capabilities, but equivalent tooling can implement the same postconditions.

## Starting a run

Provide the architect plan, then invoke `gpt-auditor`.

Every run asks two startup choices:

- Claude session: `new` or `same-name existing`
- Executor: `GPT/Codex` or `Claude/Claude Code`

Example:

```text
Use gpt-auditor on this plan.
Claude session: new
Executor: GPT/Codex

<plan>
...
</plan>
```

## Challenge evidence

Challenge rounds may inspect the live project before questioning the architect when fresh evidence would materially improve the challenge.

That can include targeted, read-only inspection of:

- source code, tests, config, schemas, migrations, and project docs
- current website/app UI
- DOM / screenshots / accessibility state
- browser or simulator behavior
- console/runtime logs
- network/API behavior
- read-only data invariants

This is **evidence-on-demand**, not a mandatory full audit every round. Pre-lock investigation may not implement the fix.

## Debate rules

- The supplied plan is the Round-0 baseline.
- At least **three challenge rounds** are required after invocation.
- Rounds 4–5 are blocker-only; there is no Round 6.
- Claude/architect owns the final architecture lock in the default profile.
- GPT may reject a valid lock only for evidence-backed correctness, safety, data-integrity, hard-constraint, or executability blockers — not preference.
- Every lock includes explicit acceptance criteria and a done-means contract for runnable product work.

## Delivery and verification

After lock, the selected executor performs implementation. The orchestrator then verifies against artifacts rather than implementation intent.

For runnable products, verification can include real user-flow/runtime checks in addition to build/test/lint/typecheck evidence. The skeptical audit explicitly looks for broken critical flows, shallow/stubbed functionality, fake integrations, data errors, UX failures, and regressions hidden by passing static checks.

A canonical run-delta manifest accounts for tracked edits, staged changes, new/untracked files, deletions/renames, and safely separable mixed-file changes.

## Completion report

The normal user-facing report is intentionally compact:

```text
Decision
Execution
Verification
Runtime
Deviations
Remaining
Git
```

Detailed recovery/audit artifacts remain outside the target repository unless the locked scope explicitly requires otherwise.

## Repository files

- `SKILL.md` — entry point and high-level rules
- `PROTOCOL.md` — durable state machine, debate, lock, execution, audit, commit, reporting
- `TRANSPORT.md` — Claude thread transport, idempotency, background DOM handling, recovery
- `TESTS.md` — E2E and scenario verification

## Verification

`TESTS.md` defines **E2E-1**, a real three-round orchestration test using a disposable git fixture. It verifies actual Claude transport, state transitions, lock creation, executor delivery, audit/regression, and one safe final fixture commit with no push.

## License

MIT. See `LICENSE`.
