# Vitest API 与 Mock 参考

## Mock 约定

mock 封装层，不要 mock 底层。只 mock 有副作用或 jsdom 未实现的依赖，其余走真实逻辑。

| 需要 mock 的对象 | mock 方式 |
|---|---|
| HTTP 请求 | mock 项目封装的 request 模块，不要 mock axios 本身 |
| localStorage / sessionStorage | `vi.stubGlobal('localStorage', { getItem: vi.fn(), setItem: vi.fn(), removeItem: vi.fn() })` |
| window.location | `vi.stubGlobal('location', { href: '', assign: vi.fn(), replace: vi.fn() })` |
| navigator.clipboard | `vi.stubGlobal('navigator', { clipboard: { writeText: vi.fn() } })` |
| URL.createObjectURL | `vi.stubGlobal('URL', { createObjectURL: vi.fn(() => 'blob:fake-url'), revokeObjectURL: vi.fn() })` |
| ResizeObserver | `vi.stubGlobal('ResizeObserver', vi.fn(() => ({ observe: vi.fn(), disconnect: vi.fn() })))` |
| Date / 定时器 | `vi.useFakeTimers()` + `vi.setSystemTime()`，测完调 `vi.useRealTimers()` |

## 模块 Mock 示例

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

## 异步测试示例

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

## 工具函数测试示例

纯函数直接调用验证返回值，不需要 mock：

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

## 快照与内联快照

谨慎使用——快照写起来容易，维护起来难。优先使用明确的断言。适用于复杂但稳定的序列化输出和回归保护。

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
