# 构建踩坑记录（全部实测）

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
| **`powershell -NoProfile -Command "New-Item -ItemType Junction ..."`** | **✅ 稳定可用（8 个插件全部用它）** |

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
| time / calculator / markdown | 否（markdown 用自写 `utf8ByteLength` 免依赖） | 不需要 node types |
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
