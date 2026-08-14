# Agent JSON Contract

三个新 Agent 和 Creator Search 适配层应尽量统一使用以下信封。

## 输入

```json
{
  "request_id": "demo-001",
  "task": "campaign_brief",
  "context": {
    "locale": "zh-CN",
    "user_id": "demo-user",
    "current_time": "2026-08-14T12:00:00Z"
  },
  "payload": {}
}
```

`task` 允许值：

```text
campaign_brief
creator_search
outreach_draft
campaign_qa
```

## 成功输出

```json
{
  "ok": true,
  "agent": "campaign_brief",
  "action": "prepare_campaign_draft",
  "requires_confirmation": true,
  "summary": "面向用户的简短结论",
  "data": {},
  "warnings": [],
  "errors": [],
  "trace": {
    "duration_ms": 12,
    "rules_applied": []
  }
}
```

## 失败输出

```json
{
  "ok": false,
  "agent": "campaign_qa",
  "action": "check_campaign_readiness",
  "requires_confirmation": false,
  "summary": "输入无法处理",
  "data": {},
  "warnings": [],
  "errors": [
    {
      "code": "INVALID_INPUT",
      "message": "payload.campaign is required",
      "field": "payload.campaign"
    }
  ],
  "trace": {
    "duration_ms": 2,
    "rules_applied": []
  }
}
```

## CLI 约定

每个新 Agent 必须同时支持：

```bash
python3 mtclaw_agents/campaign_brief/tool.py --input examples/demo_input.json
```

以及：

```bash
echo '{"request_id":"demo","task":"campaign_brief","context":{},"payload":{}}' \
  | python3 mtclaw_agents/campaign_brief/tool.py
```

标准输出只写最终 JSON。调试日志写到标准错误，避免破坏调用方解析。

## 副作用规则

| Agent | 允许副作用 |
|---|---|
| Campaign Brief | 无，只生成草稿 |
| Creator Search | 允许读取外部数据；写入 Collab 前确认 |
| Outreach Draft | 无，严禁真实发送 |
| Campaign QA | 无，只读检查 |

## 幂等与追踪

- 相同输入应产生结构一致的输出。
- QA 相同输入必须产生完全一致的业务结论。
- 所有请求携带 `request_id`。
- Router 记录被选 Agent、开始/结束时间、耗时和最终状态。
- 不在日志中输出密钥、Token、Cookie 或完整生产数据。

