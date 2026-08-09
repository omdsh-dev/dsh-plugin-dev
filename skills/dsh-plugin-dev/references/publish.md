# 发布与 hub 收录

## 0. 外部写操作授权门（审查 PD-03，必读）

以下操作**默认只生成命令/计划，不直接执行**；每项执行前必须获得当前任务的显式授权，并展示目标与影响：

- `gh repo create` / `gh repo edit --visibility`（建仓、改可见性）；
- hub 仓库写入（`catalog.source.json` 登记/提交）；
- `git commit` / `git push`（先展示待推送 diff 摘要、目标分支）；
- 任何可能公开/难逆的操作（如改 public）。

**禁止把历史任务的授权延伸到新仓库**；不确定时先问。

## 0a. 可见性政策（dsh-external 铁律，实测）

**在内测环境中任何提交到 dsh-external 的操作默认都是 private 的**：

- `gh repo create ... --public` 会被组织策略**覆盖**为 private（实测 csv/regex 创建后仍是 private）；
- 必须显式确认/设置：`gh repo view dsh-external/<repo> --json visibility`；`gh repo edit ... --visibility public|private --accept-visibility-change-consequences`；

## 1. 建仓与 description

- 在 `dsh-external` 组织建仓（默认 private，见上）；
- **description 直接影响 hub 列表文案**（hub 每 2 小时自动同步 GitHub description）。格式：

```
DSH CSV 数据工具插件：解析/查询/统计/转换 CSV 文本（RFC 4180），零依赖状态机解析器，注册 csv 工具
```

要点：`DSH xxx 插件：功能，特点，零依赖，注册 <工具名> 工具`，≤80 字符（hub 表格截断）。

## 2. topic 与 hub 收录

```sh
gh repo edit dsh-external/dsh-tool-xxx --add-topic marisa-plugin   # 可选增强，见下
```

- **实测现状**：组织内 9 个插件（含 plugin-check）实际**都没打 `marisa-plugin` topic**（`gh api .../topics` 全空）；hub 收录实际靠 `catalog.source.json` 登记（手动或等 2 小时自动同步）——topic 机制未在本组织验证，按"可选"描述，别写进验收门槛；
- 分类登记：`hub` 仓库的 `catalog.source.json` 的 `repos` 加 `{ name, category }`（plugin/collection/channel/infra/research/community）；
- 收录验证：`hub/catalog.json` 出现仓库条目（本地提交即可，自动同步会推送）。

## 3. collection 打包（工具族规模化，dsh-toolkit 的形态）

多个同风格插件打包成 collection 仓库：

```
dsh-toolkit/
├── catalog.json          # collection 清单（hub 识别 collection 分类）
├── cordis.patch.yml      # meta 包：insert tool-kit 行
├── packages/             # vendored 子包（冻结复制，保留各仓库独立演进）
└── scripts/              # build-all / test-all / install
```

- meta 包 `apply(ctx)` 依次调用各子包 `apply(ctx)`，用户一次 `dsh plugin add` 全量挂载；
- `hub/catalog.source.json` 登记 `category: collection`。

## 4. README 内容清单

- 安装命令（`dsh plugin --profile web add <path>`）；
- actions 表格 + 示例 + 边界说明；
- 验证命令（`--dump-config | grep tool-`）；
- 测试数标注（质量信号）。

## 5. 交付前闭环

1. `tsc` 零错误（注意：**报错仍会 emit**，见 build-pitfalls 坑 8——失败即停并验证产物），产物无 `.ts` 残留；
2. 测试全过；
3. `--dump-config` 见工具行；
4. `dsh run` 端到端成功；
5. code review 一轮（实现 → 评审 → 修复提交，如 dsh-tool-csv 的 C-01..C-04）；
6. 提交推送（SSH + `-u`），`gh repo view ... --json visibility` 确认可见性；
7. 新增工具后**全仓扫描旧计数**（`grep -nE '6|six|合计'`），README/description 同步（教训：sed 静默失效，见坑 12）。

## 6. 发布验收清单（Profile Bundle 生态，immediate-adjustments-bundle-profile-plan §4.7）

### 包结构

- [ ] package.json 的 `main`/`types`/`exports` 正确且指向仓库内真实存在文件（lib/ 随仓库提交，clean checkout 可运行）
- [ ] `dsh.bundle.patch` 存在；patch 文件进入发布内容（`files` 含 lib/src/cordis.patch.yml）
- [ ] row id 独立且不与官方核心 row（tools/session/llm/web/permission）冲突
- [ ] 宿主能力通过 peerDependencies 声明；无用的遗留 peer 移除

### 安装

- [ ] README 第一安装方式是 `dsh plugin --profile <profile> add`（web/headless 分开写明）
- [ ] 0808+ 用 `dsh run` 做端到端验证（exit 0）
- [ ] 重复安装可安全执行（幂等）；卸载单个子插件不影响其他插件

### 发布

- [ ] description 清晰；`marisa-plugin` topic 已设置
- [ ] collection 清单（catalog.json）已更新；hub 分类正确
- [ ] 提交作者与仓库所有者一致（whiteicey <whiteicey@users.noreply.github.com>）
- [ ] `plugin_check` 全绿（含生态四项：core-row-id / missing-profile-install-example / manual-install-only / core-modification-required）
