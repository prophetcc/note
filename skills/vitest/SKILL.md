---
name: vitest
description: "编写 Vitest 单元测试。当用户要求写测试、单测、补测试、加覆盖率，或提到 vitest/vi.mock/vi.fn 等 API 时触发。用户只说'写测试'时，应检查项目是否使用 Vitest（查看 package.json 或 vitest.config），如果是则使用此 skill。不适用于 Cypress、Playwright 等 E2E 框架或 Mocha、pytest 等其他测试框架。"
---

# 编写 Vitest 测试

生成测试的质量取决于事先收集的上下文和测试目标的具体程度。

## 编写测试前：收集上下文

在写第一个 `test()` 块之前，完成以下步骤：

### 1. 阅读源文件
完整阅读实现文件——包括 imports、辅助函数等所有内容。理解代码实际做了什么，而不是凭函数名猜测。

### 2. 查找已有测试
搜索项目中已有的测试文件（`.test.js`、`.spec.js`、`__tests__/`），了解命名规范、测试风格和导入模式。遵循现有规范，一致性比个人偏好更重要。

### 3. 阅读 Vitest 配置
查找 `vitest.config.js` 或 `vite.config.js` 中的 `test` 部分，关注 `globals`、`environment`、`setupFiles`、`coverage` 设置。

### 4. 阅读需要 mock 的依赖
如果被测函数调用了其他模块，阅读它们的实现，了解 mock 应该是什么结构。

### 5. 检查全局自定义指令（Vue 组件测试时）
扫描组件模板中的自定义指令（排除 `v-if`、`v-for`、`v-model` 等 Vue 内置指令），然后：
1. 读取全局指令注册文件（`src/directives/index.js` 或 `src/plugins/`）
2. 读取 `vitest.setup.js`，确认哪些指令已全局 mock
3. 处理策略：已 mock 的跳过；未 mock 的通过 `global.directives` 注册；第三方 UI 库（如 Element Plus）整包安装的通过 `global.plugins` 安装

> 指令处理的详细代码示例见 `references/vue-directives.md`

## 编写测试

### 核心原则

- **有意义的断言** —— 每个测试问自己："如果实现有细微错误，这个测试能捕获吗？" `toBeDefined()` 几乎验证不了任何东西，要断言具体的返回值和结构
- **测试行为，而非实现** —— 重构内部实现不应导致测试失败。只 mock 有副作用的依赖（网络请求、数据库、定时器），其余走真实逻辑
- **覆盖边界用例** —— null/undefined、空值、校验失败、边界值（零/负数/最大值）、错误路径、异步边界（rejected promise、超时）

### 测试结构

```javascript
import { describe, test, expect, beforeEach, afterEach, vi } from 'vitest'

describe('UserService', () => {
  beforeEach(() => { /* 共享 setup */ })
  afterEach(() => { vi.restoreAllMocks() })

  describe('createUser', () => {
    test('合法输入时创建用户', () => { /* ... */ })
    test('缺少姓名时抛出异常', () => { /* ... */ })
  })
})
```

- 用 `describe` 分组，`beforeEach`/`afterEach` 做 setup 和 teardown
- 测试名称简洁聚焦行为："格式化美元价格" 而非 "当输入一个数字时应该正确地将价格值格式化为美元"

### 工具函数测试

纯函数直接调用验证返回值，不需要 mock。有外部依赖时只 mock 那个依赖，其余走真实逻辑。异步工具函数（debounce、throttle）配合 `vi.useFakeTimers()` 控制时间。

> 完整的工具函数和模块 mock 示例见 `references/api-and-mock.md`

### Vue 组件测试

封装 `createWrapper` 统一处理 `global.directives`、`global.plugins`、`global.stubs`。

> 组件测试结构模板和指令处理方式见 `references/vue-directives.md`

## Vitest API

所有 mock API 使用 `vi.*`，不要用 `jest.*`。Mock 清理：在配置中设 `restoreMocks: true`（推荐），或在 `afterEach` 中调用 `vi.restoreAllMocks()`。

> Mock 约定、模块 mock、异步测试、快照的详细示例见 `references/api-and-mock.md`

## 编写测试后：运行与验证

```bash
npx vitest run src/utils/userService.test.js  # 指定文件
npx vitest run                                 # 全量（非 watch）
npx vitest run --coverage                      # 覆盖率报告
```

使用 `vitest run` 而非 `vitest`，避免 watch 模式阻塞终端。

### 覆盖率要求

尽可能全面覆盖所有分支，包括：正常路径、异常路径、边界值（null/undefined/空值/极值）。写完后运行 `npx vitest run --coverage` 确认覆盖率。如有未覆盖的分支，说明原因（不可达代码、防御性代码等）。

### 测试失败处理

排查失败原因并修复测试或源码。禁止通过删除、跳过（`.skip`）或注释掉失败用例来回避问题。反复修改仍无法通过时，保留该用例并向用户报告：失败的测试名称、错误信息、已尝试的修复方式。

允许删除的情况：用例与其他用例验证了完全相同的逻辑分支（真正冗余），删除时说明它与哪条用例重复。

## 完成前检查清单

- [ ] 每个测试都有有意义的断言（不只是 `toBeDefined`）
- [ ] 覆盖了边界用例（null、空值、无效输入、错误路径）
- [ ] 使用的是 `vi.*` API，而非 `jest.*`
- [ ] Mock 已清理（restoreMocks 或 afterEach）
- [ ] 测试能实际运行并通过
- [ ] 测试文件遵循项目命名规范
- [ ] 没有过度 mock——尽可能测试真实代码路径
- [ ] Vue 组件测试中已正确处理自定义指令（全局 setup 或单测注册）
- [ ] 运行了 coverage 并确认覆盖率，未覆盖的分支已说明原因
