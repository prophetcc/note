# 笔记

## 导航

- [待办](#待办)
- [Q1: Council 是什么](#q1-council-是什么) — OMOS 的多 Agent 集成机制（同题多解 + 投票）
- [Q2: OpenCode Plan Agent vs OpenSpec](#q2-opencode-plan-agent-vs-openspec) — 临时大脑 vs 长期账本
- [Q3: OpenSpec 做新需求时会扫之前的 spec 吗](#q3-openspec-做新需求时会扫之前的-spec-吗) — 设计要扫，执行靠 AI 自觉
- [Q4: OpenCode agent 架构 与 context 管理策略](#q4-opencode-agent-架构-与-context-管理策略) — Primary/Subagent 双层、与 Claude Code "新会话"建议的冲突、按场景+模型决策
- [Q5: 前端测试方案：Storybook + Playwright + toHaveScreenshot](#q5-前端测试方案storybook--playwright--tohavescreenshot) — 各层职责、E2E 怎么测得稳、痛点与解决方案
- [Q6: 视觉回归 CI 工作流——三种方案](#q6-视觉回归-ci-工作流三种方案) — 自托管两遍跑、本地预 approve、CI artifact、SaaS 一遍跑的取舍

> 待整理（已挪到 [tbd.md](tbd.md)）：Q7 OpenSpec + Superpowers 组合工作流；Q8 CLAUDE.md / Skill / Hook 三层强度

---

## 待办

- [ ] 学习 Claude Code Chrome 扩展：<https://code.claude.com/docs/zh-CN/chrome>（用浏览器自动化让 Claude 自己验证 UI 改动）
- [ ] 研究 OpenCode 浏览器自动化的主流做法（OpenCode 本身没有官方 Chrome 扩展）：
    - **Playwright MCP**（微软）：驱动浏览器，accessibility tree 快照，省 token — <https://github.com/microsoft/playwright-mcp>，OpenCode 配置教程 <https://opencodeguide.com/en/opencode-mcp-playwright/>
    - **Chrome DevTools MCP**（Google 官方）：调试已运行的浏览器，读 console / network / perf trace — <https://github.com/ChromeDevTools/chrome-devtools-mcp>
    - **different-ai/opencode-browser**：Chrome 扩展 + native messaging，复用真实浏览器登录态，最接近 "Claude in Chrome" — <https://github.com/different-ai/opencode-browser>
    - 备注：以上是目前查到的主流，需继续寻找更多方法（关注 [awesome-opencode](https://github.com/awesome-opencode/awesome-opencode)、[opencode.cafe](https://www.opencode.cafe/)、官方 issue #6992 进展）
- [ ] 深入学习 Vitest watch mode：自动监听 / `--changed` / `--related` / `-t` 过滤等命令的实战用法 — <https://vitest.dev/guide/cli>
- [ ] 研究 affected detection（依赖图分析做"安全增量 CI"）：
    - **Nx**（monorepo 编排，能算出"改了 A 包，B/C 受影响"）— <https://nx.dev/>
    - **Turborepo**（轻量、缓存粒度细）— <https://turbo.build/>
    - **Lage**（微软的 monorepo 编排器）— <https://github.com/microsoft/lage>
    - 备注：普通非 monorepo 项目直接全量 CI 即可，配缓存就行，不要瞎搞增量

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

---

# Q5: 前端测试方案：Storybook + Playwright + toHaveScreenshot

## 一、整体定位

每个工具只干一件事，不重叠：

```
Vitest          → 测代码逻辑
Storybook       → 测组件表现
Playwright 断言 → 测应用行为
Playwright 截图 → 测应用视觉
```

## 二、各工具的使用场景

### Vitest（基础层，90% 项目必须有）

**用来测**：
- 工具函数（format、validate、parse、calculate）
- 自定义 hooks 的状态机逻辑
- 状态管理（Redux / Zustand store）的 reducer / action
- 业务逻辑函数（购物车计算、权限判断、表达式求值）

**不要用它测**：UI 视觉、跨页面行为、真浏览器渲染。

#### 收益（为什么要装）
- **快**：几秒跑完几百个测试，watch mode 改完保存秒级反馈
- **覆盖逻辑边界**：30 个边界场景 = 30 个毫秒级 unit test，便宜
- **重构敢动手**：改 util 跑一下测试，几秒确认没坏
- **覆盖率信号强**：逻辑覆盖率直接体现在 unit 层，不用通过 UI 推断
- **几乎零成本**：`npm i vitest` 一行装好，跟 Vite 共用配置

#### 代价（不装会怎样）
- **Playwright 测试爆炸**：所有逻辑被迫通过 UI 测，100 个边界 = 100 个 E2E
- **CI 跑到死**：Vitest 几秒跑 1000 个，Playwright 跑 1000 个要几十分钟
- **逻辑覆盖率盲区**：E2E 90% 覆盖率 ≠ 逻辑 90% 覆盖
- **重构难**：改一个 util 不敢动，要看 N 个 E2E 受影响
- **测试体量倾斜**：原本 70% Vitest + 15% Storybook + 15% Playwright，没 Vitest 变成 25% Storybook + 75% Playwright，后者的运维成本是前者几十倍

#### 什么时候可以省

| 项目特征 | 能省吗 |
|---------|-------|
| 纯展示型 dashboard（API 数据 → 渲染） | ✅ |
| 静态营销网站 | ✅ |
| 简单 CRUD 后台（逻辑都在后端） | ✅ |
| Prototype / MVP 早期 | ✅（上线前补） |
| 复杂 SaaS（计费、权限、状态机） | ❌ |
| 工具类应用（编辑器、计算器） | ❌ |
| 有共享 utils / hooks 库 | ❌ |
| 多人协作项目 | ❌ |

#### 判断信号
打开项目根目录，看有没有 `utils/` `hooks/` `lib/` `services/` `helpers/` 这类**非 JSX 文件夹**：
- 有，文件 5 个以上 → 必装
- 没有，全是组件 → 可以先不上

或者问"项目里非组件代码占多少"：30%+ → 必装；10% 以下 → 可以省。

#### 替代选项
如果不想用 Vitest：
- **Jest**：写法一样，但慢、Vite 项目要折腾配置
- **Node 自带 test runner**（Node 18+ 内置 `node:test`）：零依赖，适合超小项目
- **Bun test**：用 Bun 的话内置，最快
- ❌ 别用 Mocha + Chai + Sinon 组合（2026 年没必要）

### Storybook（组件层）

**用来测/展示**：
- 原子组件（Button / Input / Modal / Tabs）的所有状态
- 业务原子组件（PriceTag / OrderStatus / UserAvatar）
- 表单字段组合
- 设计系统的 token / 主题
- 组件级的交互行为（用 play 函数）
- 组件级的可访问性（addon-a11y）
- 组件级的多状态可视化（loading / error / empty / data）

**不要用它装**：整个业务页面（需要路由+认证+全局状态的）、Layout 容器、跟后端深度耦合的组件。

**判断口诀**：这个东西**多个页面会复用吗**？是 → 进；否 → 不进。

### Playwright 功能断言（应用层 - 行为）

**用来测**：
- 关键用户旅程（登录、注册、付款、核心功能）
- 跨页面流程
- 路由跳转正确性
- 表单提交后的状态变化
- 错误处理 / fallback UI 显示
- 认证流程 / 权限控制
- 真生产构建产物的运行时行为

**断言用 `expect()`**，不要用截图——能 assert 的事都不该截图。

### Playwright `toHaveScreenshot`（应用层 - 视觉）

**用来测**：
- 关键页面的默认状态视觉
- 关键页面的错误 / 空 / loading 状态视觉
- 响应式断点（mobile / tablet / desktop）
- 主题切换（light / dark）
- 跨主要浏览器渲染差异
- 设计稿还原度

**用法约束**：
- 数量控制在 20-50 张以内
- 只截关键状态，不要每个交互都截
- mask 掉动态内容（时间戳、ID）
- disable 动画
- 等 networkidle 再截

## 三、四个工具的协作分工速查

| 测试目标 | 用什么 |
|---------|-------|
| 一个函数返回值对不对 | Vitest |
| 一个 hook 状态对不对 | Vitest |
| 一个 Button 在 5 种 prop 下长啥样 | Storybook |
| 点击一个 Modal 能不能打开 | Storybook play 函数 |
| Modal 在 dark/light 下视觉对不对 | Storybook（写两个 story）|
| 用户能不能从首页走到付款页 | Playwright 断言 |
| 表单提交失败显示错误 | Playwright 断言 |
| 整个付款页的视觉布局对不对 | Playwright 截图 |
| 移动端首页排版没炸 | Playwright 截图（移动 viewport） |
| Safari 上能不能正常用 | Playwright 跨浏览器跑 |

## 四、E2E 测试不是要不要测，而是怎么测得稳

Playwright 高保真但环境依赖——后端挂、API 变、第三方限流、网络抖动都会让测试 flaky。**纯打真后端的 E2E 测试 flaky 率通常 5-20%**。

关键不是放弃 E2E，而是**在合适的层级 mock**：

```
保真度 ↑                          稳定性 ↓
  │ ① 全打生产                          每次都 flaky
  │ ② 打 staging                        依赖 staging 数据
  │ ③ 打本地 dev server                 依赖本地后端
  │ ④ 真应用 + mock 后端  ← 甜点位      可控且稳定
  │ ⑤ Vitest 单元测试                   完全稳定但保真度低
保真度 ↓                          稳定性 ↑
```

**第 ④ 层是 95% 场景的最优解**：浏览器真、应用真、构建真，**只把外部后端 mock 掉**——拿到 95% 的 E2E 价值 + 单元测试级的稳定性。

### Playwright 里怎么 mock 后端

- **网络层拦截**（最常用）：用 `page.route('**/api/**', route => route.fulfill({ json: mockData }))` 拦所有 API 调用，给固定假数据
- **MSW 共享 handler**：mock 数据可以跨 Vitest / Storybook / Playwright 共享，避免漂移
- **故意模拟故障**：`fulfill({ status: 500 })` 测错误 UI，加 setTimeout 测 loading 状态——这是真后端做不到的

### 真后端测试的少数场景

只有这几类**必须打真后端**：
- 后端集成正确性
- 部署后的冒烟测试
- 数据库 migration 正确
- 第三方支付真实流程（用 sandbox 账号）
- 性能 / 负载测试

应对 flaky 的策略：
- **测试隔离**：每个测试用独立 user 和数据（如 `test-${Date.now()}@x.com`）
- **DB seed / reset**：每次测试前重置到已知状态
- **重试机制**：`retries: 1`（CI），但不要无脑加更多——会掩盖真 bug
- **分层 CI**：每次 PR 跑 mock E2E（强制门），每天定时跑真 staging（监控用），部署后跑生产冒烟（兜底）

## 五、痛点与解决方案

### 痛点 1：跨平台截图基线不兼容
本地 macOS 截的 png 和 CI Linux 截的字节不同（字体抗锯齿差异）。

**解决**：视觉测试只在 CI Linux 跑；本地不跑或用 Docker 跑同环境；基线只提交 Linux 版。

### 痛点 2：Playwright 测试 flaky
网络慢、动画没结束、字体没加载完导致截图不一致；后端时好时坏导致断言偶尔挂。

**解决**：
- 后端用 `page.route()` mock，不打真后端（除非冒烟）
- 截图前 `await page.waitForLoadState('networkidle')`
- 截图配 `animations: 'disabled'`
- 字体加载等 `await document.fonts.ready`
- mask 动态内容
- CI 加 `retries: 1`

### 痛点 3：首次基线没人审就生效
第一次写视觉测试，没基线时自动生成的就是基线——可能本身就有 bug。

**解决**：第一次生成的基线**作为代码 review 的一部分**——PR 里让人看 png；重要页面建议设计师 review 后再合。

### 痛点 4：PR 里看 PNG diff 体验差
GitHub 显示 PNG 二进制 diff，看不出哪儿变了。

**解决**：CI 把 Playwright HTML report 作为 artifact 上传；PR 评论贴下载链接；团队大可以上 Argos / Lost Pixel。

### 痛点 5：基线维护越来越重
改一个共享样式，几十张截图全变要逐个 approve。

**解决**：
- 视觉测试**少而精**——只测关键状态
- 共享样式改动放独立 PR 专门审视觉
- `maxDiffPixels` 给一点容忍度
- 截图加描述性文件名

### 痛点 6：Storybook 和 Playwright 测试覆盖重复
一个 Button，Storybook 测了，Playwright 路径里又点到了。

**解决**：明确分工——**Storybook 测组件本身**（5 种 prop 状态），**Playwright 测组件在真应用里被使用**（跑用户路径时它能不能正常 click）。Playwright 不要专门为 Button 写测试。

### 痛点 7：业务页面状态太多
付款页有 loading / 空购物车 / 数据 / 错误 / 优惠券 / 地址未选 等 7 种状态。

**解决**：
- 只截 **2-3 个最关键的状态**（默认 + 错误 + 一种重要变体）
- 其他状态用 Playwright 功能断言覆盖
- 状态视觉的覆盖在**组件层**做（Storybook 写 7 个 story）

### 痛点 8：Storybook stories 没人维护
组件改了 stories 没跟上；半年后全过期。

**解决**：CI 跑 storybook test runner，故事报错就 fail；团队约定组件 PR 必须带 stories；删除长期没人看的 story。

### 痛点 9：mock vs 真后端取舍
mock 跟真后端响应不一致，上线挂；全打真后端 flaky 高。

**解决**：分层 CI（前面已说）；用 OpenAPI / 类型定义生成 mock 数据避免漂移；后端有 contract 变更 → 自动通知前端重生成 mock。

### 痛点 10：CI 时间太长
所有测试合起来 PR 等 30 分钟。

**解决**：
- 分 job 并发跑
- GitHub Actions 矩阵分片 Playwright（`--shard 1/4`）
- 视觉测试只在 main 分支或 daily 跑（PR 不卡）
- Vitest 只跑改动相关测试（基于 git diff）

### 痛点 11：构建产物 bug 没人测
dev 模式跑测试都过，部署后用户报错。

**解决**：Playwright 的 webServer 跑 `npm run build && npm run preview`，**不要跑 dev server**。这样测的就是真生产构建（min + tree-shake + 环境变量替换后）。

### 痛点 12：第三方 SDK 测试
Stripe / Map / Auth0 在测试里要么真调（rate limit），要么 mock 不准。

**解决**：用各家提供的 sandbox 账号 / test mode；mock 在网络层（`page.route('**/stripe.com/**')`）；关键流程才打真 sandbox。

## 六、推荐工作流

| 阶段 | 做什么 |
|------|------|
| **本地开发** | Vitest watch 实时跑单测 + Storybook 看组件 + Playwright UI mode 调流程 |
| **提 PR** | CI 并行 5 job：static / unit / storybook test runner / playwright 断言 / playwright 视觉。任意失败不让合 |
| **合并后 / 部署后** | 每天定时跑真 staging 冒烟；部署后跑生产关键路径冒烟；Sentry 监控真错误 |

## 七、各工具占测试总数比例（健康基线）

```
Vitest 单元测试        ~70%   （最大头，覆盖逻辑）
Storybook stories     ~15%   （组件展示+交互）
Playwright 功能断言    ~12%   （关键路径）
Playwright 视觉截图    ~3%    （关键状态视觉）
```

**信号**：如果 Playwright 视觉测试占到 30% 以上，架构错了——大概率把行为测试当成视觉测试在做。

## 八、测试执行范围：本地增量 vs CI 全量

**核心原则**：本地求快增量跑（可漏，CI 兜底）；CI 必须全量跑（漏测会上线）。

### 不同 PR 类型的本地策略

| PR 类型 | 本地最少跑 | CI 跑啥 |
|--------|-----------|--------|
| 改 README / docs | 啥也不用 | lint 即可 |
| 改 utils 工具函数 | Vitest watch + 想到的相关组件 | 全量 |
| 加新组件 | 该组件的 stories + Vitest | 全量 |
| 改业务页面 | 相关 Playwright spec | 全量 + 视觉 |
| 改全局样式 / theme | Storybook 看几个组件 | **必跑视觉回归全量** |
| 改路由 / 全局 layout | Playwright 关键流程 | 全量 + 视觉 |
| 改 API 客户端 | Vitest mock 测试 | 全量 + 真后端冒烟 |

### CI 时间太长怎么办

如果全量 CI 超过 10 分钟：

1. **并发分 job**：typecheck / lint / unit / storybook / e2e 不互相依赖，并行跑
2. **Playwright 分片**：`playwright test --shard 1/4`，4 个 job 并行，单 job 时间 / 4
3. **视觉测试单独 schedule**：挪到 main 分支或 daily，PR 不卡
4. **缓存**：node_modules / Playwright 浏览器二进制 / TypeScript incremental / Vite build 都能缓存
5. **关键路径优先**：必过的（typecheck + unit + 关键 E2E）放前面 fail-fast，次要的（视觉、a11y、性能）后面

### 不要做的事

| 反模式 | 为啥不好 |
|-------|---------|
| 本地全量跑视觉测试 | 基线跨平台不兼容，大概率全红 |
| CI 上只跑改动文件相关测试 | 跨文件依赖你看不见，会漏回归（除非用 Nx/Turborepo 做真依赖图分析） |
| 本地不跑任何测试，全靠 CI | 浪费 CI 资源 + 反馈延迟 |
| watch mode 一直开着不跑 PR 前全量 | watch 可能漏掉某些场景 |
| 跳过测试急着合 | 出 bug 还是你修 |

### 推荐工作流（详细版）

**开发中**：
- Vitest watch 开着，改文件秒级反馈
- 改组件 → Storybook UI 看效果
- 改流程 → Playwright UI mode 调（`playwright test --ui`）

**提 PR 前自己跑**：
- `tsc --noEmit`（全项目，几秒）
- `biome check`（lint，全项目）
- `vitest run`（全量单测，很快）
- `npx test-storybook`（全量 stories）
- `playwright test e2e/<改动相关>.spec.ts`（只跑你改的流程）
- 视觉测试**不在本地跑**，让 CI 跑

**CI 上**（自动）：
- 全量并行跑所有 job
- 任意失败不让合（强制门）

### 关键反原则

**绝对不要**让 CI 也搞"只跑改动相关"——除非你用 **Nx / Turborepo / Lage** 做真依赖图分析，否则会放跑回归。**漏测的成本永远比多跑的成本高**。

## 九、一句话

**Storybook 管组件，Playwright 断言管行为，Playwright 截图管视觉，Vitest 管逻辑**。每层只干自己的事。最大痛点是基线维护和 flaky——靠**网络层 mock + 数量控制 + 只在 CI Linux 跑视觉**这三招解决 80%。E2E 不是要不要测，是**在第 ④ 层 mock**——保真度和稳定性的甜点。

---

# Q6: 视觉回归 CI 工作流——三种方案

## 问题来源

视觉回归测试有个固有"两遍跑"问题：自托管基线（Playwright `toHaveScreenshot`）下，**视觉变化的 PR 第一遍 CI 必然失败**（diff 检测出来了），需要人审 → 更新基线 → push → 第二遍 CI 才通过。

这是不是"每次都要跑两遍 CI"？答案是——**取决于你选哪种工作流**。

## 关键概念

**自托管基线**（Playwright `toHaveScreenshot`）：基线 png 文件提交进 git，跟代码一起版本管理。
**SaaS 基线**（Chromatic / Argos / Percy）：基线在云端数据库，git 仓库不存截图。

两遍跑的根源：自托管下"approve 新基线" = 改 git 文件 = 必须 commit + push = 触发新 CI。

## 三种方案对比

| 方案 | 跑几遍 CI | 本地需要 Linux | Review 体验 | SaaS 费用 |
|------|---------|--------------|-----------|----------|
| **A. 本地预 approve** | 1 遍 | ✅ 必须（Docker 或 Linux 机器） | reviewer 在 PR 看 png 文件，普通 | $0 |
| **B. CI artifact + 手动触发 update** | 2 遍 | ❌ 不用 | CI artifact 下载，普通 | $0 |
| **C. SaaS（Chromatic / Argos）** | 1 遍 + 网页 approve | ❌ 不用 | 网页 review 丝滑 | $50-300/月起 |

## 方案 A：本地预 approve（自托管最常用）

**流程**：

```
本地（必须 Linux 环境）：
  改代码 → 跑视觉测试 → 看 diff → 确认有意
  → npx playwright test --update-snapshots
  → git add 基线变化 + 代码改动一起 commit
  → push
   
CI：
  只跑一次 → 基线和截图一致 → 通过 ✅
```

**优点**：CI 一遍过；省 SaaS 钱。
**缺点**：
- 必须本地有 Linux 环境（Mac / Windows 开发要 Docker）
- diff review 在你本地，**reviewer 看不到 diff**——除非他也下载 png 自己看
- 容易"自己 approve 自己"，缺第二双眼睛

## 方案 B：CI artifact + 手动触发 update

**流程**：

```
第一遍 CI：跑视觉测试 → 失败 → 上传新截图作为 artifact
   ↓
你下载 artifact，看 diff
   ↓
触发一个 GitHub Actions job："accept-snapshots"
   ↓
该 job 把 artifact 里的截图覆盖到 git，commit + push
   ↓
第二遍 CI：自动跑 → 通过 ✅
```

**优点**：本地不用 Linux 环境；所有事在 CI 上完成。
**缺点**：还是两遍 CI；要写 accept-snapshots 这个 job 的脚本。

## 方案 C：SaaS（Chromatic / Argos / Percy）

**流程**：

```
你 push 改动 → CI 跑测试 → 截图上传到 SaaS
   ↓
PR 评论里贴 diff 链接
   ↓
你在网页上看 diff → 点 "Accept"
   ↓
SaaS 把新截图标为基线 → PR 状态变绿 ✅ → merge
```

**为什么 SaaS 能省一遍**：基线不在 git 里——是 SaaS 数据库的指针。Approve = 改数据库指针，不动代码、不重跑测试。

**优点**：CI 一遍过；网页 review 丝滑；设计师 / PM 都能直接 review；自带 cross-browser/viewport 矩阵管理。
**缺点**：要钱（5000 snapshots/月以上开始烧）；vendor lock-in。

## 选型建议

| 场景 | 推荐方案 |
|------|---------|
| 个人 / 小团队，预算 $0 | **方案 A**（本地 Docker 预 approve） |
| 中型团队，没 Mac/Linux 鸿沟 | **方案 B**（CI artifact） |
| 团队有设计师重度参与 review | **方案 C**（Chromatic 体验最好） |
| 中型团队，要 SaaS 体验但预算紧 | **方案 C 用 Argos**（比 Chromatic 便宜多） |
| 开源项目 | **方案 C 白嫖 Chromatic**（开源仓库无限免费） |

## 不是每次 PR 都要跑两遍

**只有视觉真改了的 PR 才会触发**。普通 PR（改逻辑、改 API、修 bug）视觉不变，一遍过。**两遍只在改 UI 时承担**——这恰恰是质量门该卡的地方。

进阶：用 GitHub Actions `paths` filter 让视觉测试只在 UI 文件改动时触发，进一步减少不必要的两遍跑。

## 一句话

**自托管 = 改 UI 的 PR 跑两遍 CI（除非本地 Linux 预 approve）；SaaS = 一遍 CI + 网页 approve**。两遍是自托管的代价，但只在改 UI 的 PR 上承担——平时绝大多数 PR 还是一遍过。**取舍核心是预算 vs 工作流体验**。

