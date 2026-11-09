# doubao-session-handoff

> 会话快聊满了，怎么把上下文交接给一个全新的会话，而不是从头讲一遍？

一个给 AI Agent 用的**会话交接技能**（Agent Skill）：把当前会话压成一份 `HANDOFF.md`，经你确认后，在新会话里用一句话接上，继续干活。

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

---

## 痛点

长会话上下文将满时，常规做法只有两个，都很难受：

| 做法 | 问题 |
|---|---|
| 硬撑着继续聊 | 后面开始丢细节、回答变浅，甚至忘记早期结论 |
| 新开会话从头讲 | 你要重述背景，不重述就会重复已经验证过的弯路 |

还有一种看起来很聪明的做法——**把旧会话全文读进新会话**。它的问题在于：读历史本身也吃新会话的上下文，等于把"快满"从一个会话搬到另一个会话，只是延后了同样的结局。

## 思路

**有损压缩 + 精确接续**，分三步：

1. **写交接文档**：在当前项目目录生成 `HANDOFF.md`，只保留新会话真正需要的东西——一句话结论、已实测的事实与证据、关键路径、可勾选的待续任务、踩过的坑、以及一段可直接粘贴的接续提示词。
2. **征询闸口**：把文档路径、接续提示词、新会话配置报给你，**等你说 OK 才开新会话**。不允许静默开、不允许先开后报。
3. **开新会话**：默认最稳的方式是你自己点「新建对话」粘贴提示词；需要程序化时再走 CLI（见下文适配层）。

核心纪律：**先精选，再按需回溯**。绝不全文灌入。

## 目录结构

```
doubao-session-handoff/
├── SKILL.md                        # 技能主文件（Agent 读取）
├── assets/HANDOFF-template.md      # 交接文档模板
├── README.md
└── LICENSE
```

## 安装

任意支持 `SKILL.md` 的 Agent（Claude Code、Doubao、其他 SKILL 加载器），把本目录放进技能目录即可：

```bash
git clone https://github.com/mysoloai/doubao-session-handoff.git
cp -r doubao-session-handoff ~/.your-agent/skills/
```

## 快速开始

对话快满时，对 Agent 说：

> 交接一下

Agent 会：

1. 在项目目录写 `HANDOFF.md`（≤ 60 行，只留结论与路径）；
2. 把接续提示词贴给你，并报新会话配置；
3. 等你确认后，你手动新建对话并粘贴；或由 Agent 用适配层命令新开。

新会话第一句就是那段提示词：

> 这是上一个会话的交接续作。请先读 `<HANDOFF 绝对路径>`，按其中「待续任务」一节继续推进，不要重复已验证的探查结论……

## 设计原则

- **有损压缩**：产物只引用路径/URL，不复制内容——复制必然漂移过期。
- **绝对路径**：交接文档、产物引用一律写绝对路径，永不依赖"会话工作空间"这类会被平台改写的相对概念。
- **实测优先**：文档里只写已验证的事实并附证据，推测必须明确标注为推测。
- **脱敏**：密钥、token、手机号、内部 ID 一律不写进交接文档。
- **闸口**：开新会话是不可逆的外部动作，必须先拿到明确同意。

## CLI 适配层（可选）

SKILL.md 里附带了一套"程序化新建会话"的实现，运行在 `doubao` CLI 的 headless 模式（私有 fork，非必需）。关键实测结论，直接抄作业：

| 坑 | 现象 | 做法 |
|---|---|---|
| `--wait --timeout` | 超时后 run 被取消，会话状态 `cancelled`、回复为空，看起来像"没在本地跑过" | 不要带 `--wait`；拿回 `conversationId` 后用 `sessions status` 轮询 |
| 客户端覆盖 workspace | 明明指定了目录，客户端随后新建自己的目录接管会话 | 文档与引用一律绝对路径，不要靠重建会话"纠正" |
| 新会话不继承项目 | `sessions create` 无 conversationId，项目归属默认为空 | 显式 `--project <id-or-name>`，且必须 `--mode work` |
| 会话列表核验 | `sessions list --limit N` 可能不含刚创建的会话（排序非 updatedAt） | 用会话查询接口核对标题与 `created_at` |
| 多设备 | `--runtime local` 绑定的是"运行 CLI 的那台机器" | 先 `runtimes` 确认 deviceName，别在 A 机派 B 机的活 |

没有这套 CLI 也完全能用——第 3 步退化为"用户手动新建对话并粘贴提示词"，这是默认路径。

## 兼容性

- 任何支持 `SKILL.md`（YAML frontmatter + Markdown 正文）的 Agent。
- CLI 适配层依赖 `doubao` CLI，缺失时自动走手动路径。

## License

MIT
