# 插件形态与生态地图

## 三种形态

| 形态 | 载体 | 安装方式 | 适用场景 |
|---|---|---|---|
| **bundle**（官方机制，固定使用） | cordis 插件包 + `cordis.patch.yml`（package.json 声明 `dsh.bundle.patch`） | `dsh plugin --profile <name> add <路径或包名>`，自动 reconcile 进 `dsh.profile.bundles` | 工具、服务、web UI 贡献——一切需要运行时逻辑的插件 |
| **registry**（plugin-registry 机制） | `dsh.plugin.json` 清单 + cordis 入口 | `dsh registry install`（需装 plugin-registry） | 进「设置页插件面板」的插件 |
| **skill** | `SKILL.md`（frontmatter + 正文 + references/） | 放进 skills 目录 | 纯提示词/流程型能力（本档案的形态） |

**默认选 bundle**：官方 profile/bundle 组合机制（0806+），`dsh plugin` 命令原生支持，hub 自动识别。

## dsh-external 生态地图（截至 2026-08-08，133 仓库）

- **工具类**：零依赖确定性工具——time/encoding/json/calculator/csv/regex/markdown，打包在 `dsh-toolkit` collection
- **能力/工作流类**：oh-my-dsh（24+ gap 插件）、official-plugins-port（23 个官方插件移植）、dsh-deep-research、dsh-inspect
- **Web UI 类**：dsh-ui-whale、dsh-web-terminal、dsh-better-sidebar、dsh-skills-manager、dsh-web-archive 等
- **会话互操作类**：dsh-session-search、dsh-session-hub、dsh-session-cluster、dsh-session-repair-skill
- **模型路由类**：dsh-plan-execute、dsh-llm-fallbacks、dsh-advisor
- **生态基建类**：marisa（管理器）、plugin-registry、hub（索引）、dsh-plus（manifest+installer）、dsh-plugin-check（检查）
- **远程渠道类**：telegram/qq/feishu/wecom/weixin + dsh-remote（SSH）
- **桌面/设备类**：deepseek-harness-desktop、dsh-android、dsh-desktop-mac、dsh-island

## 选型判断（基于 8 个插件的实践）

1. 模型**高频需要、确定性、可验证**的能力 → 工具插件（零依赖纯函数，dsh-tool-csv 是模板）
2. 需要**读文件/跑进程/查系统** → 工具插件 + `types: ["node"]`
3. 需要**注入 UI/客户端** → bundle 的 client half（web-ui 类）
4. 纯**流程/提示词** → skill（deep-standard-skill 是范例）

→ 下一步：[tool-plugin.md](tool-plugin.md)
