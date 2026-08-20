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
- Execution backend: user chooses **Codex**, **Claude Code**, or an explicitly identified **other verified coding agent** for every run.

Do not ask extra role questions by default. If the user explicitly requests a custom/reverse profile (for example a future GPT architect with Claude as challenger), record that role map explicitly and preserve the same challenge-count, lock-schema, verification, audit, safety, and commit invariants. Model superiority assumptions must never be hard-coded into the architecture itself.

## [GPT] Mandatory startup gate

Before inspecting, searching, opening, creating, or binding any Claude chat, collect **both** choices in one compact question:

1. **Claude session** — `new` or `same-name existing`.
2. **Execution backend** — `Codex`, `Claude Code`, or an explicitly identified `other verified` coding agent.

Do not proceed until both are explicit. Do not infer either choice from prior runs. If `other verified` is selected, also record a human-readable executor label and require the same workspace/capability preflight as the named backends; public docs/reports describe it generically and do not expose private/local-only tool names.

For `same-name existing`, find exactly one Claude chat whose title exactly matches the current orchestrator session title. Zero or multiple exact matches are ambiguous: stop and ask the user for the intended thread; never guess. If the current host title is unavailable, ask for the exact title before any Claude discovery.

For `new`, after Round 0 creates a concrete Claude conversation, rename it to the **exact current orchestrator session title** and verify the readback before continuing.

Under the current default role profile, the bound Claude thread must be **Opus 5 Max**. Switch and verify before Round 0 if necessary; if model verification/switching is unavailable, stop before transmitting auditor content. A custom role profile must likewise verify the explicitly selected remote model before consuming its debate turns.

## [GPT] Non-negotiable invariants

1. The supplied architect plan is the debate baseline; Round 0 imports/anchors it and does not restart framing from scratch.
2. Once invoked, complete **at least 3 challenge rounds** before execution.
3. Rounds 4–5 are allowed only for unresolved evidence-backed blockers. There is no Round 6.
4. Consensus means blocking disagreements are resolved; preferences may remain.
5. Bind and control the exact remote debate thread. Claude browser transport is **background-first**: target the exact thread/URL with DOM-aware operations and do not steal foreground focus when equivalent background control exists. Never automate GPT's own composer in the default ChatGPT-host profile.
6. Every debate message uses visible sentinels: `[[AUDITOR round=N from=X]]` … `[[END round=N]]`.
7. Match sentinels by message role, never flat page text alone.
8. Never advance state until the preceding browser/action/state postcondition is verified.
9. Before sending a round, perform the role-scoped idempotency check so retries cannot double-post.
10. Read an unread answer before any reload/recovery action.
11. Redact secrets, credentials, tokens, keys, and private customer data before external model transport.
12. No user-mediated transport fallback. Missing adapter capability is a named blocker.
13. **LOCK is not execution permission.** After a valid lock, persist the exact locked plan inside the target repo, hash that repo artifact, summarize the upcoming execution compactly, and wait for explicit user approval bound to that exact hash before any implementation write.
14. If the repo locked-plan artifact changes after approval, or a current user instruction would materially change its scope/steps/criteria, invalidate approval and require an updated lock artifact + summary + approval before continuing implementation.
15. During execution, the repo locked-plan artifact is the task-specific execution authority; session chat/history is non-authoritative context. Re-read the repo artifact on executor handoff/resume and follow it rather than reconstructing requirements from conversation memory.
16. Never silently switch execution backend after the user chooses one.
17. Do not claim completion without fresh verification evidence.
18. A new Claude session must have its concrete URL captured and exact requested title verified before debate proceeds beyond Round 0.
19. In a git repo, successful execution ends with one final auto-commit containing only the run-owned delta unless the current user explicitly disables commit for that run; never auto-push and never include pre-existing unrelated user changes.
20. Any change to `gpt-auditor/` itself is unreleasable until E2E-1 passes against the final post-fix files.
21. During debate, fresh repo/runtime/visual inspection is allowed only as targeted read-only evidence gathering; no implementation fix may occur before LOCK.

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

## [GPT] Codex execution profile

When `Codex` is selected:

- prefer a concise `AGENTS.md` as a routing map when one exists or is genuinely needed; link to deeper source-of-truth docs instead of duplicating them;
- prefer mechanical enforcement over prose reminders: tests, lint, typecheck, scripts, structural checks, CI, and explicit invariants with commands;
- for bugs, reproduce first, isolate root cause, fix minimally, then verify;
- use the smallest meaningful pre-edit check and broader post-edit/regression validation;
- after coding changes, run the relevant build/test/lint/typecheck commands;
- if execution exposes a reusable missing capability (fixture, check, observability path, script, test), persist it only when it will materially help future runs.

## [GPT] Claude Code execution profile

When `Claude Code` is selected:

- prefer a concise `CLAUDE.md` project contract when one exists or is genuinely needed;
- keep the locked done-means contract visible to the executor; do not reopen architecture during implementation;
- for long-running work, preserve concise truthful continuation state (for example `HANDOFF.md`) when it is load-bearing;
- use a build → skeptical QA → fix → QA/regression loop until the locked criteria pass;
- actively test product depth, functionality, UX clarity, completeness vs stubs/fake integrations, and runtime behavior with browser/simulator/MCP/log/network/API/data evidence where available;
- keep ceremony proportional to risk; simple work must not inherit unnecessary phase/sprint overhead.

## [GPT] Other verified execution profile

When an explicitly identified `other verified` coding agent is selected:

- require a generic executor label plus verified access to the exact workspace and equivalent file/edit, command/build/test, runtime-validation, state, and git/snapshot capabilities;
- re-read and hash-verify the approved repo locked-plan artifact on start/resume/handoff, exactly like the named backends;
- preserve the same scope, verification, skeptical-audit, approval, and one-final-commit invariants while following that agent's native project conventions;
- record `execution_environment=other_verified`; keep private/local-only tool names in local run state only when operationally necessary, never in public docs or release-facing summaries;
- never silently treat the orchestrator/chat host as this executor; it must be explicitly selected and capability-verified.

## [GPT] Plan lock and delivery authority

Under the default role profile, Claude/Opus owns architecture revisions and the final lock; GPT owns the execution gate. A lock is accepted only after Round 3 or later, passes the nine-field schema, and has testable acceptance criteria.

After validation, the exact lock must be persisted both in external run state and as the canonical repo execution artifact at `.gpt-auditor/LOCKED_PLAN.md` (or an explicitly user-selected equivalent repo path). The repo artifact is the highest-priority task-specific execution contract for this run. Existing project harness docs and session chat remain evidence/context but may be stale; do not let them silently override the repo lock. Synchronize other durable project docs after verified delivery only when they are in declared scope.

GPT may reject a valid-format lock only for an evidence-backed correctness, safety, data-integrity, hard-constraint, or executability blocker. Preference is not a veto.

## [GPT] After lock

A valid LOCK freezes architecture but does **not** authorize implementation. Before the first implementation write:

1. record the pre-write repo baseline;
2. persist the exact validated lock to `.gpt-auditor/LOCKED_PLAN.md` (or the explicitly selected equivalent repo path) and record its SHA-256 in run state;
3. complete the chosen backend capability preflight without implementation writes; never silently fall back to the other backend;
4. re-read the repo locked-plan artifact and give the user a compact 3–8 bullet execution summary covering what will change, material out-of-scope boundaries, the selected execution path in generic capability terms, and any approval-sensitive/destructive actions; do not expose private/local-only tool names in public-facing summaries;
5. ask for explicit execution approval bound to the displayed locked-plan hash;
6. only after approval, re-read and re-hash the repo artifact, confirm it still matches the approved hash, then begin implementation.

During execution, use the repo locked-plan artifact as the task-specific authority rather than session chat/history. On executor handoff, context restart, or resume, re-read that artifact before continuing. A material lock change or new user instruction that changes scope/steps/criteria invalidates prior approval; persist the revised lock, re-summarize, and obtain fresh approval before further implementation writes.

After approval, execute in bounded increments, verify meaningful changes, run an artifact-only skeptical audit, fix P0/P1 findings, rerun targeted regression plus every locked criterion, then create one final run-owned commit when safe. Routine implementation problems stay in delivery; re-enter the architect only when new evidence invalidates a locked architectural assumption or satisfying the lock would violate a hard constraint.

When the target being modified is `gpt-auditor` itself, final files must pass E2E-1 before release/commit unless the current user explicitly directs an immediate commit/release despite that project-local gate.

## [GPT] User-facing completion report

After a successful run, keep the chat report compact and evidence-backed. Do not dump internal state files or the full debate. Report exactly these seven sections in this order: **Decision → Execution → Verification → Runtime → Deviations → Remaining → Git**. Include a short `Auditor` line only when debate materially changed the plan or when the user asks for round details. Full `locked_plan.md`, `final_verification.md`, and `run_delta_manifest.md` remain external audit artifacts for drill-down.
