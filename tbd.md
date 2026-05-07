# tbd（待整理）

> Q7、Q8 从 [notes.md](notes.md) 暂迁到此处。
> Q1-Q4 等跨节引用指向 `notes.md`；Q7 ↔ Q8 互引在本文件内。

## 导航

- [Q7: OpenSpec + Superpowers 组合工作流](#q7-openspec--superpowers-组合工作流) — OpenSpec "actions, not phases" vs Superpowers hard gates，brainstorming HARD-GATE 行为，社区 bridge schema (非官方)，CLAUDE.md 硬路由 vs 软约定
- [Q8: CLAUDE.md 真的管用吗？Skill 不是要手动调用吗？](#q8-claudemd-真的管用吗skill-不是要手动调用吗) — Skill 自动 trigger ≠ Slash 手敲，三层约束强度（CLAUDE.md / Skill / Hook）
- [Q9: 放弃 OpenSpec + Superpowers 组合方案的理由](#q9-放弃-openspec--superpowers-组合方案的理由) — 两个 PR 全貌（#970 已 close、#1043 三天前合并）、案例问题、维护风险、什么时候可以重新评估

---

# Q7: OpenSpec + Superpowers 组合工作流

> 本节所有断言尽量配 **官方文档原文引用**。证据分三类：①OpenSpec 官方文档 ②Superpowers 官方 SKILL.md ③社区 bridge schema（**非官方**）。
>
> ⚠️ **重要修订**：早期版本以 Austin Xu 一篇博客为主源、写成"社区共识"，并自行推论了不少衔接（fresh-agent review、tasks→plan 接口）。本版本已重写——所有 skill 名、命令名、行为约束改用官方原文引用。

## 一、两套系统的哲学冲突（理解组合的前提）

| | OpenSpec | Superpowers |
|---|---|---|
| 哲学 | **"Actions, not phases"**（命令是能力，不是阶段） | **Hard gates**（强制顺序） |
| 官方原文 | *"Commands are things you can do, not stages you're stuck in"* — workflows.md | *"Do NOT invoke any implementation skill, write any code... until you have presented a design and the user has approved it"* — brainstorming/SKILL.md |
| 执行 | 任何命令任何时候都可以走 | 每步是下一步的硬前置 |

**这是组合的核心张力**——一边鼓励灵活，一边强制顺序。组合方案本质是**用 Superpowers 的强制框架去消化 OpenSpec 的灵活性**。

## 二、Superpowers 关键 skill 的 hard-gate 行为（来自官方 SKILL.md）

### `brainstorming`

- **描述**（自动匹配 trigger）：*"You MUST use this before any creative work — creating features, building components, adding functionality, or modifying behavior."*
- **HARD-GATE**：*"Do NOT invoke any implementation skill, write any code, scaffold any project, or take any implementation action until you have presented a design and the user has approved it. This applies to EVERY project regardless of perceived simplicity."*
- **终止规则**：*"The ONLY skill you invoke after brainstorming is writing-plans"*
- **输出**：`docs/superpowers/specs/YYYY-MM-DD-<topic>-design.md`

**含义**：装了 Superpowers，brainstorming description 匹配几乎一切"做新东西"的需求——**自动 fire**，除非 CLAUDE.md 显式路由到别的路径（详见 第五节、第六节）。

### `writing-plans`

- **描述**：*"Use when you have a spec or requirements for a multi-step task, before touching code"*
- **输入**：*"Project specification or requirements document"*——任何 spec/requirements 都行（OpenSpec 的 specs/tasks 也算）
- **输出**：`docs/superpowers/plans/YYYY-MM-DD-<feature-name>.md`

**含义**：writing-plans 接口宽松——OpenSpec 的 `specs/`、`tasks.md` 都能直接喂。这是组合时真实可用的衔接点。

## 三、OpenSpec 命令角色（来自官方 opsx.md / workflows.md）

| 命令 | 用途（官方原文） | 输出 |
|------|--------------|------|
| `/opsx:explore` | *"Think through ideas, investigate problems, compare options"* | 思考反馈，**无结构化产物** |
| `/opsx:propose` | *"Create a change and generate planning artifacts in one step"* | proposal + specs + design + tasks |
| `/opsx:apply` | *"Implement tasks, updating artifacts as needed"* | 代码实现 |
| `/opsx:verify` | *"Validate implementation against artifacts"* | 验证报告 |
| `/opsx:archive` | *"Archive when done"* | 归档 + 同步 specs |

**官方推荐序列**（workflows.md）：

- Core profile：`propose → apply → sync → archive`
- 探索式：`explore → new → continue → apply`
- 快速：`new → ff → apply → verify → archive`

> ⚠️ `/opsx:explore` 的输出是"思考反馈，无结构化产物"——它**不直接产 proposal 草稿**。要走 OpenSpec 路径，需要 `explore → propose`，让 propose 把 explore 结论物化成 artifacts。

## 四、社区 bridge schema（非官方）

[`JiangWay/openspec-schemas`](https://github.com/JiangWay/openspec-schemas) 的 `superpowers-bridge`（前身：[OpenSpec PR #970](https://github.com/Fission-AI/OpenSpec/pull/970)，已 close）定义了 8-stage 桥接：

```
1. Brainstorm        → superpowers:brainstorming
2. Proposal
3. Design (optional)
4. Specs
5. Tasks
6. Plan              → superpowers:writing-plans
7. Apply             → superpowers:subagent-driven-development
                       (transitive: test-driven-development, requesting-code-review)
8. Verify            → superpowers:openspec-verify-change
```

⚠️ **PR #970 没合进 OpenSpec 主仓库**——reviewer 反馈"耦合风险大、时序约束太死"。最终独立成外部 repo。**OpenSpec 官方目前没有钦定的 Superpowers 集成方案**，bridge 是社区方案。

apply 阶段的展开（来自 PR 描述）：

- 工作区：`using-git-worktrees`
- 执行：`subagent-driven-development`（transitive 触发 `test-driven-development` 和 `requesting-code-review`——这是真实 skill 名，不是早期版本"fresh-agent review"的推论）
- 验证：`openspec-verify-change`
- 收尾：`finishing-a-development-branch`

## 五、explore 替代 brainstorming 的真实代价

要用 OpenSpec `/opsx:explore` 替代 Superpowers `brainstorming`：

**默认情况**（不改 CLAUDE.md）：用户说新需求 → Superpowers brainstorming description（*"You MUST use this before any creative work"*）自动匹配 → brainstorming skill 强制 fire → `/opsx:explore` 被挡。

**要换路径**必须做两件事：

1. **CLAUDE.md 显式路由**——写"新需求走 `/opsx:explore + /opsx:propose` 路径，**不要 fire** `superpowers:brainstorming`"。这是**硬路由开关**（覆盖 skill 自动匹配），不是软偏好（呼应 [Q8](#q8-claudemd-真的管用吗skill-不是要手动调用吗) 的强度补充）
2. **手工接 plan**——brainstorming 不再 fire 意味着 `writing-plans` 不会被链式触发（参考 第二节的"终止规则"）。要么 CLAUDE.md 写"OpenSpec tasks 完成后调用 `superpowers:writing-plans`"，要么手敲

## 六、CLAUDE.md 模板（两种候选路径）

### 路径 A：默认顺路（brainstorming 起步，OpenSpec 收尾）

```md
- 任何新需求由 brainstorming skill 起步（自动 trigger，不要绕过）
- brainstorming 完成后由 writing-plans 接手（自动链式）
- /opsx:apply 阶段必须调用 superpowers:subagent-driven-development
  （transitive 触发 test-driven-development + requesting-code-review）
- /opsx:verify 完成后再 /opsx:archive
- specs/ 是真理之源；做新需求前 openspec list --specs
```

### 路径 B：OpenSpec 探索起步（绕过 brainstorming）

```md
- 新需求走 OpenSpec 路径：/opsx:explore → /opsx:propose
- 不要 fire superpowers:brainstorming（OpenSpec 已经做了探索）   ← 硬路由
- /opsx:propose 产出 specs/tasks 后，调用 superpowers:writing-plans 拆细
- apply 阶段调用 superpowers:subagent-driven-development
- /opsx:verify + /opsx:archive 收尾
```

> ⚠️ **强度差异**：路径 B 第一行"不要 fire"是**硬路由**（覆盖 skill 自动匹配机制），其余是**软约定**。详见 [Q8 三层强度](#三三层约束强度)。

## 七、跟其他笔记的对应

| 概念 | 在组合工作流的位置 |
|------|-----------------|
| [Q1 Council](notes.md#q1-council-是什么) | brainstorming 或 explore 阶段可让 Council 同题多解 |
| [Q2 Plan vs OpenSpec](notes.md#q2-opencode-plan-agent-vs-openspec) | Superpowers plan = 临时施工图（per-feature），OpenSpec specs = 长期账本 |
| [Q3 OpenSpec 扫描机制](notes.md#q3-openspec-做新需求时会扫之前的-spec-吗) | CLAUDE.md 写 "propose 前先 `openspec list --specs`" 做兜底 |
| [Q4 Context 资源 vs 负担](notes.md#q4-opencode-agent-架构-与-context-管理策略) | subagent-driven-development 把 context 当负担处理；OpenSpec 落盘是长期账本 |
| [Q8 三层强度](#三三层约束强度) | 路径 B 第一行硬路由 vs 其余软约定 |

## 八、什么时候不结合

| 场景 | 推荐 |
|------|-----|
| Prototype / MVP / 一次性 demo | 只用 Superpowers（OpenSpec 归档负担划不来） |
| 多人维护、长期项目、需求要审计 | 组合方案（参考 superpowers-bridge） |
| 不喜欢强 TDD 的探索性项目 | 只用 OpenSpec（spec-driven 但实现自由） |
| 简单 bug fix / 文案改动 | 都跳过，直接动手 |
| 单人 / 小项目 / 短周期 | 只用 Superpowers，spec 用 plan 凑合（呼应 [Q2](notes.md#q2-opencode-plan-agent-vs-openspec)） |

## 九、一句话

OpenSpec 哲学是 *"actions, not phases"*（灵活），Superpowers 哲学是 *hard gates*（强制）——组合本质是**用 Superpowers 的强制框架消化 OpenSpec 的灵活**。社区 [`superpowers-bridge`](https://github.com/JiangWay/openspec-schemas) 给出 8-stage 桥接但**非官方**（PR #970 被 close）。要让 explore 替代 brainstorming 必须改 CLAUDE.md **硬路由**——否则 brainstorming description（*"You MUST use this before any creative work"*）会自动匹配把你挡回去。

## 参考资料

**OpenSpec 官方**

- [opsx.md](https://github.com/Fission-AI/OpenSpec/blob/main/docs/opsx.md)（命令角色定义）
- [workflows.md](https://github.com/Fission-AI/OpenSpec/blob/main/docs/workflows.md)（"Actions, not phases" 哲学 + 推荐序列）
- [PR #970（已 close，未合并）](https://github.com/Fission-AI/OpenSpec/pull/970)
- [Issue #780: Distribute OpenSpec as a Superpowers skill pack](https://github.com/Fission-AI/OpenSpec/issues/780)

**Superpowers 官方 SKILL.md**

- [brainstorming/SKILL.md](https://github.com/obra/superpowers/blob/main/skills/brainstorming/SKILL.md)（HARD-GATE 来源）
- [writing-plans/SKILL.md](https://github.com/obra/superpowers/blob/main/skills/writing-plans/SKILL.md)
- [Superpowers GitHub](https://github.com/obra/superpowers)

**社区方案（非官方）**

- [JiangWay/openspec-schemas](https://github.com/JiangWay/openspec-schemas)（`superpowers-bridge` 当前位置）
- [Veath/openspec-spec-driven-superpowers](https://github.com/Veath/openspec-spec-driven-superpowers)
- [SYZ-Coder/superpowers-openspec-team-skills](https://github.com/SYZ-Coder/superpowers-openspec-team-skills)
- [momoshenchi/Superpowers-with-Spec](https://github.com/momoshenchi/Superpowers-with-Spec)

---

# Q8: CLAUDE.md 真的管用吗？Skill 不是要手动调用吗？

> 接 Q7 的疑虑：CLAUDE.md 里写一堆"先 brainstorm 再 propose"、"绕过 /opsx:apply 用 subagent + TDD"，看起来只是文字——skill 不就是用户敲 `/xxx` 才跑吗？agent 不会自动调用，这段配置不是白写？

## 一、问题触发：把 Claude Code 当"命令式工具集合"

这个怀疑成立的前提是——**Claude Code 是个工具盒，用户输什么命令它跑什么**。在这个心智模型下：
- skill 等同于 slash command
- 用户不敲就不跑
- CLAUDE.md 就是装饰文字

但这是 Claude Code 早期的模型，或者说"对它的旧理解"。**现代 Claude Code 是个 agent**——它能读 skill 描述、自己挑工具、根据 CLAUDE.md 调整行为。怀疑的根源是没分清"agent 主动选工具" vs "用户手动敲命令"这两件事。

把 Q7 那段 CLAUDE.md 当"白写"，本质是把 Skill 误认成 Slash command。

## 二、Skill ≠ Slash command（关键分辨）

现代 Claude Code 同一个 plugin（如 Superpowers）里**同时存在两种东西**：

| | Slash command | Skill |
|---|---|---|
| 文件位置 | `commands/<name>.md` | `skills/<name>/SKILL.md` |
| 触发方式 | **用户手敲 `/xxx`** | **agent 读 frontmatter description 自动匹配** |
| 谁来选 | 用户 | agent |
| 例子 | `/opsx:propose`、`/brainstorm` | `brainstorming`、`writing-plans`、`subagent-driven-development` |

Skill 的 YAML frontmatter 长这样：

```yaml
---
name: brainstorming
description: Use when user has a vague idea and needs structured exploration...
---
```

Claude 每次回复前会扫所有可用 skills 的 description，**匹配到就主动加载并执行**。所以：
- 用户说"想加个登录功能"（模糊） → agent 自己 trigger brainstorming skill
- 用户说"我有 tasks.md 了" → agent 自己 trigger writing-plans skill
- **不需要敲 `/brainstorm` 或 `/write-plan`**

Superpowers 之所以能形成一套工作流，**核心是 skill 自动 trigger 机制**，不是 slash command。slash command 只是给"想手动控制"的用户额外的入口。

## 三、三层约束强度

| 强度 | 机制 | 性质 | 例子 |
|-----|------|------|------|
| **弱** | CLAUDE.md 自然语言（软约定） | 偏好提示，agent 读了会尝试遵守 | "写代码先写测试" |
| **中-强** | CLAUDE.md 自然语言（**硬路由**） | 显式 "不要 fire X skill"——覆盖 skill 自动匹配 | "新需求走 OpenSpec 路径，不要 fire `superpowers:brainstorming`" |
| **中** | Skill description 自动匹配 | agent 自己挑工具，结构化触发 | brainstorming skill 自动响应模糊需求 |
| **强** | settings.json **Hook** | harness 层硬拦截，agent 绕不过 | commit 前必须跑测试 |

- CLAUDE.md 大多数用法是**软约定**（长会话会稀释、可以被绕过）
- 但 CLAUDE.md 写**显式 skill 路由**时强度会提高——能压制 skill description 自动匹配（呼应 Q7 第五节）
- Skill 自动 trigger 才是"agent 自动跑流程"的真正机制
- Hook 是命令调用前由 Claude Code 主程序运行的脚本，failed 直接 block 工具调用——**唯一的硬约束**

## 四、CLAUDE.md 里每种内容的实际效果

| 写在 CLAUDE.md 里 | 效果 | 原因 |
|---|---|---|
| "写代码先写测试 (TDD)" | ✅ 强 | 自然语言行为指令，agent 直接照做 |
| "specs/ 是真理之源，propose 前先扫" | ✅ 强 | 行为指令配合 skill |
| "新需求先 brainstorm 再 propose" | ✅ 中-强 | brainstorming skill 自动触发，CLAUDE.md 强化优先级 |
| "不要 /opsx:apply，改走 subagent" | ⚠️ 中 | agent 会偏向那个选择，但不是硬阻止 |
| "用户说 X 时执行 /yyy" | ❌ 弱 | slash command 必须人敲，写了也没用 |
| "提交前必须跑测试，否则拒绝" | ❌ 弱 | 硬约束需要 hook，CLAUDE.md 长会话会忘 |

**口诀**：软约定写 CLAUDE.md，自动触发靠 skill description，硬约束靠 hook。

## 五、Hook 才是真正的硬约束

`settings.json` 里：

```json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "Write|Edit",
      "hooks": [{ "type": "command", "command": "<检查脚本>" }]
    }]
  }
}
```

Hook 是 Claude Code 主程序在 agent 调用工具**之前**运行的命令——脚本 exit code 非 0,工具调用直接被 block。**agent 看不见、绕不过**。

例：
- commit 前 hook 跑 `npm test`，挂了拒绝写文件
- 写代码前 hook 检查 spec 文件存在，否则拒绝
- archive 前 hook 强制要求 verify 通过

## 六、所以 Q7 那段 CLAUDE.md 真的有用吗

**有用，但不是因为 CLAUDE.md "命令" agent**——是因为：

1. Superpowers 的 skills（brainstorming / writing-plans / subagent-driven-development）**自动 trigger**，不靠 CLAUDE.md
2. CLAUDE.md 里的自然语言规则（TDD、specs/ 优先）agent **会读会试着照办**
3. "绕过 `/opsx:apply` 改用 subagent" 这种**工具偏好**，写在 CLAUDE.md 里 agent 会倾向遵守
4. 真要硬保证（commit 前 verify、archive 前 tests pass）需要加 hook

把 Q7 的 CLAUDE.md 段落理解为**软约定层**——它管 80% 场景；剩下 20% 真要拦住的事，靠 hook。

## 七、修正后的 CLAUDE.md 写法

不要写这种：

```md
- 用户说新需求时执行 /brainstorm   ← ❌ slash 不会自动执行
- agent 必须调用 /opsx:propose       ← ❌ 同上
```

要写这种：

```md
- 新需求一开始要做结构化反问澄清（依赖 brainstorming skill）   ← ✅ skill 自动 trigger
- 写代码必须先写测试，按红绿循环来                              ← ✅ 行为指令
- specs/ 是真理之源，做新需求前先 openspec list --specs        ← ✅ 行为指令
- 涉及多文件大改时优先用 subagent 隔离 context                ← ✅ 工具偏好
- 不要直接执行 /opsx:apply，改走 Superpowers 的 subagent      ← ✅ 工具偏好（agent 会避开）
```

**差别是写"行为期望"和"工具偏好"，而不是"敲哪个 slash"**。

## 八、一句话

CLAUDE.md 是**软约定**，靠 agent 自觉；Skill description 是**自动匹配机制**，agent 真会主动选用，**不需要敲 slash**；Hook 是**硬拦截**，绕不过。Q7 那段 CLAUDE.md 真有用——但它只承担软约定层，自动行为靠 Superpowers 的 skill 机制（agent 自己 trigger），硬保证靠 hook。怀疑的根源是把 Claude Code 当"用户敲命令的工具盒"，而它现在是**能自己挑工具的 agent**。

## 九、回头改 Q7（已完成）

Q7 早期版本写得过于乐观，没说清强度差异。现已重写，并在 第五节、第六节 显式区分**硬路由**（覆盖 skill 自动匹配，例如 "不要 fire `superpowers:brainstorming`"）和**软约定**（行为偏好，例如 "TDD 红绿循环"）。

**关键认识**：真正驱动 brainstorm → write-plan → subagent 自动衔接的，**是 Superpowers skill description 的自动匹配机制**，不是 CLAUDE.md。CLAUDE.md 的作用是**强化 / 消歧 / 路由 / 注入项目约束**（如 specs/ 是真理之源），不是驱动流程。

---

# Q9: 放弃 OpenSpec + Superpowers 组合方案的理由

> 决策记录（2026-05-07）：经过 [Q7](#q7-openspec--superpowers-组合工作流)、[Q8](#q8-claudemd-真的管用吗skill-不是要手动调用吗) 的讨论，决定**当前不采用 OpenSpec + Superpowers 组合作为生产工作流**。本节归档放弃理由，方便日后回看决策依据。

## 一、两个关键 PR 的全貌——核心证据

### PR #970（提议合进 OpenSpec core）—— **已 close，未合并**

| 项 | 情况 |
|---|------|
| 提议者 | JiangWay（superpowers-bridge 作者） |
| 内容 | `sdd-plus-superpowers` schema 通过 `--schema` 注入 OpenSpec |
| 工作流 | brainstorm → proposal → specs → tasks → plan → apply → verify |
| **结果** | **被 OpenSpec 维护者拒绝合并** |

**Reviewer alfred-openspec 三大反对**（结构性问题，不是细节）：

1. **强耦合无保护**——*"no Superpowers version check, install check, capability detection, or fallback beyond prompt text"*。硬绑死 Superpowers 当前 skill 名，对方一改就坏。

2. **时序不可强制**——*"the engine cannot enforce the timing rule, so this depends on the model obeying a paragraph of prompt text. That feels too fragile for a packaged workflow."* OpenSpec 引擎只看结构化字段（`requires:`），看不到 prompt 文字里的时序约束。这是 OpenSpec *"actions, not phases"* 哲学的直接结果——不会改。

3. **Repo 副作用过重**——apply 步骤包含 pre-flight git commits，*"much more aggressive than OpenSpec's default apply guidance"*。

**关闭信号**：OpenSpec 维护者认为**两套系统的设计哲学不兼容到难以打包**——不是修修补补能解决的。

### PR #1043（社区目录入口）—— **已合并，但意味深长**

| 项 | 情况 |
|---|------|
| 性质 | 纯文档 PR（zero code / zero schema） |
| 改动 | `README.md` 加引导段、`docs/customization.md` 加 catalog 表 |
| **状态** | ✅ Merged 2026-05-04（**距决策约 3 天**） |
| **Catalog 当前条目数** | **1 条**（仅 superpowers-bridge） |

**OpenSpec 维护者 TabishB 批准时的原话**：

> *"Nice, I'm ok with supporting this! It's easy and lightweight. We want to try... find better ways to support community plugins/extensions. **Just not a priority at the moment**."*

读出来的几层信号：

- ✅ 接受社区方案存在
- ⚠️ **不是优先级**——意味着扩展系统短期内不会有官方投入
- ⚠️ 维护者**有意未来重做扩展机制**——意味着现在的目录格式可能被推翻

### 两个 PR 合起来读

```
2026 早期        PR #970 提议合进 core
                 ↓ 三大反对
                 关闭。schema 作者迁外部 repo
2026-05-04       PR #1043 退而求其次：在 docs 加 catalog 入口
（约 3 天前）     ↓ 合并，但维护者明说"不是优先级"
```

**含义**：**核心方案被拒，最终只拿到 docs 一行 catalog**——这是政治妥协，不是工程背书。OpenSpec 团队明显不想真承担这个组合方案的责任。

## 二、案例问题——没有独立验证

复盘搜到的所谓"实战"，剥开看：

| "案例" | 真相 |
|--------|------|
| **tennis-lineup 网球阵容 app** | **JiangWay 自己的项目**——而且**搜不到公开 GitHub 链接**（不在 JiangWay 公开 repo 列表、博客/Medium 都查不到），可能是私有 repo 或仅在内部分享中提及 |
| Finance 3 个月案例 | 也是 schema 作者自家项目，无公开溯源 |
| Austin Xu 444 行 / 86 测试 / 3 小时 | 单一博客，被各处引用造成"多源印证"假象 |
| Veath/openspec-spec-driven-superpowers | 独立尝试，**不用 superpowers-bridge** |
| momoshenchi/Superpowers-with-Spec | 同上，独立方案 |
| SYZ-Coder/superpowers-openspec-team-skills | 同上，独立方案 |
| adoptableCoho gist 集成指南 | 个人探索，未跑通真项目 |

**结论**：

- **schema 作者以外，没有人在产线用过 superpowers-bridge**
- "实战项目"连**公开 GitHub 链接都拿不出来**
- 多个独立 fork / 重新发明 = **大家都觉得现成方案不够好**——这是负面信号
- "Austin Xu 实战"被反复引用造成共识假象，实际只是**一篇博客**

## 三、设计问题——哲学冲突无法根除

PR #970 三大反对暴露的是**结构性张力**：

```
OpenSpec   "actions, not phases"   ← 引擎不强制顺序
Superpowers   "hard gates"          ← 强制每步硬前置
```

哪怕 superpowers-bridge 用 PRECHECK 双层（skill 名检查 + shell evidence）填缝，**OpenSpec 引擎永远不会原生强制 Superpowers 期望的时序**——只是用更多检查脚本逼近。**脚本越多，脆弱点越多**。

## 四、维护风险——依赖链条过长

```
你的项目
   ↓ 依赖
superpowers-bridge schema   ← 1 个 maintainer，v1，无 LTS 承诺
   ↓ 依赖
Superpowers skill 名（brainstorming / writing-plans / subagent-driven-development）
   ↓ 依赖
obra 一个人维护 Superpowers
   ↓ 依赖
Claude Code skill matching 机制
   ↓ 依赖
OpenSpec /opsx:* 命令稳定
```

**任意一环改名 / 重构，PRECHECK 会立刻击穿**，工作流停摆。OpenSpec 已明说扩展不是优先级——出问题没人兜底。

## 五、复杂度不匹配收益

实际安装 superpowers-bridge 后承担的复杂度：

- 12 步工作流（6 planning artifacts + 6 apply steps）
- PRECHECK 双层能力检测
- 新概念 `retrospective` artifact（Superpowers 没有，bridge 自创）
- CLAUDE.md fragment 模板（英 / 繁中两版）
- `/opsx:ff`、`/opsx:continue`、`/opsx:apply`、`/opsx:verify`、`/opsx:archive` 命令链
- Superpowers skill 自动 trigger 机制 + CLAUDE.md 硬路由

**对比单工具复杂度——多了一倍以上**。换来的"额外收益"是 spec 长期账本 + TDD 强制——但单用 OpenSpec 已经有账本，单用 Superpowers 已经有 TDD。**没有"1+1=3"的特别收益**。

## 六、替代方案的明确性

| 需求 | 用 |
|------|---|
| 长期项目要 spec 账本、变更审计 | **只用 OpenSpec**（按 core profile workflow，不强制 TDD） |
| 需要强制 TDD + subagent 隔离 | **只用 Superpowers**（plan 留磁盘当临时 spec） |
| 两者都想要 | 接受**人工把两边串联**的复杂度——别用 schema |
| 不确定 | 先各用一段时间，体感更准再考虑组合 |

## 七、什么时候可以重新评估

设几个**触发条件**，避免决策后继续焦虑：

| 信号 | 评估窗口 |
|------|---------|
| OpenSpec catalog 收录 ≥ 5 条社区 schema | 说明扩展生态起来了 |
| superpowers-bridge 出 v2，有非作者贡献者 | 有第二个人在乎了 |
| 出现 ≥ 3 篇**第三方**（非 JiangWay / Austin Xu）真实项目复盘 | 经过外部验证 |
| OpenSpec 团队把"扩展系统"放进 roadmap | TabishB 改口、不再是 *"not a priority"* |
| Superpowers 出 v2 / obra 公司化 | 上游稳定性提升 |

**任何一条触发**就重新评估；**全都没触发**就继续单工具。

## 八、一句话

**这是个 3 天前才有官方目录入口、唯一 catalog 条目作者就是 schema 作者本人、维护者明说"不是优先级"、核心 PR 因哲学冲突被拒、没有 schema 作者以外的产线案例（甚至作者自家"实战项目"都没有公开 GitHub 链接）的早期社区方案**——结合复杂度翻倍且没有"1+1=3"的收益，**当前不值得作为生产工作流采用**。等生态走出"作者自卖自夸"阶段再说。

## 参考资料

- [PR #970（已 close）](https://github.com/Fission-AI/OpenSpec/pull/970) — 三大反对意见来源
- [PR #1043（已 merge 2026-05-04）](https://github.com/Fission-AI/OpenSpec/pull/1043) — TabishB *"not a priority"* 评论来源
- [JiangWay/openspec-schemas](https://github.com/JiangWay/openspec-schemas) — superpowers-bridge 当前位置
- [OpenSpec docs/customization.md — Community Schemas](https://github.com/Fission-AI/OpenSpec/blob/main/docs/customization.md) — 当前 catalog（1 条）
- [Stacking OpenSpec and Superpowers — Austin Xu](https://medium.com/@austin.xyz/stacking-openspec-and-superpowers-a-combined-sdd-workflow-c03fbf47539c) — 被反复引用的"单一博客"
