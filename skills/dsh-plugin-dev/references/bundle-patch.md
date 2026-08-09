# 挂载与验证（profile / bundle / dsh run）

## Profile 机制（0806+）

- 每个 profile（`web`/`headless`/自定义）是 `$DSH_HOME/profiles/<name>/` 下的一个目录：`package.json`（含 `dsh.profile.bundles` 层列表）+ `cordis.yml`（空根）+ `cordis.patch.yml`（用户层）；
- 启动时组合 = 按 `dsh.profile.bundles` 顺序叠加各 bundle 的 patch 层 → 用户层 → `--patch` 覆盖；
- 声明了 `dsh.bundle.patch` 的依赖包会被 `dsh plugin` 自动 reconcile 进 bundles 列表。

## 挂载

```sh
# 路径用 Windows 正斜杠形式（C:/Users/...）
dsh plugin --profile web add "C:/Users/admin/Desktop/dshext/dsh-tool-xxx"
dsh plugin --profile headless add "C:/Users/admin/Desktop/dshext/dsh-tool-xxx"   # 一次性任务也要
```

挂载验证（组合配置中出现工具行）：

```sh
dsh --profile web --dump-config | grep tool-xxx
# → # == @deepseek-ai/dsh-tool-xxx
#   - id: tool-xxx
#     name: '@deepseek-ai/dsh-tool-xxx'
```

## 一次性任务（0808 起用 `dsh run`）

```sh
dsh run "用 xxx 工具做一次端到端任务"        # 输出结果，exit 0 = 成功
```

- `dsh --profile headless "task"` 是 0807 的旧用法；0808 起统一 `dsh run "task"`（默认 headless profile）；
- web profile 不接受 task（无 headless-runner 行，会明确报错）。

## 运行时依赖解析（out-of-tree 插件）

- 插件的 peer 依赖（`@deepseek-ai/dsh-tools`、`cordis` 等）经 **profile fallback** 解析：启动时 `healProfilesModuleFallback` 把 app 依赖闭包链接到 `$DSH_HOME/profiles/node_modules/`；
- 因此**运行时** cordis 解析到 monorepo 的 vendor 副本，与编译期（坑 1 的解决方案）一致——这也是为什么必须 junction 到 vendor 而不是 .pnpm；
- 更新 dsh 快照后，fallback 自动重新链接到新 staging 的包；插件是 `link:` 依赖（指向源码目录），不需要重新 `add`。

## 验证闭环

```sh
dsh --version                                    # 新快照生效
dsh --profile web --dump-config | grep <row-id>  # 插件已挂载
dsh run "..."                                    # 端到端可用（0808 修复了 0807 的一次性模式输出问题）
```

## 卸载与重名冲突（实测补充）

- 移除：`dsh plugin --profile <name> remove @deepseek-ai/<plugin>`（pnpm remove 转发）；
- **collection meta 包与独立插件重名**：profile 已单独挂载 7 个工具时再挂 dsh-toolkit 会注册重名报错——二选一（移除独立插件或逐包挂载）；meta 包 apply 具备原子回滚（任一子插件失败逆序 dispose，不残留部分状态）；
- 插件是 `link:` 依赖时，更新 lib 无需重新 `add`（改源码 → 重构建 → 重启生效）。

→ 下一步：[testing.md](testing.md)

## 安装边界（官方 Profile Bundle 生态方向，immediate-adjustments-bundle-profile-plan）

- 插件代码归插件仓库；
- 插件组合由 profile 管理；
- 插件安装通过 `dsh plugin --profile <profile> add <path>`；
- DSH 核心仓库不承载第三方源码；
- 插件不通过替换官方入口获取能力（row id 避开 tools/session/llm/web/permission）；
- 手动复制、源码补丁和直接改核心配置只作为旧版本兼容或调试方案，不能作为默认安装流程。

README 安装章节顺序（统一模板）：

1. **Profile Bundle（推荐）**：`dsh plugin --profile web add "C:/path/to/<plugin>"`（需要一次性任务再加 headless）；
2. **验证安装**：`dsh --profile web --dump-config | grep <row-id>`；
3. **运行验证**：`dsh run "用 <tool> 完成最小任务"`；
4. **手动安装与旧版本兼容**：仅旧快照/调试。

web 与 headless 是不同 profile：web 安装不会自动覆盖 headless；`dsh run` 默认使用 headless。
`plugin_check` 的 `core-row-id` / `missing-profile-install-example` / `manual-install-only` /
`core-modification-required` 四项检查会验证上述边界（`schema` action 可见完整矩阵）。
