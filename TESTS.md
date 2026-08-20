# [GPT] gpt-auditor Verification Suite

## [GPT] Test rule

This skill is specification, not executable orchestration code. Verification is therefore a mix of real end-to-end runs, scripted browser fault injection where available, and document assertions.

E2E-1 is an optional full-orchestration diagnostic. Normal release verification relies on the relevant scenario checks and document assertions; `NOT RUN` for E2E-1 does not block commit or release.

For every scenario record: date/run_id, host, Claude URL, setup, observed evidence, pass/fail, and any deviation.

## [GPT] E2E-1 — Real three-round orchestration

Purpose: prove the actual workflow, not just isolated browser calls.

Procedure:

1. Start a new run and persist `state.json` outside the target repo.
2. Supply a complete architect plan as run input. Under the default role profile this is the plan the user already obtained from Claude/Opus before invoking the auditor; do not ask Claude to create a second initial plan. Persist the exact supplied plan to `input_plan.md`.
3. Ask the two mandatory startup choices in one gate: Claude session (`new` or `same_name_existing`) and execution backend (`codex`, `claude_code`, or `other_verified` with a generic executor label). Do not touch Claude until both are explicit.
4. Bind a real authenticated Claude thread on **Opus 5 Max** for the default role profile. If `new` is selected, use the current orchestrator session title as `requested_claude_title`.
5. Send one redacted Round-0 Context Packet that imports/anchors the supplied architect plan with verified fill/send; Round 0 must not replace it with a fresh from-scratch plan. For `new`, capture the concrete Claude URL, rename the thread to the exact requested title, and verify the readback before continuing.
6. Run Challenges 1, 2, and 3 using sentinels and role-scoped parsing: GPT challenges; Claude/Opus revises and owns the final lock under the default role profile.
7. After every verified transition, inspect state and confirm role profile, execution backend, `round`, `turn_state`, blocker lists, URL/title state, and send/read markers are correct.
8. Confirm no Round 4 occurs unless Round 3 carries an evidence-backed blocker.
9. Receive and validate a nine-field lock.
10. Create/use a disposable **git fixture** under the run's external state/test area and keep it isolated from `gpt-auditor/` itself. Record its clean baseline and declared scope **before** writing the repo lock artifact.
11. Persist the exact validated lock to the fixture's `.gpt-auditor/LOCKED_PLAN.md`, re-read it, record its SHA-256, and confirm the new file is visible as run-owned delta.
12. Apply the selected execution-backend preflight to the fixture without implementation writes. Exercise at least one load-bearing harness principle for that backend without generating ceremonial files.
13. Re-read the repo locked-plan artifact, present a compact execution summary including executor/out-of-scope/destructive-action status, and obtain the **single normal user execution approval** anchored to the root plan hash plus the non-material repair envelope. Re-hash immediately before implementation and require equality. Do not ask for another generic `approve` later in the run.
14. Perform the approved small documentation-only implementation against that fixture **from the repo locked-plan artifact rather than chat memory** and create `execution_completion.md` enumerating every locked implementation step and acceptance criterion with phase-aware status/evidence. Pre-audit obligations must be done/proven; implementation steps or criteria that logically belong to skeptical audit/regression or final commit are explicitly `DEFERRED_TO_AUDIT` / `DEFERRED_TO_COMMIT`, not falsely completed/passed.
15. Send the current repo lock + completion matrix + canonical changed-path summary through the verified Claude transport. Require the Architect Completion Gate to check every item and return `ALL PASS` only when all pre-audit obligations are proven and every later-gated step/criterion is correctly accounted for; if it returns `INCOMPLETE`, close the exact gaps and repeat. Do not start skeptical audit before `ALL PASS`.
16. Run the deep skeptical audit across all eight audit dimensions, fix every P0/P1, rerun targeted regression plus every locked acceptance criterion, and write/surface the informational pre-commit delivery report containing all changes, completion-gate result, audit findings, fixes, verification/deviations, and exact commit scope. The report must not ask for approval.
17. Create exactly one final run-owned commit in the fixture, including the repo locked-plan artifact when it is part of the run-owned declared scope, verify the new HEAD/committed paths, and confirm no push occurred.

Pass: all steps have observable evidence; the supplied architect plan is preserved as the starting plan; both startup choices are explicit; exactly one GPT challenge exists for each executed challenge round in the default profile; no partial Claude response is consumed; state can explain the run without chat memory; the exact validated lock is persisted inside the target repo before implementation; backend preflight precedes the **single normal approval**; execution uses the current approved repo artifact + repair chain as its authority; the Architect Completion Gate returns `ALL PASS` before skeptical audit; all eight audit dimensions are explicitly judged; the pre-commit delivery report is surfaced without another approval prompt; new-session naming is verified when applicable; and the fixture ends with exactly one safe final commit and no push.

## [GPT] S01 — Partial stream

Setup: inspect Claude while it is still streaming and before the expected footer exists.

Expected: `get_latest_completed` returns nothing; round and `turn_state` do not advance.

## [GPT] S02 — Fill/readback failure

Setup: force or simulate a composer fill whose expected round tag is absent from readback.

Expected: no send occurs; state remains `ready_to_challenge`.

## [GPT] S03 — Duplicate send retry

Setup: execute the send routine twice for the same round.

Expected: exactly one user-role message contains the GPT round tag; second attempt skips send and waits.

## [GPT] S04 — Ambiguous send

Setup: send click occurs but confirmation is unavailable/uncertain.

Expected: state is not advanced to `challenge_sent`; recovery scans user-role messages first and produces exactly one total challenge message.

## [GPT] S05 — Quoted-tag false match

Setup: Claude reply quotes `[[AUDITOR round=N from=GPT]]`.

Expected: assistant-role quotation does not satisfy the GPT idempotency scan.

## [GPT] S06 — Restart at every turn state

Setup: interrupt and resume separately at `ready_to_challenge`, `challenge_sent`, `claude_complete`, `response_processed`, and `lock_pending`.

Expected: resume action follows the table; no lost round and no duplicate message.

## [GPT] S07 — Second run in same workspace

Setup: leave one interrupted run, then invoke the skill again without asking to resume.

Expected: a new run_id is created; old run is only offered as resumable.

## [GPT] S08 — Cross-host takeover

Setup: attempt to mutate an existing run from a different `owner_host` without a recorded handoff.

Expected: mutation stops until explicit handoff is recorded.

## [GPT] S08a — Startup gate blocks Claude discovery

Setup: start a fresh run with either mandatory startup choice missing.

Expected: no Claude tab listing/search/open/create/bind occurs; state remains `awaiting_startup_choices`. Both session mode and execution backend must be explicit.

## [GPT] S08b — Same-name session requires exact title

Setup: choose `same_name_existing` while the host cannot reliably expose the current orchestrator session title.

Expected: ask the user for the exact title before any Claude search; do not guess or open a near-match.

## [GPT] S08c — Same-name duplicate is ambiguous

Setup: choose `same_name_existing` and return two Claude chats with the exact requested title.

Expected: bind neither; ask the user for the intended thread identifier. Never choose by tab order, recency, or near-match.

## [GPT] S08d — New-session choice is explicit

Setup: choose `new` and also choose an execution backend.

Expected: only after both startup choices and the supplied plan are present may a new Claude chat be created/bound; the run records `claude_session_mode=new` plus the selected backend. After verified Round 0, capture and persist the resulting concrete conversation URL before reading the reply.

## [GPT] S08e — Recovery artifacts survive host restart

Setup: after Round 1 completes, terminate the GPT host before Round 2 is sent.

Expected: external state points to the stored redacted `context_packet.md`, exact Claude response artifact, and latest full `current_plan.md`; resume does not need prior chat memory.

## [GPT] S08f — Valid lock is persisted before execution

Setup: Claude returns a valid nine-field lock.

Expected: the exact lock block is saved to `locked_plan.md`, `locked_plan_path` is updated, and only then may `phase=locked` be written.

## [GPT] S08g — New-session concrete URL capture

Setup: choose `new`, successfully send Round 0 from `claude.ai/new`, but make the resulting concrete conversation URL unavailable.

Expected: do not read/advance Round 0; record `adapter_failure` and stop rather than continuing with an unbound thread.

## [GPT] S09 — Missing browser adapter

Setup: remove one required semantic capability.

Expected: preflight fails before Round 0 and names the missing postcondition; no Claude message is sent.

## [GPT] S10 — Flat-text-only adapter

Setup: adapter can read page text but cannot distinguish user/assistant message roles.

Expected: preflight fails role-distinguishability.

## [GPT] S11 — Preflight does not alter composer

Setup: record Claude composer content before and after preflight.

Expected: content is byte-identical; preflight sends nothing.

## [GPT] S12 — Claude model mismatch

Setup: bound thread is not Opus 5 Max.

Expected: adapter switches to Opus 5 Max and verifies it before context transmission; if switching/verifying is unavailable, preflight stops and sends nothing.

## [GPT] S13 — Mid-thread model switch

Setup: model changes after a completed earlier round.

Expected: detected on read; current round does not advance until Opus 5 Max is restored and verified.

## [GPT] S14 — Early lock

Setup: Claude emits a lock at Round 1 or 2.

Expected: invalid due to minimum-three-round rule; debate continues.

## [GPT] S15 — Malformed lock

Setup: lock misses/empties one required field.

Expected: targeted format repair requested; challenge round count unchanged.

## [GPT] S16 — Acceptance criterion missing method

Setup: criterion states an outcome but no verification method.

Expected: lock validation fails.

## [GPT] S17 — Acceptance criterion missing expected result

Setup: method exists but expected result is unstated.

Expected: lock validation fails.

## [GPT] S18 — Evidence-free GPT veto

Setup: GPT rejects a valid lock because it prefers a different design, with no blocking evidence.

Expected: veto is refused as preference; does not open Round 4.

## [GPT] S19 — Evidence-backed blocker at Round 3

Setup: GPT provides claim + evidence + resolution condition showing correctness/hard-constraint/executability risk.

Expected: Round 4 opens and discusses only that blocker.

## [GPT] S20 — Round ceiling

Setup: attempt Challenge 6.

Expected: refused; run ends LOCK or BLOCKED.

## [GPT] S21 — BLOCKED terminal

Setup: final blocker remains unresolved after allowed rounds.

Expected: `blocked.md` records both positions, resolving evidence, minimum user need, safest partial scope; dependent execution does not start.

## [GPT] S22 — Secret redaction

Setup: Context Packet source contains a fake API key or `.env`-style secret.

Expected: sensitive value is replaced with typed `[REDACTED:...]` placeholder and listed, or preflight stops if redaction destroys necessary meaning; raw value is never transmitted.

## [GPT] S23 — Oversized Context Packet

Setup: packet exceeds the configured host bound.

Expected: summarize to one message, use a verified attachment, or fail preflight; do not send ordinary multi-message chunks.

## [GPT] S24 — Thread-length recovery

Setup: Claude thread cannot continue due to length.

Expected: continuation thread is bound and receives original compact packet + full current plan + resolved decisions/justifications + open blockers + current round + contract + nine-field schema; old/new URLs recorded; same round resumes.

## [GPT] S25 — Rate limit

Setup: Claude reports rate limiting.

Expected: bounded backoff attempts; debate state unchanged; exhaustion becomes BLOCKED with error named.

## [GPT] S26 — Capacity

Setup: Claude reports capacity/unavailable state.

Expected: bounded backoff attempts; state unchanged; exhaustion becomes BLOCKED.

## [GPT] S27 — Timeout

Setup: overall response deadline expires.

Expected: inspect unread answer first, perform one bounded re-wait, never consume partial text; unresolved result becomes BLOCKED.

## [GPT] S28 — Refusal

Setup: Claude refuses the request.

Expected: no repeated rephrasing to get around the refusal; reason is recorded and run stops for user review.

## [GPT] S29 — Pre-existing dirty file delta

Setup: file has user edits before lock and the run later edits it.

Expected: baseline copy is captured before this run's first edit; audit delta includes only run changes.

## [GPT] S30 — Incidental scope extension

Setup: locked step requires an adjacent test/import/lockfile not named initially.

Expected: GPT records file + reason + locked step before first touch; no scope finding. Same touch without record is a finding.

## [GPT] S31 — Architectural scope expansion

Setup: incidental file reveals a new architecture or hard-constraint change is required.

Expected: do not treat as routine extension; trigger Architecture Escalation.

## [GPT] S32 — Destructive-action discipline

Setup: execution performs an approval-sensitive/destructive action absent from the lock field.

Expected: audit raises P0.

## [GPT] S33 — Fresh-context audit

Setup: host can cheaply start a fresh GPT context for audit.

Expected: auditor receives only goal, lock, run delta, and fresh evidence; remembered implementation rationale is not needed.

## [GPT] S34 — Same-context audit fallback

Setup: fresh context is unavailable.

Expected: audit explicitly treats remembered intent as non-evidence and grades only artifacts/verification.

## [GPT] S35 — Routine implementation issue

Setup: the selected execution backend hits a type error, CSS issue, ordinary test failure, or package-level implementation detail that does not invalidate the lock.

Expected: delivery resolves it within the selected backend; the architect debate is not re-entered.

## [GPT] S36 — Architecture escalation

Setup: new evidence proves a locked architectural assumption false or a required capability absent.

Expected: only the bounded Architecture Escalation packet is sent; the original 3–5 round debate is not restarted.

## [GPT] S37 — Completion evidence

Setup: implementation appears done but one locked acceptance criterion has not been freshly verified.

Expected: no completion claim; verify first.

## [GPT] S38 — New Claude session adopts host title

Setup: choose `new` while the current orchestrator session title is available.

Expected: after Round 0 creates the concrete Claude thread, rename that Claude conversation to the exact current orchestrator session title and verify the title before continuing. If the host title cannot be read reliably, ask the user for the exact desired title before creating the Claude thread.

## [GPT] S39 — Final auto-commit

Setup: run inside a git repo, complete execution, audit, P0/P1 fixes, regression checks, and every locked acceptance criterion successfully.

Expected: create exactly one final commit containing only this run's safely stageable delta, with a concise task-derived commit message. Do not push. Pre-existing unrelated dirty paths are excluded.

## [GPT] S40 — Mixed dirty file commit safety

Setup: the run edits a file that was already dirty before the run and the host cannot stage only the run-owned hunks/delta safely.

Expected: do not sweep the user's pre-existing edits into the automatic commit. Stop the commit step with an explicit `commit_blocked_mixed_file` result, leave the working tree intact, and report the affected path. Never use path-level commit as a workaround when it would include pre-existing user changes.

## [GPT] S41 — Self-modification uses proportional verification

Setup: any run changes files under `gpt-auditor/` itself.

Expected: run the relevant static checks, document assertions, and scenario checks that cover the changed behavior before commit/release. E2E-1 may be run as optional full-orchestration diagnostics but is not auto-required by self-modification; `NOT RUN` does not block release, and an unrun E2E is never reported as PASS.

## [GPT] S42 — Existing project harness is first-class context

Setup: target repo already contains one or more persistent harness files such as `CLAUDE.md`, `AGENTS.md`, `TASKS.md`, `TEST_CHECKLIST.md`, `QA_FINDINGS.md`, `BUG_LOG.md`, `DECISIONS.md`, `ARCHITECTURE.md`, or `HANDOFF.md`.

Expected: Round 0 treats the relevant current harness state as project evidence/constraints, does not regenerate or overwrite harness files merely because they exist, and includes only load-bearing excerpts needed for the decision.

## [GPT] S43 — Done-means contract is executable

Setup: an app/website/feature lock has high-level acceptance criteria but omits a key user flow, validation method/command, or explicit out-of-scope boundary needed to prove completion.

Expected: lock validation fails or requests targeted repair before execution. The executable delivery contract must make expected behavior, critical user flows, verification, and out-of-scope boundaries testable without inventing implementation detail.

## [GPT] S44 — Skeptical product QA catches shallow completion

Setup: implementation builds and tests cleanly but a core user flow is stubbed, display-only, fake, or visually polished while functionally incomplete.

Expected: audit fails the run with P0/P1 as severity warrants; passing build/tests alone cannot override broken product behavior.

## [GPT] S45 — Runnable app is validated as a user

Setup: the target is a runnable app or website and the host has browser/simulator/MCP/Playwright-equivalent access plus relevant console/network/API/database evidence.

Expected: verification exercises the locked critical user flows in the running product and inspects relevant runtime evidence. Static code inspection alone is insufficient when direct validation is reasonably available.

## [GPT] S46 — Existing harness state stays truthful after delivery

Setup: the locked plan explicitly includes updates to existing harness state files after implementation.

Expected: update only the in-scope state that changed (for example phase/task status, QA findings, bugs, decisions, handoff); record observed facts only, create no fake entries, and leave unrelated harness files untouched.

## [GPT] S47 — Startup gate collects session and execution backend

Setup: start a fresh default-profile run with neither startup choice recorded.

Expected: ask in one compact gate for (1) Claude session: `new` or `same_name_existing`, and (2) execution backend: `codex`, `claude_code`, or `other_verified` with an explicit generic executor label. No Claude discovery/navigation occurs until both are explicit.

## [GPT] S48 — Supplied architect plan is the debate baseline

Setup: the user invokes the auditor after pasting a complete plan produced by the architect outside the skill.

Expected: persist it to `input_plan.md`; Round 0 imports/anchors that plan and context but does not instruct the architect to produce another initial plan. Challenge #1 starts from the supplied plan.

## [GPT] S49 — Missing architect plan blocks before debate

Setup: invoke the auditor with no usable architect plan in current context or attached artifacts.

Expected: ask the user to provide the plan; do not synthesize a replacement plan and do not touch Claude yet.

## [GPT] S50 — Codex execution profile

Setup: select `codex` as execution backend for an app/repo task.

Expected: delivery treats repo-local artifacts as source of truth, uses `AGENTS.md` as a short routing map when present/needed, prefers mechanical enforcement (tests/lint/typecheck/scripts/invariants) over prose reminders, reproduces bugs before fixing, runs targeted pre-edit and broader post-edit checks, and captures reusable missing capabilities in the appropriate project artifact when load-bearing.

## [GPT] S51 — Claude Code execution profile

Setup: select `claude_code` as execution backend for a long-running app/repo task.

Expected: delivery uses a concise `CLAUDE.md` project contract when present/needed, preserves an executable done-means contract, maintains truthful handoff/continuation state when load-bearing, and follows build → skeptical QA → fix → QA/regression with real runtime validation where available.

## [GPT] S52 — Harness ceremony stays load-bearing

Setup: selected backend has no existing harness files and the locked task is a small bounded change.

Expected: do not generate a full harness file suite. Create/update only artifacts required to execute, verify, recover, or hand off the locked work. A long-running app may justify more persistent artifacts; a small fix may justify none.

## [GPT] S53 — Existing backend harness is respected

Setup: repo already has `AGENTS.md` and/or `CLAUDE.md` plus shared harness artifacts.

Expected: inspect relevant current content, surface conflicts with the lock, never silently overwrite, and update only files explicitly in declared scope. The locked plan outranks stale project-harness guidance for this run; durable project docs are synchronized only after verified delivery when in scope.

## [GPT] S54 — Role map is explicit and future-swappable

Setup: default role profile is used.

Expected: state records architect, challenger, lock owner, orchestrator/auditor, and execution backend separately. Current default remains Claude/Opus architect + GPT challenger/orchestrator. The user is not asked extra role questions by default. An explicitly requested custom/reverse profile may swap model families without changing the challenge-count, lock-schema, verification, audit, or commit invariants.

## [GPT] S55 — Execution backend capability failure

Setup: user selects `claude_code`, `codex`, or an explicitly named `other_verified` coding agent, but the host lacks the tools/adapter needed to execute that backend safely.

Expected: debate may still reach lock if architecture transport is available, but delivery stops before first implementation write with a named backend capability blocker. Never silently execute on the other backend.

## [GPT] S56 — Legacy harness skills are not runtime dependencies

Setup: `my-cld-harness` and `my-codex-harness` have been retired after migration.

Expected: `gpt-auditor` contains the load-bearing shared/backend-profile rules itself and has no live instruction to invoke either legacy skill.

## [GPT] S57 — Complete run-delta includes untracked and deleted paths

Setup: a run modifies tracked files, creates two new untracked files, and deletes one tracked file before skeptical audit.

Expected: the canonical run-delta manifest includes all tracked modifications, staged/unstaged state, untracked new files, and deletions. Scope, prohibited-path, audit, and commit checks consume that complete manifest rather than plain `git diff` output. Audit cannot declare 0 P0/P1 while any run-owned path is absent from the reviewed manifest.

## [GPT] S58 — Runtime durability is reported, not just liveness

Setup: verification starts or observes a server/worker/tunnel/job that is currently healthy but has a lease/TTL, supervisor, restart policy, or other lifecycle semantics.

Expected: final verification records current health plus durability metadata: temporary/permanent/unknown, lease expiry when known, supervisor/restart owner when known, and any action required to keep it alive. `HTTP 200 now` alone is insufficient when lifecycle evidence is available.

## [GPT] S59 — Execution deviations are explicit

Setup: execution reasonably departs from a locked operational expectation without invalidating architecture, for example a server restart becomes necessary although the lock said no restart should be required.

Expected: `final_verification.md` contains `Execution deviations` with the observed deviation, reason, effect, and whether architecture/acceptance criteria remain valid; write literal `NONE` when there were no deviations. Do not hide a deviation merely because the final product passes.

## [GPT] S60 — Baseline claims are evidence-qualified

Setup: final verification sees warnings/errors in files outside the run delta but no pre-change baseline captured those exact warnings.

Expected: do not call them `pre-existing`. State only what evidence proves (for example `originates from files outside the run-owned delta`). A `pre-existing` claim requires matching pre-change evidence or another independently verified baseline source.

## [GPT] S61 — Protocol normalization happens before Round 1

Setup: the supplied plan contains inherited/report-only instructions that conflict with current auditor protocol, such as `no commit`, while the current user did not explicitly request that override.

Expected: before Round 0 transmission, normalize protocol conflicts into a recorded `protocol_normalization` artifact/section and present the active execution contract to the architect. Current explicit user instructions outrank auditor defaults; stale or inherited plan text does not silently override active protocol. Avoid spending Round 4 on a contradiction that was mechanically knowable before debate.

## [GPT] S62 — Lock gets semantic sanity validation

Setup: a Round-3 lock is structurally complete and testable but contains an obvious product/copy correctness defect such as protecting `1 details`, an impossible numeric invariant, or mutually inconsistent labels.

Expected: request a targeted semantic lock repair without opening a new challenge round. Do not reject for subjective taste; only repair evidence-backed obvious correctness defects in the expected result itself.

## [GPT] S63 — Orchestrator host is not an execution backend

Setup: the user selects `codex`, but the orchestrator/chat host also happens to have local workspace tools.

Expected: implementation still requires the verified Codex execution path and records `execution_family=openai`, `execution_environment=codex`. The orchestrator/chat host is never substituted as an implicit executor. Claude Code records `execution_family=anthropic`, `execution_environment=claude_code`. Public docs do not require or name a private/local-only tool stack.

## [GPT] S64 — Transport and logging stay bounded

Setup: a large supplied plan/context and several rounds would tempt repeated full dumps, Base64 output, redundant tool-schema rediscovery, or broad diff printing.

Expected: keep one canonical persisted plan/context artifact, send only load-bearing bounded challenge content plus required full-plan restatements, never print raw Base64 payloads merely for transport, reuse already-discovered tool schemas unless capabilities changed, and prefer targeted manifest/diff reads. Recovery may resend the bounded rehydration packet when genuinely required.

## [GPT] S65 — Challenger reads repo when it materially improves the challenge

Setup: the supplied/revised plan makes a load-bearing claim about current architecture or behavior that can be verified from a small set of repo files/tests.

Expected: before composing the challenge, inspect only the relevant repo evidence, record source/result, and use it to sharpen the challenge. Do not read the whole repo or make implementation edits.

## [GPT] S66 — Challenger performs targeted visual/runtime audit

Setup: an app/website plan depends on a claim about current UI hierarchy, responsive behavior, navigation, runtime state, or a visible defect that cannot be resolved confidently from code alone, and a safe current runtime is available.

Expected: inspect the relevant surface with browser/simulator/Playwright/MCP-equivalent evidence (for example exact viewport + DOM/screenshot + console/network when relevant), then challenge the architect with the concrete observation and its implication. Do not turn this into a full product audit unless the disputed claim requires it.

## [GPT] S67 — Pre-lock evidence gathering stays read-only

Setup: during Round 1 or 2, investigation reveals an obvious one-line code fix.

Expected: do not patch it before LOCK. Record the evidence and challenge the plan/acceptance criteria. Diagnostic commands must not mutate meaningful project/user data; use a safe fixture/read-only path or skip the diagnostic.

## [GPT] S68 — Evidence-on-demand is not mandatory ceremony

Setup: the challenge is conceptual and all material facts are already proven in the Context Packet/current plan.

Expected: do not run repo scans, visual QA, or redundant commands just because tools are available. Compose the challenge directly from existing evidence.

## [GPT] S69 — Challenge prompt carries concise evidence, not raw dumps

Setup: targeted repo/runtime inspection produces large logs/screenshots/source context.

Expected: outgoing challenge contains only the load-bearing claim, evidence/procedure, consequence, and required plan question/change. Persist larger evidence externally when needed; do not paste raw full logs/files into Claude by default.

## [GPT] S70 — Successful completion report is compact and fixed-shape

Setup: a run finishes successfully with a verified commit, runtime QA, and one non-blocking P2.

Expected: chat output uses `Decision → Execution → Verification → Runtime → Deviations → Remaining → Git`, names the concrete execution environment and commit, includes the P2 under Remaining, uses `Deviations: NONE` when applicable, and does not dump the full lock/diff/state artifacts.

## [GPT] S71 — Completion report preserves runtime durability

Setup: a local preview is healthy but leased/temporary.

Expected: Runtime reports health plus `Durability: temporary` and known lease/TTL/restart semantics; it must not merely say `running`.

## [GPT] S72 — Blocked run cannot look DONE

Setup: execution or commit is blocked by an unresolved P0/backend/mixed-file issue.

Expected: report starts `BLOCKED`, preserves the same seven-section order, states exact evidence and minimum next action, and makes no completion/commit claim.

## [GPT] S73 — Auditor detail is proportional

Setup: Round 1–3 materially change the plan.

Expected: completion report may include one compact round summary and at most 2–4 key decisions changed; no full debate transcript. If rounds made no material change, omit the Auditor detail by default.

## [GPT] S74 — Claude transport stays background-first

Setup: an exact Claude thread URL is already known and the browser adapter supports URL-targeted DOM operations.

Expected: open/fill/send/wait/read/model-check/rename/recovery use exact-tab background operations without stealing foreground focus. Foreground activation is used only when no equivalent background postcondition is available, and that fallback is recorded as a transport deviation.

## [GPT] S75 — Assistant extraction excludes Claude UI chrome

Setup: a completed Claude assistant response contains a fenced/code block, so the transcript row also contains a Copy button/icon plus the normal message-action toolbar/timestamp/status UI.

Expected: role-scoped extraction persists only the semantic assistant response content. Copy-button glyphs, toolbar labels/icons, timestamps, thinking/status chrome, and hidden status-only text are excluded before sentinel/hash/lock parsing. A UI glyph can never become part of `locked_plan.md` or another recovery artifact.

## [GPT] S76 — LOCK is persisted inside repo before implementation

Setup: Round 3 returns a valid lock in a git target workspace.

Expected: record baseline first, then write the exact validated lock to `.gpt-auditor/LOCKED_PLAN.md` (or explicit equivalent), re-read it, record SHA-256, and include it as run-owned delta. No implementation write occurs before this succeeds.

## [GPT] S77 — User execution approval is single-gate and root-hash anchored

Setup: repo lock artifact and backend preflight are valid, but the user has not yet answered the post-lock execution summary.

Expected: show only a compact 3–8 bullet summary derived from the repo artifact, including the selected execution path in generic capability terms, material out-of-scope boundaries, destructive-action status, root hash, and the narrow non-material repair envelope; state waits at `awaiting_execution_approval`. LOCK itself, startup backend selection, or an earlier generic `go ahead` is not execution approval. An explicit response to this gate records the run's **one normal execution approval** with `approval_root_hash=current_plan_hash`; private/local-only tool names are not exposed. Later audit/commit phases do not ask `approve` again.

## [GPT] S78 — Non-material repair keeps approval; material change asks a specific decision

Setup: the user approves root hash A. Case 1: later evidence requires a lock repair to hash B that preserves goal/product direction, permissions/scopes, destructive actions, external side-effect blast radius, executor, architecture, and acceptance burden. Case 2: a proposed change crosses one of those boundaries.

Expected: Case 1 records `{old_hash:A,new_hash:B,reason,evidence}` in `repair_chain`, updates `current_plan_hash`, and continues without another generic approval prompt. Case 2 stops affected work with `execution_approval.status=change_decision_required` and asks one specific user decision describing the material change; it does not enter a repeated `approve this hash` loop.

## [GPT] S79 — Executor follows current repo plan and repair chain, not chat memory

Setup: session history contains an old requirement that conflicts with the current approved repo locked-plan artifact, or execution resumes in a fresh context after a non-material repair.

Expected: executor re-reads and hash-checks the current repo artifact, loads the recorded repair chain, and follows that execution contract. Remembered debate/session content is non-authoritative for task requirements. A current task-changing user instruction follows the material-change decision rule rather than silently altering execution.

## [GPT] S80 — Private executor tooling stays private

Setup: the run explicitly selects `other_verified` for a coding agent whose concrete local transport/tool stack is private to one operator.

Expected: state records `execution_backend=other_verified`, a generic executor label, truthful provider family when known, and verified capability evidence. Public docs, public release artifacts, and user-facing execution summaries do not name or require the private tool stack. The orchestrator/chat host is not silently substituted as executor, and the same repo-plan/hash/approval/verification/commit invariants still apply.

## [GPT] S81 — Architect Completion Gate catches omitted locked work

Setup: executor reports the task finished but its `execution_completion.md` marks or implicitly omits one locked implementation step while the rest is done.

Expected: the architect receives the current repo lock, full completion matrix, canonical changed-path summary, and relevant evidence; it returns `INCOMPLETE` naming the exact missing/unproven item. `architect_completion_gate.status=incomplete`; skeptical audit does not start. The executor completes the gap, refreshes evidence, resubmits, and only `ALL PASS` permits audit.

## [GPT] S82 — Architect ALL PASS requires evidence or a valid later-gate deferment

Setup: executor's completion matrix claims every item is done but two pre-audit rows have no evidence reference, while another acceptance criterion can only be verified during the later skeptical audit.

Expected: architect may not return `ALL PASS` from unsupported pre-audit claims. It returns `INCOMPLETE`/requests targeted evidence for those exact rows. The audit-only criterion is explicitly `DEFERRED_TO_AUDIT`, not falsely marked verified. A valid `ALL PASS` response covers every locked Implementation step and Acceptance criterion line by line, proves all pre-audit obligations, validates any later-gate deferments, and is consumed only with the expected completion-check attempt sentinels.

## [GPT] S83 — Skeptical audit has explicit eight-dimension depth

Setup: Architect Completion Gate has passed and the implementation builds cleanly, but runtime/UX/data/regression evidence is mixed.

Expected: audit explicitly records `PASS|FAIL|N/A` with reason/evidence for all eight dimensions: lock conformance, functional/runtime, data/auth/permissions/side effects, regression, UX/IA/affordance/visual, implementation depth/wiring, scope/delta hygiene, and verification quality. It actively tries to falsify completion; green build/tests do not override a broken observed flow. P0/P1 block delivery and are fixed before regression/commit; P2 is reported but not auto-fixed by default.

## [GPT] S84 — Pre-commit report is visible and not another approval gate

Setup: execution, Architect Completion Gate, audit, required fixes, and regression all pass and one final commit is ready.

Expected: before commit, `precommit_report.md` is written and the user is shown all run-owned changes, completion-gate result, every audit finding, P0/P1 fixes, verification/deviations, remaining P2/known issues, exact commit scope, and intended message. The report does not ask for `approve` and does not create a wait state; unless the user already said stop/cancel/do-not-commit or a blocker exists, the same run proceeds to the single final commit.

## [GPT] S85 — Operator review precedes UX plan debate

Setup: a supplied UX/IA/affordance plan was written from an AI audit, but no fresh operator review of the current product is recorded.

Expected: debate does not start. The auditor asks the operator to inspect/use the product first, persists the operator findings, requires a refreshed architect plan, and only then permits Round 0. Pure perceptual findings are treated as operator evidence rather than AI debate topics.

## [GPT] S86 — Pattern completeness precedes modification scope

Setup: one route shows a duplicated title / false-affordance pattern and the initial plan scopes the fix only to that route.

Expected: before scope freeze, the auditor performs a read-only sibling-pattern inventory across analogous routes/components/states. Modification scope is decided after that inventory. A narrow write scope is allowed only with an explicit reason; observation is not limited by the write scope.

## [GPT] S87 — Operator authority cannot be substituted

Setup: an acceptance criterion says a sidebar heading must not look clickable and declares `Authority: OPERATOR`; computed styles and DOM checks look clean, but no operator perception evidence exists.

Expected: criterion status is `NOT VERIFIED`, never `VERIFIED`. Model judgment, DOM, style diff, or automated tests cannot substitute for operator evidence. `MIXED` similarly requires both evidence classes.

## [GPT] S88 — Experience work uses small review batches without approval spam

Setup: an approved run contains six UX/affordance changes inside the same approved repair envelope.

Expected: executor applies a small coherent batch (normally 1–3 issues or one repeated pattern), asks the operator to look/use the product, records feedback, then continues the next batch. The checkpoint does not use `approve` language and does not create a new execution-approval state.

## [GPT] S89 — Machine PASS is not overall completion

Setup: every MACHINE criterion and Architect Completion Gate item passes, but one required OPERATOR criterion remains `NOT VERIFIED`.

Expected: status is `TECHNICALLY COMPLETE — OPERATOR REVIEW REQUIRED`; the run names the pending operator check, does not claim `DONE`, and does not create the final commit until the operator verifies or explicitly waives that criterion.

## [GPT] S90 — Debate is limited to objective correctness

Setup: Case A is purely visual/taste work with no architecture/safety/data/correctness/executability dispute. Case B mixes a perceptual UX issue with an auth/data-integrity change.

Expected: Case A skips the adversarial 3-round debate and uses the operator loop. Case B runs the required debate only for the objective correctness track; operator findings enter as authoritative constraints/evidence and are not resolved by model consensus.

## [GPT] S91 — Completion gate does not pre-pass future audit/commit criteria

Setup: a lock contains an implementation step and criterion that only occur during skeptical audit, plus a final-commit step/criterion that only occur after audit, while all obligations due before audit are complete.

Expected: the completion matrix enumerates all of them but marks the future items `DEFERRED_TO_AUDIT` or `DEFERRED_TO_COMMIT` with protocol-supported reasons. Architect `ALL PASS` means all pre-audit obligations are executed/proven and every future-gated step/criterion is correctly accounted for; it does not pretend those future items already happened or passed. Deferred steps must later become done and deferred criteria must later become `VERIFIED` at their real gates before final completion.

## [GPT] Document assertions

After edits, inspect/grep the skill files and verify:

- public release contains the six expected root files: `SKILL.md`, `PROTOCOL.md`, `TRANSPORT.md`, `TESTS.md`, `README.md`, `LICENSE`
- every markdown section heading carries `[GPT]`, `[CLAUDE-via-paste]`, or `[HUMAN]`
- the lock schema contains nine required fields including `Approval-sensitive / destructive actions`
- no live rule requires a GPT browser-tab URL
- no numeric file/module threshold controls skill activation
- no user-mediated transport fallback is offered
- `TRANSPORT.md` requires role-scoped matching
- `PROTOCOL.md` contains literal R0–R5, lock-repair, and Architecture Escalation templates
- `TESTS.md` keeps E2E-1 as optional full-orchestration diagnostic coverage rather than a release gate
- a new Claude session is renamed to the exact current orchestrator session title before debate continues
- successful git runs auto-commit exactly one final run-owned commit and never auto-push
- mixed pre-existing dirty-file content is never swept into an automatic commit
- modifications to `gpt-auditor/` itself require relevant static/document/scenario verification, while E2E-1 remains optional
- existing project harness files are treated as evidence/constraints and are not regenerated or overwritten merely because the auditor runs
- app/website/feature locks require an executable done-means contract covering expected behavior, critical user flows, verification, and out-of-scope boundaries
- runnable products are validated with real user-flow/runtime evidence when reasonably available; clean build/tests alone cannot excuse a broken core flow
- skeptical audit explicitly checks completeness vs stubs/fake/display-only behavior and keeps harness state truthful when updates are in scope
- startup gate requires both Claude session choice and execution backend choice before Claude discovery
- a supplied architect plan is persisted and used as the Round-0 baseline; Round 0 does not regenerate the initial plan
- state separates architect/challenger/lock-owner/orchestrator from execution backend, with current default profile documented and custom/reverse roles opt-in only
- Codex, Claude Code, and explicitly selected `other_verified` execution profiles preserve their load-bearing harness principles without forcing full harness generation
- no live rule depends on `my-cld-harness` or `my-codex-harness`
- skeptical audit/scope/prohibited-path checks use a complete canonical run-delta manifest that includes untracked new files and deletions, not plain `git diff` alone
- runtime verification distinguishes liveness from lifecycle durability and reports known lease/supervisor/restart semantics
- final verification always records `Execution deviations`, using literal `NONE` when applicable
- `pre-existing` claims require actual baseline evidence; otherwise reporting is limited to what the run delta proves
- protocol conflicts mechanically knowable before debate are normalized before Round 1 while current explicit user instructions remain authoritative
- lock validation includes an objective semantic sanity repair path for obvious correctness defects without converting taste into a veto
- execution state records the concrete implementation environment (`codex`, `claude_code`, or an explicitly supported custom executor); the orchestrator/chat host is not an implicit execution backend, and public docs do not depend on a private/local-only tool stack
- transport/logging rules require canonical artifacts and bounded output; raw Base64 and redundant schema/diff dumps are not routine evidence
- assistant-role extraction persists semantic Claude response content only; transcript-row UI chrome/control glyphs are excluded before sentinel/hash/lock parsing
- after lock validation, baseline is captured before the exact lock is persisted inside the target repo as `.gpt-auditor/LOCKED_PLAN.md` (or explicit equivalent) and hashed
- LOCK is not execution permission: backend preflight passes first, then a compact repo-plan-derived summary is shown and the **single normal execution approval** is anchored to the root plan hash plus a narrow non-material repair envelope before implementation
- non-material post-approval repairs record old/new hashes and continue without another generic approval prompt; material changes stop for a specific user change decision rather than an `approve this hash` loop
- execution/resume/handoff re-reads the current approved repo locked-plan artifact + repair chain and does not reconstruct task requirements from session chat/history
- Architect Completion Gate is mandatory after execution and before skeptical audit; Claude/architect checks every locked implementation step and acceptance criterion line by line, proves all pre-audit obligations, validates any explicit `DEFERRED_TO_AUDIT` / `DEFERRED_TO_COMMIT` classifications, and only then may `ALL PASS` open audit; deferred steps/criteria remain incomplete/unverified until their real later gates
- skeptical audit explicitly judges all eight depth dimensions with `PASS|FAIL|N/A` + evidence and actively tries to falsify completion
- pre-commit delivery report is written/surfaced before the final commit, lists all changes/audit findings/fixes/verification/commit scope, and is informational rather than another approval gate
- material UX/IA/visual/affordance work requires fresh operator review before plan debate; a plan that predates that evidence is refreshed before Round 0
- pattern completeness precedes modification scope; scope fences limit writes rather than inspection
- every acceptance criterion declares `MACHINE|OPERATOR|MIXED` authority and uses `VERIFIED|NOT VERIFIED|FAILED`; operator evidence cannot be substituted by model/DOM evidence
- perceptual execution uses small operator-review batches without creating new `approve` gates
- required `OPERATOR`/`MIXED` criteria left `NOT VERIFIED` produce `TECHNICALLY COMPLETE — OPERATOR REVIEW REQUIRED` and block the final commit
- pure experience/taste tasks skip adversarial debate; mixed tasks debate only the objective correctness track
- challenge rounds may gather targeted fresh repo/runtime/visual evidence when it can materially change a blocker, lock, acceptance criterion, or prompt
- the skill/folder/state namespace is `gpt-auditor`; no live legacy-prefixed skill-name references remain
- successful and blocked user-facing reports use the fixed seven-section contract: Decision, Execution, Verification, Runtime, Deviations, Remaining, Git
- normal completion reporting is concise and does not dump full debate/lock/diff/state artifacts
- pre-lock challenge investigation is read-only and may not implement fixes or mutate meaningful project/user data
- evidence-on-demand is optional rather than mandatory ceremony; when used, the outgoing challenge carries concise provenance and implications instead of raw dumps

## [GPT] Reporting

Report a suite as:

```text
E2E-1: PASS | FAIL | NOT RUN
Scenarios: X/Y PASS
Document assertions: PASS | FAIL
Overall: PASS when all required scenario/document checks for the release pass; E2E-1 is reported separately when run
```

Never convert `NOT RUN` into an E2E pass. `NOT RUN` is acceptable because E2E-1 is optional.
