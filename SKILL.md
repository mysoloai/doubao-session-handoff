---
name: doubao-session-handoff
display_name: 会话交接
version: "1.3.0"
description: 把当前会话压缩成一份交接文档（HANDOFF.md，写到当前项目目录），再经用户明确同意后自动新开会话接续推进。触发场景：用户说「交接」「写个 handoff」「上下文快满了」「换个会话继续」「开新会话接着聊」「新开一个对话」；或上下文占用超过约 70% 需要换车。专用于豆包桌面端/本地 runtime：写项目目录文档、用户同意后用 headless 自动创建新会话（失败回退为手动粘贴接续提示词）。不用于：需要无损保留全部上下文的会话分叉、原地回退，或只是写个摘要给人看（后者直接写文档即可，不必开新会话）。
---

# doubao-session-handoff（豆包版）

把当前会话的进展压成交接文档，让零上下文的新会话靠文档重建认知，然后在新会话继续干活。

## 硬闸口（不可跳过）

**开新会话前必须先征询用户，拿到明确同意才执行。** 不允许静默新建会话、不允许先开后报。用户只说「继续」不等于授权开新会话——那只是让你接着做已有待续任务。

## Step 1　写交接文档

写进**当前项目目录**的 `HANDOFF.md`（同名已存在就覆盖自己上次写的；用户的其他文档不动）。模板见 `assets/HANDOFF-template.md`，必备章节：

| 章节 | 要求 |
|---|---|
| 一句话结论 | 现在处在什么状态、下一步走哪条路 |
| 已确认的事实 | 只写**已实测**的结论 + 证据来源，明确哪些仍是推测 |
| 关键路径 | 项目目录、技能目录、产物路径、会话 ID，一律绝对路径 |
| 待续任务 | 可勾选清单，每条都能直接执行，不写「继续推进」这类空话 |
| 坑与约束 | 踩过的坑、红线、口径，必须带过去 |
| 接续提示词 | 一段可直接粘贴的首句，指向本文件 |

写作纪律：

- **有损压缩**：只留新会话真正需要的结论。已有产物（报告、patch、评审文档）**只引用路径/URL，不复制内容**，复制必然漂移过期。
- **脱敏**：密钥、token、密码、手机号、open_id 一律删掉或打码。
- 篇幅控制在 60 行内；超了说明该拆成「文档 + 引用」，不是继续灌。

## Step 2　征询

向用户汇报三件事，然后等回复：

1. 交接文档绝对路径；
2. 准备**自动新开**会话，首条消息用这段接续提示词（贴出来给你确认）；
3. 新会话配置：默认沿用当前会话的 `--mode work --runtime local --workspace <当前项目目录>`。

用户可能想在旧会话里补几条遗漏再开，这一步不要抢跑。

## Step 3　开新会话（用户同意后）

**默认路径：用 headless 程序化新开会话**，首条消息就是交接文档里那句「接续提示词」（指向 HANDOFF.md，不贴全文）。

```bash
# 项目 id 现取，不要硬编码：读当前会话的 project_id
# im_mget_conversation → items[0].conversation.project_id（无则说明当前会话不在项目下）

doubao headless sessions create "<接续提示词>" \
  --mode work --runtime local \
  --project "<当前会话的 project_id>" \
  --workspace "<当前项目目录>" \
  --json < /dev/null
```

- **必须显式传 `--project`，否则新会话不在项目下**：`sessions create` 是新会话、没有 conversationId，CLI 里 `previousProjectId` 恒为空（源码 `headless.mjs:409`），项目归属只能靠 `--project` 指定（传项目 id 或名称，`--project none` 表示不入项目）。实测：带 `--project <project_id>` 建出的会话 `im_mget_conversation` 返回对应 `project_id`；不带则无 project 字段。
- **`--mode chat` 不能带项目**：源码里 chat 模式下带 project 会直接报错，开在项目里的会话必须用 `--mode work`。
- **不要加 `--wait`**：等待超时后 CLI 会直接把该 run cancel 掉（实测会话状态变 `cancelled`、reply 为空）。不带 `--wait` 拿回 `conversationId`，之后用 `doubao headless sessions status <id>` 轮询。
- 取回 `conversation_id`，告知用户回桌面端在该会话继续。
- 验证：`im_list_conversations` 看新会话是否已出现在账号会话列表——这才是「真开出来了」的证据，不要凭命令退出码下结论。若要求留在项目下，**再补一次 `im_mget_conversation` 核对 `project_id`**，别只看标题。
- 想直接拉起客户端可试 `doubao headless sessions open <conversation-id>`；失败不影响会话已创建这一事实。

**回退路径：headless 不可用（未登录 / CLI 报错 / run 被 cancel / 超时 / 会话没出现在列表）时，立刻改为让用户自己点「新建对话」粘贴接续提示词**，一句话说明回退原因，不要反复重试同一条命令。

## 坑与约束

1. **不要把旧会话全文灌进新会话。** 读历史本身吃新会话上下文，等于把爆满搬过去。顺序永远是「HANDOFF 精选 → 按需定向回溯」。
2. `im_get_conversation_messages` 单页最多 20 turn，超长 item 会被服务端截断（`truncated: true`），调参避免不了。
3. **headless 调用必须显式关 stdin**（尾部 `< /dev/null`），否则子进程读 EOF 会永久阻塞。
4. **不要在豆包会话内部递归派发 headless 任务**，会形成嵌套 agent 循环。若当前会话本身就是 headless 派发起来的，不要再在里面开新会话，改为把接续提示词交给用户手动开。
5. headless 本地任务默认 `AskOnRisk`，无人值守下需审批的动作会被直接拒；必要时显式传 `--permission`。
6. headless 用独立 profile 登录态（keychain 里 `headless:<profile>`），与桌面 App 隔离，但**会话数据在账号层面共享**——所以新开出来的会话桌面端看得见。
7. `doubao headless sessions create` **没有 `--title`**，标题由首条消息自动生成，别指望自定义标题。
8. **`--wait` 会毁掉会话**：等待超时会 cancel 该 run，会话直接变 `cancelled`。一律不带 `--wait`，改轮询 `sessions status`。
9. **客户端会覆盖 `--workspace`**：即使 create 时指定了目录，桌面端接管会话时会新建它自己的目录（实测 create 指到 `new-chat-11`，客户端随即建 `new-chat-12` 并接管）。因此交接文档、产物引用**一律写绝对路径**，永不依赖「会话 workspace」这个相对概念。
10. 多设备时确认目标机在账号里是 `--runtime local` 且就是用户想跑的那台；跑错机器会得到"文件不存在"的假故障。设备清单用 `doubao headless runtimes --json` 核对 deviceName。
11. **不要用 `doubao headless sessions list` 核验新会话**：实测 `--limit 3` 返回的列表不含刚创建的那条（排序不是 updatedAt）。核验一律用 `im_list_conversations` 看标题与 `created_at`。

## 与 `handoff` 技能的分工

Claude Code 侧另有一个 `handoff` 技能（写 `$TMPDIR`、开 Orca 标签）。在豆包环境里**用本技能**：文档落项目目录（便于长期保留和交接给别人），开新会话走豆包自己的会话体系，不用 Orca。

## 版本

当前版本 **1.3.0**（2026-11-11）。自检：`head -5 SKILL.md` 的 `name` 应为 `doubao-session-handoff`。

| 版本 | 日期 | 变更 |
|---|---|---|
| 1.3.0 | 2026-11-11 | Step 3 默认改为 headless 自动创建新会话；原「让用户手动粘贴」降为失败回退路径；description 与 Step 2 措辞同步 |
| 1.2.0 | 2026-11-09 | Step 3 强制 `--project`：新建会话默认不继承项目归属（源码 `headless.mjs:409`），实测带/不带 `--project` 的 project_id 差异；补充 `--mode chat` 不能带项目、开完须用 `im_mget_conversation` 核对 `project_id` |
| 1.1.0 | 2026-11-09 | 更名 `doubao-session-handoff`；新增坑 11（`sessions list` 会漏刚建的会话）；坑 10 补 `runtimes --json` 设备命令与实测 deviceName；模板接续提示词去掉硬编码章节号 |
| 1.0.1 | 2026-11-09 | 补坑 8/9/10：`--wait` 取消 run、客户端覆盖 `--workspace`、多设备跑错机器 |
| 1.0.0 | 2026-11-09 | 初版：HANDOFF 模板 + 征询闸口 + headless 开会话 |
