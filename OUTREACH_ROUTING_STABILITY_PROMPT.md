# Outreach Routing Stability Prompt

将本文件完整交给总 Bot。目标是修复 `outreach_draft` 在真实 MTClaw 下的路由与证据关联，不新增业务功能。

---

继续处理当前唯一阻塞：`outreach_draft` 在复杂嵌套 `creators` 参数下偶发没有可验证的同 session 结构化 tool history。

不要换模型，不要重写 Outreach Agent，不要降低证据标准。采用“扁平路由参数 + wrapper 内部补全数据 + 明确 session 关联”。

## 一、先区分两类失败

在修改前，通过脱敏日志判断本次失败属于：

1. Router 没选择 `outreach_draft`；或
2. Router 已执行工具，但 `run_stage_demo.py` 没有按正确 session 找到 history。

记录以下非敏感信息：

- request/session id；
- response model；
- selected tool（若存在）；
- HTTP status；
- tool history 中出现的 session key；
- Router debug log 是否出现 `TOOL: outreach_draft`；
- Router debug log 是否出现对应 `TOOL RESULT`。

不得输出 Authorization、API Key、完整环境变量或达人私密信息。

## 二、把 Outreach 的 Router Schema 改成扁平参数

不要再要求路由模型生成完整的嵌套 creators 数组。

将 MTClaw 注册的 `outreach_draft` 参数收敛为：

```json
{
  "type": "object",
  "properties": {
    "campaign_id": {
      "type": "string",
      "description": "Existing Collab campaign ID."
    },
    "batch_size": {
      "type": "integer",
      "description": "Number of matched creators to prepare drafts for. Use 3 for the demo.",
      "minimum": 1,
      "maximum": 3,
      "default": 3
    },
    "locale": {
      "type": "string",
      "enum": ["zh-CN", "en-US"],
      "default": "zh-CN"
    }
  },
  "required": ["campaign_id"],
  "additionalProperties": false
}
```

工具描述保持短而明确：

```text
Prepare unsent outreach email drafts for existing matched creators in a Collab campaign. Use this when the user asks to draft, prepare, tier, or plan creator outreach without sending.
```

不要在 Function Router schema 中放 Campaign 对象、Creator 对象、证据数组或邮件模板。

## 三、wrapper 内部补全数据

`outreach_draft.sh` 或对应 Python runtime adapter 收到：

```json
{
  "campaign_id": "...",
  "batch_size": 3,
  "locale": "zh-CN"
}
```

后，在 wrapper 内部：

1. 通过 Collab 现有 CLI/service 查询 Campaign；
2. 通过 `match list` 查询该 Campaign 的 Match；
3. 只选择前 3 条有效 Match；
4. 获取 Outreach Agent 真正需要的最小达人字段；
5. 转换成现有 `outreach_draft/tool.py` 合同；
6. 调用已经通过测试的 Outreach Agent；
7. 返回统一 JSON。

只允许使用：

- creator id；
- 输入中已有的姓名；
- region；
- platforms；
- categories；
- followers；
- evidence；
- missing fields。

不得编造邮箱、历史合作、受众画像、互动率或真实姓名。

如果 Campaign 不存在、Match 少于 1 条或候选数据为空，返回结构化错误，不生成虚假草稿。

所有成功输出必须：

```text
action=generate_outreach_drafts
requires_confirmation=true
sent=false
status=draft
```

## 四、修复 session 关联

`run_stage_demo.py` 为整次彩排生成唯一 session，例如：

```text
hicool-demo-<timestamp-or-uuid>
```

每次请求 Function Router 时都显式传入同一个顶层字段：

```json
{
  "sessionId": "hicool-demo-..."
}
```

如果当前 MTClaw 版本使用 `sessionKey`，按实际源码支持字段传入；优先级以当前源码的 `derive_session_key()` 为准。

在脚本启动后先做一个最小 session probe：

1. 发送数量查询；
2. 查询 tool history；
3. 确认返回记录的 session key 与请求一致。

如果 history 仍显示 `default`：

- 不要把它误判为工具未执行；
- 同时使用本次 HTTP 响应、Router debug log 中的 request/session 信息和 tool result 做三方核验；
- 在证据中明确标注 session correlation 限制；
- 不得伪造精确 session 匹配。

## 五、诊断测试：先 forced，再 auto

### A. Forced tool test

仅用于验证 schema、wrapper 和执行链路，使用：

```json
{
  "tool_choice": {
    "type": "function",
    "function": {
      "name": "outreach_draft"
    }
  }
}
```

输入只包含：

```json
{
  "campaign_id": "<真实campaign_id>",
  "batch_size": 3,
  "locale": "zh-CN"
}
```

必须验证：

- wrapper 被执行；
- 找到 3 个 Match；
- 返回草稿；
- `sent=false`；
- 无外部发送。

Forced test 不能作为 Router 自动选择成功的最终比赛证据。

### B. Auto routing test

Forced test 通过后，恢复：

```text
tool_choice=auto
```

使用简短、明确的自然语言：

```text
为 Campaign <campaign_id> 中已有的 3 位 Match 分层并生成联络邮件草稿，先不要发送。
```

成功标准：

- Qwen 路由模型自行选择 `outreach_draft`；
- 工具参数只包含 `campaign_id`、`batch_size` 和可选 `locale`；
- 真实 wrapper 执行；
- 返回 `sent=false`。

只有 Auto routing test 才能作为比赛证据。

## 六、测试

新增至少以下测试：

1. 扁平参数正常执行；
2. Campaign 不存在；
3. Campaign 没有 Match；
4. 只选择最多 3 个 Match；
5. 空候选不生成草稿；
6. `sent` 永远为 false；
7. session id 被写入请求；
8. history 不匹配时能区分执行失败与证据关联失败。

运行现有测试和新增测试，确保不破坏：

- Campaign Brief；
- Creator Search；
- Campaign QA；
- Count Query。

## 七、重新彩排

修复后执行：

1. 单独 forced Outreach test；
2. 单独 auto Outreach test；
3. 完整四 Agent 彩排；
4. 重启 Function Router；
5. 再次完整四 Agent 彩排。

完整彩排必须依次出现：

```text
campaign_brief
creator_search
outreach_draft
campaign_qa blocked
campaign_qa ready 或 ready_with_warnings
campaign_count_query
```

任何一步缺失结构化证据时继续失败退出，不得用自然语言回答冒充工具结果。

## 八、证据与停止条件

更新脱敏证据文件，包含：

- Outreach forced test 结果，标记为 diagnostic only；
- Outreach auto routing 结果，标记为 competition evidence；
- selected tool；
- arguments keys；
- `sent=false`；
- draft count；
- session correlation 状态；
- Router 重启后复跑结果。

不得保存邮件完整正文到公开证据；只保存草稿数量、状态和脱敏摘要。

当 auto routing 连续两次完整彩排成功后，立即停止代码修改并进入录屏。

## 最终报告

报告：

1. 原失败属于 Router 未选择还是 history 关联失败；
2. 修改后的 Outreach schema；
3. wrapper 如何从 Campaign/Match 补全数据；
4. forced test 结果；
5. auto routing test 结果；
6. 两次完整彩排结果；
7. session correlation 是否精确；
8. 是否已经具备录屏条件。

