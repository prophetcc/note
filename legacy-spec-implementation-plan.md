# 纯前端老项目补充 Spec 的实施方案

## 目标

为纯前端老项目补充 spec，让团队和内部模型能够理解当前页面、组件、交互、状态和接口契约的真实行为。

本方案只关注“补充 spec”这件事，不覆盖测试补充、代码修改、需求变更实施、OpenSpec change 执行流程或 ADR。补 spec 必须采用人机协作方式，AI 不能只靠扫描代码一次性生成最终 spec；AI 需要持续向业务负责人或项目维护者询问关键业务问题。

需要维护两套互补 spec：

- 通用前端业务 Spec：用于沉淀页面/组件理解、当前行为、代码事实、不确定点和上下文。
- OpenSpec Baseline Spec：用于沉淀已确认的当前前端能力承诺。

## 关键边界

1. 当前没有新需求或行为变更，只补充现状 spec。
2. 不创建 `openspec/changes/`。
3. 不写 `proposal.md`。
4. 不写 `tasks.md`。
5. 不写 delta spec。
6. 不补测试。
7. 不修改业务代码。
8. 不写 ADR。
9. AI 必须在发现业务含义不明确、代码行为冲突、页面文案无法解释、权限/状态/接口语义不清时暂停并提问。

## OpenSpec 中 Spec 的两种含义

OpenSpec 里有两类文件都叫 `spec.md`，但语义不同：

```text
openspec/specs/<capability>/spec.md
```

这是 baseline spec，表示当前系统已经确认的能力，是最终沉淀后的当前行为。

```text
openspec/changes/<change-name>/specs/<capability>/spec.md
```

这是 delta spec，表示某次变更相对于 baseline 的新增、修改或删除。

当前任务只补老项目前端现状，没有新需求，所以只写顶层 baseline spec：

```text
openspec/specs/<capability>/spec.md
```

不要创建 change 目录，也不要使用 `ADDED Requirements`、`MODIFIED Requirements`、`REMOVED Requirements`。

## 当前阶段推荐目录结构

```text
docs/
  business-specs/
    user-list-filtering.md
    order-detail-page.md
    permission-gated-actions.md

openspec/
  specs/
    user-list-filtering/
      spec.md
    order-detail-page/
      spec.md
    permission-gated-actions/
      spec.md
```

## 人机协作要求

AI 不能把代码扫描结果直接当成最终业务事实。补充 spec 时必须循环执行：

```text
阅读局部代码和页面行为
  -> 提炼当前理解
    -> 向用户提出业务问题
      -> 根据回答更新通用前端业务 Spec
        -> 只把已确认规则提炼进 OpenSpec baseline
```

AI 应该持续向用户询问以下类型的问题：

- 业务目标问题：这个页面/流程最终服务哪个用户动作？成功标准是什么？
- 术语问题：页面里的字段、状态、标签、按钮在业务上分别是什么意思？
- 规则问题：某个按钮为什么可见、隐藏、禁用或可点击？
- 状态问题：loading、empty、error、success、disabled 等状态分别应该如何表现？
- 接口问题：某个请求参数或响应字段在业务上代表什么？哪些字段是前端必须依赖的？
- 权限问题：不同角色、租户、环境或配置下 UI 行为是否不同？
- 异常问题：接口失败、无数据、部分数据缺失、重复点击、刷新页面时应该如何处理？
- 历史问题：代码里的兼容分支、特殊判断、TODO/FIXME 是否仍然有效？
- 取舍问题：如果当前代码行为和业务理解冲突，应该以哪个为准？

AI 提问时应遵守：

- 一次只问少量高价值问题，避免把整页问题一次性抛给用户。
- 问题必须带上下文，说明来自哪个页面、组件、代码片段或 UI 行为。
- 问题应尽量可回答，例如“这个按钮在无权限时应该隐藏还是禁用？”而不是“这里业务是什么？”
- 用户没有确认的问题，只能写进通用前端业务 Spec 的“不确定点”。
- 未确认内容不能写进 OpenSpec baseline。
## 提问 Skill 选择

补充 spec 时采用 `grill-me` 作为主要提问协议，不默认使用 `grill-with-docs`。

### 为什么选择 grill-me

`grill-me` 的核心行为是：围绕一个计划或设计持续追问，直到双方达成共同理解；每次只问一个问题；如果问题能通过探索代码库回答，就先探索代码库而不是打扰用户。

这与当前目标匹配：

- 当前目标是补充 spec，不是实现新需求。
- 当前目标需要 AI 持续追问业务问题。
- 当前目标要求模型能先读本地代码，再问用户无法从代码确认的问题。
- 当前目标不需要额外创建 ADR 或维护新的上下文文档。

### 为什么不默认使用 grill-with-docs

`grill-with-docs` 也具备持续追问、术语澄清、场景压力测试和代码交叉验证能力，但它还会维护 `CONTEXT.md`，并在满足条件时建议 ADR。

这与当前边界不完全匹配：

- 当前方案只补通用前端业务 Spec 和 OpenSpec baseline。
- 当前方案明确不写 ADR。
- 当前方案不希望引入 `CONTEXT.md` 作为第三套文档。

因此，默认不使用 `grill-with-docs`。只有当用户明确要求维护项目术语表、`CONTEXT.md` 或 ADR 时，才考虑使用 `grill-with-docs`。

### grill-me 触发时机

`grill-me` 不应只在任务开始前触发。补充 spec 的整个过程都应保留 `grill-me` 检查点，但 AI 不应一直提问而不沉淀内容。只要 AI 准备把代码观察上升为业务事实，就必须判断是否需要提问。

默认行为：

1. 能从代码确认的事实，先写入通用前端业务 Spec。
2. 不能确认但不阻塞理解的内容，写入“不确定点”。
3. 只有会影响 capability 边界、OpenSpec baseline、关键业务规则或用户可见行为的内容，才触发 `grill-me` 提问。
4. 每次 `grill-me` 提问后，AI 必须根据用户回答更新通用前端业务 Spec。
5. 当问题不影响当前 spec 的最低可用版本时，不应阻塞继续整理。

AI 在以下情况必须触发 `grill-me` 提问：

1. 无法从代码确认业务含义。
2. 同一行为在代码、页面文案、接口字段或用户描述中存在冲突。
3. 需要决定 capability 名称或边界。
4. 准备把规则从通用前端业务 Spec 提炼进 OpenSpec baseline。
5. 发现特殊分支、兼容逻辑、TODO/FIXME、魔法值。
6. 发现权限、状态、URL query、缓存、表单校验等规则不明确。
7. 需要判断某个行为是业务规则、历史遗留，还是实现细节。
8. 当前页面或流程的关键用户路径无法仅凭代码解释。

AI 每完成一个局部阅读单元后，都必须判断是否需要触发 `grill-me`。

局部阅读单元包括：

- 一个页面
- 一个用户流程
- 一个核心组件
- 一个状态模块
- 一个接口封装
- 一个权限判断逻辑
- 一个 URL query 或缓存同步逻辑
- 一个表单校验或提交逻辑

### grill-me 提问节奏

AI 不应一直提问而不沉淀内容，也不应为了赶进度跳过关键业务确认。

默认节奏：

1. 每个局部阅读单元可以连续提问 1 到 7 轮。
2. 每轮最多问 1 到 3 个强相关问题。
3. 每轮提问前，AI 必须先说明：已从代码确认了什么、当前推断是什么、哪些问题会影响 spec 正确性。
4. 用户回答后，AI 必须先更新通用前端业务 Spec，再决定是否继续提问。
5. 如果连续 7 轮后仍有不确定点，AI 应把剩余问题写入“不确定点”，继续整理不受影响的部分。
6. 只有当不确定点会影响 OpenSpec baseline 或 capability 边界时，才暂停等待用户确认。
写入 OpenSpec baseline 前，必须至少进行一次 `grill-me` 确认，确认：

- capability 名称是否正确。
- capability 覆盖范围是否正确。
- 哪些规则已经确认。
- 哪些规则只能留在通用前端业务 Spec 的不确定点。
- 是否存在代码行为和业务预期冲突。
### grill-me 在本方案中的使用方式

内部模型在补充每个页面、流程或组件 spec 时，应按 `grill-me` 的方式工作：

1. 先阅读相关路由、页面、组件、hooks/composables、状态模块和接口封装。
2. 能从代码确认的问题，先从代码中确认。
3. 不能从代码确认的问题，再向用户提问。
4. 每次只问一个关键问题，最多不超过 3 个紧密相关问题。
5. 每个问题都要附带推荐答案或当前推断，方便用户确认或纠正。
6. 用户确认后，更新通用前端业务 Spec。
7. 只有已确认内容才能进入 OpenSpec baseline。

### 可借鉴 grill-with-docs 的提问技巧

虽然不默认使用 `grill-with-docs`，但可以借鉴它的提问技巧：

- 当术语模糊时，要求用户选择一个精确含义。
- 当同一术语在代码、页面文案和用户描述中含义冲突时，立即指出冲突。
- 用具体场景追问边界，例如“无权限用户看到按钮时是隐藏、禁用还是展示提示？”
- 当用户描述和代码行为冲突时，明确提问：“以当前代码为准，还是以业务预期为准？”

借鉴这些技巧时，不创建 `CONTEXT.md`，不创建 ADR，不把术语表当作第三套 spec。
## 两套 Spec 的分工

### 通用前端业务 Spec

通用前端业务 Spec 是理解底稿。它可以记录调研过程、代码推断、不确定点和业务上下文。

适合写：

- 页面或用户流程目标
- 路由和入口
- 相关页面、组件、hooks/composables
- 当前 UI 行为
- 当前交互规则
- 当前状态管理规则
- 当前接口调用和响应处理
- 当前权限、可见性和禁用规则
- 当前加载、空状态和错误状态
- 当前表单校验规则
- 当前 URL query、本地缓存或状态恢复行为
- 示例场景
- 异常情况
- 副作用，例如路由跳转、弹窗、toast、埋点、缓存更新、URL 更新
- 相关代码位置
- 历史问题
- 不确定点
- 与 OpenSpec 的映射关系

### OpenSpec Baseline Spec

OpenSpec baseline spec 是当前能力承诺。它只写已经确认的前端行为，不写调研过程、不确定点和代码猜测。

适合写：

- Requirement
- Scenario
- GIVEN / WHEN / THEN
- 已确认的 UI 行为
- 已确认的交互规则
- 已确认的状态规则
- 已确认的接口请求触发和响应处理
- 已确认的 URL、缓存、权限、表单规则

## Capability 边界确认规则

AI 不能自行决定 OpenSpec capability 名称或边界。哪些页面、流程、组件或交互规则属于同一个 capability，必须经过用户确认。

在创建或映射 OpenSpec baseline 前，AI 必须先提出 capability 候选：

```md
## Capability 候选

### 候选名称
`user-list-filtering`

### 覆盖范围
- 用户列表页的筛选表单
- 筛选条件和 URL query 同步
- 提交筛选后的列表刷新
- 页面刷新后的筛选条件恢复

### 不覆盖范围
- 用户详情页
- 用户编辑表单
- 用户权限配置
- 列表行操作按钮权限

### 命名依据
该 capability 描述的是用户列表筛选能力，而不是某个具体组件。

### 需要用户确认
1. 这个 capability 名称是否合适？
2. 上述覆盖范围是否正确？
3. 哪些行为应该拆到其他 capability？
```

用户确认前：

- 不创建 `openspec/specs/<capability>/spec.md`。
- 不在通用前端业务 Spec 中写死 OpenSpec Mapping。
- 不把多个页面或流程强行合并为一个 capability。
- 不把同一业务能力拆成多个过细 capability。

命名建议仅作为候选，不作为最终决定：

- 使用 kebab-case，例如 `user-list-filtering`。
- 优先使用业务能力或用户流程命名，不优先使用组件名。
- 页面级能力可以使用页面加行为，例如 `user-detail-permissions`。
- 如果一个页面包含多个独立业务能力，应先询问用户是否拆分。
- 如果多个组件共同完成一个用户流程，应优先归入同一个 capability，但必须经用户确认。
## 通用前端业务 Spec 模板

```md
# [页面/流程/组件名称] 前端业务 Spec

## 目标

描述该页面、流程或组件解决什么用户问题。

## 入口

- 路由:
- 页面入口:
- 组件入口:
- 用户操作入口:

## 当前行为

描述前端代码当前实际行为。必须区分事实来源。

### 来源

- 代码位置:
- 页面验证:
- 接口响应样本:
- 浏览器行为:
- 历史文档:
- 产品确认:

## 相关页面与组件

- 页面:
- 组件:
- hooks/composables:
- 状态模块:
- 工具函数:

## 交互规则

### 已确认规则

- ...

### 从代码推断的规则

- ...

## 状态规则

- 初始状态:
- loading 状态:
- empty 状态:
- error 状态:
- success 状态:
- disabled 状态:
- selected/expanded/active 状态:

## 接口契约

- 请求触发时机:
- 请求参数来源:
- 成功响应处理:
- 失败响应处理:
- 重试规则:
- 取消请求规则:

## URL、缓存和本地状态

- URL query/path 参数:
- localStorage/sessionStorage:
- 内存缓存:
- 状态管理 store:
- 页面刷新后的恢复行为:

## 权限与可见性

- 可见条件:
- 禁用条件:
- 隐藏条件:
- 无权限提示:

## 表单与校验

- 字段:
- 默认值:
- 必填规则:
- 格式校验:
- 提交前校验:
- 提交后反馈:

## 示例场景

### 场景: [场景名称]

- GIVEN ...
- WHEN ...
- THEN ...
- AND ...

## 异常情况

- 条件:
- UI 行为:
- 提示文案:
- 状态变化:
- 是否允许重试:

## 副作用

- 路由跳转:
- 弹窗:
- toast/message:
- URL 更新:
- 缓存更新:
- 状态管理更新:
- 埋点:

## 相关代码

- ...

## 不确定点

- ...

## OpenSpec Mapping

- Capability:
- Baseline:
```

## OpenSpec Baseline Spec 模板

路径示例：

```text
openspec/specs/user-list-filtering/spec.md
```

模板：

```md
# [Capability Name] Specification

## Requirements

### Requirement: [能力规则名称]
The system SHALL [用一句话描述已确认的当前前端能力承诺].

#### Scenario: [场景名称]
- GIVEN ...
- WHEN ...
- THEN ...
- AND ...
```

示例：

```md
# User List Filtering Specification

## Requirements

### Requirement: Apply keyword filter
The system SHALL filter the user list by keyword when the user submits the search form.

#### Scenario: User searches by keyword
- GIVEN the user list page is open
- AND the keyword input contains "alice"
- WHEN the user submits the search form
- THEN the list request includes keyword "alice"
- AND the table displays the returned results

### Requirement: Preserve filters in URL query
The system SHALL keep active filters in the URL query after search.

#### Scenario: User applies status filter
- GIVEN the user list page is open
- WHEN the user selects status "enabled"
- AND submits the search form
- THEN the URL query includes status "enabled"
- AND refreshing the page restores the selected status
```

## 补充 Spec 的执行流程

### 阶段 1: 选择 Spec 对象

不要让模型凭空判断“经常出 bug”。如果用户没有明确指定页面或流程，模型只能根据本地项目中可观察到的前端复杂度选择优先级。

优先选择：

- 表单、筛选、列表、详情、编辑、审批、权限展示等高频页面。
- 状态分支多的组件。
- URL query、缓存或状态管理复杂的页面。
- 接口调用链路复杂的页面。
- 路由入口多、组件层级深、条件渲染多的页面。
- 代码路径分散、命名混乱、存在大量条件判断的模块。
- 存在 TODO、FIXME、临时兼容逻辑或明显历史分支的模块。
- 用户明确指定的页面、流程或组件。

### 阶段 2: 阅读前端代码和页面行为

需要重点观察：

- 路由入口
- 页面组件
- 子组件
- hooks/composables
- 状态管理模块
- 接口封装
- 权限判断
- 表单校验
- URL query 同步
- localStorage/sessionStorage 使用
- loading、empty、error、success 状态
- 弹窗、toast、跳转、埋点等副作用

### 阶段 3: 向用户询问业务问题

AI 在形成初步理解后，必须向用户提出关键业务问题。

要求：

- 先问影响 spec 正确性的核心问题。
- 每次提问数量要少，优先问 1 到 3 个。
- 问题要带具体上下文。
- 用户未确认前，不得把推断写进 OpenSpec baseline。

### 阶段 4: 编写通用前端业务 Spec

先写通用前端业务 Spec，记录当前行为、事实来源、代码推断和不确定点。

要求：

- 已确认规则和代码推断规则分开写。
- 不确定点必须显式列出。
- 重要交互规则必须写示例场景。
- 相关代码位置必须列出。

### 阶段 5: 提炼 OpenSpec Baseline

从通用前端业务 Spec 中提取已确认规则，写入 `openspec/specs/<capability>/spec.md`。

要求：

- 只写已确认的当前前端能力。
- 使用 `## Requirements`。
- 每条规则使用 `### Requirement: ...`。
- 每条重要规则至少包含一个 `#### Scenario: ...`。
- 不写不确定点。
- 不写调研过程。
- 不使用 `ADDED Requirements`、`MODIFIED Requirements`、`REMOVED Requirements`。

## 内部模型执行要求

当内部模型被要求为纯前端老项目补 spec 时，应按以下规则执行：

1. 只做 spec 补充，不补测试，不改代码，不创建需求变更。
2. 先识别页面、用户流程、路由入口和关键组件，不要从单个代码文件开始写。
3. 形成初步理解后，必须向用户持续询问关键业务问题。
4. 先创建或更新通用前端业务 Spec。
5. 将事实来源标注清楚。
6. 明确区分已确认规则、代码推断规则和不确定点。
7. 对重要交互规则补充 Given / When / Then 场景。
8. 在创建或映射 OpenSpec baseline 前，必须先提出 capability 名称和边界候选，并等待用户确认。
9. 只将已确认规则提炼进 OpenSpec baseline。
10. 当前没有新需求或行为变更时，不创建 OpenSpec change。
11. 不生成 `proposal.md`、`tasks.md` 或 delta spec。
12. 不在 OpenSpec baseline 中写调研过程、猜测和待确认问题。
13. 在通用前端业务 Spec 中维护 OpenSpec Mapping。
14. 不要生成数据库、后台任务、消息队列、服务端状态流转等后端描述，除非它们以接口契约形式影响前端行为。

## 判断标准

一份合格的通用前端业务 Spec 应满足：

- 能让新成员理解该页面或流程现在怎么运行。
- 能指出哪些行为来自代码事实，哪些来自产品或业务确认。
- 能列出主要交互规则、状态规则、异常状态和副作用。
- 能暴露不确定点。
- 能映射到 OpenSpec capability。

一份合格的 OpenSpec baseline 应满足：

- 只包含已确认的当前前端能力承诺。
- 使用 Requirement 和 Scenario 表达行为。
- 不包含分析过程和不确定点。
- 不要求创建 `openspec/changes/`。
- 不使用 delta sections。

## 推荐起步动作

从用户明确指定的对象开始；如果用户没有指定，则从本地代码中可观察复杂度较高的前端页面/用户流程开始，例如列表筛选、详情页操作、编辑表单、权限按钮展示、审批页面、导入导出页面或搜索结果页。模型不能凭空断言某页面经常出 bug，只能说明选择依据是代码复杂度、页面复杂度或用户流程重要性。

执行顺序：

```text
选择页面或用户流程
  -> 阅读路由、页面、组件、hooks、状态和接口代码
    -> 写通用前端业务 Spec
      -> 标注事实来源和不确定点
        -> 提炼 OpenSpec baseline
```