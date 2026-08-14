# P0 MTClaw Deployment Prompt

将本文件完整交给负责部署的总 Bot。模型 API Key 由操作者通过服务器安全文件或其他私密方式提供，不得写入本文件、群聊或 GitHub。

---

继续执行 HICOOL 2026 Collab × MTClaw P0 集成。

模型服务已经准备，由操作者另行通过安全方式提供。严禁要求操作者在群聊中发送 API Key，严禁在日志、命令输出、截图、Git 提交或最终报告中打印密钥。

## 一、模型配置

使用：

```text
ROUTING_BASE_URL=https://dashscope.aliyuncs.com/compatible-mode/v1
ROUTING_MODEL=qwen3.7-plus

UPSTREAM_BASE_URL=https://dashscope.aliyuncs.com/compatible-mode/v1
UPSTREAM_MODEL=qwen3.7-plus
```

API Key 由操作者安全写入服务器，例如：

```text
<SECURE_ENV_PATH>
```

预期环境变量：

```text
ROUTING_BASE_URL
ROUTING_MODEL
ROUTING_API_KEY
UPSTREAM_BASE_URL
UPSTREAM_MODEL
UPSTREAM_API_KEY
```

加载时不要执行 `env`、`printenv`、`cat` 或 `set -x`，不得输出密钥。

## 二、先验证模型锚点

在部署 MTClaw 前完成两个测试。

### 1. 普通 Chat Completions

```text
POST https://dashscope.aliyuncs.com/compatible-mode/v1/chat/completions
```

确认：

- HTTP 200；
- 返回 `choices`；
- 返回模型名；
- 没有鉴权错误。

### 2. Tool Calling

发送一个包含 `tools` 和 `tool_choice=auto` 的请求，例如要求查询 Campaign 状态。

成功标准：

- HTTP 200；
- `choices[0].message.tool_calls` 存在；
- `function.name` 与测试工具一致；
- `arguments` 是可解析 JSON。

测试报告只允许输出：

- HTTP 状态；
- model；
- has_choices；
- has_tool_calls；
- selected_tool；
- error code。

严禁输出 Authorization Header、API Key 或完整环境变量。

如果 `qwen3.7-plus` 不返回 `tool_calls`：

1. 检查请求格式；
2. 用 `tool_choice` 强制指定测试函数；
3. 查看百炼官方 Function Calling 支持；
4. 必要时改用当前账户中明确支持 Function Calling 的 `qwen-plus`；
5. 不得伪造 `tool_calls`。

只有真实 `tool_calls` 测试通过后才能继续。

## 三、部署独立 MTClaw Function Router

使用现有源码：

```text
<MTCLAW_ROOT>
```

当前已知 HEAD：

```text
861fe37
```

先检查 Git 状态和官方 README，不覆盖未提交修改。

MTClaw Function Router 可以独立作为 OpenAI-compatible Provider 运行，本轮不等待 OpenClaw Gateway 配置。

优先采用独立兼容模式：

```text
fr_context_history.enabled=false
fr_context_preserve.enabled=false
```

固定演示不依赖严格多会话状态，因此暂不要求 session-bridge。

安装：

```bash
cd <MTCLAW_ROOT>
python3 -m venv .venv
. .venv/bin/activate
pip install .
```

配置：

```text
~/.function-router/config.json
~/.function-router/functions.jsonl
~/.function-router/scripts/
```

启动后验证：

```text
GET http://127.0.0.1:18790/health
```

确认：

- 服务健康；
- `tools_loaded` 数量正确；
- 没有模型鉴权错误；
- 没有配置文件解析错误。

## 四、注册真实工具

现有轻量 Agent 位于：

```text
<COLLAB_DEMO_ROOT>/mtclaw_agents/
```

注册：

```text
campaign_brief
creator_search
outreach_draft
campaign_qa
campaign_count_query
```

同时增加两个需要确认的写入工具：

```text
campaign_create_confirmed
match_add_confirmed
```

`functions.jsonl` 中函数名必须与脚本名一致：

```text
~/.function-router/scripts/campaign_brief.sh
~/.function-router/scripts/creator_search.sh
~/.function-router/scripts/outreach_draft.sh
~/.function-router/scripts/campaign_qa.sh
~/.function-router/scripts/campaign_count_query.sh
~/.function-router/scripts/campaign_create_confirmed.sh
~/.function-router/scripts/match_add_confirmed.sh
```

每个 wrapper：

- stdin 读取 JSON；
- stdout 只输出 JSON；
- 日志写 stderr；
- 成功 exit 0；
- 失败非 0；
- 不得输出密钥。

不要重写已经通过测试的 Python Agent，只做最薄 wrapper。

## 五、真实 Collab 写入

禁止直接写数据库，必须复用 Collab 现有 CLI/service：

Campaign 创建：

```bash
npm run collab -- campaign create ...
```

Campaign 回查：

```bash
npm run collab -- campaign get <campaign_id>
```

Match 写入：

```bash
npm run collab -- match add <campaignId> <creatorId> \
  --reason "..." --quotedRate 0
```

Match 回查：

```bash
npm run collab -- match list <campaignId>
```

写入工具必须要求：

```text
confirmed=true
```

如果缺失或不是 `true`，返回：

```json
{
  "ok": false,
  "error": {
    "code": "CONFIRMATION_REQUIRED"
  }
}
```

不得产生写入。

## 六、通过真实 MTClaw 触发

所有路由证据必须来自：

```text
POST http://127.0.0.1:18790/v1/chat/completions
```

不得把现有 `mtclaw_agents/router.py` 称为 MTClaw 路由证据。

固定验证：

1. “帮我整理 DJI 美国和巴西推广活动” → `campaign_brief`
2. “寻找 TikTok 和 Instagram 户外旅行创作者” → `creator_search`
3. “生成第一批候选人的联络草稿，先不要发送” → `outreach_draft`
4. “检查活动现在能不能启动” → `campaign_qa`
5. “活动有多少位已确认达人” → `campaign_count_query`

保存脱敏证据：

- 原始请求，但移除 Authorization；
- 原始响应；
- selected tool；
- Function Router 日志；
- duration；
- wrapper 结果；
- tool history。

## 七、固定比赛闭环

将达人目标从 20 人改为第一批 3 人，避免伪造或因数量不足阻塞。

完整流程：

1. MTClaw 路由到 `campaign_brief`。
2. 返回 Campaign 草稿和缺失字段。
3. 不带 `confirmed=true` 调用创建工具，确认被拒绝。
4. 用户确认后创建真实 Campaign。
5. 返回真实 `campaign_id`。
6. `campaign get` 回查成功。
7. MTClaw 路由到 `creator_search`。
8. 优先实时搜索；失败时明确使用 `fallback_cached_result`。
9. 用户确认后写入至少 3 个 Match。
10. `match list` 回查成功。
11. MTClaw 路由到 `outreach_draft`。
12. 输出 `draft`，`sent=false`。
13. MTClaw 路由到 `campaign_qa`。
14. 第一次 `blocked`。
15. 补充信息后第二次 `ready` 或 `ready_with_warnings`。
16. 数量查询走 `campaign_count_query` 短链路。

## 八、Creator Search 规则

优先调用现有真实 Creator Search，并设置严格超时。

成功：

```text
source.mode=live
```

失败：

```text
source.mode=fallback_cached_result
```

必须同时返回：

- 缓存采集时间；
- 需在外联前刷新；
- 不得把缓存描述为实时结果。

严禁伪造候选。

## 九、停止条件

当以下条件全部满足后立即停止新增功能：

- 百炼模型真实返回 `tool_calls`；
- Function Router health 正常；
- MTClaw 能选择四个 Agent；
- Campaign 被真实创建并回查；
- 至少 3 个 Match 被写入并回查；
- Outreach `sent=false`；
- QA `blocked` → `ready`；
- 简单数量查询走短链路；
- live/fallback 标签真实；
- 固定流程完整运行一次；
- 已保存脱敏证据。

不要继续做 UI 美化、复杂多 Agent、20 人搜索或大规模重构。

## 十、最后工作

完成固定流程后：

1. 保存一键启动命令；
2. 保存一键 smoke test；
3. 重启服务后复跑一次；
4. 录制完整备用演示视频；
5. 准备网络失败时的 fallback 演示；
6. 等 MTT AIBOOK 可访问后复制相同配置并复跑。

## 最终报告

报告必须包含：

1. 模型普通调用是否通过；
2. `tool_calls` 是否真实返回；
3. MTClaw health；
4. `tools_loaded`；
5. 五条路由实际选择结果；
6. 真实 `campaign_id`；
7. Campaign 回查结果；
8. Match 写入和回查数量；
9. QA 状态变化；
10. Creator Search 的 live/fallback 状态；
11. 测试命令和结果；
12. 备用视频路径；
13. 尚未完成项目。

所有结果必须脱敏。任何地方不得出现完整 API Key。
