# 笔记

## 日期: 2026-05-02

**来源文章**: <https://www.dataleadsfuture.com/how-i-use-opencode-oh-my-opencode-slim-and-openspec-to-build-my-own-ai-coding-environment/>

---

## 待办

- [ ] 学习 Claude Code Chrome 扩展：<https://code.claude.com/docs/zh-CN/chrome>（用浏览器自动化让 Claude 自己验证 UI 改动）
- [ ] 研究 OpenCode 浏览器自动化的主流做法（OpenCode 本身没有官方 Chrome 扩展）：
    - **Playwright MCP**（微软）：驱动浏览器，accessibility tree 快照，省 token — <https://github.com/microsoft/playwright-mcp>，OpenCode 配置教程 <https://opencodeguide.com/en/opencode-mcp-playwright/>
    - **Chrome DevTools MCP**（Google 官方）：调试已运行的浏览器，读 console / network / perf trace — <https://github.com/ChromeDevTools/chrome-devtools-mcp>
    - **different-ai/opencode-browser**：Chrome 扩展 + native messaging，复用真实浏览器登录态，最接近 "Claude in Chrome" — <https://github.com/different-ai/opencode-browser>
    - 备注：以上是目前查到的主流，需继续寻找更多方法（关注 [awesome-opencode](https://github.com/awesome-opencode/awesome-opencode)、[opencode.cafe](https://www.opencode.cafe/)、官方 issue #6992 进展）

---

# Q1: Council 是什么

**是什么**
Oh-My-OpenCode-Slim 提供的多 Agent 集成机制，本质是 ensemble learning（集成学习）。把同一个任务同时分发给多个 Agent（可以是不同模型），各自给出答案后计算置信区间、综合出最终结果。

**核心思想**
"Two heads are better than one"。单模型有偏见和盲区，多模型并行思考再汇总，能降低错误率、提高决策质量。

**适用场景**（成本高，不要滥用）
- 复杂架构决策（选型、设计权衡）
- 难调的 bug（单模型反复修不好的）
- 需要多角度交叉验证的问题
- 日常小改动用 Orchestrator 即可

**怎么用**
- Agent 选择里把主 Agent 切成 Council
- 测试连通性：选 Council 后输入 `test Council connectivity`，可看到每个子 Agent 各自响应
- 在 `~/.local/share/opencode/auth.json` 配置不同 provider，让成员模型异构，效果更好

**Council vs Orchestrator**
- **Orchestrator**：分工协作 — 把任务拆给 Explorer / Librarian / Fixer 等不同角色，做不同子任务
- **Council**：同题多解 — 多个 Agent 做**同一个**任务，再汇总投票

**配图**

![Once you've got Council set up, you can test the connectivity of each sub-agent.](https://storage.ghost.io/c/33/67/33678c00-2c15-4961-93e9-497b427e2006/content/images/2026/04/image-5.png)

> Once you've got Council set up, you can test the connectivity of each sub-agent.

---

# Q2: OpenCode Plan Agent vs OpenSpec

> OpenSpec 没有 Agent 概念，它是 slash 命令和工作流阶段（`/opsx-explorer` → `/opsx-propose` → `/opsx-apply` → `/opsx-verify` → `/opsx-archive`），由 OpenCode 的某个 Agent 来执行。

| 维度 | OpenCode **Plan Agent** | **OpenSpec** |
|------|-------------------------|--------------|
| **产物** | 一份临时 plan.md（会话内） | 仓库目录：`specs/`、`changes/<name>/proposal.md` + `design.md` + `tasks.md` + spec deltas |
| **持久化** | 会话结束即丢 | 提交进仓库，跨会话/跨 PR 存在；`specs/` 是真理之源 |
| **增量管理** | 每次从零生成 | Delta Specs（ADDED / MODIFIED / REMOVED），改老项目时知道哪些是新增/修改/废弃 |
| **并行变更** | 多个 plan 互相覆盖 | 多个 change 可同时改同一 spec 的不同需求而不冲突 |
| **工作流** | Plan → Build | Explorer → Propose → Apply → **Verify**（对照 tasks 验证实施）→ Archive |
| **跨工具** | 仅 OpenCode 内 | 跨工具：Claude Code / Cursor / Copilot / Cline / Windsurf |
| **适用规模** | 一次性任务、小改动、个人 demo | 多人协作、长期维护、需要变更审计的项目 |

**一句话**：Plan = 临时大脑；OpenSpec = 长期账本。

**参考资料**
- [Fission-AI/OpenSpec (GitHub)](https://github.com/Fission-AI/OpenSpec)
- [OpenSpec 官网](https://openspec.dev/)

---

# Q3: OpenSpec 做新需求时会扫之前的 spec 吗

**结论：会，但机制目前不算特别可靠。**

### 现状

**设计上是要扫的**
- `specs/` 是"真理之源"，新对话开始时 AI 应该读取它
- `config.yaml`（项目规则、技术栈、约定）会**主动注入到每次 proposal 请求**
- 新建变更时，AI 读 `proposal.md` 和 `specs/` 来理解现有需求
- 这是 OpenSpec 解决"AI 跨会话丢上下文"问题的核心机制

**执行上有缺陷**（来自官方 issue #901）
- 当前指令很模糊："Check `openspec/specs/` for existing spec names"
- AI 经常**跳过这一步**（缺少强制执行）
- 真去读时是把整份 spec 文件塞进 context，**浪费 token**
- spec 数量一多就扛不住

**改进方向**（issue 提议）
- 用 CLI 命令 `openspec list --specs --json --detail` 拿轻量目录（id、title、overview、requirementCount）
- 不必读全文，AI 也能判断"这次变更应该修改哪些已有 spec"

### 实操建议
- **小项目**（spec 数量个位数）：基本能正常工作
- **中大型项目**：
  - 在 `proposal.md` 或对话里**主动点名**相关 spec（"这次改动涉及 specs/auth、specs/billing"）
  - 关注 issue #901 进展
- **写新需求前**：先 `openspec list --specs` 自己看一眼有哪些已有 spec，避免重复或冲突

### 一句话
"会扫"是设计意图，当前实现依赖 AI 自觉、对大项目不友好；做大项目时建议手动引用相关 spec 名做兜底。

**参考资料**
- [Issue #901: Automatic spec catalog discovery during proposal creation](https://github.com/Fission-AI/OpenSpec/issues/901)
- [OpenSpec Deep Dive](https://redreamality.com/garden/notes/openspec-guide/)

---

# Q4: OpenCode agent 架构 与 context 管理策略

> 合并主题：OpenCode 的 agent 双层架构 + 它跟 Claude Code "新会话"建议的冲突 + 怎么按场景和模型决策。

## 一、Agent 的两种类型（架构事实）

OpenCode 官方架构把 agent 分成**两种完全不同**的东西，OMOS 在两层各加了几个：

| 类别 | 在 OpenCode 里的角色 | Context 行为 | 例子 |
|------|---------------------|------------|------|
| **Primary Agent**（主 agent） | 你直接对话的对象，处理主会话 | **共享 context**，全部消息历史在同一条流里。切换只换系统提示 + 工具权限集 | OpenCode 自带：**Build**、**Plan**<br>OMOS 加的：**Orchestrator**、**Council** |
| **Subagent**（子 agent） | 主 agent 委派出去的执行者 | **独立 child session**，自己的 context window，跑完返回摘要给父会话（`session_child_first` / `session_parent`） | OpenCode 自带：**General**、**Explore**<br>OMOS 加的：**Explorer**、**Librarian**、**Fixer** |

**架构证据**
- [OpenCode Agents 官方文档](https://opencode.ai/docs/agents/)
- 源码 `packages/opencode/src/session/message-v2.ts`：`AssistantMessage.agent` 是 per-message 字段，证明同 session 共享 context

## 二、与 Claude Code "新会话"建议的冲突

### 问题来源

Claude Code 官方最佳实践文档里"Let Claude interview you"一节的建议：

> 一旦规范完成，启动新会话来执行它。新会话有干净的 context，完全专注于实现，你有一个书面规范可以参考。
>
> — [Claude Code 最佳实践（中文）](https://code.claude.com/docs/zh-CN/best-practices) / [英文版](https://code.claude.com/docs/en/best-practices)

**摘要**：Claude Code 把 SPEC → 实现拆成两段——前一段在当前会话里让 Claude 用 `AskUserQuestion` 工具反复采访你、写出 SPEC.md；后一段**手动开新会话**，从干净 context 开始读 SPEC.md 干活。理由是规划阶段累积的对话会污染实现阶段的 context window，导致性能下降。

### 冲突点

OpenCode 裸用 Plan→Tab→Build 的流程，**与上面这条建议是相反的**——Tab 切换共享 context（见上节），规划讨论全部带进 Build 阶段。这不是工具 bug，是两边对 context 性质的判断不同。

### 两种设计哲学

| 维度 | Claude Code | OpenCode（裸 Tab 切换） |
|------|-------------|----------------------|
| **主张** | 规划完 → 开新会话执行 | 规划完 → Tab 切 Build 继续干 |
| **假设** | 规划阶段对话是**噪音**，会污染执行 context | 规划阶段对话是**连续性**，不要丢 |
| **后果** | 干净 context，但需要 SPEC.md 把决策外化 | 决策都在记忆里，但 context 越来越脏 |

## 三、Context 是资源还是负担——按场景 + 按模型判断

哪边对？**都对，看场景 + 看模型**。Context 不是天然的好东西也不是天然的坏东西。

### 按场景

| Context 是**资源**（保留它）<br>👉 Tab 切换 / 同会话继续 | Context 是**负担**（清掉它）<br>👉 开新会话 / 派 subagent |
|---|---|
| 规划阶段很短，几轮对话就敲定 | 规划阶段冗长，反复改主意、否过几个方案 |
| 探索性任务，决策还在变，前面的"为什么不选 X"是有用的 | 决策已敲定，"如何选到这里"对实现毫无价值 |
| 调试场景，前面失败的尝试能给下一次提供线索 | 已经在同一问题上纠正超过 2 次，旧上下文是污染源 |
| 小改动，重启的 setup 成本 > 污染成本 | 多文件大 feature，每个无关 token 都在拉低实现质量 |
| 决策很难外化到文档（隐性默契、口头共识） | 决策可以清晰落到 SPEC.md / OpenSpec 磁盘文件 |
| 紧凑的一次性任务，半小时内搞完 | 跨天/跨 session 的长跑工作 |

### 按模型

同样的 context，不同模型容忍度差很多：

| 模型特征 | Context 倾向 | 操作建议 |
|---------|------------|---------|
| **强模型 + 长 context window**（Opus 4.7 1M、Sonnet 4.6） | 偏**资源** | 噪音容忍度高，能从大量上下文里挑出有效信息；可以让 context 累积久一点，少 `/clear` |
| **弱模型 / 小 context window**（Haiku、本地小模型、32K 级窗口） | 偏**负担** | 一旦塞满早期指令就会"被遗忘"，性能断崖式下降；要更激进地 `/clear`、更早开新 session |
| **专门 coding-tuned 模型** | 偏**资源** | 对代码相关 context 抗干扰强，无关闲聊也能忽略 |
| **通用模型干 coding 活** | 偏**负担** | 容易被无关对话带偏，干净 context 收益大 |
| **跑在 auto / 非交互模式** | 偏**负担** | 没人盯着纠正，context 越脏越容易跑飞，更要清 |

**模型层面的实操含义**：
- 用 Opus 1M：Tab 切换 + 长会话基本没问题，OpenCode 那套连续工作流能跑得很顺
- 用 Haiku 或本地 7B/13B：Claude Code 这套"短会话 + SPEC 落盘 + 新 session"的纪律必须严格执行，否则模型扛不住
- 同一个流程在不同模型上的效果不一样——不要把"在 Opus 上验证过的 prompt 工作流"无脑搬到弱模型上

### 判断口诀

- 问自己"如果删掉规划阶段的对话，光看 SPEC，实现能不能干"——能干 → context 是负担，开新会话；不能干 → context 是资源，留着
- 问自己"前面的对话里有多少是有效决策、多少是兜兜转转的过程"——过程比例高 → 清掉；决策密度高 → 留着
- 问自己"我用的模型能扛多长的脏 context"——扛得住 → 给点连续性；扛不住 → 严格隔离

## 四、三条路径对应关系

| OpenCode 流程 | 对应 Claude Code 建议 | 适用场景 |
|--------------|---------------------|---------|
| 裸 Plan → Tab → Build | ❌ 与 Claude Code 主张相反，规划噪音进执行 | 小改动、规划很短、用强模型 |
| Plan → 写 OpenSpec 到磁盘 → 新 session Build | ✅ 等价于 SPEC.md + 新会话 | 大 feature、规划冗长、跨会话工作 |
| Build → 派 Subagent 干子任务 | ✅ 等价于 Claude Code 的 subagent | 有界子任务（调研、verify、批量小修） |

## 五、推论

OpenSpec 把 spec 落盘到 `specs/`，本质上就是 OpenCode 社区**承认 Tab 切换不够用**的产物——光靠 primary agent 内部传递规划决策有 context 污染问题，所以把 spec 外化成磁盘文件，让"另开 session 执行"变得无痛（呼应 Q2：Plan = 临时大脑，OpenSpec = 长期账本）。
