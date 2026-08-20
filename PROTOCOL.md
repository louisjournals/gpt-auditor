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
  "owner_host": "orchestrator_host|codex|other",
  "phase": "startup_choice|preflight|context|debate|locked|repo_lock_persisted|backend_preflight|awaiting_execution_approval|executing|architect_completion_check|auditing|fixing|regression|precommit_reporting|release_verifying|committing|done|blocked",
  "round": 1,
  "turn_state": "awaiting_startup_choices|ready_to_challenge|challenge_sent|claude_complete|response_processed|lock_pending|awaiting_execution_approval",
  "role_profile": "default_claude_architect_gpt_challenger|custom",
  "roles": {
    "architect": "Claude Opus 5 Max",
    "challenger": "GPT-5.6",
    "lock_owner": "Claude Opus 5 Max",
    "orchestrator_auditor": "GPT-5.6"
  },
  "execution_backend": "codex|claude_code|other_verified",
  "execution_backend_label": "Codex|Claude Code|<generic custom label>",
  "execution_family": "openai|anthropic|other",
  "execution_environment": "codex|claude_code|other_verified",
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
  "repo_locked_plan_path": ".gpt-auditor/LOCKED_PLAN.md",
  "repo_locked_plan_hash": "sha256:...",
  "execution_approval": {"status": "not_requested|awaiting|approved|change_decision_required", "approval_root_hash": null, "current_plan_hash": null, "repair_envelope": null, "repair_chain": [], "approved_at": null, "evidence": null},
  "task_tracks": {"objective_correctness": true, "operator_experience": false},
  "operator_review": {"preplan_status": "not_applicable|missing|required|complete", "evidence_path": null, "pending_criteria": []},
  "pattern_inventory_path": null,
  "baseline": {"head_sha": null, "dirty_paths": [], "snapshot_manifest": [], "evidence": {}},
  "declared_scope": [],
  "scope_extensions": [],
  "touched_paths": [],
  "run_delta_manifest_path": "...",
  "execution_completion_path": "...",
  "architect_completion_gate": {"status": "not_started|incomplete|passed", "attempt": 0, "checked_plan_hash": null, "missing_items": [], "evidence": null},
  "execution_deviations": [],
  "runtime_assets": [],
  "precommit_report_path": "...",
  "final_verification_path": "...",
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

## [GPT] Problem-domain split and operator calibration

Before debate, classify the task into two independent tracks:

- **Objective correctness** — architecture, safety, security, data integrity, auth/permissions, logic, contracts, migrations, runtime correctness, regression, executability. This track may use the adversarial multi-round protocol.
- **Operator experience** — perceived affordance, visual hierarchy, information architecture clarity, taste, usability, whether something *looks* clickable/important/confusing, and similar human perception. Model consensus is not authoritative for this track.

For any material operator-experience track, require fresh operator evidence **before the architect plan is treated as ready for debate**. The operator should inspect/use the current product normally and record what they notice, try to click, misunderstand, or dislike without being constrained to the existing acceptance-criteria list. Persist that evidence externally and set `operator_review.preplan_status=complete`.

If a supplied UX/IA/visual/affordance plan predates that review, do not spend challenge rounds polishing it. Stop before Claude debate, obtain the operator review, then require a refreshed architect plan that incorporates the findings. This preserves the intended order: initial audit → operator review → pattern inventory → plan.

If the task is purely operator-experience work with no objective correctness/safety/data/architecture/executability dispute, skip the adversarial debate entirely and use the small-batch operator loop. If mixed, debate only the objective track; operator findings enter Round 0 as authoritative constraints/evidence and are not preference questions for the models to resolve.

## [GPT] Pattern completeness before scope freeze

**Scope fences constrain writes, not observation.** Before freezing modification scope around any discovered defect instance, inspect the relevant sibling pattern across the product/repo: shared component usages, same visual treatment, same route family, same state machine, same account-dependent surface, or other directly analogous instances. Persist a concise `pattern_inventory.md` externally and set `pattern_inventory_path`.

A finding on one page must not become a one-page plan merely because that page was the first observed instance. The inventory may conclude that only one instance should be changed, but that decision comes **after** the pattern scan and must state why. Pattern discovery is read-only unless/until the refreshed plan includes additional changes.

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

Normalization may resolve procedural contradictions, not architecture disagreements. Objective architecture/product choices remain for the correctness-track debate. Operator-authority perceptual judgments remain operator evidence and are not resolved by model voting.

## [GPT] Startup-choice gate

Every new auditor run starts in `phase=startup_choice` and `turn_state=awaiting_startup_choices`. Before any Claude chat discovery or browser navigation, collect **both** choices in one compact gate:

1. Claude session: `new` or `same_name_existing`; and
2. execution backend: `codex`, `claude_code`, or `other_verified` with a human-readable executor label.

Do not inspect/search/open/create/bind Claude chats until both choices are explicit and the supplied architect plan is present. Persist each choice only after the user explicitly selects it. Never inherit a previous run's execution backend or session mode.

Resolve `requested_claude_title` from the exact current orchestrator session title for both session modes; if the host cannot read that title reliably, ask the user for the exact title before any Claude search or new-session creation.

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
| `awaiting_execution_approval` | Re-read and hash the repo locked-plan artifact, show the compact execution summary if needed, and wait for explicit user approval. Do not make implementation writes. |

`challenge_sent` may be written only after `verified_send` succeeds. If a send is ambiguous, state remains `ready_to_challenge`; recovery starts with the idempotency scan.

## [GPT] Recovery artifacts

Persist the evidence needed to resume without chat memory:

- save the exact supplied architect plan to `input_plan.md`, set `input_plan_path`, then hash it into `input_plan_hash` before any debate transmission;
- save pre-debate protocol decisions to `protocol_normalization.md` and set `protocol_normalization_path` before Round 0;
- save the **redacted** Round-0 packet to `context_packet.md`, set `context_packet_path`, then hash that exact stored packet into `context_packet_hash` before transmission;
- after every completed Claude response in the default role profile, save the exact assistant-role text to `messages/round-N-claude.md` and set `last_claude_response_path` **before** writing `turn_state=claude_complete`;
- after processing any nonterminal response that establishes/restates the current plan (Round 0, 1, 2, and blocker-resolution Round 4 when used), save the latest full current plan to `current_plan.md` and update `current_plan_path` before writing `response_processed`;
- after a valid lock, save the exact `=== LOCKED PLAN ===` block to external `locked_plan.md`, set `locked_plan_path`, and only then set `phase=locked`;
- before any repo write, record the implementation baseline; then persist the exact validated lock into the repo as `.gpt-auditor/LOCKED_PLAN.md` (or an explicitly user-selected equivalent), record `repo_locked_plan_path` + SHA-256, and set `phase=repo_lock_persisted`;
- persist execution approval separately from debate state; the one normal approval is anchored to `approval_root_hash` plus the explicit non-material repair envelope, while `current_plan_hash` and any repair chain remain durable execution evidence.

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
7. Operator-review evidence and pattern inventory when the experience track is material
8. Prior attempts and results
9. Hard constraints
10. Explicit non-goals
11. Unknowns
11. Draft acceptance criteria
12. Debate contract — minimum 3 challenge rounds, blocker-only R4/R5, sentinel format, lock schema

Keep the packet to one message. If it is too large, summarize harder or use a host-supported attachment only when attachment success can be verified. Do not split the packet into ordinary sequential chat messages.

### [CLAUDE-via-paste] Round 0 template

```text
[[AUDITOR round=0 from=GPT]]
DEFAULT ROLE PROFILE
Architect / lock owner: Claude Opus 5 Max
Independent challenger + orchestrator/auditor: GPT-5.6
Execution backend: <codex|claude_code|other_verified> — if `other_verified`, include the generic executor label

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

If LOCK, output the complete nine-field `=== LOCKED PLAN ===` now. Every acceptance criterion must include `Authority: MACHINE|OPERATOR|MIXED`, a verification method appropriate to that authority, and an expected result.
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

- `Authority: MACHINE`, `Authority: OPERATOR`, or `Authority: MIXED`;
- a verification method appropriate to that authority: automated check, artifact/data invariant, runtime interaction, or operator observation with a stated procedure; and
- the expected result.

Criterion status is exactly `VERIFIED`, `NOT VERIFIED`, or `FAILED`. `OPERATOR` criteria require explicit operator evidence to become `VERIFIED`; DOM/computed-style/tests/model judgment cannot substitute. `MIXED` requires both machine and operator evidence. Never infer `VERIFIED` from absence of failure.

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

## [GPT] Repo locked-plan artifact and execution approval gate

A valid architecture lock is necessary but **not sufficient** to begin implementation.

After the external `locked_plan.md` is valid:

1. **Baseline first.** Record git HEAD, complete working-tree status, declared scope, and any needed mixed-file snapshots before the repo locked-plan artifact is written. The repo plan file is itself a run-owned change and must remain visible in the canonical run delta.
2. **Persist the execution contract in the repo.** Write the exact validated `=== LOCKED PLAN ===` block to `.gpt-auditor/LOCKED_PLAN.md` by default. Use another repo-relative path only when the user explicitly selects one or an established project convention requires an equivalent path. Never overwrite a pre-existing unrelated file silently.
3. **Hash the exact repo artifact.** Compute SHA-256 over the stored bytes and persist `repo_locked_plan_path` + `repo_locked_plan_hash`. Re-read the file after writing; do not trust the write call alone.
4. **Backend preflight before asking approval.** Verify the selected executor can operate on the workspace without making implementation writes. If capabilities are missing, block instead of asking the user to approve an impossible execution.
5. **Resolve known lock instability before asking approval.** Use the preflight and currently available repo/runtime evidence to settle observable baseline/invariant contradictions before presenting the approval gate. Do not ask the user to approve while a known lock repair is still pending.
6. **Summarize from the repo artifact, not chat memory.** Re-read `.gpt-auditor/LOCKED_PLAN.md` and present a compact 3–8 bullet summary of what will be executed. Include material out-of-scope boundaries, the selected execution path in generic capability terms, and any `Approval-sensitive / destructive actions`. Do not re-dump the full plan or expose private/local-only tool names.
7. **Ask for the one normal execution approval.** State the repo path and SHA-256. Set `phase=awaiting_execution_approval`, `turn_state=awaiting_execution_approval`, and `execution_approval.status=awaiting`. LOCK itself and startup backend selection are not approval. A normal run must not repeatedly ask the user to type `approve` after this gate.
8. **Bind approval to a root hash plus a narrow repair envelope.** On an explicit `approve`, `approved`, `proceed`, or unambiguous equivalent responding to this gate, persist `approval_root_hash`, `current_plan_hash`, approval evidence, timestamp, and the repair envelope. Before the first implementation write, re-read/re-hash the repo plan and require exact equality with both recorded hashes.
9. **Permit only non-material post-approval repairs without another approval prompt.** An architect lock repair may update `current_plan_hash` and append `{old_hash,new_hash,reason,evidence}` to `repair_chain` without another user approval only when the auditor and architect both verify that it: preserves the user goal and product direction; does not add permissions/OAuth scopes, destructive or approval-sensitive actions, production/data mutation blast radius, or a different executor; does not introduce a new architectural dependency/system; and does not weaken the locked acceptance burden. Re-persist/re-hash the repo lock and continue under the existing approval. Do not show another generic approval prompt for such repairs.
10. **Material changes require a specific change decision, not approval spam.** If a proposed repair falls outside the envelope, or the user materially changes goal/scope/permissions/destructive actions, set `execution_approval.status=change_decision_required` and stop affected work. Ask one concrete decision describing exactly what changed and why. Do not present another generic `approve this hash` loop. If the user authorizes the change, record that decision and establish the revised execution contract before continuing.
11. **Chat is non-authoritative during execution.** For task requirements, the current approved repo locked-plan artifact plus recorded repair chain outranks remembered debate/session content. Current user messages may pause/cancel work immediately; a task-changing instruction follows the change-decision rule above rather than silently mutating execution.
12. **Resume from artifact.** Every executor handoff, fresh context, or resumed run must re-read the repo locked-plan artifact, verify `current_plan_hash`, and load any recorded repair chain before continuing implementation.

Execution-stage authority is therefore:

1. system/host safety constraints;
2. current explicit user control instructions (`stop`, `pause`, `cancel`, or a task-changing instruction that triggers the material-change decision rule);
3. the current **approved repo locked-plan artifact** plus its recorded approval/repair chain;
4. current repo facts/harness evidence that do not contradict that execution contract;
5. session chat/history and stale inherited guidance as non-authoritative context.

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

After the repo locked-plan artifact is persisted and hashed, set `phase=backend_preflight` and verify the **selected** execution backend can operate on the exact target workspace. Debate transport and execution transport are separate capabilities. Never treat claude.com browser chat as equivalent to Claude Code execution, and never silently fall back to the other backend. This preflight may inspect/read but must not make implementation writes; user execution approval is requested only after preflight passes.

If required backend capabilities are missing, record a named backend blocker and stop before the first implementation write. The locked architecture remains valid unless the missing capability itself invalidates a locked assumption.

Before the first implementation write, persist concrete execution provenance:

- Codex: `execution_family=openai`, `execution_environment=codex`;
- Claude Code: `execution_family=anthropic`, `execution_environment=claude_code`;
- a custom explicitly supported executor may use `other_verified` plus a human-readable note in state.

The default public workflow does not treat the orchestrator chat host as an execution backend and must not require or name a private/local-only tool stack. Final reporting names the concrete `execution_environment` that actually performed the implementation.

Use the lightest load-bearing project harness. Existing repo artifacts are preferred; create a new harness artifact only when it materially improves execution, verification, recovery, or handoff. Do not generate a ceremonial full suite for a small task.

### [GPT] Codex execution profile

When `execution_backend=codex`:

- begin implementation/resume by reading the approved repo locked-plan artifact and verifying its hash; do not reconstruct task requirements from session chat/history;
- treat repo-local files as source of truth; use a concise `AGENTS.md` as a routing map when present or genuinely needed, linking outward instead of copying a manual inline;
- prefer mechanical enforcement over reminders: tests, lint, typecheck, scripts, structural checks, CI, or documented invariants with executable verification;
- for bugs, reproduce before fixing, isolate root cause, fix minimally, verify the fix, then run broader regression;
- run the smallest meaningful pre-edit check and the relevant build/test/lint/typecheck after coding changes;
- if execution exposes a reusable missing capability (fixture, test, script, observability path, check), persist it in the right project artifact only when it is load-bearing for future runs;
- update `TASKS.md`, `BUG_LOG.md`, `DECISIONS.md`, `ARCHITECTURE.md`, or other project state only when those files exist/are needed and the update is factual and in declared scope.

### [GPT] Claude Code execution profile

When `execution_backend=claude_code`:

- begin implementation/resume by reading the approved repo locked-plan artifact and verifying its hash; do not reconstruct task requirements from session chat/history;
- use a concise `CLAUDE.md` project contract when present or genuinely needed;
- hand the executor the repo locked-plan/done-means contract rather than asking it to re-plan architecture;
- keep long-running continuation state concise and truthful (for example `HANDOFF.md`) when it materially helps a fresh context continue;
- use build → skeptical QA → fix → QA/regression until every locked criterion passes;
- QA must actively look for missing core functionality, stubs, fake/display-only integrations, broken flows, shallow implementation, weak information hierarchy, and visual polish hiding functional failure;
- when possible, run the product and inspect it as a user with browser/simulator/MCP/screenshots, console/runtime logs, network/API checks, database/data invariants, and end-to-end flows;
- keep ceremony proportional to risk: simple changes do not require phases/sprints or a full harness suite.

### [GPT] Other verified execution profile

When `execution_backend=other_verified`:

- require an explicit human-readable executor label plus a verified path to the exact target workspace;
- require equivalent file/edit, command/build/test, runtime-validation, state, and git/snapshot capabilities before approval;
- begin implementation/resume by reading and hash-verifying the approved repo locked-plan artifact; session chat remains non-authoritative for task requirements;
- use that executor's native project instructions/harness conventions where they exist, while preserving the same lock, scope, verification, audit, approval, and commit invariants;
- record `execution_family` truthfully and `execution_environment=other_verified`; keep any private/local-only implementation detail in local run state rather than public docs or user-facing release text;
- never silently reinterpret the orchestrator/chat host itself as this executor; `other_verified` must be an explicitly selected and capability-verified coding agent path.

### [GPT] Shared harness decisions intentionally retained/removed

Retain from the former backend-specific harness approach: durable repo state, done-means contracts, skeptical evaluation, reproduce-first debugging, mechanical checks, runtime validation, truthful handoff, and load-bearing risk scaling.

Do **not** retain redundant ceremony that conflicts with this auditor: no mandatory giant discovery questionnaire when plan+repo already answer it, no human stops after every phase beyond the single mandatory **post-lock execution approval gate**, no per-phase commits, no full harness generation by default, and no second architecture authority beside the approved repo locked plan. The auditor's one-final-commit rule remains authoritative.

## [GPT] Execution after lock

1. Record baseline **before writing the repo locked-plan artifact**.
2. Persist and hash the exact validated lock in the repo at `.gpt-auditor/LOCKED_PLAN.md` (or the explicitly selected equivalent path).
3. Complete selected-backend preflight without implementation writes; never use an unverified or different backend.
4. Resolve any known lock-readiness/baseline contradiction, then re-read the repo artifact, summarize the upcoming execution compactly, and obtain the **single normal user execution approval** anchored to the root hash plus the non-material repair envelope.
5. Immediately before the first implementation write, re-read/re-hash the repo plan and require it to match the approved root/current hash.
6. Prepare only the load-bearing execution-harness context/artifacts required by the selected profile, using the repo plan as the task contract.
7. Execute the approved repo locked plan in bounded increments. On handoff/resume, re-read `current_plan_hash` and the repair chain before continuing.
8. Verify meaningful increments with targeted evidence and the backend's actual build/test/runtime paths.
9. Solve routine implementation problems locally; for bugs, reproduce/root-cause before fix when applicable.
10. Do not reopen debate for type errors, CSS, ordinary test fixes, package details, or implementation choices that remain inside the lock.
11. If new evidence requires a lock correction, classify it against the approved repair envelope. Non-material repair follows the recorded repair-chain path without another approval prompt; a material change stops for a specific user change decision.
12. When the executor believes execution is complete, build the Architect Completion Gate artifact and do not begin skeptical audit until the architect returns `ALL PASS`.

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

## [GPT] Operator experience loop

When execution includes material UX/IA/visual/affordance work, do not hide it inside one giant implementation batch. Use a **small coherent batch** — normally 1–3 perceptual issues or one repeated pattern — then pause further experience changes and ask the operator to inspect/use the product. This is a review/feedback checkpoint, **not another execution approval** and must not use `approve` language.

Record operator feedback as evidence and update affected `OPERATOR`/`MIXED` criteria to `VERIFIED`, `NOT VERIFIED`, or `FAILED`. If feedback requests another change that stays inside the approved repair envelope, continue the next small batch without a new generic approval. If it is a material contract change, use the specific change-decision path.

Passing every machine check does not end the run while operator-authority criteria remain `NOT VERIFIED`. The truthful state is `TECHNICALLY COMPLETE — OPERATOR REVIEW REQUIRED`; do not claim `DONE` or create the final commit until required operator review is complete, unless the current user explicitly waives that operator criterion.

## [GPT] Architect Completion Gate — mandatory before audit

When the executor believes the locked work is finished, set `phase=architect_completion_check`. This gate exists to catch **omitted locked work before skeptical audit begins**; it is not another architecture debate and not a second user approval.

1. Re-read the current repo locked-plan artifact and verify `current_plan_hash` plus any recorded repair chain.
2. Write `execution_completion.md` in the external run-state directory and set `execution_completion_path`. It must enumerate **every locked Implementation step and every Acceptance criterion** one by one. Implementation steps use `DONE|MISSING|NOT_PROVEN|DEFERRED_TO_AUDIT|DEFERRED_TO_COMMIT|N/A`. Acceptance criteria use `VERIFIED_PRE_AUDIT|NOT_VERIFIED|FAILED|DEFERRED_TO_AUDIT|DEFERRED_TO_COMMIT|N/A`. Every row needs concise evidence/reasoning, relevant changed paths, commands/runtime checks, and any deviation. `N/A` or a deferred status requires a lock/protocol-supported reason.
3. Build/update the canonical run-delta manifest so the architect sees the complete run-owned path set.
4. Send the architect/lock owner a bounded packet containing the exact current `=== LOCKED PLAN ===` block, the completion matrix, canonical changed-path summary, and only the evidence needed to judge coverage. Do not send implementation rationale as proof.
5. The architect must check every implementation step and acceptance criterion line by line. An executor assertion alone is not proof. Each implementation `DONE` and each `VERIFIED_PRE_AUDIT` criterion needs an evidence reference or directly inspected read-only evidence. An implementation step or criterion that logically occurs only during skeptical audit/regression or after the audit at the commit gate must be explicitly `DEFERRED_TO_AUDIT` or `DEFERRED_TO_COMMIT`; that is **accounted for, not completed/verified**. The architect must reject a deferred status if the item could and should already have been executed/proven before audit.
6. The architect returns only `ALL PASS` or `INCOMPLETE`. `ALL PASS` means: every pre-audit obligation is executed/proven, every later implementation step or criterion is explicitly deferred to the correct protocol gate, every criterion due before audit is proven at the appropriate authority, and no locked item is silently omitted. `INCOMPLETE` must list the exact missing/unproven/mis-deferred step or criterion and the evidence/action needed to close it.
7. On `INCOMPLETE`, set `architect_completion_gate.status=incomplete`; return the exact gaps to the same executor, complete them within the current execution contract, refresh the matrix, and repeat this gate. Do not start skeptical audit.
8. Only on `ALL PASS` set `architect_completion_gate.status=passed` with the checked plan hash/evidence, then enter skeptical audit. Deferred criteria remain mandatory and must later become `VERIFIED` (or an explicit user waiver where allowed) before final commit.

Architect Completion Gate is a **coverage/completeness gate**, not the skeptical audit. `ALL PASS` means the executor did not omit locked work before audit and all future-gated obligations are explicitly accounted for; it does **not** mean every final acceptance criterion has already passed or that the implementation is defect-free. For `OPERATOR` or `MIXED` criteria that are due before audit, Claude may mark them complete only when the matrix includes explicit operator evidence; it must return `INCOMPLETE` rather than substituting its own perception.

### [CLAUDE-via-paste] Architect completion check template

```text
ARCHITECT COMPLETION CHECK — no architecture redesign

Current locked plan:
<exact repo LOCKED_PLAN block>

Executor completion matrix:
<every implementation step + acceptance criterion with phase-aware status/evidence>

Canonical run-owned paths:
<complete manifest summary>

Check every locked implementation step and acceptance criterion one by one. Do not infer completion from the executor's claim alone; require evidence. Pre-audit obligations must already be executed/proven. Criteria that logically belong to skeptical audit/regression or final commit may be `DEFERRED_TO_AUDIT` / `DEFERRED_TO_COMMIT`, but only when the deferment is protocol-correct and explicit. Deferred means accounted for, not verified.

Return exactly one verdict:
ALL PASS
or
INCOMPLETE
- <exact missing/unproven/mis-deferred item + required evidence/action>
```

## [GPT] Audit and finish

Audit against artifacts, not implementation intent:

- original user goal;
- **current approved repo locked-plan artifact / done-means contract**, root approval hash, current plan hash, and repair chain;
- `execution_completion.md` plus the architect's `ALL PASS` completion-gate result;
- complete canonical run-specific delta;
- fresh test/runtime/verification evidence;
- relevant existing project-harness state when present.

The orchestrator/auditor owns this pass independently from the executor. Use a fresh audit context when the host supports it cheaply; otherwise explicitly ignore remembered implementation rationale unless backed by artifacts. The audit's job is to **try to falsify “done”**, not to confirm the executor's story.

Every skeptical audit must explicitly consider all eight dimensions below and mark each `PASS|FAIL|N/A`; `N/A` requires a reason. Depth is risk-proportional inside each dimension, but none may be silently skipped:

1. **Lock conformance & coverage** — the delivered behavior and run-owned changes match the current approved lock/repair chain; no locked requirement was reinterpreted or quietly dropped despite the Architect Completion Gate.
2. **Functional & runtime correctness** — critical user flows, success/failure states, edge cases, real interactions, and actual runtime behavior work. For runnable products, exercise the product as a user when reasonably possible; build/tests alone are insufficient.
3. **Data, auth, permissions & side-effect integrity** — data invariants, migrations, authorization boundaries, OAuth/scopes/permissions, external writes, background workers, retries/idempotency, and destructive-action limits are correct when relevant. Verify that unrelated production/business data was not mutated.
4. **Regression & neighboring-flow risk** — adjacent flows, shared components/contracts, responsive states, error paths, backward compatibility, and relevant pre-existing tests still behave correctly.
5. **UX / IA / affordance / visual correctness** — when UI is in scope, check information hierarchy, labels, navigation, discoverability, false affordances, responsive behavior, empty/error/loading states, and visual quality against the locked direction. User/operator perception of affordance is valid evidence; computed styles alone cannot overrule an observed false affordance.
6. **Implementation depth & wiring** — look for stubs, TODOs, dead paths, fake/display-only integrations, placeholder success states, mocks leaking into production behavior, disconnected UI, incomplete error handling, and shallow implementation that merely looks finished. AI/integration features must drive real promised behavior rather than a demo surface.
7. **Scope, delta & pattern completeness** — every run-owned path in the canonical manifest is reviewed; no undeclared/unrelated change, accidental generated file, secret/private path, mixed dirty-file sweep, or out-of-scope side effect is hidden. Re-run a **pattern-escape check** for each material defect/fix: inspect sibling instances/components/routes/states to ensure the plan did not fix only the first observed instance while the same defect remains elsewhere. Observation may exceed modification scope.
8. **Verification quality & authority** — each acceptance criterion is supported by evidence that actually proves it; tests are meaningful rather than tautological, runtime evidence is current, baseline claims are qualified, and green build/unit tests cannot override a directly observed broken core flow. Verify that `OPERATOR`/`MIXED` criteria have actual operator evidence rather than model/DOM substitution.

Use the strongest available evidence appropriate to the task: browser/simulator/Playwright/MCP interaction, screenshots where useful, console/runtime logs, network/API traces, database/data invariants, background-job state, static/code inspection, tests, typecheck/lint/build, and end-to-end critical flows. Do not perform risky production mutations merely to make an audit deeper.

Severity:

- **P0 — blocker:** safety/data-integrity/security/destructive-action violation, unapproved material scope/permission change, missing core promised outcome, or another defect that makes delivery unsafe/invalid.
- **P1 — must-fix:** locked acceptance criterion unmet, critical/important flow broken, substantial regression, misleading/fake core behavior, or high-impact UX/implementation defect that prevents credible completion.
- **P2 — optional:** non-blocking maintainability, polish, resilience, or improvement that is outside the minimum locked done-means contract.

Output:

```text
P0 — blockers
P1 — must-fix
P2 — optional
Audit dimensions — 1..8 PASS|FAIL|N/A + concise evidence
Acceptance criteria status
Regression risks
```

Fix P0/P1 only unless the user separately requests optional work. Then rerun targeted checks plus every locked acceptance criterion. Do not run another broad audit without new evidence.

After regression passes, write `final_verification.md` in the external run-state directory, set `final_verification_path`, and include concise evidence for:

- concrete `execution_environment`;
- approved repo locked-plan root hash, current plan hash, any non-material repair chain, and confirmation that execution stayed inside the approved repair envelope;
- Architect Completion Gate result and checked plan hash;
- acceptance criteria status;
- all eight skeptical-audit dimension statuses and remaining P0/P1/P2 counts;
- canonical run-delta coverage (all run-owned paths reviewed);
- regression checks;
- **Execution deviations** — literal `NONE` or each departure from the locked operational expectation with reason, effect, and whether architecture/criteria remain valid;
- **Runtime assets / durability** when servers, workers, tunnels, schedulers, preview jobs, or similar processes matter: current health plus lifecycle status (`temporary|durable|unknown`), known lease/TTL expiry, supervisor/restart owner, and any action required to keep it alive;
- baseline-qualified warnings/known issues using only claims supported by actual pre-change evidence.

`HTTP 200 now` proves liveness, not durability. If the runtime is a temporary leased preview, say so explicitly. If durability metadata is unavailable, report `unknown` rather than implying permanence.

A sensitive/destructive action outside the lock's enumerated field is P0.

If the locked scope includes persistent project-harness files, update them only after the implementation/audit state is known: task/phase status from observed completion, QA findings from the actual audit, bugs from reproduced failures, decisions from actual locked decisions, and `HANDOFF.md` from the real continuation state. Do not create ceremonial entries or mark work complete before verification.

## [GPT] Pre-commit delivery report — visible, not an approval gate

Before the one final commit, set `phase=precommit_reporting`, write `precommit_report.md` in external run state, set `precommit_report_path`, and surface the same concise report to the user. It must contain:

- **All changes made** — every run-owned path in the final canonical manifest, each with a one-line factual description;
- **Architect Completion Gate** — `ALL PASS`, checked current plan hash, and any gap/fix cycles completed before audit;
- **Audit findings** — every P0/P1/P2 found, grouped by severity and audit dimension; literal `NONE` for an empty severity;
- **Fixes applied** — what changed to close each P0/P1, with verification evidence;
- **Verification/regression** — locked acceptance-criteria result, meaningful tests/build/runtime QA, and audit-dimension result;
- **Deviations / remaining** — execution deviations plus unresolved P2/known issues;
- **Commit scope** — exact safely stageable run-owned paths and intended commit message.

This report is **informational, not another approval prompt or wait state**. The earlier execution approval already authorizes the normal final run-owned commit. After surfacing the report, continue to commit in the same run unless the user has already said `stop`, `cancel`, or `do not commit`, or a real blocker appears. Never ask `approve` again merely because the run reached commit.

## [GPT] Final auto-commit

When the target workspace is a git repository, successful completion includes **one final commit** only after execution, required operator-review checkpoints, Architect Completion Gate `ALL PASS`, skeptical audit, required P0/P1 fixes, regression checks, every locked acceptance criterion is `VERIFIED` (or explicitly waived by the current user), and the visible pre-commit delivery report **unless the current user explicitly disabled commit for this run**. A stale `no commit` inside an inherited plan is not such an override.

Rules:

- set `phase=committing` only after verification is complete, all required `OPERATOR`/`MIXED` criteria are verified or explicitly waived, Architect Completion Gate is `passed`, and `precommit_report_path` has been written/surfaced;
- the commit contains only the safely stageable run-owned subset of the canonical run-delta manifest; pre-existing unrelated dirty paths are never included;
- use a concise task-derived commit message unless the user explicitly supplied one;
- do not create intermediate implementation/fix commits by default; consolidate the verified run into the one final commit;
- never push automatically;
- before commit, reconcile staged paths against the canonical run-delta manifest and prohibited-path rules, including new/untracked files;
- after commit, verify the new HEAD and committed path set against the manifest, then persist `final_commit_sha` and set `phase=done`.

A file that was already dirty before the run is a **mixed file**. It may be auto-committed only if the host can stage exactly the run-owned hunks/delta without including the pre-existing user edits. If exact run-only staging cannot be proven, set `error_state=commit_blocked_mixed_file`, leave the working tree intact, and stop before creating a partial or unsafe final commit. Never use path-level staging/commit as a workaround that sweeps pre-existing user work into the commit.

If the workspace is not a git repository, no commit is required; finish after verification.

## [GPT] User-facing completion report contract

After the final commit decision is resolved (committed, intentionally skipped by current user instruction, safely blocked, or waiting on required operator review), produce one concise chat report. The prose report is the user's primary completion surface; internal artifacts remain the audit record.

Use exactly these sections and include only verified facts:

```text
DONE | TECHNICALLY COMPLETE — OPERATOR REVIEW REQUIRED | BLOCKED

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
9. When machine/architect checks are complete but required operator-authority criteria remain `NOT VERIFIED`, use `TECHNICALLY COMPLETE — OPERATOR REVIEW REQUIRED`, list exactly what the operator still needs to inspect, and do not create the final commit.

## [GPT] Self-modification verification

Changes under `gpt-auditor/` follow the same proportional verification rule as other specification changes. Before commit/release, run the relevant static checks, document assertions, and scenario checks that cover the changed behavior.

E2E-1 remains available as an **optional** full orchestration diagnostic when transport/state-machine regressions are suspected or when the operator explicitly requests it. It is not automatically triggered by self-modification, and `NOT RUN` does not block commit/release. Never describe an unrun E2E as passed.

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
- implementation write before the exact validated lock is persisted and hashed inside the repo
- implementation write before selected-backend capability preflight passes
- implementation write before the one normal execution approval is anchored to the root lock hash and repair envelope
- silently continuing after a post-approval lock change that was not classified/recorded in the non-material repair chain, or after a material change without a specific user change decision
- asking the user for another generic execution `approve` after the normal approval merely because a non-material lock repair, operator-review checkpoint, audit phase, or commit phase occurred
- model/DOM/computed-style evidence marking an `OPERATOR` acceptance criterion `VERIFIED` without explicit operator evidence
- freezing a narrow modification scope around the first defect instance before a read-only sibling-pattern inventory
- starting skeptical audit before required small-batch operator review evidence is recorded for material experience work
- claiming `DONE` or creating the final commit while required operator-authority criteria remain `NOT VERIFIED`
- reconstructing execution requirements from session chat/history instead of re-reading the current approved repo locked-plan artifact + repair chain on start/resume/handoff
- starting skeptical audit before `architect_completion_gate.status=passed` for the current plan hash
- audit/scope/prohibited-path completion based only on plain `git diff` without accounting for untracked/deleted paths
- final verification missing an `Execution deviations` section
- reporting a runtime as durable when only current liveness is proven
- calling an issue `pre-existing` without matching baseline evidence
- spending a challenge round on a protocol contradiction that was mechanically knowable and should have been normalized before Round 0
- accepting a lock with an objective semantic correctness defect after the semantic sanity gate
- reporting an orchestrator/chat host as though it were the executor instead of the verified `execution_environment`
- execution after terminal BLOCKED
- completion claim without fresh evidence
- auto-pushing a repository
- creating the final auto-commit before required verification passes or before the pre-commit delivery report is written and surfaced
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
