# Self-Recording Handoff Prompt

将本文件完整交给总 Bot。目标是让操作者在自己的 Mac 上安全、简单地录制备用演示视频。不要为录屏临时开发公开网站，也不要暴露 Function Router、Collab 或模型服务到公网。

---

在 Outreach 自动路由修复、完整四 Agent 彩排连续通过两次之后，进入录屏交接阶段。

如果尚未满足以下条件，先停止录屏准备并明确报告：

- MTClaw health 正常；
- `campaign_brief` 自动路由成功；
- `creator_search` 自动路由成功；
- `outreach_draft` 自动路由成功；
- `campaign_qa` 能真实展示 `blocked → ready`；
- `campaign_count_query` 走短链路；
- 完整流程重启后仍成功；
- 所有输出已脱敏。

## 一、优先采用“用户本机录制终端”

不要创建公网链接。向操作者提供：

1. SSH Host 的安全别名或由操作者已经掌握的登录方式；
2. 登录后进入演示目录的命令；
3. 一条启动完整演示的命令；
4. 一条只做快速彩排的命令；
5. 一条重启 Function Router 后复跑的命令。

不要在群里公开服务器 IP、端口、用户名、私钥路径或密码。实际 SSH 信息通过私聊或操作者现有 SSH 配置传递。

理想使用形式：

```bash
ssh -t <PRIVATE_SSH_ALIAS> '<RUN_STAGE_DEMO_COMMAND>'
```

或登录后：

```bash
cd <COLLAB_DEMO_ROOT>
python3 mtclaw_agents/mtclaw_runtime/run_stage_demo.py \
  --campaign-id <DEMO_CAMPAIGN_ID>
```

实际路径和 Campaign ID 只在私密交接中提供，不提交到公开 GitHub。

## 二、为录屏增加 presentation mode

让 `run_stage_demo.py` 支持：

```text
--presentation
--no-delay
--campaign-id
```

`--presentation` 模式要求：

- 每一步使用清晰中文标题；
- 输出自动换行，适合 100–120 列终端；
- 每一步停顿 1–2 秒；
- 只显示业务摘要，不显示原始长 JSON；
- 不显示 HTTP Authorization；
- 不显示 API Key；
- 不显示 `.env` 路径或内容；
- 不显示服务器绝对路径；
- 不显示个人信息；
- 不显示完整邮件正文，只显示主题、层级、draft/sent 状态；
- Creator Search fallback 时明确显示缓存状态和时间；
- 错误时立即停止，不能跳过失败步骤。

建议终端输出结构：

```text
COLLAB × MTCLAW — HICOOL 2026 DEMO

[0/7] MTClaw Health
health: ok
tools loaded: 12

[1/7] Campaign Brief
selected agent: campaign_brief
...

[2/7] Creator Search
selected agent: creator_search
source mode: fallback_cached_result
candidates: 3
...

[3/7] Outreach Draft
selected agent: outreach_draft
drafts: 3
sent: false
...

[4/7] Campaign QA — Before Fix
conclusion: blocked
...

[5/7] Campaign QA — After Fix
conclusion: ready
...

[6/7] Short Query
confirmed creators: 0
matches evaluating: 3

[7/7] Real Collab Evidence
campaign lookup: ok
match count: 3
```

可以使用 ANSI 颜色，但在无颜色终端也必须可读。

## 三、向操作者提供录制前检查

总 Bot最终回复中必须明确告诉操作者：

- 演示命令已经连续通过几次；
- Function Router 重启后是否通过；
- 预计演示时长；
- Creator Search 是 live 还是 fallback；
- Campaign ID 是否为演示专用；
- 是否会产生新的写入；
- 是否需要在录制前重置演示数据。

录屏命令默认应避免重复创建 Campaign 或 Match。优先使用已存在演示数据进行只读展示；如果需要写入，必须在执行前明确提示。

## 四、用户在 Mac 上的录制方式

总 Bot向操作者提供以下简短说明：

1. 关闭通知和无关窗口；
2. 打开终端并将字体调到 18–22；
3. 终端窗口调整成 16:9；
4. 运行一次 `--no-delay` 快速彩排；
5. 执行 `clear`；
6. 按 `Command + Shift + 5`；
7. 选择“录制所选部分”；
8. 框选终端；
9. 选择是否录制麦克风；
10. 开始录制后运行 `--presentation` 命令；
11. 完成后点击菜单栏停止按钮。

视频目标：

- 横版 16:9；
- 90–120 秒；
- 分辨率至少 1280×720；
- 字体清晰；
- 不出现 API Key；
- 不出现通知；
- 四个 Agent 名称全部出现；
- QA 明确展示 `blocked → ready`；
- fallback 时不得说成 live。

## 五、是否需要“可打开的链接”

默认不提供公网链接，也不要把以下服务暴露到 `0.0.0.0` 或公网：

- Function Router 18790；
- Collab 数据库；
- 内部 CLI；
-模型端点代理；
- Router debug log。

如果操作者确实需要浏览器打开，只有在现有 Collab UI 已经稳定运行时，提供 SSH 本地端口转发：

```bash
ssh -N -L <LOCAL_PORT>:127.0.0.1:<REMOTE_COLLAB_PORT> <PRIVATE_SSH_ALIAS>
```

然后只在操作者本机打开：

```text
http://127.0.0.1:<LOCAL_PORT>
```

不要创建公开隧道、临时公网域名或匿名访问链接。比赛前夜不新增 UI 时，优先录终端。

## 六、Bot 自动录制作为降级方案

如果操作者无法 SSH 或无法稳定录制，总 Bot可以生成自动录制版本，但它是降级方案。

优先方式：

1. 使用终端录制工具保存 session；
2. 将终端 session 渲染为 16:9 视频；
3. 或使用现有浏览器自动化录制已有 UI；
4. 不为录屏新建复杂 UI。

自动生成视频必须：

- 不含密钥；
- 不含 `.env`；
- 不含服务器绝对路径；
- 不含通知；
- 不含完整 debug log；
- 不含虚构结果；
- 不超过 2 分钟；
- 提供文件大小、时长、编码和 SHA-256；
- 在交付前完整播放检查一次。

## 七、录制后的检查与交付

操作者录完后，总 Bot应提供视频检查命令，例如使用 `ffprobe` 验证：

- 时长；
- 分辨率；
- 视频编码；
- 音频轨道。

目标：

```text
duration <= 120 seconds
aspect ratio = 16:9
video codec = H.264
pixel format = yuv420p
audio = AAC（如果有讲解）
```

如果需要转码，保留原文件，输出新副本，不覆盖唯一原片。

最终至少保存：

1. 比赛电脑本地副本；
2. 独立云盘或 U 盘副本；
3. PPT 嵌入版；
4. 单独 MP4 应急版。

## 八、总 Bot最终回复格式

只需回复操作者：

1. 是否已满足录屏前置条件；
2. 私密 SSH 登录方式应从哪里获取；
3. 快速彩排命令；
4. 正式录制命令；
5. 预计时长；
6. 是否会产生写入；
7. Creator Search 的 live/fallback 状态；
8. 录制失败时的自动视频降级方案；
9. 视频完成后的检查命令。

不要再新增功能。录屏准备完成后停止开发。

