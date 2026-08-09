---
name: dsh-plugin-dev
description: DeepSeek Harness（DSH）插件开发的踩坑与做法记录——覆盖插件形态选择、defineTool 工具开发、构建踩坑（vendor cordis 双副本 / tsconfig 三件套 / Windows junction）、cordis.patch.yml 与 profile 挂载、测试、发布与 hub 收录。当开发新插件、复现旧做法、或遇到"ctx.tools 报错""产物导入还是 .ts""dsh plugin add 怎么用"等问题时参考。
license: MIT
metadata:
  author: whiteicey
  version: "0.1.0"
---

# DSH 插件开发记录（dsh-plugin-dev）

一句话：**工具插件 = cordis 插件包 + `cordis.patch.yml`（bundle 形态）挂进 profile 就生效；但 bundle 不是唯一形态——registry（dsh.plugin.json）/ skill（SKILL.md）/ collection（catalog.json）各有合法协议，按需求选型（审查 PD-02）。**

本档案记录 dsh-external 插件开发中实际发生过的坑与验证过能用的做法。内容按实际执行的顺序组织。

## 开发流程（9 个插件沿用）

1. **选形态** → [references/overview.md](references/overview.md)
   按需求决策矩阵选 bundle / registry / skill / collection / infra（随 profile 启动 → bundle；registry 面板生命周期 → registry；纯提示词 → skill）。合规事实源：`plugin_check` 的 `schema` action 输出"检测项 × 形态适用矩阵"。

2. **搭骨架** → [references/tool-plugin.md](references/tool-plugin.md)
   文件树、package.json 关键字段、`cordis.patch.yml`、`src/index.ts` 的 `name`/`inject`/`apply` + `defineTool` 契约。

3. **写工具** → [references/tool-plugin.md](references/tool-plugin.md#definetool-契约)
   `defineTool` 参数 schema、`output.render`、`timeoutMs`、action 分发模式。

4. **构建（先看踩坑记录）** → [references/build-pitfalls.md](references/build-pitfalls.md)
   vendor cordis 双副本（`ctx.tools` 类型失效的根因）、tsconfig 三件套、Windows junction 方案。dsh-tool-csv 的 tsconfig 是验证过的零成本起点。

5. **挂载与验证** → [references/bundle-patch.md](references/bundle-patch.md)
   `dsh plugin --profile web|headless add <path>` → `--dump-config | grep tool-` → `dsh run "..."` 端到端。

6. **测试与发布** → [references/testing.md](references/testing.md) · [references/publish.md](references/publish.md)
   register.spec 契约测试 + 逻辑 spec；GitHub description（默认 private 政策）、hub 收录（catalog.source.json 登记）、collection 打包。

## 踩坑速查（详见 build-pitfalls.md）

| 症状 | 根因 | 解决方案 |
|---|---|---|
| `Property 'tools' does not exist on type 'Context'` | cordis 解析到 `.pnpm` 副本，与 monorepo 的 `vendor/cordis` 是两个模块，类型增强不合并 | 构建时 cordis 解析到 `vendor/cordis` |
| `TS5097: import path can only end with '.ts'` | tsconfig 缺 `allowImportingTsExtensions` | 补 + `rewriteRelativeImportExtensions`（产物自动 `.js`） |
| 产物 `lib/index.js` 里还是 `./x.ts` | 同上缺 rewrite | 重新构建，验证产物 |
| `TS2591: Cannot find name 'Buffer'` | 缺 node types | `types: ["node"]`，junction 直达 `.pnpm/@types+node@*`（Windows 用 PowerShell `New-Item -ItemType Junction`；`ln -s`/`mklink /J` 在本机均失败，见坑 1b） |
| `dsh: profile web takes no task` | web profile 无 headless-runner 行 | 一次性任务改用 `dsh run`（0808+） |
| 会话文件"只有 header" | 误用单帧解码 API（`decompressZstdFrame`）；dsh 会话是多帧 zstd 追加写入 | 用 `scanZstdFrames` + decoder 逐帧解（真实导入路径 `@deepseek-ai/dsh-session-persistence-jsonl/src/zstd.ts`） |
| `tsc` 报错却生成了坏产物 | `noEmitOnError` 默认 false，**报错仍 emit** | 构建失败即停（`|| exit 1`）+ 产物统一验 `.ts` 残留（坑 8） |
| 建仓后以为是 public | `gh repo create --public` 被组织策略覆盖 | 显式 `gh repo view/edit --visibility` 确认——**dsh-external 默认全 private** |

## 交付前验证闭环

```sh
node <monorepo>/node_modules/typescript/bin/tsc -p tsconfig.json      # 零错误
grep -rE "from './[^']+\.ts'" lib/ || echo "产物无 .ts 残留"            # 产物干净
dsh --profile web --dump-config | grep <工具行 id>                      # 已挂载
dsh run "用 <工具名> 工具做一次端到端任务"                                # 实际可用
node <monorepo>/node_modules/vitest/vitest.mjs run tests               # 测试通过
```
