---
name: doubao-session-handoff
display_name: 会话交接
version: "1.2.0"
description: 把当前会话压缩成一份交接文档（HANDOFF.md，写到当前项目目录），再经用户明确同意后开一个新会话继续推进。触发场景：用户说「交接」「写个 handoff」「上下文快满了」「换个会话继续」「开新会话接着聊」「新开一个对话」；或上下文占用超过约 70% 需要换车。不用于：需要无损保留全部上下文的会话分叉、原地回退，或只是写个摘要给人看（后者直接写文档即可，不必开新会话）。
license: MIT
---

# doubao-session-handoff

把当前会话的进展压成交接文档，让零上下文的新会话靠文档重建认知，然后在新会话继续干活。

## 硬闸口（不可跳过）

**开新会话前必须先征询用户，拿到明确同意才执行。** 不允许静默新建会话、不允许先开后报。用户只说「继续」不等于授权开新会话——那只是让你接着做已有待续任务。

## Step 1　写交接文档

写进**当前项目目录**的 `HANDOFF.md`（同名已存在就覆盖自己上次写的；用户的其他文档不动）。模板见 `assets/HANDOFF-template.md`，必备章节：

| 章节 | 要求 |
|---|---|
| 一句话结论 | 现在处在什么状态、下一步走哪条路 |
| 已确认的事实 | 只写**已实测**的结论 + 证据来源，明确哪些仍是推测 |
| 关键路径 | 项目目录、产物路径、会话 ID，一律绝对路径 |
| 待续任务 | 可勾选清单，每条都能直接执行，不写「继续推进」这类空话 |
| 坑与约束 | 踩过的坑、红线、口径，必须带过去 |
| 接续提示词 | 一段可直接粘贴的首句，指向本文件 |

写作纪律：

- **有损压缩**：只留新会话真正需要的结论。已有产物（报告、patch、评审文档）**只引用路径/URL，不复制内容**，复制必然漂移过期。
- **脱敏**：密钥、token、密码、手机号、内部 ID 一律删掉或打码。
- **篇幅 60 行内**；超了说明该拆成「文档 + 引用」，不是继续灌。

## Step 2　征询

向用户汇报三件事，然后等回复：

1. 交接文档绝对路径；
2. 准备新开会话，首条消息用这段接续提示词（贴出来）；
3. 新会话配置（手动路径下只是告知；CLI 路径下是 `--mode work --runtime local --workspace <当前项目目录>` 等参数）。

用户可能想在旧会话里补几条遗漏再开，这一步不要抢跑。

## Step 3　开新会话（用户同意后）

**默认路径（推荐）：让用户自己点「新建对话」，把接续提示词粘进去。** 最稳，不依赖任何 CLI 登录态，也不受平台改写 workspace 的影响。

需要程序化开时，走 doubao CLI headless（可选适配层）：

```bash
# 项目 id 现取，不要硬编码：读当前会话的 project_id（无则说明当前会话不在项目下）

doubao headless sessions create "<接续提示词>" \
  --mode work --runtime local \
  --project "<当前会话的 project_id>" \
  --workspace "<当前项目目录>" \
  --json < /dev/null
```

- **必须显式传 `--project`，否则新会话不在项目下**：`sessions create` 是新会话、没有 conversationId，CLI 内部 `previousProjectId` 恒为空，项目归属只能靠 `--project` 指定（项目 id 或名称，`--project none` 表示不入项目）。
- **`--mode chat` 不能带项目**：chat 模式下带 project 会直接报错，开在项目里的会话必须用 `--mode work`。
- **不要加 `--wait`**：等待超时后 CLI 会直接把该 run cancel 掉（会话状态变 `cancelled`、reply 为空）。不带 `--wait` 拿回 `conversationId`，之后用 `doubao headless sessions status <id>` 轮询。
- 取回 `conversation_id`，告知用户回客户端在该会话继续。
- 验证：用会话列表/详情接口确认新会话真的存在（标题 + `created_at`）——这才是「真开出来了」的证据，不要凭命令退出码下结论。若要求留在项目下，再补一次详情查询核对 `project_id`。
- 想直接拉起客户端可试 `doubao headless sessions open <conversation-id>`；失败不影响"会话已创建"这一事实。

## 坑与约束

1. **不要把旧会话全文灌进新会话。** 读历史本身吃新会话上下文，等于把爆满搬过去。顺序永远是「HANDOFF 精选 → 按需定向回溯」。
2. 读历史接口通常单页最多 20 turn，超长 item 会被服务端截断（`truncated: true`），调参避免不了。
3. **headless 调用必须显式关 stdin**（尾部 `< /dev/null`），否则子进程读 EOF 会永久阻塞。
4. **不要在同一 Agent 会话内部递归派发 headless 任务**，会形成嵌套 agent 循环。若当前会话本身就是 headless 派发起来的，不要再在里面开新会话，改为把接续提示词交给用户手动开。
5. headless 本地任务默认 `AskOnRisk`，无人值守下需审批的动作会被直接拒；必要时显式传 `--permission`。
6. headless 用独立 profile 登录态，与桌面 App 隔离，但**会话数据在账号层面共享**——所以新开出来的会话客户端里看得见。
7. `sessions create` **没有 `--title`**，标题由首条消息自动生成，别指望自定义标题。
8. **`--wait` 会毁掉会话**：等待超时会 cancel 该 run，会话直接变 `cancelled`。一律不带 `--wait`，改轮询 `sessions status`。
9. **客户端会覆盖 `--workspace`**：即使 create 时指定了目录，桌面端接管会话时会新建它自己的目录并接管。因此交接文档、产物引用**一律写绝对路径**，永不依赖「会话 workspace」这个相对概念。
10. 多设备时确认目标机是用户想跑的那台；跑错机器会得到"文件不存在"的假故障。设备清单用 `doubao headless runtimes --json` 核对 deviceName。
11. **不要用 `sessions list --limit N` 核验新会话**：实测该列表可能不含刚创建的那条（排序不是 updatedAt）。核验用会话查询接口核对标题与 `created_at`。

## 版本

当前版本 **1.2.0**。

| 版本 | 日期 | 变更 |
|---|---|---|
| 1.2.0 | 2026-11-09 | Step 3 强制 `--project`：新建会话默认不继承项目归属；补充 `--mode chat` 不能带项目、开完须核对 `project_id` |
| 1.1.0 | 2026-11-09 | 更名 `doubao-session-handoff`；新增坑 11（`sessions list` 会漏刚建的会话）；坑 10 补设备命令；模板接续提示词去掉硬编码章节号 |
| 1.0.1 | 2026-11-09 | 补坑 8/9/10：`--wait` 取消 run、客户端覆盖 `--workspace`、多设备跑错机器 |
| 1.0.0 | 2026-11-09 | 初版：HANDOFF 模板 + 征询闸口 + headless 开会话 |
