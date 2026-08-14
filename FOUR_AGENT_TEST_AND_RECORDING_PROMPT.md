# Four-Agent Test and Backup Recording Prompt

将本文件完整交给负责部署与验收的总 Bot。实际服务器路径由操作者私下提供或由 Bot 在当前工作区发现。不得把服务器绝对路径、API Key 或私有数据提交到公开仓库。

---

继续完成比赛前最后收尾，不新增业务功能。

当前重点：

1. 修复 Campaign QA 的 Router 参数规范化；
2. 明确验证 Creator Search Agent 没有遗漏；
3. 创建一键运行的四 Agent 演示脚本；
4. 输出适合屏幕录制的脱敏内容；
5. 完成 `blocked → ready` 的真实 MTClaw 证据。

## 一、修复 QA 字段规范化

在 `campaign_qa` 的 MTClaw 薄适配层加入显式字段映射：

```text
product_name → product
creators_needed → creator_target
target_count → creator_target
creator_count → creator_target
end_date → deadline
total_budget → budget
```

要求：

- 只做字段名称规范化，不修改业务值；
- 原字段和目标字段同时存在时，优先使用合同定义字段；
- 空字符串转换为 `null`；
- 空候选对象必须拒绝或忽略，不能算作有效候选；
- 不得伪造候选、预算、日期或状态。

新增针对字段映射的测试。

通过真实 MTClaw：

```text
POST http://127.0.0.1:18790/v1/chat/completions
```

留下两条日志证据：

1. 信息缺失 → `campaign_qa` → `blocked`
2. 字段补齐 → `campaign_qa` → `ready` 或 `ready_with_warnings`

## 二、确认四个 Agent

检查以下工具同时存在于 `functions.jsonl`，并被 health 统计加载：

```text
campaign_brief
creator_search
outreach_draft
campaign_qa
```

同时验证短链路：

```text
campaign_count_query
```

Creator Search 测试必须确认：

- MTClaw 实际选择 `creator_search`；
- `source.mode=fallback_cached_result` 或诚实的 `live`；
- 返回 3 位候选；
- 每位候选有来源证据或明确缺失信息；
- fallback 时返回缓存采集时间；
- fallback 时明确提示外联前重新验证；
- 不得把 fallback 描述为 live。

## 三、创建一键演示脚本

在 `<COLLAB_DEMO_ROOT>` 中创建：

```text
mtclaw_agents/mtclaw_runtime/run_stage_demo.py
```

调用方式：

```bash
python3 mtclaw_agents/mtclaw_runtime/run_stage_demo.py \
  --campaign-id <真实campaign_id>
```

脚本必须调用真实：

```text
http://127.0.0.1:18790/v1/chat/completions
```

不得调用旧的 `mtclaw_agents/router.py` 代替 MTClaw。

### STEP 0 - Health

- 显示 MTClaw health；
- 显示 `tools_loaded`；
- 不显示配置文件、环境变量或 API Key。

### STEP 1 - Campaign Brief

输入：

```text
为 DJI 运动相机在美国和巴西做推广，平台为 TikTok 和 Instagram，
先寻找第一批 3 位户外旅行创作者，请整理成 Campaign 草稿。
```

显示：

- `selected_agent=campaign_brief`
- 已识别字段
- 缺失字段
- `requires_confirmation=true`

### STEP 2 - Creator Search

输入：

```text
寻找第一批 3 位匹配这个 Campaign 的创作者，保留来源证据和排除原因。
```

显示：

- `selected_agent=creator_search`
- `source.mode`
- 3 位候选的简短摘要
- fallback 时显示缓存采集时间
- fallback 警告

不得把缓存说成实时结果。

### STEP 3 - Outreach Draft

输入：

```text
为这 3 位候选人分层并生成第一批联络草稿，先不要发送。
```

显示：

- `selected_agent=outreach_draft`
- A/B/C 分层
- 只显示一封代表性草稿
- `sent=false`
- `requires_confirmation=true`

### STEP 4 - Campaign QA blocked

使用缺少预算或截止日期的输入。

显示：

- `selected_agent=campaign_qa`
- `conclusion=blocked`
- 最多显示 3 个 blocker
- 显示 `recommended_action`

### STEP 5 - Campaign QA ready

补充：

- product
- budget
- deadline
- `creator_target=3`
- 3 位有效候选或 Match

显示：

- `selected_agent=campaign_qa`
- `conclusion=ready` 或 `ready_with_warnings`
- `blocker_count=0`

### STEP 6 - Short Query

询问：

```text
这个活动目前有多少位已确认达人？
```

显示：

- `selected_agent=campaign_count_query`
- `confirmed_creator_count=0`
- 说明现有 3 个 Match 状态为 `EVALUATING`，不伪造 confirmed。

### STEP 7 - Real Business Evidence

显示：

- 真实 `campaign_id`
- Campaign 回查成功
- `match_count=3`
- Match 状态摘要

## 四、录制友好输出

`run_stage_demo.py` 默认使用简洁输出：

- 每一步有明显标题；
- 使用中文；
- 不输出完整 HTTP 请求；
- 不输出 Authorization；
- 不输出 API Key；
- 不输出完整调试日志；
- 不输出过长 JSON；
- 每一步停顿 1–2 秒，方便录屏；
- 错误时显示明确红色或 `ERROR` 文本；
- 支持 `--no-delay` 供自动测试。

可以使用 ANSI 颜色，但必须在普通终端可读。

## 五、验证

运行现有测试和新增字段映射测试，例如：

```bash
python3 -m unittest \
  mtclaw_agents/tests/test_fixed_demo.py \
  mtclaw_agents/tests/test_mtclaw_runtime.py
```

然后运行：

```bash
python3 mtclaw_agents/mtclaw_runtime/run_stage_demo.py \
  --campaign-id <真实campaign_id>
```

要求完整成功两次：

1. 正常运行一次；
2. Function Router 重启后再运行一次。

## 六、保存证据

生成脱敏文件：

```text
mtclaw_agents/artifacts/final-four-agent-evidence.json
mtclaw_agents/artifacts/final-demo-transcript.txt
```

必须包含：

- health
- tools_loaded
- 四个 `selected_agent`
- Creator Search `source.mode`
- Outreach `sent=false`
- QA `blocked → ready`
- campaign_id
- `match_count=3`
- `confirmed_creator_count=0`
- 测试结果

不得包含 API Key、Authorization Header、完整环境变量或服务器敏感信息。

## 七、录制备用视频

目标时长：90–120 秒。

录制前：

- 清空终端；
- 将终端字号调整到 18–22；
- 隐藏聊天、API Key 页面和无关标签；
- 不打开 `.env`；
- 不展示原始 debug log；
- 关闭桌面通知；
- 终端窗口调整为 16:9；
- 先完整彩排一次。

推荐节奏：

| 时间 | 内容 |
|---:|---|
| 0–8 秒 | MTClaw health、tools loaded |
| 8–25 秒 | Campaign Brief |
| 25–48 秒 | Creator Search、3 位候选、source.mode |
| 48–65 秒 | Outreach Draft、sent=false |
| 65–90 秒 | QA blocked → 补充 → ready |
| 90–105 秒 | Campaign ID、3 个 Match |
| 105–115 秒 | 数量查询为 0，说明 Match 为 EVALUATING |
| 115–120 秒 | 结束画面 |

搜索为 fallback 时，讲解口径：

```text
MTClaw 已路由到 Creator Search。为了避免现场网络波动，本次使用带时间戳和来源证据的最近成功搜索缓存；任何真实外联前仍会刷新验证。
```

不得说成实时搜索。

录制后检查：

- 视频不超过 2 分钟；
- 没有显示 API Key；
- 没有个人通知；
- 字体清晰；
- 声音正常；
- 四个 Agent 名称都出现；
- fallback 时 `fallback_cached_result` 没被裁掉；
- QA 明确展示 `blocked → ready`。

视频至少保存到比赛电脑本地和另一份独立备份。

## 八、最终报告

报告：

1. QA 字段映射修复文件；
2. 四个 Agent 是否全部真实通过 MTClaw；
3. Creator Search 是 live 还是 fallback；
4. 一键演示命令；
5. 两次运行结果；
6. 证据文件路径；
7. 是否已经具备录屏条件；
8. 备用视频文件路径与时长。

完成以上内容后停止开发，不做 UI 美化。

