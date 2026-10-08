# open-branch-task

**QoderWork / QwenWork skill：把主线对话里冒出来的支线切成独立任务，主线不中断。**

> **TL;DR (EN)** — A [QwenWork](https://qwenwork.cn) skill that cuts a side task out of the current main-line conversation. It extracts branch context into a handoff markdown file (placed according to the current repository's own conventions, never silently), then returns a copy-paste prompt you paste into a new chat to start the branch. Main line keeps flowing.

---

## 为什么做这个

在主线任务里泡久了，总会冒出支线：

- "这个 MIME 白名单要重新评估"
- "顺便调研下竞品"
- "先记下这个坑，主线跑完再说"

小事顺带做了就行，但**稍微大一点的支线塞进主线**，两边互相稀释——主线进度被冲淡，支线的关键上下文又被遗忘。切去新对话框做？新对话框一开就是失忆状态，你复述背景的成本比直接做完还高。

`open-branch-task` 把"切支线"这件事变成**一次召唤、一份交接文件、一段可复制的开场提示词**：

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

Clone 到 QwenWork 的 skills 目录（macOS / Linux）：

```bash
git clone https://github.com/lovelycherrybb/open-branch-task.git \
  ~/.qwenworkcn/skills/open-branch-task
```

Windows 用户改路径：

```powershell
git clone https://github.com/lovelycherrybb/open-branch-task.git `
  "$env:USERPROFILE\.qwenworkcn\skills\open-branch-task"
```

重启 QoderWork 会话（或新开一个），在技能列表里就能看到 `open-branch-task` / "开新任务做支线"。

> 也可以直接在 QoderWork 的技能面板里手动 import，把这两个文件（`SKILL.md` + `.skill-metadata.yaml`）放到 `~/.qwenworkcn/skills/open-branch-task/` 下即可。

## 使用

召唤这个 skill 后，agent 会按三步走：

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

探测完 → 列 2–3 个候选路径给你挑（AskUserQuestion），**你选定之前不 Write**。

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
- **不污染 memory**（`~/.qwenworkcn/awareness/`）——交接文件是项目产物，不是长期记忆。

## 目录结构

```
open-branch-task/
├── SKILL.md                # 技能主文档（agent 执行规范）
├── .skill-metadata.yaml    # 预置的三条推荐 query（UI 快捷入口）
├── README.md               # 本文件
├── LICENSE                 # MIT
└── .gitignore
```

## 常见问题

**Q：为什么交接文件不直接写成 memory 或者 issue？**
A：memory 是**跨会话长期沉淀**用的，支线是**一次性事务**，塞进 memory 会污染以后的会话。GitHub issue 太重，切个支线还要联网、还要选 repo。落项目内的 md 是最合适的：跟着仓库走、可 review、可归档。

**Q：切出去之后主线真的不会被打断吗？**
A：不会。skill 只在主线里做三件事：抽信息、问你落点、写文件+返回提示词。做完立刻交回主线注意力，不留尾巴。

**Q：同一份 handoff 文件能被多次继续用吗？**
A：可以。你甚至可以在同一个 handoff 上开两三个不同对话框接力做——每个新对话框都读同一份文件、往"完成情况"栏追加。文件本身就是接力棒。

**Q：支线的六栏模板能改吗？**
A：能，但改之后要在 `SKILL.md` 里同步。当前模板针对"跨对话框无损交接"设计，改动建议只做加栏不做减栏。

## 反馈

Issue 提在本仓库：<https://github.com/lovelycherrybb/open-branch-task/issues>

## License

[MIT](LICENSE) © 2026 龚成
