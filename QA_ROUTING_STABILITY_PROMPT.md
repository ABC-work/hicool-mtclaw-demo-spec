# P0 — Campaign QA Routing Stability Prompt

Continue in the existing isolated competition worktree. The current configuration and first three successful stages must be preserved:

- routing model: `qwen3.7-flash`
- upstream model: `qwen3.7-plus`
- Campaign Brief: real same-session MTClaw tool result confirmed
- Creator Search: real same-session result confirmed, honestly labelled `fallback_cached_result`
- Outreach Draft: real same-session result confirmed, three drafts, `sent=false`, confirmation required

Do not change the selected models, redesign the first three Agents, increase global timeout, call QA directly from the demo script, or accept a natural-language response as QA evidence.

## Goal

Make Campaign QA reliably produce two structured, same-session results through the real MTClaw Function Router:

1. `blocked`, with real blockers;
2. `ready`, after valid prepared corrections are supplied.

## Required QA Router Contract

Apply the same input-convergence strategy that stabilized Outreach. The routing model must receive a small, flat, strict schema. Prefer inputs such as:

```json
{
  "campaign_id": "<campaign_id>",
  "qa_mode": "blocked|ready",
  "current_time": "<ISO-8601 timestamp>"
}
```

If a real Campaign ID is not appropriate for the read-only rehearsal, use a stable `demo_run_id` plus `qa_mode`. Do not pass nested Campaign objects, candidate arrays, outreach drafts, or arbitrary JSON through the routing model.

The registered QA tool schema must:

- require only the minimal flat fields;
- use explicit enums where possible;
- reject unknown or empty critical values;
- clearly describe that the tool performs campaign launch-readiness validation;
- avoid aliases that previously caused `product_name/product` or `creators_needed/creator_target` drift.

## Wrapper Responsibilities

The QA wrapper, not the routing model, must assemble the full canonical QA contract by loading or constructing data from trusted existing artifacts:

- Campaign Brief result for the same demo run;
- Creator Search result for the same demo run;
- Outreach Draft result for the same demo run;
- existing Campaign/Match service queries where required;
- explicit `current_time` supplied by the demo, never machine time hidden inside the QA logic.

Normalize known aliases inside the wrapper only:

- `product_name -> product`
- `creators_needed -> creator_target`

Reject empty candidate objects and inconsistent run/session IDs. Do not invent budget, deadline, creator count, evidence URLs, or confirmation state.

For the `blocked` demonstration, use the honest incomplete campaign state and return the actual blockers, for example missing budget/deadline/target count or insufficient candidates.

For the `ready` demonstration, apply an explicit, visible prepared patch containing valid demo values. Record exactly which fields were added or changed. The QA tool must independently evaluate the corrected canonical object and return `ready`; the frontend/script must not rewrite `blocked` to `ready`.

## MTClaw Invocation and Evidence

For both QA calls:

- use the real `POST /v1/chat/completions` endpoint;
- send the existing aligned `x-openclaw-session-key` and `sessionId`;
- use short, unambiguous Chinese instructions that name the QA intent;
- require current run/session correlation;
- retrieve the structured `campaign_qa` TOOL RESULT;
- record selected tool, arguments, duration, status, warnings/errors, and conclusion;
- fail visibly if `function_name=null`, `forwarded_after_routing`, timeout, missing tool history, mismatched session, or invalid result occurs.

Do not use forced tool choice as the final auto-routing acceptance evidence. A forced invocation may be used once for schema diagnosis, but final evidence must come from normal MTClaw routing.

## Tests

Add or update tests for:

- flat schema accepts valid `campaign_id/demo_run_id`, `qa_mode`, and `current_time`;
- invalid `qa_mode` is rejected;
- alias normalization;
- empty candidates are rejected;
- incomplete canonical state returns `blocked` with at least one structured blocker;
- corrected canonical state returns `ready` with zero blockers;
- conclusion is never overridden by the demo script/UI;
- current session/run mismatch fails;
- invalid or absent MTClaw TOOL RESULT fails.

## Acceptance Sequence

1. Restart the Function Router only if the registered QA schema changed.
2. Verify `/health` and expected tool count.
3. Run three short QA routing probes for `blocked`; all must select `campaign_qa` and return structured results.
4. Run three short QA routing probes for `ready`; all must select `campaign_qa` and return structured results.
5. Run the full four-Agent Stage Demo twice consecutively.
6. Restart the Function Router.
7. Run the full four-Agent Stage Demo once more.

Every full run must show:

- Campaign Brief completed;
- Creator Search completed and fallback-labelled when applicable;
- Outreach completed with `sent=false`;
- QA `blocked` structured result;
- visible prepared field corrections;
- QA `ready` structured result;
- all relevant results correlated to the current run/session.

## Final Report

Return:

- exact QA flat schema;
- wrapper data sources and normalization mapping;
- blocked and ready probe table, including latency and selected function;
- tests and results;
- two consecutive full-run results;
- post-restart full-run result;
- sanitized evidence artifact paths;
- exact Stage Demo command;
- `SAFE TO RECORD: YES` only if every acceptance item passes, otherwise `NO` with the exact remaining blocker.

Do not begin HTML polish or video recording during this fix unless the existing implementation work can proceed independently without consuming the same Router test capacity. Functional acceptance comes first.
