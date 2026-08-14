# HICOOL 2026 Collab × MTClaw Demo Spec

这是一个面向 **HICOOL 2026 MTClaw 智能体赛道预选赛**的紧急演示版实现任务书。

目标不是在比赛前完成完整产品，而是在现有 Collab 和 Creator Search 基础上，尽快交付一条稳定、真实、可解释的端到端演示链路。

## 给总 Bot 的指令

请先完整阅读：

1. [`BOT_PROMPT.md`](BOT_PROMPT.md) - 总实施 Prompt
2. [`CONTRACT.md`](CONTRACT.md) - 四个 Agent 的统一 JSON 协议
3. [`ACCEPTANCE.md`](ACCEPTANCE.md) - 演示和验收清单
4. [`P0_MTCLAW_DEPLOYMENT_PROMPT.md`](P0_MTCLAW_DEPLOYMENT_PROMPT.md) - 真实 MTClaw P0 部署
5. [`FOUR_AGENT_TEST_AND_RECORDING_PROMPT.md`](FOUR_AGENT_TEST_AND_RECORDING_PROMPT.md) - 四 Agent 测试与备用视频录制
6. [`OUTREACH_ROUTING_STABILITY_PROMPT.md`](OUTREACH_ROUTING_STABILITY_PROMPT.md) - Outreach 扁平参数与路由稳定性修复
7. [`SELF_RECORDING_HANDOFF_PROMPT.md`](SELF_RECORDING_HANDOFF_PROMPT.md) - 用户自行录制与安全访问交接

按 `BOT_PROMPT.md` 执行，不要扩大范围。

## 当前已知代码

### Creator Search / Creator Match Subagent 基础

在执行环境中将 Creator Search 仓库路径配置为：

```text
<CREATOR_SEARCH_ROOT>
```

来源：`airacle-ai/Collab` 的 `collab-search` 分支。

已知本地版本：`0d5bd50`；已知远端分支版本：`cb35bdad`。

重点代码：

```text
skills/influencer-search/
  SKILL.md
  scripts/creator_search.py
  scripts/creator_quality.py
  scripts/provider_router.py
  scripts/build_creator_report.py

creator_search_app/
  server.py
  runner.py
  briefs.py
  store.py
  static/

tests/
  test_creator_search.py
  test_creator_search_api.py
  test_creator_search_runner.py
  test_creator_search_store.py
```

### 主 Collab

将主 Collab 仓库路径配置为 `<COLLAB_ROOT>`。

现有数据库内达人搜索与确定性匹配：

```text
lib/services/creators.ts
lib/services/campaigns.ts
lib/match-score.ts
app/matching/page.tsx
```

### 独立 Creator Intel（仅作参考，不作为明日主线）

如需参考独立 Creator Intel，将其路径配置为 `<CREATOR_INTEL_ROOT>`。

GitHub：<https://github.com/airacle-ai/airacle-creator-intel>

## 明日演示主线

```text
自然语言 Brief
  → MTClaw Function Router
  → Campaign Brief Subagent
  → 用户确认 Campaign 草稿
  → Creator Search Subagent（复用现有真实实现）
  → Collab 去重与确定性 Match
  → Outreach Draft Subagent
  → Campaign QA Subagent
  → 简单数量查询直接调用工具
```

## 不在范围内

- 真实邮件群发
- 新建另一套 Creator Search
- 完整通用型 Agent 平台
- 大规模前端重构
- 模型编造缺失的达人数据
- 在 Router 中复制业务规则
- 为追求远端最新版本而盲目更新现有可运行搜索代码

## 安全约束

- 此公开仓库不得提交密钥、Token、Cookie、生产数据或私有源码。
- 所有数据写入、批量操作和外部发送必须先获得用户确认。
- 公开达人数据缺失时必须标记为待补充，不能推断或编造。
- Outreach Agent 只生成草稿，不执行发送。
