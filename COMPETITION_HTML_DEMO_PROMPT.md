# HICOOL Competition HTML Demo Console — Execution Prompt

You are the implementation owner on the competition server. Build a professional, locally hosted HTML demo console for the existing **Collab × MTClaw** system. This is a real competition deliverable, not a visual mock-up.

Read these repository documents completely before changing code:

- `BOT_PROMPT.md`
- `CONTRACT.md`
- `ACCEPTANCE.md`
- `P0_MTCLAW_DEPLOYMENT_PROMPT.md`
- `FOUR_AGENT_TEST_AND_RECORDING_PROMPT.md`
- `OUTREACH_ROUTING_STABILITY_PROMPT.md`
- `SELF_RECORDING_HANDOFF_PROMPT.md`

Inspect the actual server repository, runtime configuration, registered tool schemas, and existing demo scripts. Preserve unrelated and uncommitted work. Do not ask the operator to copy secrets into chat, code, HTML, Git, screenshots, or logs.

## Objective

Deliver one browser-based **Collab × MTClaw Live Demo Console** that:

1. visibly demonstrates why MTClaw is needed;
2. invokes the four real Agent tools through the real MTClaw Function Router;
3. displays structured, auditable results instead of fabricated frontend animation;
4. supports a reliable 90–120 second stage demonstration and local screen recording;
5. remains honest when Creator Search uses cached fallback data;
6. looks formal and credible on a 16:9 competition projector.

The four required Agents are:

1. Campaign Brief
2. Creator Search / Match
3. Outreach Draft
4. Campaign QA

Do not replace the real MTClaw route with the legacy manual `router.py`. Do not make four direct HTTP calls from the frontend and label them “MTClaw routing.”

## Required Architecture

Use the smallest dependable architecture:

```text
Browser UI
  -> local demo backend (same origin; no secret in browser)
  -> real MTClaw POST /v1/chat/completions
  -> MTClaw-selected registered tool
  -> existing Agent wrapper
  -> existing Collab CLI/service layer when a confirmed write is requested
```

Requirements:

- Bind to `127.0.0.1` by default.
- Provide a configurable port, preferably `8787`.
- Serve frontend and demo API from the same backend.
- Read secrets only from the existing protected server environment file.
- Never return API keys or secret headers to the browser.
- Add sensible request timeouts and structured error responses.
- Do not introduce a large frontend framework unless it is already installed and proven stable.
- Prefer a low-dependency Python or existing Node server with static HTML/CSS/JS.
- Do not expose the console publicly merely to produce a link. The operator will use SSH port forwarding.

## Mandatory User Experience

Create a single 16:9-friendly dashboard with a formal dark navy / electric blue / restrained gold visual system. It must be readable from a projector, with high contrast, large type, generous spacing, and no playful gradients or consumer-chat styling.

The page must contain:

### 1. Header / Runtime status

Show real values retrieved from the runtime:

- `MTClaw Function Router: Online / Offline`
- routing model name
- upstream model name when safely available
- tool count, expected to include the registered competition tools
- current demo mode: `LIVE ROUTING`, `FALLBACK DATA`, or `OFFLINE REPLAY`
- session ID
- a conspicuous **Reset Demo** control

Never hard-code “Online” or a successful tool count.

### 2. Campaign task panel

Provide a prefilled, editable Chinese task such as:

> 为 DJI 运动相机在美国和巴西寻找适合 TikTok 和 Instagram 的户外运动及旅行创作者，粉丝量 5k–50k。先生成活动方案和候选人，再生成外联草稿并完成上线前检查。所有写入和发送动作必须先经人工确认。

Controls:

- **Run Full Demo** — preferred stage path
- **Run Current Step**
- **Reset Demo**
- optional **Presentation Mode** that hides technical noise without hiding evidence

### 3. MTClaw orchestration lane

Show a horizontal or vertical pipeline:

```text
User Request
  -> MTClaw Router
  -> Campaign Brief
  -> Creator Search
  -> Outreach Draft
  -> Campaign QA
  -> Human Confirmation
  -> Collab
```

Each step must show a real state:

- waiting
- routing
- running
- completed
- blocked
- failed

For every completed step display:

- selected tool name
- duration
- same-session evidence marker
- `requires_confirmation`
- warnings/errors count

Do not animate a step as completed before its backend result is received and validated.

### 4. Structured result area

Use polished cards/tables, not a raw JSON dump as the primary view. A collapsible **Technical Evidence** drawer may show sanitized JSON.

Campaign Brief must display:

- product
- regions
- platforms
- follower range
- audience/categories
- objectives
- missing fields such as budget, deadline, and target creator count
- draft and confirmation status

Creator Search must display exactly what the tool returns:

- initial candidate batch, expected to be three for the fixed demo
- creator name/handle
- platform, region, follower count when present
- rationale/quality indicators
- evidence URL
- source mode and collection timestamp
- exclusion reasons where supplied

If the tool returns `fallback_cached_result`, show a visible amber label and this meaning in concise Chinese:

> 最近一次成功搜索的带证据缓存；不是实时搜索，任何真实外联前必须重新验证。

Never relabel fallback data as live search.

Outreach Draft must display:

- A/B/C segmentation or the real grouping returned
- personalized basis
- subject/body preview
- `draft=true`
- `sent=false`
- `requires_confirmation=true`

There must be no send button in the competition build unless it is permanently disabled and clearly labelled “比赛演示：禁止发送.” Never call an email, DM, or external sending service.

Campaign QA must demonstrate a meaningful transition:

- first run: `blocked`, with clear blockers such as missing deadline/budget or insufficient candidates;
- after applying prepared demo inputs: `ready`, only when the real QA tool returns ready;
- show exactly which fields changed;
- never turn a failed or missing tool result into ready in the frontend.

Collab write evidence, if used, must show:

- human confirmation boundary;
- real campaign ID returned through the existing CLI/service path;
- real Match count and statuses;
- if three Matches are `EVALUATING`, display confirmed count as `0`, not `3`.

### 5. Trust and safety strip

Keep these visible throughout the demo:

- No fabricated creator data
- Human confirmation before writes
- Outreach drafts only — not sent
- Evidence URL and timestamp retained
- Cached search clearly labelled

## Real MTClaw Evidence Rules

A step passes only when all of the following are true:

1. the request went through the real Function Router HTTP endpoint;
2. the routing runtime selected or invoked the intended registered tool;
3. the structured tool result is attributable to the current session/request;
4. the frontend displays fields from that tool result;
5. there is no contradictory tool error in the same run.

Implement a backend response envelope that includes, where available:

```json
{
  "ok": true,
  "session_id": "...",
  "request_id": "...",
  "selected_agent": "campaign_brief",
  "tool_name": "campaign_brief",
  "status": "completed",
  "duration_ms": 1234,
  "requires_confirmation": true,
  "warnings": [],
  "errors": [],
  "data": {}
}
```

Use an explicit session ID for the full demo. If MTClaw does not expose sufficient structured evidence in its HTTP response, correlate only with the runtime’s actual structured history/log mechanism using request/session identifiers. Do not scrape an unrelated recent log line.

## Reliability Requirements

Provide three modes, clearly labelled:

### Live Routing

All four tools run through the real MTClaw endpoint. This is the preferred competition path.

### Fallback Data

The Router remains live, but Creator Search may return its existing `fallback_cached_result`. This is acceptable and must be labelled.

### Offline Replay

Optional last-resort display using a sanitized artifact from a previously successful complete run. It must be labelled **OFFLINE REPLAY — NOT A LIVE RUN** at all times. It is for presentation continuity only and cannot be used as acceptance evidence.

Additional safeguards:

- disable repeated clicks while a step runs;
- show a useful timeout/failure message;
- allow retrying only the failed step;
- allow resetting to a clean deterministic demo state;
- keep the last successful sanitized artifact locally;
- prevent duplicate Campaign or Match creation by default;
- require an explicit confirmation UI before any real write;
- make the default stage run read-only if the existing campaign already provides adequate evidence.

## Stage and Recording Mode

Add `?presentation=1` or an equivalent control that:

- fits into one 16:9 viewport at 1920×1080;
- uses at least approximately 18 px body text and larger key metrics;
- hides developer-only controls and verbose logs;
- keeps source mode, confirmation state, `sent=false`, QA status, and tool evidence visible;
- uses smooth, restrained transitions that reflect actual backend events;
- does not auto-scroll unpredictably;
- supports browser fullscreen.

Provide one deterministic **Run Full Demo** sequence suitable for a 90–120 second recording. It may pause briefly between successful real steps for readability, but it must not fabricate duration or success.

## Required Server Deliverables

Place the implementation inside the isolated competition worktree, in a clearly named directory such as:

```text
mtclaw_agents/demo_console/
  server.py or server.js
  static/index.html
  static/app.js
  static/styles.css
  README.md
  tests/
```

Also provide:

- one start command;
- one health command;
- one automated end-to-end rehearsal command;
- one restart-and-rehearse command;
- SSH tunnel instructions using placeholders, for example:

```bash
ssh -L 8787:127.0.0.1:8787 <SERVER_ALIAS>
```

- the operator URL:

```text
http://127.0.0.1:8787/?presentation=1
```

- a concise recording checklist for macOS;
- sanitized screenshots or a short visual QA report;
- a sanitized demo evidence JSON artifact;
- no secrets or credentials in any deliverable.

Do not promise a publicly reachable URL unless public deployment is explicitly authorized. A local URL via SSH tunnel is the expected result.

## Tests and Acceptance Criteria

Add tests for:

- UI/backend health;
- MTClaw unavailable state;
- session/request correlation;
- Campaign Brief successful route;
- Creator Search fallback labelling;
- Outreach `draft=true`, `sent=false`, confirmation required;
- QA blocked and ready responses;
- invalid or missing tool result must fail visibly;
- duplicate-click/write protection;
- secret redaction;
- reset and rerun behavior.

Before reporting completion:

1. run existing Agent/runtime tests;
2. run the new demo-console tests;
3. run the full four-Agent sequence twice consecutively;
4. restart the Function Router;
5. run the full sequence once again;
6. verify the browser at 1920×1080 with no clipped text, overlap, horizontal scrolling, or unreadable evidence;
7. confirm browser source/network output contains no API key;
8. confirm no outreach was sent;
9. confirm all cached data is labelled;
10. save sanitized evidence for all three successful rehearsals.

If Outreach still lacks a same-session structured tool result, do not declare the console complete. Apply the narrow schema/wrapper strategy in `OUTREACH_ROUTING_STABILITY_PROMPT.md`, retest the real auto-routing path, and report the exact remaining blocker if it persists.

## Required Final Report

Return a concise operator-facing report containing:

- implementation directory and Git commit;
- exact start command;
- exact SSH tunnel command with a placeholder for the server alias;
- local presentation URL;
- four-Agent rehearsal results;
- restart/rehearsal result;
- Creator Search source mode;
- confirmation and `sent=false` proof;
- campaign/Match evidence if shown;
- test commands and results;
- recording checklist;
- all unresolved issues stated honestly.

Do not say “all four Agents are working” unless each Agent has a validated real MTClaw same-session tool result in the final rehearsal. Do not describe an attractive frontend, cached JSON, direct wrapper call, or natural-language model response as proof of MTClaw execution.
