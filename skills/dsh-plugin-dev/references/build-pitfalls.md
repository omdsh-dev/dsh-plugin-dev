# 构建踩坑记录（全部实测）

## 2026-08-12：npm rc.1 路径下的适用性变化

- 坑 1 / 1b / 3 / 5（vendor cordis 双副本、junction、monorepo tsc 路径）**仅适用于 monorepo/旧 snapshot 场景**；
  npm 路径（`@deepseek-ai/dsh@0.0.1-rc.1`）下：peer 全为 scoped（`@deepseek-ai/cordis`），`npm install` 即得唯一 Cordis 身份与 devDependencies 工具链，无需 junction；
  新坑是"双 Cordis 分裂"——unscoped `cordis` import/peer 与 scoped 并存时 `ctx.tools`/`ctx.invariants` 类型与运行时身份不一致（dsh-tools 类型只增强 `@deepseek-ai/cordis`）。
- 坑 6（zstd 真实导入路径）：npm tarball 不含 `src/`，`@deepseek-ai/dsh-session-persistence-jsonl/src/zstd.ts` 在 npm 安装下 404——deep 模式降级 `decoder-unavailable`，frame-level 扫描不受影响；测试改用 `node:zlib` zstd 生成帧，官方差分在 npm 环境条件跳过。


## 坑 1：cordis 双副本 → `Property 'tools' does not exist on type 'Context'`

**问题**：插件编译报 `ctx.tools` 不存在，但 `defineTool` 导入正常。

**根因**：DSH monorepo **vendor 了 cordis**（`vendor/cordis`），monorepo 内所有包的 `.d.ts` 都从 vendor 解析 `cordis`。插件若从 `node_modules/.pnpm/cordis@.../` 副本解析 `cordis`，TypeScript 把两个物理副本视为**两个不同模块**——`@deepseek-ai/dsh-tools` 的 `declare module 'cordis' { interface Context { tools } }` 增强无法合并到插件的 `Context` 上。

**解决方案**：构建时 cordis 解析到 `vendor/cordis`（运行时 profile fallback 链接的也是它，天然一致）。

```sh
# 插件目录（构建期用，可 gitignore；Windows 建 junction 的正确姿势见"坑 1b"）
mkdir -p node_modules/@deepseek-ai
# cordis → <monorepo>/vendor/cordis（不是 .pnpm 副本）
# @deepseek-ai/dsh-tools → <monorepo>/packages/core/tools
```

验证：`tsc` 报错消失。**这是 out-of-tree 插件最致命的坑。**

## 坑 1b（Windows）：junction 创建的三个尝试，只有 PowerShell 成功

| 方法 | 实测结果（本机 git-bash） |
|---|---|
| `ln -s` / `ln -sfn` | `Operation not permitted`（无开发者模式/管理员权限） |
| `cmd /c mklink /J ...` | MSYS 参数转换破坏 `/J`（报"文件名、目录名或卷标语法不正确"）；设 `MSYS2_ARG_CONV_EXCL='*'` 也救不回来 |
| **`powershell -NoProfile -Command "New-Item -ItemType Junction ..."`** | **✅ 稳定可用（9 个 bundle 插件全部用它）** |

```sh
powershell -NoProfile -Command "New-Item -ItemType Junction -Path '$WD/dsh-tool-xxx/node_modules/cordis' -Target '$MONOREPO\vendor\cordis' | Out-Null"
```

注意：`@types` 不能整体 junction 到 monorepo 的 `node_modules/@types`（内部是 pnpm 符号链接，tsc 解析失败），必须直达 `.pnpm` 真实路径（见坑 3）。

## 坑 2：tsconfig 三件套

**问题**：

| 配置 | 缺失时的症状 |
|---|---|
| `"allowImportingTsExtensions": true` | `import ... from './x.ts'` 报 TS5097 |
| `"rewriteRelativeImportExtensions": true` | 产物 `lib/index.js` 里还是 `./x.ts`，**运行时 ESM 崩溃** |
| `"lib": ["ES2024"]` | `isWellFormed` 等新 API 报 TS2550 |

**解决方案**（dsh-tool-csv 的可用形态）：

```jsonc
{
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["ES2024"],
    "module": "esnext",
    "moduleResolution": "bundler",
    "outDir": "lib",
    "rootDir": "src",
    "declaration": true,
    "declarationDir": "lib/types",
    "allowImportingTsExtensions": true,
    "rewriteRelativeImportExtensions": true,
    "strict": true,
    "skipLibCheck": true
  },
  "include": ["src"],
  "exclude": ["tests", "lib"]
}
```

## 坑 3：node types（用 `Buffer`/`node:` 时）

**问题**：`Buffer`、`TextDecoder`、`node:crypto` 需要可解析的 `@types/node`。

**根因（Windows junction 陷阱）**：junction 到 monorepo 的 `node_modules/@types`（整体）**失败**——内部是 pnpm 符号链接，tsc 无法穿透。必须**直达 `.pnpm` 真实路径**（命令用坑 1b 的 PowerShell 形式）：

```sh
powershell -NoProfile -Command "New-Item -ItemType Junction -Path '$WD/node_modules/@types/node' -Target '$MONOREPO\node_modules\.pnpm\@types+node@22.20.0\node_modules\@types\node' | Out-Null"
```

**哪些工具真的需要（实测核对，勿照抄旧说法）**：

| 插件 | 用了 Buffer/node API？ | tsconfig 做法 |
|---|---|---|
| time / calculator / markdown | 否（markdown 用自写 `utf8ByteLength` 免依赖） | 不需要（但 2026-08-09 统一修复时已显式声明 types:["node"]，无害；markdown 是唯一真正无 node types 的） |
| encoding / json / csv / regex | **是**（`Buffer.byteLength`/`Buffer.from`） | `"types": ["node"]`（csv/regex 显式；encoding/json 靠无 `types` 字段时的隐式 @types 包含——较脆弱，建议显式） |

要点：tsconfig **不写 `"types"` 字段**时，TS 会隐式包含 node_modules/@types 下全部包——encoding/json 靠这个"碰巧"能编译；显式 `"types": ["node"]` 更稳。想彻底免 node types 就学 markdown：手写字节计数，不用 Buffer。

## 坑 4：lib 布局与 package.json 不一致

**问题**：`main`/`types` 声明 `lib/...` 但 tsconfig 无 `outDir` → 产物落到 `src/` 旁，运行时找不到入口。

**解决方案**：三件套齐全后，验证产物：

```sh
grep -rE "from './[^']+\.ts'" lib/ || echo "产物无 .ts 残留"
```

## 坑 5：构建命令（插件无 build script）

```sh
node <monorepo>/node_modules/typescript/bin/tsc -p tsconfig.json
```

## 坑 6（运行时，非构建）：多帧 zstd 会话文件

**问题**：诊断会话文件时用 `decompressZstdFrame` 只能解出第一个 frame——dsh 会话是**每批追加一个新 zstd frame**（0808 起 200ms 窗口）。实测一个 19MB 会话 = 12 万帧。单帧 API 会把"有完整事件的会话"误判成"只有 header"（#376 调查中误判过 33 个会话，后经官方读取路径复核推翻）。

**解决方案**（注意真实导入路径——包名带 `-jsonl` 后缀、zstd 未构建进 lib，`./src/*` 导出子路径）：

```ts
import { scanZstdFrames, createZstdFrameDecoder } from '@deepseek-ai/dsh-session-persistence-jsonl/src/zstd.ts'
const { frames } = scanZstdFrames(buf)
for (const f of createZstdFrameDecoder().decode(buf, frames)) { /* 逐帧 */ }
```

## 坑 7（0808 起）：环境变量层

**问题**：`DSH_*` 特殊变量放 `~/.dsh/.env` 会导致启动报错（启动方式相关变量必须由运行环境传入）；凭据已迁移到 `$DSH_HOME/.credentials.yaml`（LLM 配置只存引用）。

**解决方案**：`DSH_*` 一律由启动环境（wrapper/export）传入，`.env` 只放普通变量。

## 坑 8：tsc 报错仍然 emit 产物（noEmitOnError 默认 false）

**问题**：`tsc -p` 遇到类型错误（如 TS5097）**照样写出 lib/**——曾把 calculator 原仓库的 gitignored `lib/index.js` 覆盖成含 `./evaluate.ts` 导入的坏产物，导致 profile 启动时 `ERR_MODULE_NOT_FOUND`。

**解决方案**：构建脚本对失败必须**非零退出且不信任产物**（`tsc ... || exit 1`）；或用 `--noEmitOnError`；批量构建后统一验证产物无 `.ts` 残留（toolkit build-all 的产物校验就是为此加的）。

## 坑 9：worker 线程内直接跑 .ts（node 原生 type-stripping）

**问题**：需要"可终止的同步执行"（如正则 ReDoS 硬超时）时，worker 入口可以是 `.ts` 源码本身——Node ≥23.6 原生 strip-types 能加载；vitest 里 `import.meta.url` 指向 `src/`，产物里指向 `lib/`。

**解决方案**：worker URL 按源码/产物自适应：

```ts
const workerUrl = new URL(import.meta.url.endsWith('.ts') ? './worker.ts' : './worker.js', import.meta.url)
new Worker(workerUrl)
```

## 坑 10：V8 正则语义细节（写断言前先 `node -e` 实测）

| 现象 | 实测值 |
|---|---|
| 空 pattern 的 `RegExp.source` | `'(?:)'`（不是 `''`） |
| `(\d*)?` 空匹配的捕获组值 | `null`（不是 `''`，也不是 undefined） |
| `$10`（只有 1 个组） | `$1` + `'0'`（前缀回退） |
| `$0` / `$<未知名>` | **字面保留**（不展开） |
| `(a+)+$` + 40KB 失败输入 | 同步执行永久挂死（无回溯上限兜底）——测试必须放 worker/子进程 |

## 坑 11：vitest 的两个环境陷阱

1. **盘符大小写**：`node c:/.../vitest.mjs`（小写盘符）会报 `Tests no tests`——路径必须 `C:/` 大写（脚本里 `sed 's|^/\([a-zA-Z]\)/|\u\1:/|'`）；
2. **输出尾部空白行**：`vitest ... | tail -3` 会截掉 `Tests` 汇总行——用 `grep -E 'Test Files|Tests '` 取结果行。

## 坑 12：sed 批量替换静默失效

**问题**：`sed -i 's|旧|新|'` 匹配串写错时不报错、不替换（曾致 toolkit 的 package.json description 漏改，审查才抓出）。

**解决方案**：批量替换后必须 `grep` 验证命中数与新旧内容（或替换前后各 grep 一次对比）。

→ 下一步：[bundle-patch.md](bundle-patch.md)

## 坑 13：Windows 路径分隔符陷阱（resolve 比较全失败）

**问题**：`path.resolve()` 在 Windows 返回反斜杠（`C:\...`），而外部传入的 root 常是正斜杠（`C:/...`）——`resolved.startsWith(root + sep)` 恒为 false，导致"路径逃逸"误报（plugin-check 曾让全部 8 个插件误判 missing-main-or-types）。

**解决方案**：比较前对 root 也做 `resolve()`：`const rootResolved = resolve(root)`；或统一 `replace(/\/g, '/')` 后再比。

## 坑 14：工具/门禁设计教训（plugin-check 审查轮，2026-08-09）

1. **先定形态矩阵，再写规则**：bundle/registry/skill/collection/infra 各有合法协议；把一种形态当唯一协议必然误报（registry 原生插件可只有 `dsh.plugin.json + index.mjs`；bundle 可插入多个包，patch name 与包名不一致是合法的）。
2. **tsconfig extends 必须解析**：共享 base 模式（dsh-toolkit 子包）下，只读本地 compilerOptions 会系统性误报；解析失败标 skipped 而非确定性 fail。
3. **导入扫描覆盖全模式**：`from` / `import()` / `require()` / 副作用 `import` / `.tsx/.mts/.cts`；正则扫描还要**跳过注释行**（注释里的模式串会误报——plugin-check 自己的 lib 注释就命中过）。
4. **声明路径 containment**：main/types/patch 的 `../` 与绝对路径可逃逸仓库根；必须词法 + realpath 双重校验（同 session-health 的路径围栏）。
5. **YAML 行解析的坑**：行内注释（`id: a # comment`）、引号内 `#`、`config` 嵌套字段、`- update:` section——官方 PatchOptions 允许 config 与 id-targeted update，不要把合法字段当错误。
6. **npm 名校验**：只查前缀会放行 `dsh-`、`@scope/` 空名与 `a/b/c` 多段；用完整 scoped/unscoped 规则再叠加组织政策。
7. **测试要含真实多形态正向 fixtures**：只用自证模板做正向用例会自我循环（checker 全绿但误杀官方示例）；把 plugin-registry examples / collection / skill 纳入差分。
8. **报告统计语义**：`checks.total/passed` 应是"固定检查项的执行结果"（按形态适用矩阵），不是 issue 数。
9. **发布/写操作授权门**：gh repo create/edit、hub 写入、commit/push 默认只生成计划，执行前显式确认（见 publish.md §0）。

## 坑 15：Myers 行级 diff 的内存/时间预算（dsh-tool-diff，2026-08-09）

**问题**：朴素 Myers 需要保存每层 diagonal 的 V 数组快照做回溯。50K 行全部互异的对抗输入（n+m=100K，V 数组 20 万项 × 2001 层 trace ≈ 1.6GB）直接打爆内存；大量相同行错位排列时蛇步重复扫描也会超时。

**解决方案（四道防线，全部 O(n) 或常数）**：
1. **公共前后缀修剪**：先吃掉两侧相同的行（O(n+m)），只在剩余的小规模片段上跑 Myers；
2. **hash 快速拒绝**：修剪后若两侧剩余行无公共 hash（Set 判交），D 必超预算，立即回退"全删全增"；
3. **规模上限**：剩余 np+mp > 4000 直接回退（trace 内存 ≈ 2001×8001×4B ≈ 64MB 封顶）；
4. **蛇步总预算**：累计蛇步 > 2000 万中断并回退。

回退结果语义仍正确（操作序列合法），只是非最小——对工具输出可接受，文档明示。

**配套坑**：公共后缀重新拼接必须**正序**（`before[n-suf..n-1]`）——写成倒序（`n-1-i`）会让尾部 equal 行乱序（`ttt,sss,rrr`），patch 上下文错位、校验失败。这是"反向收集再整体 reverse"类算法最经典的 off-by-reverse。

## 坑 16：结构化 diff 的重复报告（markdown 块对齐）

**问题**：先按"完整块文本"做 Myers 对齐、再额外做同型块 replace 检查，会把同一个变化报成 remove+add+replace 三份（12 条变更 vs 实际 4 处）。

**解决方案**：对齐键用**块类型 token**（p/ul/code/table/...）而不是整块文本——同型块内容变化自然落成 equal 对，再在 equal 对上做内容比较出 replace；只有结构增删才出 remove/add。代码块另走 codeBlockChanges（语言+行数），不进 blockChanges。

## 坑 17：JSON 深度防线的位置（parse 之前，非递归）

**问题**：递归 diff 里做深度检查时，两个"深度相同但引用不同"的深嵌套结构会在 64 层处误报 replace（identical 输入也输出变更）。

**解决方案**：`JSON.parse` **之前**用 O(n) 非递归括号扫描（状态机跳过字符串字面量与转义）计算最大嵌套深度，超限直接抛错——既防调用栈溢出又避免 parse 本身卡死；diff 递归里的深度检查只留作兜底。

## 坑 18：driver 加载 lib 不是 src（改完源码要先构建）

**问题**：tool-driver 直连测试通过 `@deepseek-ai/dsh-tool-diff`（package.json main → lib/index.js）加载——改了 src 不重建 lib，driver 输出全是旧行为，容易误判"修复没生效"。

**解决方案**：改源码后 `tsc -p tsconfig.json` 重建再跑 driver；单测（vitest 直接 import src）不受影响，两者结果不一致时先怀疑 lib 过期。

## 坑 19：toolkit 并入新子包时的计数矩阵

**问题**：工具数变化（7→8→10）要同步改 7 处：meta `SUBPLUGINS`、`catalog.json`、README 工具表与合计（392→486）、`build-all.sh` 的 `EXPECTED` 列表与"7 个子包"文案、`test-all.sh` 注释、`package.json` description。漏一处就出现"README 说 8 个但 build-all 只验 7 个"的不一致。

**解决方案**：改完 `grep -n "7 个\|392\|EXPECTED"` 全局复查；build-all 的 EXPECTED 完整性校验 + 产物 .ts 导入扫描是最后防线。

## 坑 20：toolkit meta 与单插件的 profile 冲突验证法

**问题**：headless/web 已逐个挂载过同名单插件时，挂 toolkit meta 会报 `tool "time" is already registered`——没法直接 `dsh run` 验证 meta 聚合。

**解决方案**：临时换装法——`dsh plugin --profile headless remove` 掉 8 个单插件 → `add` toolkit → `dsh run` 验证 diff/csv 等经 meta 可用 → 再 remove toolkit + 重新 add 8 个单插件恢复原状。driver-toolkit.mjs（cordis+ToolRegistry 真实管道，只注册 meta）是无需动 profile 的快速回归。
