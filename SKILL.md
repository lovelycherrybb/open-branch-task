---
name: open-branch-task
name_en: Open Branch Task
name_zh: 开新任务做支线
description: Cut a side task out of the current main-line conversation. Extracts branch context, writes a handoff markdown file at a location that follows the current repo's conventions, and returns a copy-paste prompt the user pastes into a new chat to start the branch. Use when the user says 切支线 / 开新任务做这个 / 切出去做 / 把这个支线单独做 / handoff / 交接 / 起个新对话做这个 / 这事先放一下另开任务, and the branch is large enough to warrant its own chat.
description_en: Cut a side task out of the current main-line conversation. Extracts branch context, writes a handoff markdown file at a location that follows the current repo's conventions, and returns a copy-paste prompt for a new chat.
description_zh: 从主线对话里切出一条支线：抽取当前上下文 → 按所在仓库的落点规范写出交接 md → 返回一段可直接粘贴到新任务对话框的提示词。用于用户说"切支线 / 单独开个对话做 / handoff / 这个另开任务做"等场景。
argument-hint: Describe the branch task briefly, or say "just what we were discussing" to use current context
argument-hint-en: Describe the branch task briefly, or say "just what we were discussing" to use current context
argument-hint-zh: 简单说下要切的支线是什么，或者说"就是刚才聊的那条"沿用当前上下文
user-invocable: true
---

# 开新任务做支线（open-branch-task）

把主线对话里冒出来的**较大支线**切出去：写一份交接 md，返回一段提示词，用户复制到新任务对话框就能开工。主线不中断。

**核心定位：主线只切，不干。** 这个 skill 在主线对话框里**绝不执行支线任务本身**——不改代码、不做调研、不展开任何支线动作。支线的执行只发生在用户把提示词粘贴过去的新对话框里。

## 什么时候用

用户在主线里说：

- 切支线 / 切出去做 / 单独开个对话 / 另开任务做 / handoff / 交接
- 这条支线我另开一个窗口做
- 先不聊这个，起个新对话框
- 明确点名要"用 open-branch-task / 开新任务做支线"

**不用**：只是问问题、只是让 agent 顺手做一件小事、或者只是想让 agent 记住某事。那些不需要切。

## 三步工作流

### Step 1：从当前对话抽取交接要素

在动手写文件之前，先从主线对话里提取以下六栏内容。**任何一栏信息不完整，先向用户问一句补齐，不要靠猜**（USER 铁律："严禁把未核实系统现状写成事实"）。

| 栏 | 内容 | 常见坑 |
|---|---|---|
| **支线是什么** | 一句话说清目标（用户视角，不是任务描述） | 别写成"实现 xxx 模块"，写"能达成什么效果" |
| **已经到哪了** | 主线里为这条支线已经讨论/尝试过什么、当前结论是什么 | 别把主线无关进度塞进来 |
| **已排除** | 试过但否决的方向、明确不做的选择、否决原因 | 这一栏最容易漏，但对新对话框最有价值 |
| **相关文件/路径** | 涉及的仓库、文件绝对路径、外部资源链接 | 相对路径要写清是相对哪里 |
| **完成判定** | 什么算交完差（验收标准，尽量可验证） | 别只写"做完为止" |
| **回主线时怎么交回** | 做完要把结论落到哪里、主线如何读到 | 默认写回本 md 的"完成情况"栏 |

### Step 2：决定交接文件的落点（关键，不能拍脑袋）

用户明确要求"必须遵守当前仓库规范不能乱生成"。落点探测按下面顺序做，**每一步的结论都记录**：

1. **判断是否在某项目/仓库内**
   ```bash
   pwd
   git rev-parse --show-toplevel 2>/dev/null || echo "NOT_IN_GIT_REPO"
   ```
   若不在任何 git 仓库（只在临时 workspace）→ 直接落到 `outputs/handoff/YYYY-MM-DD-<主题>.md`，跳到 Step 3。

2. **在仓库内：先读仓库自己的规范文档**（按存在顺序探测）
   - 仓库根 `AGENTS.md` / `CLAUDE.md` / `CONTRIBUTING.md` / `README.md` 里是否提到 handoff / 交接 / 沉淀 / docs 结构 / ADR / research 目录
   - 仓库根 `docs/` 是否存在、下面有哪些子目录（`research/`、`adr/`、`notes/`、`handoff/`、`journal/` 等）
   - 是否已经存在同类历史文件（Glob `docs/**/*.md`、`.handoff/**/*.md`）——**照抄它们的目录位置和命名风格**

3. **产出落点候选**（至少 2 个），例如：
   - 候选 A：`docs/research/YYYY-MM-DD-<主题>.md`（因为仓库已有 docs/research/ 且历史文件都是这格式）
   - 候选 B：`.handoff/YYYY-MM-DD-<主题>.md`（仓库根没有明显约定，这是通用保守位置）
   - 候选 C：让用户自己指定

4. **用 AskUserQuestion 让用户在候选里挑一个**（或写自定义路径）。**没有用户确认之前绝不落文件。**

> **禁止**：直接写到项目根、直接写到 `~/`、直接写到 MEMORY.md/USER.md、擅自新建仓库顶层目录。

### Step 3：写文件 + 返回提示词

**A. 写交接 md 文件**

用用户挑定的路径写文件。内容按 Step 1 的六栏模板。文件开头加一段元信息：

```markdown
# 支线交接：<一句话目标>

- **来源主线**：<主线对话框标题或 chatId，能拿到就写>
- **切出时间**：<ISO 时间，用 `date` 命令拿>
- **状态**：待开工 / 进行中 / 已完成
- **如何更新本文件**：支线对话框每完成一步，把结论追加到"完成情况"栏
```

**B. 返回给用户的提示词**（复制即用）

在主线对话框里，用如下格式给用户一段"开箱即用"的提示词。用户复制整段，去新对话框粘贴即可：

```
【新任务开场提示词，复制下面这段到新对话框】

──────────────── 复制开始 ────────────────
读这个交接文件，按里面的目标开工：

<交接文件的绝对路径>

按文件里的六栏信息推进：
1. 支线是什么（目标）
2. 已经到哪了
3. 已排除（不要重复踩坑）
4. 相关文件/路径
5. 完成判定（做到什么算交差）
6. 回主线时怎么交回 —— 做完把结论追加回同一个文件的"完成情况"栏，并把"状态"改为"已完成"

开始之前先复述一遍你从文件里读到的目标、边界、验收，让我确认。
──────────────── 复制结束 ────────────────

回到主线：主线继续推进不受影响。支线做完后，你说一句"去看 <交接文件路径> 的完成情况栏"，我就把结论接回主线。
```

**C. 收尾（必须做）**

吐完提示词块后，补一句：

> 提示词在上面，整段复制去新任务对话框即可。这条支线在主线里到此为止，我不开始执行它。主线继续？

然后停在主线状态，**不做任何支线动作**。用户说"继续主线"就回到主线工作。

## 支线做完：合并回主线（回收）

用户从支线对话框回来后，在**主线对话框**里说：

> 去看 `<交接文件路径>` 的完成情况

agent 应该：

1. Read 交接文件，重点读"完成情况"栏
2. 把关键结论**摘要**回主线（不是全文倾倒）
3. 询问用户：这条支线是否彻底关闭？如果关闭，把文件"状态"改为"已完成"，可选归档（比如移到 `docs/research/done/` 或 `.handoff/archive/`，遵循仓库既有约定）
4. 如果结论值得跨会话长期沉淀，再单独问用户是否要抽要点进 memory（**不主动写 memory**，先问）

## 铁律

- **主线只切，不干（最高优先级）**。skill 在主线对话框里只做三件事：抽取六栏、定落点、写交接文件 + 返回提示词块。**绝不在主线里开始执行支线任务**——不改代码、不跑分析、不做调研、不"顺手先做一点"。哪怕支线看起来五分钟就能做完，也不做。发现自己在主线里就要动手执行支线 = 立刻停下，回到吐提示词块的收尾。
- **落点必须让用户确认**。宁可多问一句，不要在没探测仓库规范前就 Write 文件。
- **不擅自造词**。所有字段名、栏名、动作名沿用用户既有说法：主线、支线、切、交接、handoff、完成情况、已排除、完成判定。
- **一次只切一条支线**。用户如果同时提了两条，先问哪条先切，不要合并写成一个文件。
- **不污染所在 agent 平台的全局长期记忆**（如 QoderWork 的 `~/.qwenworkcn/awareness/` 下 MEMORY.md / USER.md / memory/*.md、Claude Code 的 `~/.claude/CLAUDE.md` 与 Projects memory、其他 agent 的等价机制）。交接文件是项目产物，写在项目/仓库里。
- **提示词里的路径必须是绝对路径**。用户复制到新对话框时，相对路径没有上下文会读不到。
- **主线不中断**。切完支线立刻把注意力交回主线，不要留在支线细节里继续讨论。

## 示例：一个典型来回

主线里，用户说：

> 等一下，这个 DeckSmith 预览端点的 MIME 白名单要重新评估，先切出去做个支线。

agent 应答：

1. 抽取六栏内容，缺"完成判定" → 问用户一句
2. 探测仓库 → DeckSmith 有 `docs/research/YYYY-MM-DD-主题.md` 惯例（memory 里也记着）
3. AskUserQuestion 让用户在候选里挑，或指定自定义
4. 写文件
5. 返回提示词块

用户在主线里说："OK 继续主线"，主线继续走。
