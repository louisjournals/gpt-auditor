# [GPT] gpt-auditor Transport Contract

## [GPT] Core direction

Debate transport and execution-backend transport are separate concerns. Under the current default role profile, browser automation is one-directional:

1. GPT reads Claude/Opus's completed architect reply from the bound Claude thread.
2. GPT reasons/challenges in its current native host turn.
3. GPT writes the next challenge/contract message to the bound Claude architect thread.

GPT's own chat composer is never part of the browser transport contract. The selected post-lock execution backend (`gpt_codex` or `claude_claude_code`) does not change these default debate semantics. A custom/reverse role profile must define an equivalent verified remote-thread mapping before use.

## [GPT] Required semantic operations

A host adapter is compliant only if it can satisfy all postconditions below.

### [GPT] `verified_fill(payload)`

Postcondition: the bound Claude composer readback contains the payload's expected round sentinel before send.

Failure: return failure and do not click send or advance state.

### [GPT] `verified_send(round_tag)`

Postcondition: a **user-role** message in the bound Claude thread contains the exact GPT round tag.

Failure/ambiguity: do not blindly resend. Return control to the idempotency scan.

### [GPT] `get_latest_completed(expected_round)`

Return only the latest **assistant-role** Claude message when all are true:

- it contains `[[AUDITOR round=<expected_round> from=CLAUDE]]`
- it contains `[[END round=<expected_round>]]`
- the message is no longer streaming
- text is stable across two bounded polls

Otherwise return no completed message.

### [GPT] `wait_until_complete(expected_round, deadline)`

Poll with bounded waits until `get_latest_completed` succeeds or an explicit error/deadline state is reached. A single short browser wait expiring is not enough to classify a long Opus response as failed.

### [GPT] `verified_rename_thread(title)` — required for `new`

Postcondition: after Round 0 creates a concrete Claude conversation, the bound thread title exactly equals the current GPT/Codex session title requested for this run. Read the title back after rename; a best-effort click is not enough.

If the adapter cannot rename and verify Claude conversations, `new` mode is not compliant for this run. Do not continue under Claude's auto-generated title.

## [GPT] Role-scoped parsing

Role comes from DOM/message structure, not text proximity.

- GPT-sent challenges must be verified inside **user-role** message nodes.
- Claude replies must be read from **assistant-role** message nodes.
- A Claude reply quoting a GPT sentinel must never satisfy the GPT-send idempotency check.
- For Claude assistant messages, extract the **semantic response content**, not the entire transcript-row `innerText`. Claude transcript rows can contain thinking/status chrome, timestamps, message-action toolbars, code-block Copy buttons, screen-reader status nodes, and private-use icon glyphs that are not model output. Bind to the response-content subtree (for the current Claude DOM, the `.font-claude-response` content root or an equivalent verified semantic root), exclude interactive/control descendants such as buttons, toolbars, and hidden status-only nodes, then preserve the rendered text structure of the remaining content.
- Validate sentinels, hashes, lock parsing, and recovery artifacts against that sanitized semantic response text. UI-control glyphs or labels must never become part of a persisted Claude response or locked plan.

An adapter that can return only flattened page text, cannot distinguish roles, or cannot separate assistant content from embedded UI controls fails preflight.

## [GPT] Sentinels

Every debate message uses visible tags:

```text
[[AUDITOR round=N from=GPT]]
...
[[END round=N]]
```

Claude is instructed to answer with:

```text
[[AUDITOR round=N from=CLAUDE]]
...
[[END round=N]]
```

Round 0 uses the same pattern with `round=0`.

## [GPT] Idempotent send sequence

Before sending GPT Round N:

1. Query **user-role** messages for `[[AUDITOR round=N from=GPT]]`.
2. If present, do not send; switch to waiting for Claude Round N.
3. If absent, run `verified_fill`.
4. Click/send only after fill postcondition succeeds.
5. Run `verified_send`.
6. Only after send postcondition succeeds may state move to `challenge_sent`.

If the click may have succeeded but the postcondition was not observed, keep state at `ready_to_challenge`. On recovery, repeat step 1 first. This prevents duplicate posting without pretending an ambiguous send failed.

## [GPT] Startup choices before preflight

This gate happens **before** Claude discovery/preflight. The supplied architect plan must already be present/persistable, and the user must explicitly choose both:

1. Claude session: create a **new Claude session** or find/reuse a **same-name existing session** whose title exactly matches the current GPT/Codex session title; and
2. execution backend: **GPT/Codex** (`gpt_codex`) or **Claude/Claude Code** (`claude_claude_code`).

Until both choices are explicit and the architect plan is available, do not list/search/open/create/bind Claude chats. Do not reuse previous-run choices. Resolve the exact current GPT/Codex session title for both session modes; if the host cannot read it reliably, ask for the exact title before searching or creating Claude. For `same_name_existing`, require exactly one exact-title Claude match; zero or multiple exact matches are ambiguous and must be resolved by the user before binding.

For `same_name_existing`, bind the exact matched conversation URL before Round 0. For `new`, a verified authenticated `claude.ai/new` page may be the temporary bootstrap target; after the verified Round-0 send, capture the resulting concrete Claude conversation URL, persist it as `claude_chat_url`, append it to `thread_lineage`, then run `verified_rename_thread(requested_claude_title)`. Do not wait for/read/advance the architecture reply until both concrete URL binding and exact-title verification succeed.

Under the default role profile, verify the selected Claude model is **Opus 5 Max**. If another model is selected, switch to Opus 5 Max and verify the selector/readback before Round 0. If the adapter cannot verify or switch the model, fail preflight without transmitting auditor content. A custom role profile must verify the explicitly recorded remote model instead.

## [GPT] Preflight

Before Round 0, verify without altering the Claude composer:

- a supplied architect plan exists, has been persisted to `input_plan.md`, and is the declared debate baseline
- browser adapter capabilities exist
- adapter can distinguish user-role and assistant-role messages
- the user has explicitly chosen `new` or `same_name_existing`
- the user has explicitly chosen `gpt_codex` or `claude_claude_code`
- the exact current GPT/Codex session title is known for deterministic matching/naming
- for `same_name_existing`, the exact Claude thread URL is bound and reachable; for `new`, an authenticated new-chat bootstrap target is reachable and the adapter supports `verified_rename_thread`
- Claude session is authenticated
- selected Claude model is **Opus 5 Max** under the default role profile; if not, switch and verify before continuing
- composer exists and is addressable
- state directory is writable
- Context Packet imports the supplied plan, has been redacted, and is within the allowed transport bound or has a verified attachment path
- intended GPT model/class is confirmed when the host exposes it

If any required capability is missing, stop before sending Claude anything and name the missing capability. Do not downgrade to user-mediated browser transport automatically.

## [GPT] Execution-backend adapter boundary

The browser adapter above proves only debate transport. Post-lock execution needs a separate verified backend capability check.

For `gpt_codex`, the selected host/backend must be able to operate on the exact workspace with the required file, command/build/test, runtime-validation, and git/snapshot primitives. Record the actual environment as `chatgpt_localops` when the current ChatGPT host drives LocalOps, or `codex` only when Codex itself executes. Both map to `execution_family=openai`.

For `claude_claude_code`, there must be an actual Claude Code execution path connected to the exact workspace with equivalent file/command/build/test/verification capabilities and observable results. Record `execution_family=anthropic`, `execution_environment=claude_code`. A claude.ai chat tab by itself is **not** Claude Code and does not satisfy this requirement.

If the selected execution backend cannot be verified, preserve the lock but stop before any implementation write with `backend_adapter_failure`. Never switch to the other backend automatically.

## [GPT] ChatGPT + LocalOps mapping

Use LocalOps capabilities by postcondition, not by tool name alone.

**Background-first rule:** when a browser primitive accepts an exact `tabUrl`/tab identity, use it against the Claude thread without activating or focusing that tab. Opening, filling, sending, waiting, reading, model verification, rename, and continuation recovery should stay off the user's foreground whenever equivalent background DOM control exists. Foreground activation is a fallback only when the required postcondition cannot be achieved otherwise; record that fallback as a transport deviation.

Typical mapping:

- discover/bind Claude thread: `browser_list_tabs`
- role-scoped thread inspection: `browser_run_js` against the exact Claude URL
- broad diagnostic text only when useful: `browser_get_text`
- fill with readback: `browser_fill`
- send: `browser_click`
- rename + title verification for new sessions: browser DOM/click/fill primitives with exact title readback
- bounded waiting: `browser_wait_for`, repeated as needed for the overall deadline

`browser_get_text` alone is insufficient for idempotency because it flattens roles. Prefer role-scoped DOM inspection for send/receive verification.

## [GPT] Codex mapping

Codex may expose different browser primitives. Map its available controls to the same semantic operations.

Required capabilities are:

- exact-tab selection
- role-scoped DOM/message inspection
- composer fill with readback or equivalent confirmation
- click/send
- exact Claude-thread rename + title readback when `new` mode is selected
- bounded wait/poll

If these postconditions cannot be satisfied, the Codex host is not compliant for this run.

## [GPT] Transport and evidence budget

Keep transport/logging bounded without weakening recovery:

- persist one canonical supplied-plan artifact and one canonical Round-0 context packet; do not repeatedly print or reconstruct them in tool output;
- challenge messages contain only load-bearing objections/questions plus the required round contract; the architect's required full-plan restatement is captured once per completed round in the recovery artifact;
- never print raw Base64 payloads merely to move or prove a file/attachment. Use verified file/attachment primitives or hashes/paths instead;
- reuse already-discovered tool schemas/capability mappings during the run unless the host/toolset changed or a call proves the capability stale;
- prefer role-scoped DOM checks and targeted manifest/diff reads over broad page dumps, full-repo diffs, or repeated full-plan output;
- continuation-thread recovery may send the bounded rehydration packet because it replaces lost context; ordinary rounds must not imitate recovery verbosity.

Efficiency is subordinate to correctness: do not omit evidence required by the canonical run-delta, lock, or recovery invariants merely to save tokens.

## [GPT] Completion detection

A response is not complete merely because text exists.

Accept only when:

1. the expected Claude footer sentinel is present in the assistant-role message;
2. the DOM/message is not marked streaming;
3. text is unchanged across two bounded polls.

If the sentinel is absent after the overall deadline, classify the observed error rather than parsing partial text.

## [GPT] Error and recovery table

| Error state | Retry class | Action | Debate state |
|---|---|---|---|
| `rate_limited` | bounded retry | Back off and retry within configured attempt budget; then BLOCKED. | unchanged |
| `capacity` | bounded retry | Back off and retry within configured attempt budget; then BLOCKED. | unchanged |
| `timeout` | one bounded re-wait | Read first; do not reload before checking for an unread completed answer. If still unresolved, BLOCKED. | unchanged |
| `model_switched` | pause | Stop. Restore **Opus 5 Max** before resuming. Check model again on later reads. | unchanged |
| `thread_length_exceeded` | recover | Open continuation thread, rehydrate per `PROTOCOL.md`, record lineage. | unchanged |
| `adapter_failure` | non-retryable | Stop and name the missing/broken debate-transport postcondition. | unchanged |
| `backend_adapter_failure` | non-retryable | Preserve any valid lock; stop before implementation and name the missing selected-backend capability. Never fall back to the other backend. | locked / backend_preflight |
| `refusal` | non-retryable | Record refusal reason and stop for user review. Do not repeatedly rephrase to bypass it. | unchanged |
| `ambiguous_send` | inspect first | Run role-scoped idempotency scan before any further send. | remains `ready_to_challenge` until verified |

## [GPT] Read-before-reload invariant

Never reload or replace a Claude thread that may contain an unread answer. Inspect the bound thread first. Only after the current response is captured or explicitly classified as absent may recovery navigate/reload.

## [GPT] Model check

Under the default role profile, verify **Opus 5 Max** during preflight and again when reading each completed architect round. If Claude has switched models, do not consume that reply until Opus 5 Max is restored. Do not silently downgrade.

For an explicitly requested custom/reverse profile, state must record the selected remote model and role before debate; verify that model on every consumed remote round. Role/profile changes are user-directed configuration, not automatic fallback.

## [GPT] Continuation thread

When a continuation thread is required, bind the new exact URL before sending any continuation payload. After the rehydration message is verified in the new thread, update `thread_lineage` and continue the same round/turn state; do not reset to Round 1.
