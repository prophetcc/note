# openspec 目录位置

## 问题

openspec 的目录只能是项目的根目录吗？

## Issue #697

**标题**：Feedback: Support custom openspec/ directory path
**作者**：@eimaj
**日期**：2026-02-11
**指派**：@TabishB
**状态**：Open，标签 enhancement
**评论**：2 条（均为 `1code-async[bot]` 自动分析，无 maintainer 人工回应）
**链接**：https://github.com/Fission-AI/OpenSpec/issues/697

### Issue body 原文

> "Allow config.yaml or env var to relocate the openspec/ folder to a custom path like ./ai/openspec/. In larger monorepos, keeping AI/spec tooling under a dedicated directory (e.g., ./ai/) helps reduce root-level clutter and keeps related concerns grouped together."

### 请求要点

| | 内容 |
|---|------|
| 想要什么 | 通过 `config.yaml` 或环境变量配置 `openspec/` 目录的位置 |
| 示例路径 | `./ai/openspec/`（放在 `./ai/` 父目录下） |
| 动机 | 大 monorepo 减少根目录杂物，把 AI/spec 工具聚拢在专门目录 |
| 不想要什么 | 改目录名（仍叫 `openspec`），只改父路径 |

### 自动机器人补的技术分析

- **5 个相关 issue**：#664、#654、#581、#449、#350——#664 最综合，可能作为汇总点
- **9+ 处硬编码**：根源是 `src/core/config.ts:1` 的 `OPENSPEC_DIR_NAME` 常量
- **现成参考模式**：XDG 支持、schema 解析分层、Zod 配置校验
- **实现建议**：加 config 字段、路径解析集中化、解决 bootstrap chicken-and-egg 问题、env var 备选

### Maintainer 回应

无。TabishB 已认领但未正式回复。

## Issue #664

**标题**：[Proposal]: Configurable specs path (`specsPath` in config.yaml)
**状态**：Open
**链接**：https://github.com/Fission-AI/OpenSpec/issues/664

### 提案要点

| | 内容 |
|---|------|
| 想要什么 | 在 `openspec/config.yaml` 加 `specsPath` 字段，自定义主 spec 目录位置 |
| 默认值 | `openspec/specs`（向后兼容） |
| 示例值 | `docs/specs`、`contracts/` |
| 作用范围 | **只挪 `specs/` 子目录**——`openspec/` 其他内容（changes/、archive/、schemas/）保持不动 |
| 理念 | "Specs are project contracts, not OpenSpec artifacts"——specs 是项目契约不属于 OpenSpec 工具产物 |

### 关键机制

- **`{{specsPath}}` 占位符**：schema 模板里写这个字符串，runtime 由 instruction loader 替换成配置值
- **跨平台路径规范化**：支持 `/` 和 `\` 分隔符
- **三种路径表示**：绝对路径（文件 I/O）、相对 POSIX（prompts/AI 指令）、相对原生（console 输出）
- **向后兼容**：老 schema 硬编码 `openspec/specs` 仍可跑，但不响应 `specsPath` 配置；自动替换 + 警告作者迁移

### Schema 来源对此功能的影响

| Schema 类型 | 位置 | 处理方式 |
|------------|------|---------|
| Built-in（如 spec-driven） | OpenSpec 自带 | 已更新使用 `{{specsPath}}` |
| Project-local | `openspec/schemas/` | 作者可以用 `{{specsPath}}` |
| User-level | `~/.local/share/openspec/schemas/` | 作者可以用 `{{specsPath}}` |
| 老的自定义 schema | 用硬编码 `openspec/specs` | 仍能跑，但不响应配置——需迁移到占位符 |

### 实操

```yaml
# openspec/config.yaml
specsPath: docs/specs
```

```bash
openspec update    # 重新生成 skill 文件，替换占位符为新路径
```

### 和 #697 的关系

| Issue | 配置什么 | 范围 |
|-------|---------|------|
| #664 | 主 spec 目录位置 | **只是 specs/ 子目录** |
| #697 | 整个 openspec/ 目录位置 | 整个 openspec/ 目录 |

#664 解决一半「目录可配置」问题；#697 是更激进的整目录可配置，仍未实现。
