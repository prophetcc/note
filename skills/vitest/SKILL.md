---
name: vitest
description: "编写高质量的 Vitest 单元测试。当用户要求编写测试、单元测试或测试用例，且项目使用 Vitest 时触发此 skill。当用户提到 'vitest'、'vi.fn'、'vi.mock'、'vi.spyOn'，或希望为基于 Vite 的项目增加测试覆盖率时也会触发。即使用户只是说'给这个文件写测试'或'增加测试覆盖率'，也应检查项目是否使用 Vitest（查看 package.json 或 vitest.config 文件），如果是则使用此 skill。"
---

# 编写 Vitest 测试

本 skill 指导你编写高质量的 Vitest 单元测试。方法基于 Vitest 官方指南：生成测试的质量很大程度上取决于你事先收集的上下文以及测试目标的具体程度。

## 编写测试前：收集上下文

上下文是影响测试质量的最关键因素。跳过这一步会产生模糊、脆弱的测试。在写第一个 `test()` 块之前，完成以下所有步骤：

### 1. 阅读源文件
完整阅读实现文件——包括 imports、辅助函数等所有内容。你需要理解代码实际做了什么，而不是仅凭函数名猜测。

### 2. 查找已有测试
搜索项目中已有的测试文件（`.test.js`、`.spec.js`、`__tests__/` 目录），从中了解：
- 命名规范（`.test.js` 还是 `.spec.js`，就近放置还是独立的 `__tests__/` 目录）
- 测试风格（是否使用 `describe` 块，setup/teardown 如何处理）
- 导入模式和通用工具函数

遵循你找到的现有规范。一致性比个人偏好更重要。

### 3. 阅读 Vitest 配置
查找 `vitest.config.js` 或 `vite.config.js` 中的 `test` 部分，重点关注：
- `globals: true` —— 如果设置了，则不需要导入 `describe`、`test`、`expect`
- `environment` —— `'jsdom'`、`'node'`、`'happy-dom'` 会影响可用的 API
- `setupFiles` —— 测试前运行的共享 setup 文件（可能设置了 mock、全局状态、全局指令等）
- `coverage` 设置、`include`/`exclude` 模式

### 4. 阅读需要 mock 的依赖
如果被测函数调用了其他模块，阅读它们的实现，以便了解 mock 应该是什么结构。

### 5. 检查全局自定义指令（Vue 项目）
如果是 Vue 组件测试，必须检查组件模板中使用的自定义指令（`v-xxx`）：

1. **扫描组件模板**，找出所有 `v-` 前缀的自定义指令（排除 Vue 内置指令如 `v-if`、`v-for`、`v-model`、`v-show`、`v-bind`、`v-on`、`v-slot`、`v-html`、`v-text`、`v-pre`、`v-cloak`、`v-once`、`v-memo`）
2. **读取全局指令注册文件**（通常在 `src/directives/index.js` 或 `src/plugins/` 下），确认项目有哪些自定义指令
3. **读取 `vitest.setup.js`（或 setupFiles 指定的文件）**，确认哪些指令已经全局 mock 过
4. **处理策略**：
   - 如果指令已在 setup 文件中 mock —— 无需额外处理
   - 如果指令未被 mock —— 在测试中通过 `global.directives` 注册 mock 或真实指令
   - 如果指令来自第三方 UI 库（如 Element Plus）—— 项目整包安装的情况下，通过 `global.plugins` 安装即可自动注册其指令

不处理自定义指令会导致 Vue 渲染警告或测试失败。

## 编写测试

### 有意义的断言

最常见的错误是写出不验证任何有用内容的测试。一个只检查 `toBeDefined()` 或 `not.toThrow()` 的测试几乎提供不了任何信心。

```javascript
// 即使 createUser 返回垃圾数据，这个测试也会通过
test('创建用户', () => {
  const user = createUser('Alice', 'alice@example.com')
  expect(user).toBeDefined() // 什么也验证不了
})

// 这个测试真正验证了函数的契约
test('创建的用户包含正确字段', () => {
  const user = createUser('Alice', 'alice@example.com')
  expect(user).toMatchObject({
    name: 'Alice',
    email: 'alice@example.com',
  })
  expect(user.id).toBeTypeOf('string')
})
```

对每个测试问自己："如果实现有细微的错误，这个测试能捕获到吗？"如果答案是否，就加强断言。

### 测试行为，而非实现

测试应该从外部验证代码做了什么，而不是内部如何做的。如果有人重构了内部实现但没改变行为，测试应该仍然通过。

过度测试实现的迹象：
- 断言内部辅助函数被调用的确切次数
- 把所有依赖都 mock 掉，而不是只 mock 有副作用的依赖
- 直接测试私有方法
- 重命名内部变量就会导致断言失败

优先使用集成式测试来验证真实的代码路径。只 mock 那些在测试中确实不可行的东西（网络请求、数据库、定时器）。

### 边界用例

每个函数都要明确覆盖这些情况：
- **Null / undefined 输入** —— 是抛出异常、返回默认值，还是优雅处理？
- **空输入** —— 空字符串、空数组、空对象
- **校验失败** —— 无效邮箱格式、缺少必填字段、超出范围的数字
- **边界值** —— 零、负数、最大值
- **错误路径** —— 依赖抛出异常时会发生什么？
- **异步边界用例** —— 被拒绝的 promise、超时、并发调用

### 测试结构

```javascript
import { describe, test, expect, beforeEach, afterEach, vi } from 'vitest'

describe('UserService', () => {
  beforeEach(() => {
    // 共享 setup
  })

  afterEach(() => {
    vi.restoreAllMocks()
  })

  describe('createUser', () => {
    test('合法输入时创建用户', () => { /* ... */ })
    test('缺少姓名时抛出异常', () => { /* ... */ })
    test('邮箱格式无效时抛出异常', () => { /* ... */ })
    test('存储前对密码进行哈希', () => { /* ... */ })
  })

  describe('deleteUser', () => {
    test('根据 id 删除用户', () => { /* ... */ })
    test('用户不存在时抛出异常', () => { /* ... */ })
  })
})
```

- 使用 `describe` 对相关测试进行分组
- 使用 `beforeEach` / `afterEach` 进行 setup 和 teardown
- 测试名称保持简洁，聚焦行为："格式化美元价格" 比 "当输入一个数字时应该正确地将价格值格式化为美元" 更好

### 工具函数 / 模块测试结构

纯函数是最容易测试的代码——没有 DOM、没有组件生命周期，直接调用验证返回值。

```javascript
import { describe, test, expect } from 'vitest'
import { formatPrice, parseQuery, debounce } from '@/utils/helpers'

describe('formatPrice', () => {
  test('正常数值格式化', () => {
    expect(formatPrice(1234.5)).toBe('¥1,234.50')
  })

  test('零值', () => {
    expect(formatPrice(0)).toBe('¥0.00')
  })

  test('负数', () => {
    expect(formatPrice(-99.9)).toBe('-¥99.90')
  })

  test('无效输入返回默认值', () => {
    expect(formatPrice(null)).toBe('¥0.00')
    expect(formatPrice(undefined)).toBe('¥0.00')
    expect(formatPrice(NaN)).toBe('¥0.00')
  })
})
```

如果工具函数有外部依赖（如调用 API、读写 localStorage），只 mock 那个外部依赖，其余走真实逻辑：

```javascript
import { describe, test, expect, vi, beforeEach } from 'vitest'
import { getUserToken, saveToken } from '@/utils/auth'

describe('auth 工具函数', () => {
  beforeEach(() => {
    vi.stubGlobal('localStorage', {
      getItem: vi.fn(),
      setItem: vi.fn(),
      removeItem: vi.fn(),
    })
  })

  test('getUserToken 从 localStorage 读取', () => {
    localStorage.getItem.mockReturnValue('abc123')
    expect(getUserToken()).toBe('abc123')
  })

  test('saveToken 写入 localStorage', () => {
    saveToken('xyz')
    expect(localStorage.setItem).toHaveBeenCalledWith('token', 'xyz')
  })
})
```

工具函数测试的重点：
- **覆盖边界值** —— null、undefined、空字符串、空数组、极大/极小数字
- **不需要 mock 的就别 mock** —— 纯函数直接调用，断言返回值
- **异步工具函数**（如 debounce、throttle）—— 配合 `vi.useFakeTimers()` 控制时间

### Vue 组件测试结构

```javascript
import { describe, test, expect, afterEach, vi } from 'vitest'
import { mount } from '@vue/test-utils'
import MyComponent from '../MyComponent.vue'

describe('MyComponent', () => {
  const createWrapper = (props = {}, options = {}) => {
    return mount(MyComponent, {
      props,
      global: {
        directives: {
          // 注册 setup 文件中未覆盖的自定义指令
          'custom-directive': { mounted() {}, updated() {} },
        },
        plugins: [],
        stubs: {},
        ...options,
      },
    })
  }

  afterEach(() => {
    vi.restoreAllMocks()
  })

  test('正确渲染', () => {
    const wrapper = createWrapper({ title: '测试' })
    expect(wrapper.text()).toContain('测试')
  })
})
```

## Vitest API —— 用对 API

这是 Vitest，不是 Jest。API 看起来相似但有区别。

| 应该用（Vitest） | 不要用（Jest） |
|---|---|
| `vi.fn()` | `jest.fn()` |
| `vi.mock('module')` | `jest.mock('module')` |
| `vi.spyOn(obj, 'method')` | `jest.spyOn(obj, 'method')` |
| `vi.useFakeTimers()` | `jest.useFakeTimers()` |
| `vi.stubGlobal('fetch', fn)` | 手动全局赋值 |
| `vi.clearAllMocks()` | `jest.clearAllMocks()` |
| `vi.restoreAllMocks()` | `jest.restoreAllMocks()` |

### Mock 清理

Mock 在测试间泄漏会导致莫名其妙的失败。两种方式二选一：
- 在 vitest 配置中设置 `restoreMocks: true`（推荐——全局生效），或
- 在 `afterEach` 中调用 `vi.restoreAllMocks()`

### 模块 Mock

```javascript
// Mock 整个模块
vi.mock('./database', () => ({
  query: vi.fn(),
  connect: vi.fn(),
}))

// 带工厂函数的 mock
vi.mock('./config', () => {
  return {
    getConfig: vi.fn(() => ({ port: 3000 })),
  }
})

// 部分 mock —— 保留真实实现，只覆盖特定导出
vi.mock('./utils', async (importOriginal) => {
  const actual = await importOriginal()
  return {
    ...actual,
    formatDate: vi.fn(() => '2024-01-01'),
  }
})
```

### 异步测试

```javascript
test('获取用户数据', async () => {
  const user = await fetchUser(1)
  expect(user.name).toBe('Alice')
})

test('网络错误时拒绝 promise', async () => {
  vi.stubGlobal('fetch', vi.fn(() => Promise.reject(new Error('Network error'))))
  await expect(fetchUser(1)).rejects.toThrow('Network error')
})
```

### 快照与内联快照

谨慎使用——快照写起来容易，维护起来难。优先使用明确的断言。快照适用于：
- 复杂但稳定的序列化输出（AST、渲染后的 HTML）
- 对特定输出的回归保护

```javascript
test('序列化配置', () => {
  expect(generateConfig()).toMatchInlineSnapshot(`
    {
      "port": 3000,
      "host": "localhost",
    }
  `)
})
```

## 自定义指令处理参考

### 全局 setup 方式（推荐用于全项目通用指令）

在 `vitest.setup.js` 中统一 mock：

```javascript
// vitest.setup.js
import { config } from '@vue/test-utils'

config.global.directives = {
  loading: { mounted() {}, updated() {} },
  permission: { mounted() {}, updated() {} },
  copy: { mounted() {}, updated() {} },
}
```

### 单测文件方式（用于个别组件特有的指令）

```javascript
import { vCustomDirective } from '@/directives/customDirective'

mount(MyComponent, {
  global: {
    directives: {
      'custom-directive': vCustomDirective,
    },
  },
})
```

### 空 mock 方式（不关心指令行为时，推荐）

```javascript
mount(MyComponent, {
  global: {
    directives: {
      loading: {},
      permission: {},
    },
  },
})
```

### 第三方 UI 库指令（如 Element Plus）

项目整包安装 UI 库时，直接通过 `global.plugins` 安装，指令会随之自动注册：

```javascript
import ElementPlus from 'element-plus'

mount(MyComponent, {
  global: {
    plugins: [ElementPlus],
  },
})
```

也可以在 `vitest.setup.js` 中全局配置，避免每个测试重复写：

```javascript
// vitest.setup.js
import { config } from '@vue/test-utils'
import ElementPlus from 'element-plus'

config.global.plugins = [ElementPlus]
```

## 编写测试后：运行测试

写完测试后务必立即运行。不运行的测试比没有测试更糟糕——它们给人虚假的信心。

```bash
# 运行指定测试文件
npx vitest run src/utils/userService.test.js

# 运行所有测试（非 watch 模式，适合 CI）
npx vitest run

# 运行并生成覆盖率报告
npx vitest run --coverage
```

使用 `vitest run`（而不是单独的 `vitest`）以避免交互式 watch 模式阻塞终端。

如果测试失败，先修复再继续。最常见的问题：
- 导入路径错误
- 缺少 mock 设置
- 异步测试缺少 `await`
- 基于过时假设来测试实现
- **Vue 组件测试中未注册自定义指令**

## 完成前检查清单

- [ ] 每个测试都有有意义的断言（不只是 `toBeDefined`）
- [ ] 覆盖了边界用例（null、空值、无效输入、错误路径）
- [ ] 使用的是 `vi.*` API，而非 `jest.*`
- [ ] Mock 已清理（restoreMocks 或 afterEach）
- [ ] 测试能实际运行并通过
- [ ] 测试文件遵循项目命名规范
- [ ] 没有过度 mock——尽可能测试真实代码路径
- [ ] Vue 组件测试中已正确处理自定义指令（全局 setup 或单测注册）
