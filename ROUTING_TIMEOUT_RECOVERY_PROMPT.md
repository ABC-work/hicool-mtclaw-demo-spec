# P0 — MTClaw Routing Timeout Recovery Prompt

Continue in the existing isolated competition worktree. Do not redesign the four Agents and do not hide the failure in the HTML. The current P0 blocker is a real MTClaw routing-model timeout at Campaign Brief after 120 seconds.

## Goal

Make the real four-Agent stage path complete reliably within competition timing, while retaining truthful same-session MTClaw evidence. Do not solve this by increasing the timeout beyond 120 seconds, calling wrappers directly, replaying cached success as live, or marking skipped steps successful.

## Required Actions

1. Preserve the confirmed fixes already made:
   - flattened Outreach inputs: `campaign_id`, `batch_size`, `locale`;
   - both `x-openclaw-session-key` and `sessionId` aligned with `derive_session_key()`.

2. Inspect the current Function Router process and logs. Confirm whether the previous timed-out request is still occupying a worker. Restart the Router cleanly if necessary, then verify `/health` and registered tool count.

3. Using the already configured provider credentials without printing them, enumerate or test only models actually available to this account. Build a small routing-model probe that tests each candidate for:
   - OpenAI-compatible tool calling;
   - correct selection of `campaign_brief` from a short Chinese request;
   - valid JSON arguments;
   - latency;
   - three consecutive successes.

4. Prefer the lowest-latency available model that passes tool-calling tests. The routing model may be smaller/faster than the upstream model. Do not assume a model name is valid; record the tested model ID and sanitized results. Keep the existing upstream model unless changing it is necessary.

5. Reduce the routing prompt and tool exposure per stage:
   - keep the user request short and unambiguous;
   - expose only the tools relevant to the current stage if MTClaw supports scoped tool registration/request filtering;
   - keep schemas small, flat, and strict;
   - do not send creator arrays to the routing model;
   - let wrappers fetch Campaign/Match data internally by ID.

6. Use bounded operational behavior suitable for a live stage:
   - preflight health check;
   - one warm-up request before the audience-facing run;
   - per-stage timeout target of 20–30 seconds;
   - at most one retry after a cleanly classified timeout;
   - no overlapping retry while the first request may still be running;
   - visible structured failure if both attempts fail.

7. Add a circuit breaker to the demo console/script. A timeout must not leave the UI spinning indefinitely. Show `MTClaw routing timeout — retry available`; do not silently invoke a direct wrapper.

8. Keep two clearly separated paths:
   - `LIVE ROUTING`: only real current-session MTClaw tool results count as success;
   - `OFFLINE REPLAY`: sanitized prior evidence, always labelled as not live and never counted as acceptance evidence.

9. Re-run the fixed demo only after the routing probe passes. Acceptance requires:
   - full four-Agent sequence twice consecutively;
   - clean Function Router restart;
   - one more complete sequence;
   - each step correlated to the same run/session;
   - Creator Search source mode shown honestly;
   - Outreach remains `draft=true`, `sent=false`, `requires_confirmation=true`;
   - QA returns real `blocked`, then real `ready` after valid prepared inputs.

## Evidence

Save sanitized artifacts containing:

- routing model IDs tested;
- latency and pass/fail for each probe, without keys;
- chosen routing model and reason;
- Router health/tool count;
- request/session IDs for each final rehearsal;
- selected tool, duration, warnings/errors, and structured result for every stage;
- restart timestamp and post-restart rehearsal result.

## Stop Conditions

Stop and report the exact blocker if:

- no available model completes three consecutive tool-calling probes;
- the provider remains unavailable after one clean Router restart and bounded retry;
- same-session structured tool evidence is absent;
- any stage requires a direct wrapper call to appear successful.

Do not claim the HTML or video is ready until the three required full rehearsals pass. If live routing cannot be stabilized before the recording deadline, prepare the explicitly labelled Offline Replay for continuity and state that the live route is unavailable; never disguise it.

## Final Report

Return only an operator-ready summary:

- selected routing/upstream model;
- probe table with three-run results and latency;
- exact Router restart/start command;
- exact full-demo command;
- results of the two consecutive runs and post-restart run;
- remaining blocker, if any;
- whether it is safe to begin recording: `YES` or `NO`.
