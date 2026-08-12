# 插件形态与生态地图

## 三种形态

| 形态 | 载体 | 安装方式 | 适用场景 |
|---|---|---|---|
| **bundle**（官方机制，固定使用） | cordis 插件包 + `cordis.patch.yml`（package.json 声明 `dsh.bundle.patch`） | `dsh plugin --profile <name> add <路径或包名>`，自动 reconcile 进 `dsh.profile.bundles` | 工具、服务、web UI 贡献——一切需要运行时逻辑的插件 |
| **registry**（plugin-registry 机制） | `dsh.plugin.json` 清单 + cordis 入口 | `dsh registry install`（需装 plugin-registry） | 进「设置页插件面板」的插件 |
| **skill** | `SKILL.md`（frontmatter + 正文 + references/） | 放进 skills 目录 | 纯提示词/流程型能力（本档案的形态） |

**默认选 bundle**：官方 profile/bundle 组合机制（0806+），`dsh plugin` 命令原生支持，hub 自动识别。

## dsh-external 生态地图（截至 2026-08-12）

- **工具类**：零依赖确定性工具——time/encoding/json/calculator/csv/regex/markdown/diff/stat/schema，打包在 `dsh-toolkit` collection
- **能力/工作流类**：oh-my-dsh（24+ gap 插件）、official-plugins-port（23 个官方插件移植）、dsh-deep-research、dsh-inspect
- **Web UI 类**：dsh-ui-whale、dsh-web-terminal、dsh-better-sidebar、dsh-skills-manager、dsh-web-archive 等
- **会话互操作类**：dsh-session-search、dsh-session-hub、dsh-session-cluster、dsh-session-repair-skill
- **模型路由类**：dsh-plan-execute、dsh-llm-fallbacks、dsh-advisor
- **生态基建类**：marisa（管理器）、plugin-registry、hub（索引）、dsh-plus（manifest+installer）、dsh-plugin-check（检查）
- **远程渠道类**：telegram/qq/feishu/wecom/weixin + dsh-remote（SSH）
- **桌面/设备类**：deepseek-harness-desktop、dsh-android、dsh-desktop-mac、dsh-island

## 选型决策矩阵（审查 PD-02/X-01 修复：bundle 不是唯一运行时形态）

| 需求 | 形态 | 识别标志 | 说明 |
|---|---|---|---|
| 官方 profile 安装、随 profile 启动、`dsh plugin add` | **bundle / tool-bundle** | `package.json`（+ `dsh.bundle.patch` → `cordis.patch.yml`） | 工具、服务、web UI 运行时逻辑；工具插件（import `@deepseek-ai/dsh-tools`）即 tool-bundle |
| registry 面板生命周期（enable/disable）、catalog 安装 | **registry 原生插件** | `dsh.plugin.json`（可只有 `index.mjs`，无 package.json） | 官方 plugin-registry 示例 greeter/loop/navbar/task-status 即此形态——**不要**默认生成 bundle |
| 纯提示词/流程 | **skill** | `SKILL.md`（frontmatter） | 本档案即 skill 形态 |
| 多个同风格插件打包 | **collection** | `catalog.json`（collection/plugins 字段） | dsh-toolkit 形态 |
| 多包/基础设施仓库（无 bundle 入口） | **infra** | `package.json` 无 `main` | dsh-my-rsi 形态；**不应套插件规则 fail** |
| 同一项目可做官方 bundle + registry 增量兼容 | 双形态 | 两者文件并存 | 安装通道互斥，文档注明 |

> **共享规则矩阵**（X-01）：`plugin_check` 的 `schema` action 输出与本文一致的
> "检测项 × 形态适用矩阵"——插件开发以它为合规事实源；新增形态先更新矩阵再实现。

## 选型判断（基于 9 个插件的实践）

1. 模型**高频需要、确定性、可验证**的能力 → 工具插件（零依赖纯函数，dsh-tool-csv 是模板）
2. 需要**读文件/跑进程/查系统** → 工具插件 + `types: ["node"]`
3. 需要**registry 面板生命周期** → registry 原生形态（`dsh.plugin.json`）
4. 需要**注入 UI/客户端** → bundle 的 client half（web-ui 类）
5. 纯**流程/提示词** → skill（deep-standard-skill 是范例）

→ 下一步：[tool-plugin.md](tool-plugin.md)
