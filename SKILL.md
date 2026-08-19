---
name: gpt-auditor
description: Use when the user explicitly invokes gpt-auditor, supplies an architect plan for adversarial review before coding, asks for a Claude–GPT debate, or a coding/product task has high-cost architecture, data-model, migration, auth, public-contract, irreversible, high-blast-radius, or unresolved root-cause risk. Do not use automatically for routine low-risk edits.
---

# [GPT] gpt-auditor

## [HUMAN] When to use

Use for explicit requests such as `use gpt-auditor`, `review this plan with Claude and GPT`, or `run the Claude-GPT debate`, and for risky work where a wrong decision would be expensive.

The normal workflow begins **after an architect has already framed the problem and produced a plan**. The user may paste that plan, attach `plan.md`, or otherwise provide it in the current context. If no usable architect plan is present, ask for it before touching Claude; do not silently generate a replacement plan.

Skip automatic activation for routine copy changes, obvious one-file fixes, formatting, simple CSS, or other low-risk execution where architecture is already settled. Once explicitly invoked, however, keep the minimum-three-challenge protocol.

## [GPT] Role architecture

Treat roles separately from model brands:

- **Architect / lock owner** — owns the starting plan, revisions, and final architecture lock.
- **Independent challenger** — adversarially tests the plan for three required challenge rounds.
- **Orchestrator / delivery auditor** — controls state, transport, execution gate, verification, skeptical audit, fixes, regression, and completion.
- **Execution backend** — performs post-lock implementation; selected independently from the architect.

Current default role profile:

- Architect / lock owner: **Claude Opus 5 Max on claude.com**.
- Independent challenger + orchestrator/auditor: **GPT-5.6** in the current host.
- Execution backend: user chooses **GPT/Codex** or **Claude/Claude Code** for every run.

Do not ask extra role questions by default. If the user explicitly requests a custom/reverse profile (for example a future GPT architect with Claude as challenger), record that role map explicitly and preserve the same challenge-count, lock-schema, verification, audit, safety, and commit invariants. Model superiority assumptions must never be hard-coded into the architecture itself.

## [GPT] Mandatory startup gate

Before inspecting, searching, opening, creating, or binding any Claude chat, collect **both** choices in one compact question:

1. **Claude session** — `new` or `same-name existing`.
2. **Execution backend** — `GPT/Codex` or `Claude/Claude Code`.

Do not proceed until both are explicit. Do not infer either choice from prior runs.

For `same-name existing`, find exactly one Claude chat whose title exactly matches the current GPT/Codex session title. Zero or multiple exact matches are ambiguous: stop and ask the user for the intended thread; never guess. If the current host title is unavailable, ask for the exact title before any Claude discovery.

For `new`, after Round 0 creates a concrete Claude conversation, rename it to the **exact current GPT/Codex session title** and verify the readback before continuing.

Under the current default role profile, the bound Claude thread must be **Opus 5 Max**. Switch and verify before Round 0 if necessary; if model verification/switching is unavailable, stop before transmitting auditor content. A custom role profile must likewise verify the explicitly selected remote model before consuming its debate turns.

## [GPT] Non-negotiable invariants

1. The supplied architect plan is the debate baseline; Round 0 imports/anchors it and does not restart framing from scratch.
2. Once invoked, complete **at least 3 challenge rounds** before execution.
3. Rounds 4–5 are allowed only for unresolved evidence-backed blockers. There is no Round 6.
4. Consensus means blocking disagreements are resolved; preferences may remain.
5. Bind and control the exact remote debate thread. Never automate GPT's own composer in the default ChatGPT-host profile.
6. Every debate message uses visible sentinels: `[[AUDITOR round=N from=X]]` … `[[END round=N]]`.
7. Match sentinels by message role, never flat page text alone.
8. Never advance state until the preceding browser/action/state postcondition is verified.
9. Before sending a round, perform the role-scoped idempotency check so retries cannot double-post.
10. Read an unread answer before any reload/recovery action.
11. Redact secrets, credentials, tokens, keys, and private customer data before external model transport.
12. No user-mediated transport fallback. Missing adapter capability is a named blocker.
13. No mandatory human confirmation at lock; normal safety/approval controls still apply to sensitive actions.
14. Never silently switch execution backend after the user chooses one.
15. Do not claim completion without fresh verification evidence.
16. A new Claude session must have its concrete URL captured and exact requested title verified before debate proceeds beyond Round 0.
17. In a git repo, successful execution ends with one final auto-commit containing only the run-owned delta unless the current user explicitly disables commit for that run; never auto-push and never include pre-existing unrelated user changes.
18. Any change to `gpt-auditor/` itself is unreleasable until E2E-1 passes against the final post-fix files.
19. During debate, fresh repo/runtime/visual inspection is allowed only as targeted read-only evidence gathering; no implementation fix may occur before LOCK.

## [GPT] Required files

- `PROTOCOL.md` — durable state, imported-plan context, debate, lock, backend profiles, execution, escalation, audit, commit.
- `TRANSPORT.md` — remote-thread adapter contract, role-scoped parsing, idempotency, preflight, recovery/error handling.
- `TESTS.md` — end-to-end and scenario verification for the skill itself.

Read all three before starting a run.

## [GPT] Challenge evidence on demand

Challenge rounds are allowed to gather fresh evidence before composing the next challenge. When a disputed assumption can be verified from the live project, the challenger/orchestrator may inspect the repo and, for runnable products, visually/runtime-audit the current website or app so the challenge sent to the architect is grounded and precise.

Use this only when the evidence could materially change a blocker, architecture choice, acceptance criterion, or implementation prompt. Prefer the smallest useful read-only investigation: relevant source/tests/config, current DOM/UI, screenshots, browser/simulator interaction, console/runtime logs, network/API behavior, or data invariants. Do **not** run a broad audit every round merely because tools are available.

Pre-lock investigation is evidence gathering, not execution: do not edit implementation files, mutate production/user data, change architecture, or fix the problem before LOCK. A safe local runtime may be started only when needed to observe current behavior and when doing so does not create material side effects; record any runtime lifecycle/deviation evidence that later matters.

Summarize only the load-bearing observations in the challenge prompt with source/procedure and what they prove. If evidence contradicts the supplied plan, challenge the plan with that evidence rather than arguing from intuition.

## [GPT] Harness principles absorbed into delivery

This skill owns the load-bearing execution-harness concepts directly; separate harness-generator skills are not runtime dependencies.

Shared rules for either backend:

- repo-local artifacts are the durable source of truth; chat memory is not;
- inspect existing `AGENTS.md`, `CLAUDE.md`, `TASKS.md`, `TEST_CHECKLIST.md`, `QA_FINDINGS.md`, `BUG_LOG.md`, `DECISIONS.md`, `ARCHITECTURE.md`, `HANDOFF.md`, scripts, tests, and CI only as relevant;
- never overwrite existing harness/project-governance files silently; surface conflicts with the lock;
- use the **lightest load-bearing harness**: do not generate a full file suite for a small bounded task;
- for medium/long-running work, persist only the state needed to execute, verify, recover, and hand off truthfully;
- every app/website/feature lock must have an executable done-means contract: expected behavior, critical user flows, validation methods/commands, expected results, and explicit out-of-scope boundaries;
- no silent dependency or architecture expansion outside the lock; escalate when it changes a locked assumption/hard constraint;
- a clean build is evidence, not proof of product correctness; validate runnable critical flows as a user when reasonably possible;
- normalize mechanically knowable protocol conflicts before Round 0 so stale inherited instructions do not waste a challenge round; current explicit user instructions remain authoritative;
- every skeptical audit/scope/prohibited-path gate must use a complete canonical run-delta manifest that includes tracked changes, new/untracked files, and deletions; plain `git diff` is insufficient;
- final verification distinguishes current runtime liveness from durability/lease/supervisor semantics, records `Execution deviations` (`NONE` when applicable), and never calls something `pre-existing` without baseline evidence;
- lock validation checks both testability and objective semantic correctness of expected results; obvious defects get a targeted repair, not a new debate round;
- keep transport/log output bounded: one canonical plan/context artifact, targeted reads/diffs, no routine raw Base64 dumps, and no redundant capability/schema rediscovery when nothing changed.

## [GPT] GPT/Codex execution profile

When `GPT/Codex` is selected:

- prefer a concise `AGENTS.md` as a routing map when one exists or is genuinely needed; link to deeper source-of-truth docs instead of duplicating them;
- prefer mechanical enforcement over prose reminders: tests, lint, typecheck, scripts, structural checks, CI, and explicit invariants with commands;
- for bugs, reproduce first, isolate root cause, fix minimally, then verify;
- use the smallest meaningful pre-edit check and broader post-edit/regression validation;
- after coding changes, run the relevant build/test/lint/typecheck commands;
- if execution exposes a reusable missing capability (fixture, check, observability path, script, test), persist it only when it will materially help future runs.

## [GPT] Claude/Claude Code execution profile

When `Claude/Claude Code` is selected:

- prefer a concise `CLAUDE.md` project contract when one exists or is genuinely needed;
- keep the locked done-means contract visible to the executor; do not reopen architecture during implementation;
- for long-running work, preserve concise truthful continuation state (for example `HANDOFF.md`) when it is load-bearing;
- use a build → skeptical QA → fix → QA/regression loop until the locked criteria pass;
- actively test product depth, functionality, UX clarity, completeness vs stubs/fake integrations, and runtime behavior with browser/simulator/MCP/log/network/API/data evidence where available;
- keep ceremony proportional to risk; simple work must not inherit unnecessary phase/sprint overhead.

## [GPT] Plan lock and delivery authority

Under the default role profile, Claude/Opus owns architecture revisions and the final lock; GPT owns the execution gate. A lock is accepted only after Round 3 or later, passes the nine-field schema, and has testable acceptance criteria.

The locked plan is the highest-priority task-specific execution contract for this run. Existing project harness docs remain evidence/context but may be stale; do not let stale guidance silently override the lock. Synchronize durable project docs after verified delivery only when they are in declared scope.

GPT may reject a valid-format lock only for an evidence-backed correctness, safety, data-integrity, hard-constraint, or executability blocker. Preference is not a veto.

## [GPT] After lock

Apply the chosen execution backend; never silently fall back to the other backend. Persist concrete execution provenance separately from the coarse family choice: `chatgpt_localops`, `codex`, or `claude_code` (plus provider family). If the selected backend lacks required execution capabilities, finish the debate if possible but block before the first implementation write and name the missing capability.

Execute in bounded increments, verify meaningful changes, run an artifact-only skeptical audit, fix P0/P1 findings, rerun targeted regression plus every locked criterion, then create one final run-owned commit when safe. Routine implementation problems stay in delivery; re-enter the architect only when new evidence invalidates a locked architectural assumption or satisfying the lock would violate a hard constraint.

When the target being modified is `gpt-auditor` itself, final files must pass E2E-1 before release/commit unless the current user explicitly directs an immediate commit/release despite that project-local gate.

## [GPT] User-facing completion report

After a successful run, keep the chat report compact and evidence-backed. Do not dump internal state files or the full debate. Report exactly these seven sections in this order: **Decision → Execution → Verification → Runtime → Deviations → Remaining → Git**. Include a short `Auditor` line only when debate materially changed the plan or when the user asks for round details. Full `locked_plan.md`, `final_verification.md`, and `run_delta_manifest.md` remain external audit artifacts for drill-down.
