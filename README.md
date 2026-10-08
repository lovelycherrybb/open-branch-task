# open-branch-task

**Agent Skill：把主线对话里冒出来的支线切成独立任务，主线不中断。**

> **TL;DR (EN)** — A cross-agent **Agent Skill** (following the `SKILL.md` spec originally introduced by Anthropic and now supported by multiple coding agents). It cuts a side task out of the current main-line conversation, extracts branch context into a handoff markdown file (placed strictly according to the current repository's own conventions — probed, then confirmed with you, never silently), and returns a copy-paste prompt you paste into a new chat to start the branch. Works on Claude Code, Codex CLI, Gemini CLI, Cursor, GitHub Copilot (agent mode), OpenClaw, QoderWork and anything else that reads `SKILL.md`. Main line keeps flowing.

---

## 为什么做这个

在主线任务里泡久了，总会冒出支线：

- "这个 MIME 白名单要重新评估"
- "顺便调研下竞品"
- "先记下这个坑，主线跑完再说"

小事顺带做了就行，但**稍微大一点的支线塞进主线**，两边互相稀释——主线进度被冲淡，支线的关键上下文又被遗忘。切去新对话框做？新对话框一开就是失忆状态，复述背景的成本比直接做完还高。

`open-branch-task` 把"切支线"变成**一次召唤、一份交接文件、一段可复制的开场提示词**：

```
主线对话 ──[切支线]──> 落 handoff.md ──> 生成提示词块
                                    ↓
                             你复制提示词去新对话框
                                    ↓
                             新对话框按六栏开工
                                    ↓
                             做完把结论追加回同一份 handoff.md
                                    ↓
                             回到主线："读一下 handoff 的完成情况"
```

## 什么时候用

你在主线对话里说以下任一：

- 切支线 / 切出去做 / 单独开个对话做
- 开新任务做这个 / 另开任务做
- handoff / 交接
- 这事先放一下，另开对话框

Agent 就会走这个 skill。

**不用**：只是问问题、顺手做件小事、或者只是想让 agent 记住某事——那些不需要切。

## 安装

Clone 到你所用 agent 的 skills 目录。各家路径整理如下（截至 2026-10 各家公开文档）：

| Agent | 用户级安装目录 | 项目级安装目录 |
|---|---|---|
| **Claude Code** (Anthropic) | `~/.claude/skills/` | `.claude/skills/` |
| **Codex CLI** (OpenAI) | `~/.codex/skills/` | `.codex/skills/` |
| **Gemini CLI** (Google) | `~/.gemini/skills/` | `.gemini/skills/` |
| **Cursor** (Anysphere) | 无用户级；仅项目级 | `.cursor/skills/` |
| **GitHub Copilot** (agent mode) | `~/.config/github-copilot/skills/` | `.github/skills/` |
| **OpenClaw** | `~/.openclaw/skills/` | `.openclaw/skills/` |
| **QoderWork / QwenWork** | `~/.qwenworkcn/skills/` | — |

**通用一行命令模板**（把 `<目标路径>` 换成上表任一格）：

```bash
git clone https://github.com/lovelycherrybb/open-branch-task.git \
  <目标路径>/open-branch-task
```

**具体例子**（Claude Code，macOS/Linux）：

```bash
git clone https://github.com/lovelycherrybb/open-branch-task.git \
  ~/.claude/skills/open-branch-task
```

**Windows（Claude Code 举例；QoderWork 用户把路径换成 `$env:USERPROFILE\.qwenworkcn\skills\` 即可）**：

```powershell
git clone https://github.com/lovelycherrybb/open-branch-task.git `
  "$env:USERPROFILE\.claude\skills\open-branch-task"
```

**装完之后**：

- Claude Code / Codex / Gemini / Copilot / OpenClaw / QoderWork：新开一个会话即可加载
- Cursor：需要 reload window（`Cmd+Shift+P` → Reload Window）
- 项目级安装（`.claude/skills/`、`.cursor/skills/` 等）：只对当前项目生效；用户级安装对所有项目生效

## 跨 agent 兼容性

本 skill 的 `SKILL.md` 严格遵循 Agent Skills 开放规范，只用**跨 agent 通用字段**：

| Frontmatter 字段 | 跨 agent 表现 |
|---|---|
| `name` | 所有 agent 都能读 |
| `description` | 所有 agent 都能读（skill 发现和触发的关键字段） |
| `argument-hint` | 主流 agent 大部分支持；不支持的会静默忽略 |
| `name_en` / `name_zh` / `description_en` / `description_zh` | QoderWork 双显示扩展；其他 agent 静默忽略 |
| `user-invocable` | QoderWork 特有；其他 agent 静默忽略 |

`.skill-metadata.yaml`（本仓库额外附带）是 **QoderWork 专属**的快捷 query UI 配置文件。其他 agent 完全忽略这个文件，不影响 skill 功能，只是没有"点击使用即预填 query"的 UI 便利。

**已知的功能降级**：

- **Cursor** 不解析 `allowed-tools` 字段（本 skill 未用到，无影响）；仅项目级
- **Copilot** 触发词匹配最保守，你可能需要显式说 "use the open-branch-task skill" 才召唤
- 各家 agent 对"未知字段"的处理都是静默忽略，因此**这份 skill 在任何读 `SKILL.md` 的 agent 上都能工作**，功能一致，只是缺少平台特化的 UI 入口

## 使用

召唤 skill 后，agent 按三步走：

**Step 1 · 抽取交接六栏**

从当前对话（或你在指令里现交代的内容）提取：

1. **支线是什么** — 一句话说清目标
2. **已经到哪了** — 已讨论 / 已尝试 / 当前结论
3. **已排除** — 试过但否决的方向（这条最容易漏也最值钱）
4. **相关文件 / 路径** — 涉及的仓库、文件绝对路径、外部链接
5. **完成判定** — 什么算交完差，尽量可验证
6. **回主线时怎么交回** — 默认写回同一文件的"完成情况"栏

任何一栏信息不完整，agent 会先问你补齐再落文件，不擅自猜。

**Step 2 · 探测仓库规范决定落点**

这是这个 skill 相对其他"todo 交接法"最硬的一条约束：**交接文件绝不擅自落到你没确认过的位置**。

探测顺序：

```bash
git rev-parse --show-toplevel    # 当前是否在仓库内
```

在仓库内 → 读仓库自己的规范文档：

- `AGENTS.md` / `CLAUDE.md` / `CONTRIBUTING.md` / `README.md` 里是否提到 handoff / 沉淀 / docs 结构 / ADR / research 目录
- `docs/` 下是否已有 `research/` `adr/` `notes/` `handoff/` `journal/` 等子目录
- 是否有历史同类文件（Glob 一遍）——照抄它们的目录位置和命名风格

探测完 → 列 2–3 个候选路径给你挑（通过当前 agent 的提问 UI 或在对话里明示候选），**你选定之前不 Write**。

不在任何仓库（只在临时 workspace）→ 保守落到 `outputs/handoff/YYYY-MM-DD-<主题>.md`。

**Step 3 · 写文件 + 返回可复制提示词**

按你选定的路径写出交接 md（六栏齐全），然后在对话里返回一段"复制开始 / 复制结束"包起来的提示词块。你**整段复制**、去新任务对话框粘贴，那个 agent 就能立刻按文件里的六栏开工，不用你复述任何背景。

## 支线做完怎么合回来

在**主线**对话框里说一句：

> 去看 `<交接文件绝对路径>` 的完成情况

主线 agent 会：

1. 读文件里"完成情况"栏
2. 摘要关键结论（不全文倾倒）
3. 问你两件事：这条支线是否可以标记完成并按仓库规范归档？有没有值得长期沉淀到 memory 的内容？

## 三条硬约束（写死在 skill 里）

- **落点必先探测仓库规范再问用户**，不允许"我觉得 `~/notes/` 挺好"就乱写。
- **六栏缺一栏就要问你补齐**，不允许靠猜。
- **不污染所在 agent 平台的全局长期记忆**（如 QoderWork 的 `MEMORY.md`、Claude Code 的 `CLAUDE.md` / Projects memory、其他 agent 的等价机制）——交接文件是项目产物，不是长期记忆。

## 常见问题

**Q：为什么交接文件不直接写成 agent 的 memory 或者 GitHub issue？**
A：agent 的 memory 是**跨会话长期沉淀**用的，支线是**一次性事务**，塞进 memory 会污染以后的会话。GitHub issue 太重，切个支线还要联网、还要选 repo。落项目内的 md 是最合适的：跟着仓库走、可 review、可归档。

**Q：切出去之后主线真的不会被打断吗？**
A：不会。skill 只在主线里做三件事：抽信息、问你落点、写文件+返回提示词。做完立刻交回主线注意力，不留尾巴。

**Q：同一份 handoff 文件能被多次继续用吗？**
A：可以。你甚至可以在同一个 handoff 上开两三个不同对话框接力做——每个新对话框都读同一份文件、往"完成情况"栏追加。文件本身就是接力棒。

**Q：不同 agent 之间能混用吗？比如主线用 Claude Code，支线对话框用 Codex？**
A：能。handoff 文件就是普通 md，谁都能读；提示词就是普通文本，谁都能贴。唯一前提是两边都用同一个文件系统/工作目录。

**Q：支线的六栏模板能改吗？**
A：能，改 `SKILL.md` 里的模板段即可。当前模板针对"跨对话框无损交接"设计，改动建议只做加栏不做减栏。

## 目录结构

```
open-branch-task/
├── SKILL.md                # Agent Skill 主文档（跨 agent 通用规范）
├── .skill-metadata.yaml    # QoderWork 专属预置 query（其他 agent 忽略）
├── README.md               # 本文件
├── LICENSE                 # MIT
└── .gitignore
```

## 反馈

Issue 提在本仓库：<https://github.com/lovelycherrybb/open-branch-task/issues>

## License

[MIT](LICENSE) © 2026 龚成
