# Vitest API 与 Mock 参考

## Vitest vs Jest API 对照

| 应该用（Vitest） | 不要用（Jest） |
|---|---|
| `vi.fn()` | `jest.fn()` |
| `vi.mock('module')` | `jest.mock('module')` |
| `vi.spyOn(obj, 'method')` | `jest.spyOn(obj, 'method')` |
| `vi.useFakeTimers()` | `jest.useFakeTimers()` |
| `vi.stubGlobal('fetch', fn)` | 手动全局赋值 |
| `vi.clearAllMocks()` | `jest.clearAllMocks()` |
| `vi.restoreAllMocks()` | `jest.restoreAllMocks()` |

## 模块 Mock

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

## 异步测试

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

## 快照与内联快照

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

## 工具函数测试示例

纯函数直接调用验证返回值：

```javascript
import { describe, test, expect } from 'vitest'
import { formatPrice } from '@/utils/helpers'

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

有外部依赖时，只 mock 那个外部依赖：

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
