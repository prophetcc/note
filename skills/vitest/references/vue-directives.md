# Vue 自定义指令处理参考

## 全局 setup 方式（推荐用于全项目通用指令）

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

## 单测文件方式（用于个别组件特有的指令）

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

## 空 mock 方式（不关心指令行为时，推荐）

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

## 第三方 UI 库指令（如 Element Plus）

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

## Vue 组件测试结构模板

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
