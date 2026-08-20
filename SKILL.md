---
name: gpt-auditor
description: Use when the user explicitly invokes gpt-auditor, supplies an architect plan for adversarial review before coding, asks for a Claude–GPT debate, or a coding/product task has high-cost architecture, data-model, migration, auth, public-contract, irreversible, high-blast-radius, or unresolved root-cause risk. Do not use automatically for routine low-risk edits.
---

# [GPT] gpt-auditor

## [HUMAN] When to use

Use for explicit requests such as `use gpt-auditor`, `review this plan with Claude and GPT`, or `run the Claude-GPT debate`, and for risky work where a wrong decision would be expensive.

The normal workflow begins **after an architect has already framed the problem and produced a plan**. The user may paste that plan, attach `plan.md`, or otherwise provide it in the current context. If no usable architect plan is present, ask for it before touching Claude; do not silently generate a replacement plan.

For work that materially involves UX, information architecture, visual hierarchy, affordance, or subjective usability, the operator must enter the loop **before the plan is frozen**. If the supplied plan was produced without a fresh operator review of the current product, do not start debate yet: ask the operator to inspect the product normally, capture what they actually notice/try/misread, then obtain a refreshed architect plan that incorporates that evidence. AI disagreement cannot settle a perceptual question whose authority is the operator.

Skip automatic activation for routine copy changes, obvious one-file fixes, formatting, simple CSS, or other low-risk execution where architecture is already settled. For a **pure experience/taste task with no objective correctness, safety, data, architecture, or executability dispute**, skip the multi-round adversarial debate and use the small-batch operator loop instead. For mixed tasks, debate only the objective correctness track; operator experience findings are constraints/evidence, not topics for AI preference debate.

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
2. For the **objective correctness track**, complete **at least 3 challenge rounds** before execution. Pure perceptual/experience work does not consume challenge rounds merely to debate taste.
3. Rounds 4–5 are allowed only for unresolved evidence-backed objective blockers. There is no Round 6.
4. Consensus means objective blocking disagreements are resolved. Operator-authority perceptual judgments are not settled by model consensus.
5. Bind and control the exact remote debate thread. Claude browser transport is **background-first**: target the exact thread/URL with DOM-aware operations and do not steal foreground focus when equivalent background control exists. Never automate GPT's own composer in the default ChatGPT-host profile.
6. Every debate message uses visible sentinels: `[[AUDITOR round=N from=X]]` … `[[END round=N]]`.
7. Match sentinels by message role, never flat page text alone.
8. Never advance state until the preceding browser/action/state postcondition is verified.
9. Before sending a round, perform the role-scoped idempotency check so retries cannot double-post.
10. Read an unread answer before any reload/recovery action.
11. Redact secrets, credentials, tokens, keys, and private customer data before external model transport.
12. No user-mediated transport fallback. Missing adapter capability is a named blocker.
13. **LOCK is not execution permission.** After a valid lock, persist the exact locked plan inside the target repo, hash it, resolve known pre-approval instability, summarize upcoming execution compactly, and wait for the **single normal execution approval** before any implementation write.
14. That approval is anchored to the displayed root hash plus a narrow non-material repair envelope. Evidence-backed repairs inside that envelope update the current plan hash/repair chain without another generic approval prompt; a material change stops for a specific user change decision.
15. During execution, the current approved repo locked-plan artifact + repair chain is the task-specific execution authority; session chat/history is non-authoritative context. Re-read it on executor handoff/resume rather than reconstructing requirements from conversation memory.
16. Never silently switch execution backend after the user chooses one.
17. **Architect Completion Gate is mandatory before skeptical audit.** The executor must enumerate every locked implementation step and every acceptance criterion with phase-aware status/evidence. The architect/lock owner returns `ALL PASS` only when all pre-audit obligations are executed/proven and every later audit/commit **step or criterion** is explicitly deferred to its correct protocol gate; missing, unproven, silently omitted, or mis-deferred items go back to the executor until closed. `ALL PASS` is execution-completeness permission to start audit, not a claim that post-audit work already happened or post-audit criteria already passed. The architect may not promote an operator-authority criterion to verified without recorded operator evidence.
18. Skeptical audit must explicitly cover the eight defined audit dimensions, include a pattern-escape/sibling-instance check, and actively try to falsify completion; no clean-build shortcut.
19. Do not claim completion without fresh verification evidence.
20. A new Claude session must have its concrete URL captured and exact requested title verified before debate proceeds beyond Round 0.
21. In a git repo, successful execution ends with one final auto-commit containing only the run-owned delta unless the current user explicitly disables commit for that run; immediately before that commit, surface the informational pre-commit delivery report (all changes, completion-gate result, audit findings, fixes, verification/deviations, exact commit scope). Do not ask for another approval and never auto-push or include pre-existing unrelated user changes.
22. Any change to `gpt-auditor/` itself is unreleasable until E2E-1 passes against the final post-fix files.
23. During debate, fresh repo/runtime/visual inspection is allowed only as targeted read-only evidence gathering; no implementation fix may occur before LOCK.
24. **Scope limits changes, not observation.** When one defect instance is found, inspect the full relevant pattern/sibling surface before freezing modification scope; do not let a narrow fix scope hide repeated instances.
25. Acceptance criteria must declare verification authority: `MACHINE`, `OPERATOR`, or `MIXED`. Operator-authority evidence cannot be substituted by DOM/style/test evidence. Use `VERIFIED`, `NOT VERIFIED`, or `FAILED`; never infer `VERIFIED`.
26. For experience work, execute in small coherent batches (normally 1–3 perceptual issues or one repeated pattern), then ask the operator to look/use the product before the next batch. This is review/feedback, not another execution approval.
27. Passing all machine criteria is not sufficient for completion when operator-authority criteria remain. Report `TECHNICALLY COMPLETE — OPERATOR REVIEW REQUIRED` until the operator verifies them.

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
- pattern completeness precedes modification scope: one observed defect instance triggers read-only inspection of sibling instances/components/pages before deciding what to change; scope fences constrain writes, never where the auditor is allowed to look;
- a clean build is evidence, not proof of product correctness; validate runnable critical flows as a user when reasonably possible;
- normalize mechanically knowable protocol conflicts before Round 0 so stale inherited instructions do not waste a challenge round; current explicit user instructions remain authoritative;
- every skeptical audit/scope/prohibited-path gate must use a complete canonical run-delta manifest that includes tracked changes, new/untracked files, and deletions; plain `git diff` is insufficient;
- final verification distinguishes current runtime liveness from durability/lease/supervisor semantics, records `Execution deviations` (`NONE` when applicable), and never calls something `pre-existing` without baseline evidence;
- lock validation checks both testability and objective semantic correctness of expected results; every acceptance criterion declares `Authority: MACHINE|OPERATOR|MIXED`; obvious defects get a targeted repair, not a new debate round;
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
5. ask for the **single normal execution approval**, anchored to the displayed root lock hash plus the narrow non-material repair envelope;
6. only after approval, re-read and re-hash the repo artifact, confirm it still matches both the approved root/current hash, then begin implementation.

During execution, use the repo locked-plan artifact as the task-specific authority rather than session chat/history. On executor handoff, context restart, or resume, re-read that artifact before continuing. The normal run has **one execution-approval prompt only**. That approval is anchored to the displayed root lock hash plus a narrow non-material-repair envelope. An evidence-backed post-approval repair may continue without another approval only when it preserves the user goal/product direction, does not add permissions/OAuth scopes/destructive actions or broaden external side effects, does not switch executor, and does not weaken the locked acceptance burden; record the old/new hashes and repair evidence. A material change outside that envelope stops for a specific user change decision instead of repeatedly asking `approve`.

After the executor believes the pre-audit implementation is complete, it must build an evidence-backed completion matrix covering **every implementation step and every acceptance criterion**, then send the current repo locked plan plus that matrix to the architect/lock owner for a line-by-line **Architect Completion Gate**. Obligations due before audit must already be executed/proven. Implementation steps or criteria that belong only to skeptical audit/regression or the final commit gate are explicitly marked `DEFERRED_TO_AUDIT` / `DEFERRED_TO_COMMIT` and remain incomplete/unverified until that later gate. The architect returns `ALL PASS` only when nothing required before audit is missing and every future-gated step/criterion is correctly accounted for; otherwise it returns the exact missing/unproven/mis-deferred items. The executor closes those gaps and repeats this gate. **Skeptical audit may not start until the architect returns `ALL PASS`.**

Then run a deep artifact/runtime skeptical audit that actively tries to falsify completion across lock conformance, functional/runtime behavior, data/auth/security integrity, regression risk, UX/IA/affordance quality when relevant, implementation depth/wiring, complete run-delta scope, and verification-evidence quality. Fix P0/P1 findings, rerun targeted regression plus every locked criterion, and prepare a concise **pre-commit delivery report** for the user listing all run-owned changes, architect-completion result, audit findings, fixes, verification/deviations, and exact commit scope. This report is informational, **not another approval gate**. Then create one final run-owned commit when safe. Routine implementation problems stay in delivery; re-enter the architect only when new evidence invalidates a locked architectural assumption or satisfying the lock would violate a hard constraint.

When the target being modified is `gpt-auditor` itself, final files must pass E2E-1 before release/commit unless the current user explicitly directs an immediate commit/release despite that project-local gate.

## [GPT] User-facing completion report

After a successful run, keep the chat report compact and evidence-backed. Do not dump internal state files or the full debate. Report exactly these seven sections in this order: **Decision → Execution → Verification → Runtime → Deviations → Remaining → Git**. Include a short `Auditor` line only when debate materially changed the plan or when the user asks for round details. Full `locked_plan.md`, `final_verification.md`, and `run_delta_manifest.md` remain external audit artifacts for drill-down.
