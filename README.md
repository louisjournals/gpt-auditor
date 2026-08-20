# gpt-auditor

`gpt-auditor` is a cross-model architecture and delivery workflow for high-risk coding and product changes.

It starts from an **existing architect plan**, challenges that plan across multiple rounds, obtains a locked implementation contract, executes through a selected coding backend, verifies the result, runs an independent skeptical audit, fixes blocking findings, regresses the final state, and reports the outcome with evidence.

## Default workflow

```text
Initial audit / existing architect input
        ↓
Operator review first when UX/IA/affordance is material
        ↓
Pattern inventory before scope freeze
        ↓
Refreshed architect plan
        ↓
Round 0 — import / synchronize the plan
        ↓
Objective-correctness challenge #1 → Claude/Opus revision
        ↓
Objective-correctness challenge #2 → Claude/Opus revision
        ↓
Objective-correctness challenge #3 → Claude/Opus LOCK
        ↓
Persist `.gpt-auditor/LOCKED_PLAN.md` + hash
        ↓
Short execution summary → ONE USER APPROVAL
        ↓
Selected executor re-reads approved repo plan
        ↓
Execute + evidence matrix
        ↓
Architect Completion Gate → ALL PASS
        ↓
Deep skeptical audit → Fix P0/P1 → Regression
        ↓
Pre-commit delivery report (informational)
        ↓
One safe run-owned commit
        ↓
Concise completion report
```

Default role profile:

- **Architect / lock owner:** Claude Opus 5 Max on claude.com
- **Independent challenger + orchestrator/auditor:** GPT-5.6
- **Execution backend:** chosen per run — Codex, Claude Code, or another explicitly verified coding agent

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

> **Important:** Full automation requires an orchestrator host that can run the challenger/auditor role, an authenticated Claude session, and a real coding-agent execution path for the target workspace (Codex, Claude Code, or another explicitly verified agent); a plain chat-only environment cannot run the complete workflow.

The default profile expects:

1. A supplied architect plan. `gpt-auditor` does not replace the initial planning step.
2. An orchestrator host capable of running the GPT challenger/auditor role.
3. Browser automation that can control an exact Claude thread in the background, inspect message roles, fill/send prompts, and read completed replies without relying on the active tab.
4. An authenticated Claude session with **Opus 5 Max** available for the default architect role.
5. A verified execution backend for the target workspace:
   - Codex,
   - Claude Code, or
   - another explicitly identified coding agent that can satisfy the same workspace and verification postconditions.
6. Git when the target workspace is a repository.

The transport contract is capability-based rather than tied to a private tool stack. Any host may implement it if it can satisfy the required browser, workspace, verification, state, and git postconditions.

## Starting a run

Provide the architect plan, then invoke `gpt-auditor`.

Every run asks two startup choices:

- Claude session: `new` or `same-name existing`
- Executor: `Codex`, `Claude Code`, or `other verified` (with a generic executor label)

Example:

```text
Use gpt-auditor on this plan.
Claude session: new
Executor: Codex

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
- At least **three challenge rounds** are required for the objective-correctness track. Pure taste/perception work is not forced through adversarial rounds.
- Rounds 4–5 are blocker-only for unresolved objective blockers; there is no Round 6.
- Claude/architect owns the final architecture lock in the default profile.
- GPT may reject a valid lock only for evidence-backed correctness, safety, data-integrity, hard-constraint, or executability blockers — not preference.
- Every lock includes explicit acceptance criteria and a done-means contract for runnable product work. Each criterion declares `Authority: MACHINE`, `OPERATOR`, or `MIXED`; operator-authority criteria cannot be verified by model/DOM evidence alone.
- Scope constrains what may be changed, not what may be inspected. A defect instance triggers a read-only sibling-pattern scan before modification scope is frozen.

## Delivery and verification

LOCK freezes the architecture but does not authorize implementation. After lock validation, the auditor records the repo baseline, persists the exact plan to `.gpt-auditor/LOCKED_PLAN.md`, hashes it, resolves any known pre-approval instability, verifies the selected backend, summarizes the upcoming work in a few bullets, and asks for the **one normal execution approval**. That approval is anchored to the displayed root hash plus a narrow non-material-repair envelope.

Only after approval may the selected executor begin implementation. The executor re-reads the current repo locked-plan artifact on start/resume/handoff and treats it—not remembered session chat—as the task-specific execution contract. Evidence-backed non-material repairs inside the approved envelope are recorded as a hash repair chain and do **not** trigger another generic `approve` prompt. A truly material change stops for a specific user change decision rather than approval spam.

When perceptual UX/IA/affordance work is in scope, execution is intentionally iterative: make a small coherent batch, let the operator use/look at it, record `VERIFIED` / `NOT VERIFIED` / `FAILED`, then continue. These are review checkpoints, not repeated execution approvals. Passing every machine criterion does not mean the experience is accepted; outstanding operator-authority criteria produce `TECHNICALLY COMPLETE — OPERATOR REVIEW REQUIRED`, not `DONE`.

When the executor believes the **pre-audit implementation** is complete, it creates an evidence matrix for every locked implementation step and acceptance criterion and sends that matrix plus the current locked plan to the architect/lock owner. Pre-audit obligations must already be proven; implementation steps or criteria that only occur during skeptical audit/regression or the final commit gate are explicitly deferred to that later gate, not falsely completed/passed. Skeptical audit cannot begin until the architect checks every item line by line and returns **ALL PASS** for execution completeness.

The skeptical audit is intentionally deep: it actively tries to falsify completion across lock conformance, functional/runtime behavior, data/auth/permission integrity, regression risk, UX/IA/affordance quality when relevant, implementation depth/wiring, complete run-delta hygiene, and verification-evidence quality. For runnable products, real user-flow/runtime checks are preferred over relying on build/test/lint/typecheck alone. P0/P1 findings are fixed and regressed before delivery.

Immediately before the single final commit, the user sees a concise informational report of **all run-owned changes, Architect Completion Gate result, audit findings, fixes, verification/deviations, remaining P2/known issues, and exact commit scope**. This is not another approval gate; the normal run still has one execution approval.

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

`TESTS.md` defines scenario/document verification plus **E2E-1**, an optional real three-round orchestration diagnostic using a disposable git fixture. E2E-1 is useful for deep transport/state-machine regression testing, but it is not a mandatory release gate or automatically required when `gpt-auditor` modifies itself.

## License

MIT. See `LICENSE`.
