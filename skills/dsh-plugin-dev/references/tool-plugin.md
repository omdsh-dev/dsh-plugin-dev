# 工具插件开发记录（defineTool 契约）

以 dsh-tool-csv 为模板（组织内 9 个 bundle 插件同模式，均验证可用）。

## 文件树

```
dsh-tool-xxx/
├── LICENSE
├── README.md            # 安装/用法/示例/边界说明
├── cordis.patch.yml     # bundle patch：insert 一行
├── package.json
├── tsconfig.json
├── src/
│   ├── index.ts         # 插件入口：name/inject/apply + defineTool
│   └── impl.ts          # 纯逻辑（导出供测试）
└── tests/
    ├── register.spec.ts # 注册契约测试
    └── impl.spec.ts     # 逻辑测试
```

## package.json 关键字段

```jsonc
{
  "name": "@deepseek-ai/dsh-tool-xxx",
  "main": "lib/index.js",
  "types": "lib/types/index.d.ts",
  "scripts": {                                   // 审查 PD-01：模板必须带（否则 plugin-check 报 no-build-script/no-build-entry）
    "build": "tsc -p tsconfig.json",
    "prepack": "npm run build",
    "test": "vitest run tests"
  },
  "peerDependencies": {
    "@deepseek-ai/dsh-tools": "^0.1.1-rc.1",
    "@deepseek-ai/cordis": "^4.0.1",
    "@deepseek-ai/dsh-invariants": "^0.1.1-rc.1"
  },
  "dsh": { "bundle": { "patch": "./cordis.patch.yml" } },   // bundle 声明，profile 自动挂载
  "exports": {
    ".": { "types": "./lib/types/index.d.ts", "default": "./lib/index.js" },
    "./cordis.patch.yml": "./cordis.patch.yml",
    "./package.json": "./package.json"
  },
  "files": ["lib", "src", "cordis.patch.yml"]
}
```

> 自包含构建（审查 PD-05）：若要在不依赖外部 DSH monorepo 的环境跑 `npm run build/test`，
> 补 devDependencies（typescript / vitest / @types/node）+ lockfile；out-of-tree 开发则沿用
> 本档案的 monorepo tsc/vitest 路径。

## cordis.patch.yml

```yaml
# dsh bundle patch: inserts this plugin into a profile's layer stack (0806+).
- insert:
    - id: tool-xxx
      name: '@deepseek-ai/dsh-tool-xxx'
```

## 插件入口（src/index.ts）

```ts
import type { Context } from '@deepseek-ai/cordis'
import { defineTool } from '@deepseek-ai/dsh-tools'

export const name = '@deepseek-ai/dsh-tool-xxx'
export const inject = ['tools']

export function apply(ctx: Context): void {
  ctx.tools.register(defineTool({
    name: 'xxx',
    description: 'Human-readable description sent to the model. 明确边界与安全提示。',
    parameters: {
      action: {
        type: 'string',
        required: true,
        enum: ['a', 'b'],
        description: 'Operation to perform.',
      },
      // ... 更多参数
    },
    output: {
      schema: { type: 'string' },   // 统一 JSON 文本输出（见下）
      render: (_args, value) => [{ type: 'text', text: value }],
    },
    execute: args => Promise.resolve(runAction(args)),
    timeoutMs: 2000,
  }))
}
```

## defineTool 契约（实测确认的字段）

- `name`：工具名（必须唯一，模型可见）
- `description`：给模型的说明；把边界写进去（如"value 是精确匹配""不要传密钥"）
- `parameters`：对象，每个属性一个 schema：
  - 字符串：`{ type: 'string', required?, enum?, const?, description?, title?, default?, examples? }`
  - 数字：`{ type: 'number' | 'integer', ... }`；布尔：`{ type: 'boolean' }`
- `output.schema`：规范化输出 schema（`{ type: 'string' }` 最常用——不同 action 返回形状不同时统一 JSON 文本）
- `output.render(args, value)`：返回 `ContentBlock[]`（`[{ type: 'text', text: String(value) }]`）
- `timeoutMs`：执行预算——**注意（实测）：对同步阻塞体是协作式**。`exec.signal` 只能取消"让出事件循环"的异步体；`(a+)+$` 这类灾难性回溯会占死事件循环，timeoutMs 救不了。需要硬超时就把执行放进**可终止的 worker**（regex 插件 R-01 模式：`new Worker(workerUrl)` + 预算到期 `worker.terminate()`，报 `xxx: execution timed out`）
- `execute(args)`：返回 Promise；同步逻辑用 `Promise.resolve(...)` 包

## 输出模式（9 个插件，三种并存——勿写成"全部统一"）

**实测核对（2026-08-09）：不是"全部统一 JSON 文本"**——按返回形状选 schema：

| 模式 | output.schema | render | 插件 |
|---|---|---|---|
| A. JSON 文本字符串（主流） | `{ type: 'string' }` | `(_a, v) => [{ type: 'text', text: v }]`（v 已是 JSON 字符串） | csv / regex / markdown / session-health / plugin-check |
| B. JSON 对象 | `{ type: 'json' }` | `JSON.stringify(value)`（time 用 `JSON.stringify(value, null, 2)`） | time / encoding / json |
| C. 标量 | `{ type: 'number' }` | `String(value)` | calculator |

选择原则：多 action 且返回形状统一为 JSON → 模式 A（模型消费等价、实现测试最简单）；单值/单 action → 模式 B/C 更直接。文档早期"csv/regex/time 都是 string"的说法**已勘误**（time 实际是 json schema）。

1. **错误用 throw**：`execute` 抛 `new Error('xxx: 原因')`；**实测：dsh 工具管道把工具失败作为 error content block 返回（`ctx.tools.execute` 不 reject）**，schema 校验失败（未知 action 等）也走同一路径——driver 断言按 content 文本核对；
2. **确定性**：纯函数、无副作用、无网络——参数会记入会话日志，工具描述中明确警告敏感输入。

## 工具管道直调验证（tool-driver 模式，用户认可的"真实测试"）

绕过 LLM 直接打真实管道——与官方 tool-bash 测试同构的最小服务栈：

```js
import { Context } from '@deepseek-ai/cordis'
import SystemPrompt from '@deepseek-ai/dsh-system-prompt'
import ToolRegistry from '@deepseek-ai/dsh-tools'
const ctx = new Context()
await ctx.plugin(SystemPrompt); await ctx.plugin(ToolRegistry); await ctx.plugin(MyPlugin)
const r = await ctx.tools.execute({ signal, callId, name, arguments: args })
```

覆盖：正常路径全 actions + 错误路径全分支 + 输入上限；配合 `dsh run` 做真实 LLM 端到端。

## action 分发模式

```ts
function runAction(args: Args): string {
  switch (args.action) {
    case 'parse': return JSON.stringify(parseCsv(args.csv, opts))
    case 'query': return JSON.stringify(queryRows(...))
    // ...
    default: throw new Error(`xxx: unknown action ${String(args.action)}`)
  }
}
```

## RC1 多模态 command envelope

官方 `@deepseek-ai/dsh-commands@next` 已将命令输入从 text-only 扩展为完整 envelope。命令定义必须在 `input.images` 明确声明图片能力；执行端传递 `EncodedImageAttachment[]`，Host 通过 `admitEncodedImages` 做 canonical base64、格式、数量和总字节 admission，再把冻结有序 `ImageBlock[]` 放到 invocation。图片发给未声明的命令时必须得到错误结果，不能静默丢弃；直接命令结果也不会自动进入 model history。

官方 `@deepseek-ai/dsh-attachment@next` 当前图片格式为 PNG/JPEG/WebP/GIF，持久化引用只能使用 `ImageAttachmentRef`，不得把浏览器路径、object URL、provider URL 或 base64 写入 session event。

## RC1 client graph 与 settings namespace

浏览器包的值导入使用 `exports["./client"]` subpath；`@deepseek-ai/dsh-client-modules@next` 会按 lazy-CJS graph 先注册 dynamic providers 再 materialize consumers。不要用裸包名导入另一个 client bundle，以免产生第二份模块实例。

`@deepseek-ai/dsh-client-web-react@next` 与 `@deepseek-ai/dsh-client-schema-form@next` 目前官方 `next` 仍解析到 0.1.0-rc.7；没有 RC1 版时不要虚构升级。新代码应按实际 RC1 包（例如 `dsh-client-modules`、`dsh-client-runtime` 与对应 `client`/slot contract）迁移，待官方提供替代包后再改依赖。

Settings namespace 是唯一 key：Host `register(ns, schema, ...)` 与浏览器 `settings.plugin.item` 的 key 必须一致；wire 读取使用 `describe({ redactSecrets: true })`。脱敏视图上的删除用 `mutate`，不要用不完整对象 `replace`。

## rc.1 设置页插件卡片（`settings.plugin.item` keyed slot）

当插件同时提供 Host 设置 namespace 与浏览器设置卡片时，三处名称必须是同一个值（例如 `plugin-foo`）：

1. **客户端 keyed slot**：`settings.plugin.item` 是 keyed slot，注册时必须用 `key: SETTINGS_NS`，且 `SETTINGS_NS` 必须等于 settings namespace；不要把它写成旧式 `id` 或任意插件名。keyed slot 的 key 是分发与去重依据。

   ```ts
   const SETTINGS_NS = 'plugin-foo'
   ctx.slots.inject('settings.plugin.item', () => ctx.slots.register({
     name: 'settings.plugin.item',
     key: SETTINGS_NS,
     order: 0,
     locale: 'settings.plugin-foo',
   }, PluginSettingsItem))
   ```

2. **Host 同名 schema**：Host 半必须以同名 namespace 注册 schema（通常是 `ctx.settings.register(settingsNamespace(SETTINGS_NS), schema, { base, applies })`）。schema 是表单的事实来源；客户端用 `settings.describe` 收到序列化 schema 和脱敏后的 `value/base/user/secrets/revision`，不要在浏览器重写一份 Host schema。

3. **Host API proxy allowlist**：Host 的 API proxy 必须把同名 namespace 加入配置客户端 allowlist，并与 schema/注册生命周期同步。未列入 allowlist 的 namespace 不会出现在 `settings.describe`，写入会得到 `settings-not-exposed`；仅注册 namespace 不等于自动对浏览器公开。所有 wire 读取都走 `redactSecrets: true`，secret 字段只能在明确的写入 payload 中发送。

设置写入优先使用 `settings.update`（保留未发送的 secret）；持有脱敏视图而需要删除单个字段时使用 `settings.mutate`，不要用重建后的不完整 `user` 做 wholesale `replace`。这条 settings 卡片契约不改变旧的 Profile Bundle 安装方式或 `defineTool` 工具注册契约。

→ 下一步：[build-pitfalls.md](build-pitfalls.md)（先看踩坑记录再构建）
