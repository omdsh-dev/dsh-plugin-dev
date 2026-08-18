---
name: dsh-plugin-dev
description: DeepSeek Harness（DSH）插件开发的踩坑与做法记录——覆盖插件形态选择、defineTool 工具开发、npm rc.7 依赖线（scoped cordis 迁移）、settings 新契约、独立构建（devDependencies + lockfile）、cordis.patch.yml 与 profile 挂载、测试、发布与 hub 收录。当开发新插件、复现旧做法、或遇到"ctx.tools 报错""双 Cordis 身份分裂""settings 卡片不显示""npm pack 怎么验"等问题时参考。
license: MIT
metadata:
  author: whiteicey
  version: "0.1.0"
---

# DSH 插件开发记录（dsh-plugin-dev）

一句话：**工具插件 = cordis 插件包 + `cordis.patch.yml`（bundle 形态）挂进 profile 就生效；但 bundle 不是唯一形态——registry（dsh.plugin.json）/ skill（SKILL.md）/ collection（catalog.json）各有合法协议，按需求选型（审查 PD-02）。**

本档案记录 DSH 公测期（dsh-external 组织）插件开发中实际发生过的坑与验证过能用的做法。内容按实际执行的顺序组织。2026-08-13 公测结束后，相关仓库已迁移至 omdsh-dev 组织并公开。


## npm rc.7 路径（官方 npm 发布后优先）

官方已发布 `@deepseek-ai/dsh@0.1.0-rc.7`（lib 生产模式）。外部插件开发与验证优先走 npm 路径：

- 启动指定版本：`npx -p @deepseek-ai/dsh@0.1.0-rc.7 dsh web`（不要 `install -g` 全局安装）
- 类型与运行时使用 scoped Cordis：源码 `import type { Context } from '@deepseek-ai/cordis'`；
  不要保留 unscoped `cordis` import/peer（dsh-tools 类型只增强 `@deepseek-ai/cordis`，双 Cordis 会导致 ctx.tools/ctx.invariants 类型与运行时身份分裂）
- peer 固定 rc.7 范围：`@deepseek-ai/cordis: ^4.0.1`、`@deepseek-ai/dsh-tools: ^0.1.0-rc.7`、`@deepseek-ai/dsh-invariants: ^0.1.0-rc.7`（需要 invariant companion 的包）
- devDependencies 自包含：typescript / vitest / @types/node + lockfile；独立 checkout 可 `npm install` → typecheck → test → build → `npm pack`
- 安装到隔离 profile：`npm pack` → `dsh plugin --profile compat add ./<pkg>.tgz` → `dsh --profile compat --dump-config`；web/headless 分属不同 profile
- NPM_TOKEN 为只读临时令牌：仅放环境变量或临时 userconfig，绝不写入项目 `.npmrc`/提交/日志
- 网络慢时公共包可走国内镜像（registry.npmmirror.com），`@deepseek-ai` scope 仍需官方 registry + token

monorepo/vendor cordis/junction 路径保留为"源码贡献/旧 snapshot"场景，不再是外部插件开发的默认方式。

## 开发流程（9 个插件沿用）

1. **选形态** → [references/overview.md](references/overview.md)
   按需求决策矩阵选 bundle / registry / skill / collection / infra（随 profile 启动 → bundle；registry 面板生命周期 → registry；纯提示词 → skill）。合规事实源：`plugin_check` 的 `schema` action 输出"检测项 × 形态适用矩阵"。

2. **搭骨架** → [references/tool-plugin.md](references/tool-plugin.md)
   文件树、package.json 关键字段、`cordis.patch.yml`、`src/index.ts` 的 `name`/`inject`/`apply` + `defineTool` 契约。

3. **写工具/设置卡片** → [references/tool-plugin.md](references/tool-plugin.md#definetool-契约)
   `defineTool` 参数 schema、`output.render`、`timeoutMs`、action 分发模式；若提供设置页，按该文的 `settings.plugin.item` keyed-slot、同名 Host schema 与 API proxy allowlist 契约实现。

4. **构建（先看踩坑记录）** → [references/build-pitfalls.md](references/build-pitfalls.md)
   npm 路径：`npm install`（devDependencies 自包含）→ `npm run typecheck` → `npm test` → `npm run build` → `npm pack`；peer 用 scoped rc.7（`@deepseek-ai/cordis` 等）。monorepo/vendor cordis/junction 仅"源码贡献/旧 snapshot"场景。

5. **挂载与验证** → [references/bundle-patch.md](references/bundle-patch.md)
   `dsh plugin --profile web|headless add <path>` → `--dump-config | grep tool-` → `dsh run "..."` 端到端。

6. **测试与发布** → [references/testing.md](references/testing.md) · [references/publish.md](references/publish.md)
   register.spec 契约测试 + 逻辑 spec；GitHub description（默认 private 政策）、hub 收录（catalog.source.json 登记）、collection 打包。

## 踩坑速查（详见 build-pitfalls.md）

| 症状 | 根因 | 解决方案 |
|---|---|---|
| `Property 'tools' does not exist on type 'Context'` | 双 Cordis：unscoped `cordis` 与 `@deepseek-ai/cordis` 是两个模块，dsh-tools 类型只增强 scoped | npm rc.7：全链 scoped（import/peer 统一 `@deepseek-ai/cordis`）；monorepo 场景：cordis 解析到 `vendor/cordis` |
| `TS5097: import path can only end with '.ts'` | tsconfig 缺 `allowImportingTsExtensions` | 补 + `rewriteRelativeImportExtensions`（产物自动 `.js`） |
| 产物 `lib/index.js` 里还是 `./x.ts` | 同上缺 rewrite | 重新构建，验证产物 |
| `TS2591: Cannot find name 'Buffer'` | 缺 node types | `types: ["node"]` + devDependencies `@types/node`（npm 路径 npm install 即得；monorepo 场景 junction 直达 `.pnpm/@types+node@*`，见坑 1b） |
| `dsh: profile web takes no task` | web profile 无 headless-runner 行 | 一次性任务改用 `dsh run`（0808+） |
| 会话文件"只有 header" | 误用单帧解码 API（`decompressZstdFrame`）；dsh 会话是多帧 zstd 追加写入 | 用 `scanZstdFrames` + decoder 逐帧解（真实导入路径 `@deepseek-ai/dsh-session-persistence-jsonl/src/zstd.ts`） |
| `tsc` 报错却生成了坏产物 | `noEmitOnError` 默认 false，**报错仍 emit** | 构建失败即停（`|| exit 1`）+ 产物统一验 `.ts` 残留（坑 8） |
| 建仓后以为是 public | `gh repo create --public` 被组织策略覆盖 | 公测期：显式 `gh repo view/edit --visibility` 确认——**dsh-external 默认全 private**；公测后：仓库已迁 omdsh-dev 组织并公开，但新仓库仍默认 private、公开需显式授权 |

## 交付前验证闭环

```sh
npm run typecheck                                                        # 零错误（npm 路径；monorepo：node <monorepo>/node_modules/typescript/bin/tsc -p tsconfig.json）
grep -rE "from './[^']+\.ts'" lib/ || echo "产物无 .ts 残留"            # 产物干净
npm run build && npm pack                                                   # 可发布产物（tarball 检查 exports 指向存在）
dsh --profile compat --dump-config | grep <工具行 id>                      # 已挂载（rc.7 consumer）
npm run verify:execution                                                   # 工具真实注册与执行（rc.7 consumer）
npm test                                                                   # 测试通过
```
