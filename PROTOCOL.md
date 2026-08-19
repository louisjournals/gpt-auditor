# [GPT] gpt-auditor Protocol

## [GPT] Durable run state

Store run state outside the target repo:

`~/.agents/state/gpt-auditor/<workspace-hash>/<run_id>/state.json`

Create a new run by default. If interrupted runs exist, offer them as resumable; never silently resume one. A different orchestrator host may take over only after recording an explicit host handoff in state.

Write state atomically where the host permits it (temporary file, then rename). Persist only after a transition is verified.

Minimum state fields:

```json
{
  "run_id": "...",
  "workspace": "...",
  "owner_host": "chatgpt-localops|codex|other",
  "phase": "startup_choice|preflight|context|debate|locked|backend_preflight|executing|verifying|auditing|fixing|regression|release_verifying|committing|done|blocked",
  "round": 1,
  "turn_state": "awaiting_startup_choices|ready_to_challenge|challenge_sent|claude_complete|response_processed|lock_pending",
  "role_profile": "default_claude_architect_gpt_challenger|custom",
  "roles": {
    "architect": "Claude Opus 5 Max",
    "challenger": "GPT-5.6",
    "lock_owner": "Claude Opus 5 Max",
    "orchestrator_auditor": "GPT-5.6"
  },
  "execution_backend": "gpt_codex|claude_claude_code",
  "execution_family": "openai|anthropic|other",
  "execution_environment": "chatgpt_localops|codex|claude_code|other_verified",
  "execution_backend_caps": [],
  "claude_session_mode": "new|same_name_existing",
  "requested_claude_title": "...",
  "claude_chat_url": "...",
  "claude_title_verified": false,
  "claude_model": "Opus 5 Max",
  "thread_lineage": [],
  "adapter_caps": [],
  "input_plan_path": "...",
  "input_plan_hash": "...",
  "protocol_normalization_path": "...",
  "context_packet_hash": "...",
  "context_packet_path": "...",
  "current_plan_path": "...",
  "last_claude_response_path": "...",
  "open_blockers": [],
  "resolved": [],
  "locked_plan_path": "...",
  "baseline": {"head_sha": null, "dirty_paths": [], "snapshot_manifest": [], "evidence": {}},
  "declared_scope": [],
  "scope_extensions": [],
  "touched_paths": [],
  "run_delta_manifest_path": "...",
  "execution_deviations": [],
  "runtime_assets": [],
  "final_verification_path": "...",
  "self_release_gate": {"required": false, "e2e_status": "not_required|pending|passed|failed"},
  "final_commit_sha": null,
  "last_verified_send": null,
  "last_read_hash": null,
  "error_state": null
}
```

## [GPT] Supplied architect plan

The normal run begins with a plan already produced by the architect outside this skill. Accept pasted text, an attached `plan.md`, or another current-context artifact. Preserve its substance faithfully.

Before debate transport:

1. verify a usable architect plan is present;
2. save the exact supplied plan to `input_plan.md`;
3. set `input_plan_path` and hash that exact stored artifact into `input_plan_hash`;
4. treat it as the starting current plan.

If no usable plan exists, ask the user to provide it. Do not synthesize a replacement plan and do not touch Claude. Round 0 is an import/anchor step, not a second framing/planning pass.

## [GPT] Pre-debate protocol normalization

Before Round 0, compare the supplied plan against **current explicit user instructions**, current repo facts, and active auditor invariants. Resolve mechanically knowable protocol contradictions before spending a challenge round on them.

Write `protocol_normalization.md` in the external run-state directory and set `protocol_normalization_path`. Record only material items, each with source and resolution. Examples: commit policy, execution backend, protected paths, destructive-action rules, required verification, or a stale report-only instruction carried in from an earlier context.

Authority order:

1. system/host safety constraints;
2. current explicit user instructions for this run;
3. locked architecture once valid;
4. active auditor defaults/invariants;
5. inherited/stale wording inside the supplied plan or project harness.

A current explicit user instruction such as `do not commit` overrides the auditor's default final auto-commit. In contrast, stale/inherited `no commit` text in a pasted plan does **not** silently disable the active delivery protocol when the user did not request that override. Surface the normalized execution contract in Round 0 so the architect cannot later lock an already-known protocol contradiction.

Normalization may resolve procedural contradictions, not architecture disagreements. Architecture/product choices remain for the three-round debate.

## [GPT] Startup-choice gate

Every new auditor run starts in `phase=startup_choice` and `turn_state=awaiting_startup_choices`. Before any Claude chat discovery or browser navigation, collect **both** choices in one compact gate:

1. Claude session: `new` or `same_name_existing`; and
2. execution backend: `gpt_codex` or `claude_claude_code`.

Do not inspect/search/open/create/bind Claude chats until both choices are explicit and the supplied architect plan is present. Persist each choice only after the user explicitly selects it. Never inherit a previous run's execution backend or session mode.

Resolve `requested_claude_title` from the exact current GPT/Codex session title for both session modes; if the host cannot read that title reliably, ask the user for the exact title before any Claude search or new-session creation.

For `same_name_existing`, require exactly one exact-title match. Zero matches or multiple exact matches are ambiguous and must return to the user for an exact thread identifier; do not guess.

For `new`, create the Claude thread only after the title is known. After verified Round 0 produces the concrete conversation URL, rename the Claude conversation to `requested_claude_title`, verify the exact title, set `claude_title_verified=true`, and only then continue waiting for/processing the architecture reply. A rename that cannot be verified is an adapter failure.

Under the default role profile, verify the chosen Claude thread is **Opus 5 Max** before entering `preflight`. If it is not, switch and verify. If model verification/switching fails, set `phase=blocked`; do not transmit Round 0. A custom/reverse role profile must record and verify its explicitly selected remote model instead of silently reusing the default.

## [GPT] Turn-state resume table

| `turn_state` | Resume action |
|---|---|
| `awaiting_startup_choices` | Ask for both Claude session mode and execution backend; if the architect plan is missing, request it too. Do not touch Claude until all are present. |
| `ready_to_challenge` | Compose round N; run role-scoped idempotency check before any send. |
| `challenge_sent` | Do not resend. Wait for the expected Claude response. |
| `claude_complete` | Re-read the stored assistant-role response and process it. |
| `response_processed` | Advance to the next round, or to lock validation if appropriate. |
| `lock_pending` | Re-run lock validation against the stored Claude response. |

`challenge_sent` may be written only after `verified_send` succeeds. If a send is ambiguous, state remains `ready_to_challenge`; recovery starts with the idempotency scan.

## [GPT] Recovery artifacts

Persist the evidence needed to resume without chat memory:

- save the exact supplied architect plan to `input_plan.md`, set `input_plan_path`, then hash it into `input_plan_hash` before any debate transmission;
- save pre-debate protocol decisions to `protocol_normalization.md` and set `protocol_normalization_path` before Round 0;
- save the **redacted** Round-0 packet to `context_packet.md`, set `context_packet_path`, then hash that exact stored packet into `context_packet_hash` before transmission;
- after every completed Claude response in the default role profile, save the exact assistant-role text to `messages/round-N-claude.md` and set `last_claude_response_path` **before** writing `turn_state=claude_complete`;
- after processing any nonterminal response that establishes/restates the current plan (Round 0, 1, 2, and blocker-resolution Round 4 when used), save the latest full current plan to `current_plan.md` and update `current_plan_path` before writing `response_processed`;
- after a valid lock, save the exact `=== LOCKED PLAN ===` block to `locked_plan.md`, set `locked_plan_path`, and only then set `phase=locked`.

A hash without the corresponding stored recovery artifact is insufficient for deterministic resume or continuation-thread rehydration.

## [GPT] Existing project harness context

When the target repo already contains persistent project-governance artifacts (`CLAUDE.md`, `AGENTS.md`, `TASKS.md`, `TEST_CHECKLIST.md`, `QA_FINDINGS.md`, `BUG_LOG.md`, `DECISIONS.md`, `ARCHITECTURE.md`, `HANDOFF.md`), scripts, tests, or CI, inspect only what is relevant to the current decision and treat current content as evidence/constraints.

Rules:

- repo-local artifacts are the durable source of truth; chat memory is not;
- never regenerate or overwrite harness files merely because the auditor is running;
- surface contradictions between the supplied plan, user goal, repo reality, and existing harness instead of silently choosing one;
- include only load-bearing harness facts in Round 0;
- use existing task/done criteria as evidence, not unquestionable truth;
- the final locked plan is the highest-priority task-specific contract for this run; stale harness guidance cannot silently override it;
- after verified delivery, update existing harness state only when the locked plan/declared scope explicitly includes those files, and record only observed state (no fake bugs, QA findings, decisions, or completion).

## [GPT] Round 0 — Context Packet

Before Round 1, send Claude one bounded context message that **imports and anchors the supplied architect plan**. Preserve the user's goal and plan faithfully, but replace secrets, credentials, tokens, keys, and private customer data with typed placeholders such as `[REDACTED:api_key]`. List the redactions. If redaction would destroy meaning needed for the decision, fail preflight rather than transmit the sensitive value.

Required sections:

1. Role map and selected execution backend
2. User goal
3. Supplied architect plan — faithful full plan or a verified attachment; this is the starting plan
4. Normalized execution protocol — only material current-user/auditor/repo constraints that the lock must not contradict
5. Repo/stack facts
6. Relevant evidence — files/diff/errors/screenshots/data as warranted
7. Prior attempts and results
8. Hard constraints
9. Explicit non-goals
10. Unknowns
11. Draft acceptance criteria
12. Debate contract — minimum 3 challenge rounds, blocker-only R4/R5, sentinel format, lock schema

Keep the packet to one message. If it is too large, summarize harder or use a host-supported attachment only when attachment success can be verified. Do not split the packet into ordinary sequential chat messages.

### [CLAUDE-via-paste] Round 0 template

```text
[[AUDITOR round=0 from=GPT]]
DEFAULT ROLE PROFILE
Architect / lock owner: Claude Opus 5 Max
Independent challenger + orchestrator/auditor: GPT-5.6
Execution backend: <gpt_codex|claude_claude_code>

The architect plan below was already produced before this auditor run. Round 0 is synchronization, not a second planning pass. Treat this supplied plan as the current architecture baseline. Do NOT create a new plan from scratch. You may repair only transcription/format ambiguity needed to faithfully anchor it.

USER GOAL
...

SUPPLIED ARCHITECT PLAN
...

NORMALIZED EXECUTION PROTOCOL
...

REPO / STACK FACTS
...

RELEVANT EVIDENCE
...

PRIOR ATTEMPTS
...

HARD CONSTRAINTS
...

NON-GOALS
...

UNKNOWNS
...

DRAFT ACCEPTANCE CRITERIA
...

CONTRACT
- Reply with visible sentinels matching the expected round.
- Confirm the supplied plan is the current baseline and restate it faithfully as the full current plan.
- Do not add architecture changes in Round 0; GPT Challenge #1 is the first adversarial revision step.
- After each later revision, restate the full current plan; do not return only a diff.
- Do not invent objections merely to fill a round; `NO CHANGES` is valid when the plan survives a challenge.
- The final lock must use the nine-field schema supplied in Round 3.

Return the anchored current plan only. Do not implement anything.
[[END round=0]]
```

## [GPT] Evidence-on-demand before challenge messages

Before composing any challenge round, decide whether a material disputed claim is cheaply verifiable from the current project. If yes, gather the minimum fresh evidence needed before writing the challenge. If not, challenge from the existing packet/plan without ceremony.

Permitted pre-lock evidence sources include, as relevant:

- targeted reads/searches of repo source, tests, config, schemas, migrations, docs, and current harness artifacts;
- git/status/history/diff evidence needed to establish current project facts;
- read-only test/build/typecheck/lint commands when they directly test a challenged assumption;
- current website/app inspection through browser, simulator, Playwright/MCP, DOM, screenshots, accessibility tree, console/runtime logs, network/API observations, and read-only database/data invariants;
- an existing local runtime, or a safely started local preview when observation is impossible otherwise and startup has no material side effects.

Rules:

1. **Targeted, not automatic.** Do not turn each round into a full repo or visual audit. Investigate only a claim whose answer can change the challenge, lock, or acceptance criteria.
2. **Read-only before LOCK.** No implementation edits, migrations, dependency changes, production mutations, data cleanup, or bug fixes during debate evidence gathering. If a diagnostic itself would mutate meaningful state, do not run it unless a safe isolated fixture/read-only mode exists.
3. **Evidence provenance.** Record the path/command/URL/viewport/procedure and result needed to support the challenge. Screenshots/visual observations should identify the inspected surface/state rather than rely on vague impressions.
4. **Prompt precision.** Put only the load-bearing finding into the outgoing challenge: claim, evidence, why it matters, and the exact plan/criterion change or question it creates. Do not dump raw logs or whole-file content into Claude unless required.
5. **No premature solutioning.** Evidence may prove the current plan wrong; it does not authorize GPT to implement its preferred fix before the architect responds and LOCK is valid.
6. **Runtime lifecycle awareness.** If the challenger starts or discovers a temporary server/preview job, record lifecycle/lease evidence needed for later verification; do not confuse `running now` with durable service state.

This evidence-on-demand step applies to Round 1, Round 2, Round 3 blocker verification, and blocker-only Rounds 4–5. The stronger the claim, the stronger the evidence should be; preference disputes do not justify extra tool work.

## [GPT] Round 1 — Broad challenge

Challenge only load-bearing issues: problem framing, unsupported assumptions, hard constraints, architecture, failure modes, product mental model, symptom-vs-root-cause mistakes, and whether acceptance criteria can prove success. When a material assumption is directly verifiable from the repo or current product, gather that evidence first and cite the observation in the challenge.

A low-finding round is still real: probe edge cases, ask verification questions, or sharpen criteria. Do not manufacture disagreement.

### [CLAUDE-via-paste] Round 1 template

```text
[[AUDITOR round=1 from=GPT]]
ROUND 1 — CHALLENGE
Consensus is expected by Round 3; Rounds 4–5 are blocker-only and Round 5 is final.

Blockers
...

Major concerns
...

Assumptions to prove
...

Required changes / verification questions
...

For each material point return Accept / Modify / Reject with a short reason, then restate the full revised plan and remaining uncertainties. If no material change is warranted, return `NO CHANGES` and restate the full plan.
[[END round=1]]
```

## [GPT] Round 2 — Stress test

Round 1 is closed. Attack only the revised plan and decisions created by that round. Probe race conditions, stale state, data loss, permissions, migrations, backwards compatibility, hidden coupling, rollback/recovery, scaling cliffs, user-flow edge cases, scope creep, new bottlenecks, and verification gaps. Use targeted repo/runtime/visual evidence when it can settle a disputed assumption more precisely than speculation.

### [CLAUDE-via-paste] Round 2 template

```text
[[AUDITOR round=2 from=GPT]]
ROUND 2 — STRESS TEST
Round 1 is closed. Stress-test only the revised plan and decisions created by that round. Round 3 is the expected consensus round.

Remaining blockers
...

Failure scenarios
...

Trade-offs still unresolved
...

Verification gaps / required changes
...

Return Accept / Modify / Reject for each material point, then restate the full revised plan. If no material change is warranted, return `NO CHANGES` and restate the full plan.
[[END round=2]]
```

## [GPT] Round 3 — Consensus

This is the normal final debate round. Do not introduce preferences, nice-to-haves, or unrelated alternative architectures. Check only prior blockers, contradictions, hard-constraint violations, acceptance-criteria quality, executability, and genuinely new blocking evidence.

### [CLAUDE-via-paste] Round 3 template

```text
[[AUDITOR round=3 from=GPT]]
ROUND 3 — CONSENSUS
This is the normal final debate round. Do not introduce new preferences or alternative architectures.

Resolved items
...

Any remaining evidence-backed blockers
...

Return exactly one verdict:
LOCK
or
BLOCKERS REMAIN

If LOCK, output the complete nine-field `=== LOCKED PLAN ===` now.
If blockers remain, list only blockers that threaten correctness, safety, data integrity, a hard requirement, or executability, with evidence and what would resolve each one.
[[END round=3]]
```

## [GPT] Rounds 4–5 — Blockers only

Round 4 is allowed only if Round 3 carries a genuine blocker. Round 5 is the absolute final round. Resolved topics are closed.

### [CLAUDE-via-paste] Round 4 template

```text
[[AUDITOR round=4 from=GPT]]
ROUND 4 — BLOCKERS ONLY
Discuss only the unresolved blockers carried from Round 3. No preference debate or scope expansion.

For each blocker: Accept / Modify / Reject with evidence. Make only the minimum changes needed, then restate the **full current plan** so recovery does not depend on an earlier thread turn.
If all blockers are resolved, output the complete `=== LOCKED PLAN ===`.
If a blocker still prevents execution, identify it precisely after the full current plan. Round 5 is final.
[[END round=4]]
```

### [CLAUDE-via-paste] Round 5 template

```text
[[AUDITOR round=5 from=GPT]]
ROUND 5 — FINAL
There is no Round 6. Resolve only blockers carried from Round 4.

End in exactly one state:
LOCK — followed by the complete `=== LOCKED PLAN ===`
or
BLOCKED — with the minimum unresolved evidence or user decision required.
Do not continue debating after this response.
[[END round=5]]
```

## [GPT] Lock schema and authority

Under the default role profile, Claude/Opus is the architect and lock owner; GPT is the challenger, orchestrator, and execution gate. A custom/reverse profile must record equivalent role ownership explicitly before debate; model names may change, but authority must not be ambiguous.

A lock is valid only at Round 3 or later and must contain all nine non-empty fields:

```text
=== LOCKED PLAN ===
Problem
Decision
Scope
Constraints
Implementation steps
Acceptance criteria
Explicitly out of scope
Known trade-offs / accepted risks
Approval-sensitive / destructive actions
=== END LOCKED PLAN ===
```

`Approval-sensitive / destructive actions` must contain an enumerated list or the literal `NONE`.

Every acceptance criterion must state:

- a verification method appropriate to the task: automated check, artifact/data invariant, structural/IA observation, or manual observation with a stated procedure; and
- the expected result.

For app/website/feature delivery, the lock as a whole must also provide an executable **done-means contract** covering the expected behavior, critical user flows, validation methods/commands, and explicit out-of-scope boundaries. Do not force implementation details into the contract; it defines what must be true, not how to code it. If a critical flow cannot be tested from the lock, request targeted lock repair before execution.

A missing/empty field is a format defect. Request a targeted repair without consuming another challenge round.

After structural validation, run an objective **semantic sanity check** on the expected results themselves. This is not a taste/style review. Repair only obvious evidence-backed correctness defects such as broken singular/plural copy (`1 details`), impossible numeric/responsive invariants, mutually inconsistent labels/states, or an expected result that contradicts a locked fact. Request a targeted semantic repair without consuming another challenge round.

### [CLAUDE-via-paste] Semantic lock repair template

```text
LOCK SEMANTIC REPAIR — no new debate round
The architecture is accepted, but these expected results contain objective correctness defects:
...
Repair only those defects and return the complete `=== LOCKED PLAN ===`. Do not reopen architecture or introduce preferences.
```

GPT may reject a valid-format lock only with:

- **Claim** — the blocking defect
- **Evidence** — observable support
- **Resolution condition** — what would remove the blocker

Preference is not a veto. An evidence-backed rejection after Round 3 opens Round 4 blocker-only.

### [CLAUDE-via-paste] Lock repair template

```text
LOCK FORMAT REPAIR — no new debate round
The plan is otherwise accepted, but these required lock fields/criteria are malformed or missing:
...
Return only a repaired complete `=== LOCKED PLAN ===`. Do not reopen architecture discussion.
```

## [GPT] BLOCKED terminal

On terminal block, write `blocked.md` in the run state directory containing:

- blocking issue
- Claude position
- GPT position
- evidence that would settle it
- minimum user decision/evidence required
- safest partial scope, if any

Set `phase=blocked`. Do not execute any part that depends on the unresolved decision.

## [GPT] Execution-backend preflight and harness profile

After lock, set `phase=backend_preflight` and verify the **selected** execution backend can operate on the exact target workspace. Debate transport and execution transport are separate capabilities. Never treat claude.com browser chat as equivalent to Claude Code execution, and never silently fall back to the other backend.

If required backend capabilities are missing, record a named backend blocker and stop before the first implementation write. The locked architecture remains valid unless the missing capability itself invalidates a locked assumption.

Before the first implementation write, persist concrete execution provenance separately from the user's coarse backend choice:

- ChatGPT current session + LocalOps: `execution_family=openai`, `execution_environment=chatgpt_localops`;
- Codex: `execution_family=openai`, `execution_environment=codex`;
- Claude Code: `execution_family=anthropic`, `execution_environment=claude_code`;
- any other supported executor must use `other_verified` plus a human-readable note in state.

Do not report `gpt_codex` as though it proves Codex executed the task; it is only the user's backend-family choice. Final reporting must name the concrete `execution_environment`.

Use the lightest load-bearing project harness. Existing repo artifacts are preferred; create a new harness artifact only when it materially improves execution, verification, recovery, or handoff. Do not generate a ceremonial full suite for a small task.

### [GPT] GPT/Codex execution profile

When `execution_backend=gpt_codex`:

- treat repo-local files as source of truth; use a concise `AGENTS.md` as a routing map when present or genuinely needed, linking outward instead of copying a manual inline;
- prefer mechanical enforcement over reminders: tests, lint, typecheck, scripts, structural checks, CI, or documented invariants with executable verification;
- for bugs, reproduce before fixing, isolate root cause, fix minimally, verify the fix, then run broader regression;
- run the smallest meaningful pre-edit check and the relevant build/test/lint/typecheck after coding changes;
- if execution exposes a reusable missing capability (fixture, test, script, observability path, check), persist it in the right project artifact only when it is load-bearing for future runs;
- update `TASKS.md`, `BUG_LOG.md`, `DECISIONS.md`, `ARCHITECTURE.md`, or other project state only when those files exist/are needed and the update is factual and in declared scope.

### [GPT] Claude/Claude Code execution profile

When `execution_backend=claude_claude_code`:

- use a concise `CLAUDE.md` project contract when present or genuinely needed;
- hand the executor the locked plan/done-means contract rather than asking it to re-plan architecture;
- keep long-running continuation state concise and truthful (for example `HANDOFF.md`) when it materially helps a fresh context continue;
- use build → skeptical QA → fix → QA/regression until every locked criterion passes;
- QA must actively look for missing core functionality, stubs, fake/display-only integrations, broken flows, shallow implementation, weak information hierarchy, and visual polish hiding functional failure;
- when possible, run the product and inspect it as a user with browser/simulator/MCP/screenshots, console/runtime logs, network/API checks, database/data invariants, and end-to-end flows;
- keep ceremony proportional to risk: simple changes do not require phases/sprints or a full harness suite.

### [GPT] Shared harness decisions intentionally retained/removed

Retain from the former backend-specific harness approach: durable repo state, done-means contracts, skeptical evaluation, reproduce-first debugging, mechanical checks, runtime validation, truthful handoff, and load-bearing risk scaling.

Do **not** retain redundant ceremony that conflicts with this auditor: no mandatory giant discovery questionnaire when plan+repo already answer it, no mandatory human stop after every phase, no per-phase commits, no full harness generation by default, and no second architecture authority beside the locked plan. The auditor's one-final-commit rule remains authoritative.

## [GPT] Execution after lock

1. Record baseline before implementation.
2. Complete selected-backend preflight; never write through an unverified or different backend.
3. Prepare only the load-bearing execution-harness context/artifacts required by the selected profile.
4. Execute the locked plan in bounded increments.
5. Verify meaningful increments with targeted evidence and the backend's actual build/test/runtime paths.
6. Solve routine implementation problems locally; for bugs, reproduce/root-cause before fix when applicable.
7. Do not reopen debate for type errors, CSS, ordinary test fixes, package details, or implementation choices that remain inside the lock.
8. Re-enter the architect only through Architecture Escalation when new evidence proves a locked architectural assumption false, a required capability absent in a way that invalidates the design, a fundamental data-model incompatibility present, or satisfying the lock would violate a hard constraint.

## [GPT] Scope discipline, baseline evidence, and canonical run delta

At lock, record:

- git HEAD when available;
- complete baseline working-tree status (`git status --porcelain`/equivalent), including untracked paths;
- `declared_scope`;
- task-relevant baseline evidence only when needed to support later claims (for example warning set, HTTP/runtime state, existing failing test, or current UI behavior).

For a pre-existing dirty file, snapshot its baseline content **before this run first edits it** and record the snapshot externally. In non-git workspaces, snapshot declared implementation scope before execution where practical.

GPT may extend `declared_scope` before first touch when an incidental file is directly required by a locked step. Record file, reason, and step. Examples: lockfile, generated manifest, adjacent test, import-site update. If the new file implies architecture/scope change or conflicts with a hard constraint, escalate instead. A file touched without prior declaration/extension is an audit finding.

Before any skeptical diff review, prohibited-path assertion, final verification, or commit gate, build/update one canonical external `run_delta_manifest.md` (or equivalent structured artifact) and set `run_delta_manifest_path`. The manifest must account for **every** path changed relative to baseline, including:

- tracked unstaged modifications;
- staged modifications;
- newly created/untracked files;
- deletions/renames;
- mixed pre-existing dirty files, with run-owned delta separated when safely provable.

For each path record at minimum: path, change kind, baseline status, current status, whether run-owned, declared scope/extension, and review status. Hashes are recommended when practical. Plain `git diff` or `git diff --name-only` is never sufficient by itself because it omits untracked files.

All scope/prohibited-path checks and skeptical audit coverage must consume the canonical manifest. Audit may not conclude `0 P0 / 0 P1` while any run-owned manifest path is unreviewed. The final staged/committed path set must reconcile exactly with the safely stageable run-owned manifest subset.

### [GPT] Evidence-qualified baseline claims

Use precise provenance language. Call a warning/error/behavior **pre-existing** only when matching pre-change evidence (or another independently verified baseline source) proves it existed before this run. If evidence only shows it comes from an unchanged/out-of-delta file, say exactly that; do not upgrade `outside run-owned delta` into `pre-existing`.

## [GPT] Architecture Escalation

Send only the minimum packet:

```text
ARCHITECTURE ESCALATION
Locked decision: ...
New evidence: ...
Why execution is blocked: ...
Minimum decision needed: ...
```

Send this to the recorded architect/lock owner using the run's verified debate transport. This is a bounded correction, not a restart of the 3–5 round debate.

## [GPT] Audit and finish

Audit against artifacts, not implementation intent:

- original user goal
- locked plan / done-means contract
- run-specific delta
- fresh test/verification evidence
- relevant existing project-harness state when present

The orchestrator/auditor owns this pass independently from the executor. If Claude/Claude Code executed, GPT audit provides cross-model separation under the default profile. If GPT/Codex executed, use a fresh GPT context when the host supports it cheaply; otherwise explicitly ignore remembered implementation rationale unless backed by artifacts.

For runnable apps/websites, verify the product as a user when reasonably possible rather than stopping at static code/build success. Use the strongest available runtime evidence: browser/simulator/Playwright/MCP interaction, screenshots where useful, console/runtime logs, network/API checks, database/data invariants, and end-to-end critical flows.

The skeptical audit must actively check, where relevant:

- product depth and whether the promised user outcome actually exists;
- functional correctness of critical flows;
- UX clarity and information hierarchy;
- visual/design quality only against the locked product direction, not reviewer taste;
- code/data correctness and regression risk;
- completeness vs stubs, fake integrations, display-only features, placeholder success states, or polished UI hiding broken behavior;
- AI features: whether the model/integration drives real product behavior/actions where promised rather than a fake chat surface or disconnected demo.

A clean build or passing unit tests cannot overrule a directly observed broken core flow. Treat a stubbed/fake/display-only core requirement as P0/P1 according to delivery impact.

Output only:

```text
P0 — blockers
P1 — must-fix
P2 — optional
Acceptance criteria status
Regression risks
```

Fix P0/P1 only unless the user approves optional work. Then rerun targeted checks plus every locked acceptance criterion. Do not run another broad audit without new evidence.

After regression passes, write `final_verification.md` in the external run-state directory, set `final_verification_path`, and include concise evidence for:

- concrete `execution_environment`;
- acceptance criteria status;
- canonical run-delta coverage (all run-owned paths reviewed);
- regression checks;
- **Execution deviations** — literal `NONE` or each departure from the locked operational expectation with reason, effect, and whether architecture/criteria remain valid;
- **Runtime assets / durability** when servers, workers, tunnels, schedulers, preview jobs, or similar processes matter: current health plus lifecycle status (`temporary|durable|unknown`), known lease/TTL expiry, supervisor/restart owner, and any action required to keep it alive;
- baseline-qualified warnings/known issues using only claims supported by actual pre-change evidence.

`HTTP 200 now` proves liveness, not durability. If the runtime is a temporary leased preview, say so explicitly. If durability metadata is unavailable, report `unknown` rather than implying permanence.

A sensitive/destructive action outside the lock's enumerated field is P0.

If the locked scope includes persistent project-harness files, update them only after the implementation/audit state is known: task/phase status from observed completion, QA findings from the actual audit, bugs from reproduced failures, decisions from actual locked decisions, and `HANDOFF.md` from the real continuation state. Do not create ceremonial entries or mark work complete before verification.

## [GPT] Final auto-commit

When the target workspace is a git repository, successful completion includes **one final commit** after execution, audit, required fixes, regression checks, and every locked acceptance criterion pass **unless the current user explicitly disabled commit for this run**. A stale `no commit` inside an inherited plan is not such an override.

Rules:

- set `phase=committing` only after verification is complete;
- the commit contains only the safely stageable run-owned subset of the canonical run-delta manifest; pre-existing unrelated dirty paths are never included;
- use a concise task-derived commit message unless the user explicitly supplied one;
- do not create intermediate implementation/fix commits by default; consolidate the verified run into the one final commit;
- never push automatically;
- before commit, reconcile staged paths against the canonical run-delta manifest and prohibited-path rules, including new/untracked files;
- after commit, verify the new HEAD and committed path set against the manifest, then persist `final_commit_sha` and set `phase=done`.

A file that was already dirty before the run is a **mixed file**. It may be auto-committed only if the host can stage exactly the run-owned hunks/delta without including the pre-existing user edits. If exact run-only staging cannot be proven, set `error_state=commit_blocked_mixed_file`, leave the working tree intact, and stop before creating a partial or unsafe final commit. Never use path-level staging/commit as a workaround that sweeps pre-existing user work into the commit.

If the workspace is not a git repository, no commit is required; finish after verification.

## [GPT] User-facing completion report contract

After the final commit decision is resolved (committed, intentionally skipped by current user instruction, or safely blocked), produce one concise chat report. The prose report is the user's primary completion surface; internal artifacts remain the audit record.

Use exactly these sections and include only verified facts:

```text
DONE | BLOCKED

Decision
- <1–3 lines: final locked architecture / outcome that was actually delivered>

Execution
- Executor: <concrete execution_environment>
- Implemented: <short factual summary>
- Files changed: <run-owned count>

Verification
- Acceptance criteria: <N/N PASS | failures>
- Tests: <result or NOT APPLICABLE>
- Lint/typecheck/build: <only commands actually run>
- Runtime QA: <PASS/FAIL/NOT APPLICABLE>
- Regression: <PASS/FAIL>
- P0/P1 remaining: <count>

Runtime
- <relevant runtime asset only> — <health>
- Durability: <temporary|durable|unknown>
- <lease/TTL/supervisor/restart note when known>

Deviations
- NONE
  or
- <deviation + reason + impact>

Remaining
- <unresolved blocker/P2/known issue, or NONE>

Git
- Commit: <sha + subject | skipped by user | blocked>
- Working tree: <verified state>
- Push/deploy: <what actually happened; never imply push/deploy>
```

Reporting rules:

1. Omit irrelevant command labels rather than fabricating `PASS`; `NOT APPLICABLE` is valid when useful.
2. `Runtime` may be `NONE` when no persistent/local runtime matters. Never report liveness without known durability semantics when those semantics are available.
3. `Deviations` is mandatory and uses literal `NONE` when empty.
4. `Remaining` includes unresolved blockers and optional P2/known issues that materially matter; do not bury a known defect.
5. Git reporting names the concrete commit SHA only after verifying HEAD/committed paths. If current user instruction disabled commit, say so explicitly.
6. Do not dump full round transcripts, full lock, full diffs, or internal state paths into the normal completion report.
7. When debate materially changed the plan, add one compact line after `Decision`: `Auditor — R1: <n> material changes · R2: <n> · R3: LOCK` (and R4/R5 only if actually used), followed by at most 2–4 key decisions that changed. Otherwise omit it.
8. On terminal `BLOCKED`, use the same section order but replace implementation/verification claims with the exact blocker, evidence, and minimum next action; never format a blocked run as `DONE`.

## [GPT] Self-modification release gate

If this run changes any file under `gpt-auditor/`, set `self_release_gate.required=true` and `e2e_status=pending` as soon as that scope is known.

Before the final commit/release:

1. finish all intended fixes and static/document checks;
2. start a **fresh auditor run using the final post-fix skill files**;
3. run E2E-1 end to end against its disposable external fixture target, not against `gpt-auditor/` itself;
4. require E2E-1 PASS plus the release-required document/scenario checks;
5. persist `e2e_status=passed` only on fresh evidence.

By default, `NOT RUN`, FAIL, or incomplete E2E evidence blocks the final commit/release. A current explicit user instruction to commit/release this skill despite the project-local E2E gate may override that local rule; record the override as an execution deviation and do not misreport E2E as passed. The release E2E is deliberately non-self-modifying so it does not recursively trigger another self-release gate.

## [GPT] Assertions

Treat these as state-machine failures, not suggestions:

- round > 5
- acceptance of a lock before Round 3
- state advance without verified transport/state postcondition
- global flat-text sentinel matching
- controlling an unbound Claude thread
- touching Claude before the supplied architect plan is persisted and **both** startup choices are explicit
- starting debate by asking the default architect to generate a fresh initial plan instead of importing the supplied plan
- making implementation edits or mutating meaningful project/user data during pre-lock challenge evidence gathering
- claiming a repo/runtime/visual fact in a challenge when direct evidence was gathered but its source/procedure/result was not recorded
- continuing a `new` session after Round 0 without capturing/persisting the concrete Claude conversation URL **and** verifying the exact requested Claude title
- consuming or sending a default-profile architecture round on any Claude model other than Opus 5 Max
- executing on a backend different from the user's recorded `execution_backend`
- implementation write before selected-backend capability preflight passes
- audit/scope/prohibited-path completion based only on plain `git diff` without accounting for untracked/deleted paths
- final verification missing an `Execution deviations` section
- reporting a runtime as durable when only current liveness is proven
- calling an issue `pre-existing` without matching baseline evidence
- spending a challenge round on a protocol contradiction that was mechanically knowable and should have been normalized before Round 0
- accepting a lock with an objective semantic correctness defect after the semantic sanity gate
- reporting `gpt_codex` as the concrete executor instead of the verified `execution_environment`
- execution after terminal BLOCKED
- completion claim without fresh evidence
- auto-pushing a repository
- creating the final auto-commit before required verification/self-release E2E passes
- sweeping pre-existing user edits into the final auto-commit
- debate reopened for routine implementation

## [GPT] Continuation-thread rehydration

If Claude reports thread-length exhaustion, open a continuation thread and send one rehydration message containing:

- original compact Context Packet
- exact/faithful supplied architect plan reference
- full current plan
- recorded role profile and execution backend
- resolved decisions with justifications
- open blockers
- current round
- role/round contract
- nine-field lock schema

Record old and new Claude URLs in `thread_lineage`. This recovery message may exceed the ordinary debate-message target size because it replaces the lost thread context.
