# OpenSpec + Superpowers 组合方案：调研与决策

> 综合 OpenSpec 官方文档、Superpowers 官方 SKILL.md、相关 PR 讨论与社区 bridge schema，分析两套工具组合的可行性。
>
> **结论**：当前不采用作为生产工作流。
>
> ⚠️ **证据声明**：本文所有断言尽量配官方文档原文引用。证据分三类——① OpenSpec 官方文档 ② Superpowers 官方 SKILL.md ③ 社区 bridge schema（**非官方**）。早期版本曾以单一博客（Austin Xu）为主源、写成"社区共识"，并自行推论了不少衔接（fresh-agent review、tasks→plan 接口）。本文已重写，所有 skill 名、命令名、行为约束改用官方原文引用。

---

# 第一部分：背景与机制

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

**含义**：装了 Superpowers，brainstorming description 匹配几乎一切"做新东西"的需求——**自动 fire**，除非 CLAUDE.md 显式路由到别的路径。

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
- 执行：`subagent-driven-development`（transitive 触发 `test-driven-development` 和 `requesting-code-review`）
- 验证：`openspec-verify-change`
- 收尾：`finishing-a-development-branch`

## 五、explore 替代 brainstorming 的真实代价

要用 OpenSpec `/opsx:explore` 替代 Superpowers `brainstorming`：

**默认情况**（不改 CLAUDE.md）：用户说新需求 → Superpowers brainstorming description（*"You MUST use this before any creative work"*）自动匹配 → brainstorming skill 强制 fire → `/opsx:explore` 被挡。

**要换路径**必须做两件事：

1. **CLAUDE.md 显式路由**——写"新需求走 `/opsx:explore + /opsx:propose` 路径，**不要 fire** `superpowers:brainstorming`"。这是**硬路由开关**（覆盖 skill description 自动匹配机制），不是软偏好。
2. **手工接 plan**——brainstorming 不再 fire 意味着 `writing-plans` 不会被链式触发（参考第二节的"终止规则"）。要么 CLAUDE.md 写"OpenSpec tasks 完成后调用 `superpowers:writing-plans`"，要么手敲。

## 六、机制小结

OpenSpec 哲学是 *"actions, not phases"*（灵活），Superpowers 哲学是 *hard gates*（强制）——组合本质是**用 Superpowers 的强制框架消化 OpenSpec 的灵活**。社区 [`superpowers-bridge`](https://github.com/JiangWay/openspec-schemas) 给出 8-stage 桥接但**非官方**（PR #970 被 close）。要让 explore 替代 brainstorming 必须改 CLAUDE.md **硬路由**——否则 brainstorming description（*"You MUST use this before any creative work"*）会自动匹配把 OpenSpec 路径挡回去。

机制理论上可行。下面分析为什么当前不采用。

---

# 第二部分：决策——当前不采用

## 一、两个关键 PR 的全貌（核心证据）

### PR #970（提议合进 OpenSpec core）—— 已 close，未合并

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

### PR #1043（社区目录入口）—— 已合并，但意味深长

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
| **tennis-lineup 网球阵容 app** | 项目本身**无法溯源**——名字最早出现在 WebSearch 工具的 AI 摘要里，但没有公开 GitHub 链接、没有可访问的博客原文、连作者本人是否真的做过这个项目都无法独立验证 |
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

| 信号 | 含义 |
|------|---------|
| OpenSpec catalog 收录 ≥ 5 条社区 schema | 扩展生态起来了 |
| superpowers-bridge 出 v2，有非作者贡献者 | 有第二个人在乎了 |
| 出现 ≥ 3 篇**第三方**（非 JiangWay / Austin Xu）真实项目复盘 | 经过外部验证 |
| OpenSpec 团队把"扩展系统"放进 roadmap | TabishB 改口、不再是 *"not a priority"* |
| Superpowers 出 v2 / obra 公司化 | 上游稳定性提升 |

**任何一条触发**就重新评估；**全都没触发**就继续单工具。

## 八、结论

**这是个 3 天前才有官方目录入口、唯一 catalog 条目作者就是 schema 作者本人、维护者明说"不是优先级"、核心 PR 因哲学冲突被拒、没有 schema 作者以外的产线案例（甚至作者自家"实战项目"都没有公开 GitHub 链接）的早期社区方案**——结合复杂度翻倍且没有"1+1=3"的收益，**当前不值得作为生产工作流采用**。等生态走出"作者自卖自夸"阶段再说。

---

# 参考资料

**OpenSpec 官方**

- [opsx.md](https://github.com/Fission-AI/OpenSpec/blob/main/docs/opsx.md)（命令角色定义）
- [workflows.md](https://github.com/Fission-AI/OpenSpec/blob/main/docs/workflows.md)（"Actions, not phases" 哲学 + 推荐序列）
- [docs/customization.md — Community Schemas](https://github.com/Fission-AI/OpenSpec/blob/main/docs/customization.md)（当前 catalog，1 条）
- [PR #970（已 close，未合并）](https://github.com/Fission-AI/OpenSpec/pull/970) — 三大反对意见来源
- [PR #1043（已 merge 2026-05-04）](https://github.com/Fission-AI/OpenSpec/pull/1043) — TabishB *"not a priority"* 评论来源
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

**反复被引用的单一博客（不构成多源印证）**

- [Stacking OpenSpec and Superpowers — Austin Xu](https://medium.com/@austin.xyz/stacking-openspec-and-superpowers-a-combined-sdd-workflow-c03fbf47539c)
