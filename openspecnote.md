# OpenSpec 指令速查

> OpenSpec 是一个轻量级的 Spec-Driven Development (SDD) 框架，适用于 AI 编程助手，无需 API key 或 MCP。
> 官方仓库：https://github.com/Fission-AI/OpenSpec

---

## 一、Slash 指令（共 11 个）

| 指令 | 说明 |
|---|---|
| `/opsx:explore` | 调查想法、思考问题、厘清需求（只读模式，不写代码） |
| `/opsx:propose <idea>` | 创建变更提案，自动生成 specs、design、tasks |
| `/opsx:new` | 创建新的变更脚手架（目录和元数据），等待产物生成 |
| `/opsx:continue` | 按依赖链逐个创建下一个产物，增量推进 |
| `/opsx:ff` | 快进，一次性生成所有规划产物 |
| `/opsx:apply` | 执行任务，编写代码并更新产物 |
| `/opsx:verify` | 验证实现是否匹配产物（完整性、正确性、一致性） |
| `/opsx:sync` | 将变更中的 delta specs 同步到主 specs 目录（可选，archive 会自动处理） |
| `/opsx:archive` | 将已完成的变更归档（自动检测未同步的 delta spec 并提示合并） |
| `/opsx:bulk-archive` | 批量归档多个已完成的变更，检测并解决 spec 冲突 |
| `/opsx:onboard` | 引导式教程，使用实际代码库演示完整 OpenSpec 工作流 |

---

## 二、CLI 命令（`openspec ...`）

### 初始化与更新

| 命令 | 说明 |
|---|---|
| `openspec init [path] [options]` | 初始化项目的 OpenSpec，创建目录结构并配置 AI 工具集成 |
| `openspec update [path] [options]` | 升级 CLI 后更新指令文件 |

### 工作区管理

| 命令 | 说明 |
|---|---|
| `openspec workspace setup` | 创建工作区并关联项目 |
| `openspec workspace list` | 列出已知的 OpenSpec 工作区 |
| `openspec workspace link [name] <path>` | 将已有仓库/目录关联到工作区 |
| `openspec workspace relink <name> <path>` | 修复或变更已有链接的本地路径 |
| `openspec workspace doctor` | 检查工作区在当前机器上的可用性 |
| `openspec workspace open [name]` | 通过存储的默认打开方式打开工作区 |

### 浏览与查看

| 命令 | 说明 |
|---|---|
| `openspec list` | 列出项目中的变更或 specs |
| `openspec view` | 交互式仪表盘，浏览 specs 和变更 |
| `openspec show [item-name]` | 显示变更或 spec 的详细信息 |

### 验证

| 命令 | 说明 |
|---|---|
| `openspec validate [item-name]` | 验证变更和 specs 的结构问题 |

### 生命周期

| 命令 | 说明 |
|---|---|
| `openspec archive [change-name]` | 归档已完成的变更，合并 delta specs |
| `openspec status` | 显示变更的产物完成状态 |

### 工作流信息

| 命令 | 说明 |
|---|---|
| `openspec instructions [artifact]` | 获取创建产物或执行任务的增强指令 |
| `openspec templates` | 显示 schema 中所有产物的已解析模板路径 |
| `openspec schemas` | 列出可用的工作流 schema 及描述 |

### Schema 管理

| 命令 | 说明 |
|---|---|
| `openspec schema init <name>` | 创建新的项目级 schema |
| `openspec schema fork <source> [name]` | 复制已有 schema 到项目中进行定制 |
| `openspec schema validate [name]` | 验证 schema 的结构和模板 |
| `openspec schema which [name]` | 显示 schema 的解析来源 |

### 配置管理

| 命令 | 说明 |
|---|---|
| `openspec config path` | 显示配置文件路径 |
| `openspec config list` | 列出所有配置值 |
| `openspec config get` | 获取指定配置值 |
| `openspec config set` | 设置配置值 |
| `openspec config unset` | 移除配置值 |
| `openspec config reset` | 重置为默认配置 |
| `openspec config edit` | 在编辑器中打开配置 |
| `openspec config profile` | 选择工作流 profile |

### 工具命令

| 命令 | 说明 |
|---|---|
| `openspec feedback <message>` | 提交反馈（创建 GitHub issue） |
| `openspec completion <shell>` | 管理 shell 自动补全 |

---

## 三、每个变更生成的产物

| 产物 | 用途 |
|---|---|
| `proposal.md` | 描述为什么要变更、变更什么 |
| `specs/` 目录 | 需求和场景 |
| `design.md` | 技术方案文档 |
| `tasks.md` | 实现清单 |

---

## 四、支持的 AI 编程工具（29 个）

Claude Code, Cursor, Windsurf, GitHub Copilot, Cline, RooCode, Gemini CLI, Codex, Amazon Q Developer, Continue, Kilo Code, Trae, Junie, Kiro, Qwen Code, Kimi CLI, Auggie, ForgeCode, CoStrict, Factory Droid, iFlow, Lingma, OpenCode, Pi, Qoder, Antigravity, Bob Shell, CodeBuddy, Crush

---

## 五、Profile 与指令可用性

默认的 core profile 只包含 5 个指令：`explore`、`propose`、`apply`、`sync`、`archive`。

其余指令（`new`、`continue`、`ff`、`verify`、`bulk-archive`、`onboard`）属于 expanded profile，需要切换：

```bash
openspec config profile   # 选择 expanded
openspec update           # 更新指令文件
```

### `/opsx:new` vs `/opsx:propose`

| | `/opsx:new` | `/opsx:propose` |
|---|---|---|
| 创建目录 | 是 | 是 |
| 自动生成产物 | 不会，需要后续 `continue` 或 `ff` | 一步到位全部生成 |
| 适用场景 | 想逐步控制每个产物的生成 | 快速开始 |

`/opsx:new` 用法：

```
/opsx:new <change-name>
```

只创建 `openspec/changes/<change-name>/` 目录脚手架，不生成任何产物文件。后续流程：

```
/opsx:new add-logout-button
→ Created change. Ready to create: proposal

/opsx:continue              # 逐个生成产物
→ 或
/opsx:ff                    # 一次全部生成

/opsx:apply                 # 开始写代码
```

---

## 六、全局选项

| 选项 | 说明 |
|---|---|
| `--version`, `-V` | 显示版本号 |
| `--no-color` | 禁用彩色输出 |
| `--help`, `-h` | 显示帮助 |
