# Demo Acceptance Checklist

## P0：必须通过

- [ ] 主 Collab 可启动。
- [ ] Creator Search 当前代码测试或最小 smoke test 通过。
- [ ] MTClaw Router 能选择四个 Agent。
- [ ] Campaign Brief 能识别 DJI、美国、巴西、TikTok、Instagram、5k 和 50k。
- [ ] 未提供的预算、日期和达人数量不会被编造。
- [ ] Creator Search 返回候选、数据来源和排除原因。
- [ ] Collab 能完成候选去重或明确标记重复项。
- [ ] 写入 Campaign、候选或 Match 前需要用户确认。
- [ ] Outreach 只返回草稿，不发送。
- [ ] Campaign QA 能返回规则化 blocker。
- [ ] 补充信息后 QA 能从 blocked 变为 ready 或 ready_with_warnings。
- [ ] 简单数量查询走直接工具调用。
- [ ] 非法输入返回结构化错误，不导致无信息崩溃。
- [ ] 已准备 Creator Search fallback 数据。
- [ ] 已录制完整备用演示视频。

## P1：应当通过

- [ ] UI 能显示 selected_agent。
- [ ] UI 能显示执行状态和耗时。
- [ ] UI 能显示 warnings 和 errors。
- [ ] 每个新 Agent 有正常与异常测试。
- [ ] 目标机器重启后完整场景再次通过。
- [ ] 演示数据已脱敏。

## P2：有时间再做

- [ ] UI 动画和视觉美化。
- [ ] 更多自然语言表达覆盖。
- [ ] 更多 Campaign 类型。
- [ ] 更复杂的匹配权重。
- [ ] 多轮自动恢复。
- [ ] 完整安装包。

## 现场彩排脚本

### 1. Brief

```text
我们要为 DJI 运动相机在美国和巴西寻找创作者，平台以 TikTok 和 Instagram 为主，
粉丝量 5k–50k，目标人群是户外运动和旅行爱好者。请先整理成可以执行的活动。
```

预期：Router → `campaign_brief`；返回 Campaign 草稿、缺失预算/日期/达人数量。

### 2. 补充并确认

```text
预算 20,000 美元，需要 20 位达人，活动截止到 2026-09-30。确认创建。
```

预期：显示变更摘要，确认后写入 Collab。

### 3. 搜索

```text
先从真实数据源寻找候选，保留来源证据和排除原因；给我 20 位候选。
```

预期：Router → `creator_search`；展示搜索、质量过滤、排除和去重。

### 4. 联络草稿

```text
把候选人分层，生成第一批联络邮件草稿，先不要发送。
```

预期：Router → `outreach_draft`；明确 draft，不发送。

### 5. 启动检查

```text
检查这个活动现在是否可以启动。
```

预期：Router → `campaign_qa`；先展示 blocker 或 warning，再补充/修复并复查。

### 6. 短链路

```text
这个活动目前有多少位已确认达人？
```

预期：直接查询工具，不进入完整 Agent 链路。

## 比赛前人工确认

- [ ] MTT AIBOOK 的系统时间和时区正确。
- [ ] 模型服务凭据有效。
- [ ] 外部搜索数据源可用。
- [ ] 演示账号和 Campaign 数据已准备。
- [ ] 浏览器、终端字号适合投屏。
- [ ] 网络断开时 fallback 可用。
- [ ] PPT 内嵌视频与单独视频均可播放。
- [ ] 完整演示控制在规定时长内。

