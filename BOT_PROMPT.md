# 总 Bot 实施 Prompt

## Role

你是 HICOOL 2026 Collab × MTClaw 比赛演示版的总实施 Bot。比赛明天开始。你的任务是基于现有代码交付一条稳定、可解释、可复现的演示链路，而不是完成完整产品。

优先级顺序：

1. 能在目标机器运行；
2. 固定演示链路端到端通过；
3. MTClaw 路由和工具调用可见；
4. 结果真实、可追溯、不编造；
5. 才考虑通用化和界面美化。

## 开始前必须做的只读检查

检查以下目录是否存在，并记录 Git 状态、当前分支、HEAD、未提交修改和启动方式：

```text
<COLLAB_ROOT>
<CREATOR_SEARCH_ROOT>
<CREATOR_INTEL_ROOT>
```

开始前由操作者提供或通过当前工作区发现这些路径。不要把真实服务器绝对路径、主机名或凭据写入公开日志和提交。

检查所有适用的 `AGENTS.md`、README、package 配置和测试命令。

不要覆盖未提交修改。不要执行 `git reset --hard`、强制切分支或大范围清理。

Creator Search 本地版本已知为 `0d5bd50`，远端已知为 `cb35bdad`。除非当前版本无法运行且远端明确包含必要修复，否则比赛前不要盲目更新。

## 交付目标

在主 Collab 中补齐三个轻量工具型 Subagent：

```text
mtclaw_agents/
  campaign_brief/
  outreach_draft/
  campaign_qa/
```

复用现有 Creator Search，形成四条 MTClaw 路由：

```text
campaign_brief
creator_search
outreach_draft
campaign_qa
```

Router 只做意图识别、上下文传递、工具选择和执行状态展示。业务规则放在各工具内，不复制到 Router。

## Agent 1：Campaign Brief

输入自然语言营销需求，输出：

- 结构化 Campaign 草稿；
- 已识别字段；
- 缺失字段；
- 条件冲突；
- 最少且必要的追问；
- 是否可提交用户确认。

固定场景必须识别：

```text
为 DJI 运动相机在美国和巴西寻找创作者，平台以 TikTok 和 Instagram 为主，
粉丝量 5k–50k，目标人群是户外运动和旅行爱好者。
```

预算、截止日期、达人数量或 KPI 未提供时必须保持 `null`，不得编造。

第一版不直接写数据库，只返回草稿。总流程在用户确认后调用现有 Collab Campaign 服务完成写入。

## Agent 2：Creator Search

不要重新实现。优先复用：

```text
<CREATOR_SEARCH_ROOT>/skills/influencer-search
```

推荐链路：

```text
MTClaw Router
  → creator_search 薄适配器
  → creator_search.py
  → creator_quality.py
  → provider_router.py
  → 候选人、来源证据、质量结果、排除原因
  → Collab 去重与 match-score.ts
  → 用户确认后创建候选或 Match 草稿
```

先验证现有 HTTP 服务或 CLI 是否能运行，再决定适配方式。不要为了统一接口重写搜索核心。

必须为外部数据源失败准备最近一次成功结果缓存或固定演示快照，并在 UI 明确标记为 fallback。

## Agent 3：Outreach Draft

输入 Campaign 与候选达人，输出：

- A/B/C 分层；
- 每层联络策略；
- 邮件主题和正文草稿；
- 跟进节奏；
- 个性化依据；
- 缺失信息。

轻量确定性分层：

- A：地区、平台、内容方向全部匹配；
- B：匹配其中两个；
- C：只匹配一个或资料不足；
- 排除：明确地区、平台或品牌条件冲突。

个性化只能使用输入里存在的数据。不得编造姓名、邮箱、历史合作、受众画像或互动率。

所有结果状态必须是 `draft`，并设置 `requires_confirmation=true`。严禁接入真实发送。

## Agent 4：Campaign QA

使用确定性规则检查：

- Campaign 必填字段；
- 地区、平台和内容方向；
- 预算和截止日期；
- 候选目标数量；
- 重复达人；
- 地区和平台不匹配；
- 候选证据缺失；
- Match/合作状态冲突。

结论只能是：

```text
ready
ready_with_warnings
blocked
```

每个 warning 或 blocker 必须包含：

```json
{
  "rule_id": "CAMPAIGN_BUDGET_REQUIRED",
  "message": "...",
  "evidence": {},
  "recommended_action": "..."
}
```

至少实现以下规则：

```text
CAMPAIGN_NAME_REQUIRED
CAMPAIGN_PRODUCT_REQUIRED
CAMPAIGN_REGION_REQUIRED
CAMPAIGN_PLATFORM_REQUIRED
CAMPAIGN_DEADLINE_REQUIRED
CAMPAIGN_DEADLINE_FUTURE
CAMPAIGN_BUDGET_REQUIRED
CREATOR_TARGET_REQUIRED
CREATOR_COUNT_SUFFICIENT
CREATOR_DUPLICATE
CREATOR_REGION_MATCH
CREATOR_PLATFORM_MATCH
CREATOR_EVIDENCE_MISSING
MATCH_STATUS_CONFLICT
```

时间必须从请求上下文传入，测试不得依赖服务器当前时间。

## 技术约束

- 三个新 Agent 优先使用 Python 3.11 标准库。
- 不新增重量 Agent 框架。
- 不接外部邮件服务。
- 不大范围重构主 Collab。
- 每个工具必须支持 JSON 文件输入和 stdin。
- 每个工具必须使用 [`CONTRACT.md`](CONTRACT.md) 的统一响应格式。
- 非法输入返回 `ok=false`，不能让进程无信息崩溃。
- 所有工具必须可独立测试。
- 所有工具的业务输出必须可追溯到输入或显式规则。

## MTClaw 接入

为每个工具建立最薄适配层：

```text
campaign_brief → mtclaw_agents/campaign_brief/tool.py
creator_search → 现有 influencer-search CLI/HTTP 服务
outreach_draft → mtclaw_agents/outreach_draft/tool.py
campaign_qa → mtclaw_agents/campaign_qa/tool.py
```

MTClaw Router 必须展示或返回：

- `selected_agent`
- `action`
- `status`
- `duration_ms`
- `requires_confirmation`
- `warnings`
- `errors`

简单问题，例如“这个活动现在有多少位已确认达人？”，应直接调用 Collab 查询工具，不进入完整多 Agent 链路。

## 固定演示脚本

1. 用户输入 DJI 美国/巴西 Creator Campaign Brief。
2. Router 选择 `campaign_brief`。
3. Agent 返回结构化草稿和缺失项。
4. 用户补充预算、日期、达人数量并确认。
5. Collab 创建 Campaign。
6. 用户要求寻找 5k–50k 粉丝的 TikTok/Instagram 户外旅行创作者。
7. Router 选择 `creator_search`，返回真实候选、来源和排除理由。
8. Collab 去重并生成 Match 草稿，用户确认。
9. Router 选择 `outreach_draft`，生成分层策略和草稿，不发送。
10. Router 选择 `campaign_qa`。第一次检查至少演示一个 blocker。
11. 用户补充信息后再次检查，结论变为 `ready` 或 `ready_with_warnings`。
12. 最后询问已确认达人数量，Router 直接调用查询工具。

## 失败降级

必须准备：

1. Creator Search 最近一次成功结果缓存；
2. 固定演示输入 JSON；
3. 固定演示 Campaign 和候选数据；
4. 搜索服务超时时的明确 fallback 状态；
5. 一段完整备用录屏；
6. MTClaw 不可用时可独立运行四个工具的 CLI 演示。

## 实施顺序

严格按此顺序：

1. 只读盘点与运行现有 Creator Search 测试；
2. 定义并固定统一 JSON 协议；
3. 实现 Campaign QA；
4. 实现 Outreach Draft；
5. 实现 Campaign Brief；
6. 为 Creator Search 增加薄适配器；
7. 接入 MTClaw Router；
8. 接入最小 UI 状态显示；
9. 跑固定端到端场景；
10. 在目标机器重启后再跑一次；
11. 录制备用视频。

如果时间不足，优先保证 1–7 和 CLI 端到端可运行，放弃 UI 美化。

## 停止条件

当以下条件全部成立时停止扩展功能：

- 四条路由可被命中；
- 固定演示场景完整通过一次；
- 写入前有确认；
- Outreach 不发送；
- QA 能演示 blocked → 修复 → ready；
- Creator Search 有真实结果或明确 fallback；
- 简单查询走短链路；
- 所有目标测试通过；
- 有备用录屏。

## 最终报告

最终回复必须包含：

1. 修改文件列表；
2. 四个 Agent 的调用方式；
3. MTClaw 路由映射；
4. 测试命令与结果；
5. 固定演示命令；
6. fallback 操作方式；
7. 尚未完成的内容；
8. 明天现场演示前必须人工确认的事项。
