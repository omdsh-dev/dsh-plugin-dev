# 测试做法（8 个插件沿用的模式）

## 1. 注册契约测试（tests/register.spec.ts）

验证插件暴露的 cordis 契约 + 工具定义形状。用 `vi.mock` 屏蔽 dsh-tools，零依赖可跑：

```ts
import { describe, expect, it, vi } from 'vitest'
vi.mock('@deepseek-ai/dsh-tools', () => ({ defineTool: (opts: unknown) => opts }))
import { name, inject, apply } from '../src/index.ts'

describe('xxx: plugin registration contract', () => {
  it('exports the cordis plugin contract', () => {
    expect(name).toBe('@deepseek-ai/dsh-tool-xxx')
    expect(inject).toContain('tools')
    expect(typeof apply).toBe('function')
  })

  it('registers the tool with schema + render', () => {
    let captured: unknown
    const ctx: any = { tools: { register: (def: unknown) => { captured = def; return () => {} } } }
    apply(ctx)
    const def = captured as any
    expect(def.name).toBe('xxx')
    expect(def.parameters.action.required).toBe(true)   // enum/required 关键断言
    expect(typeof def.output.render).toBe('function')
    expect(def.timeoutMs).toBeGreaterThan(0)
  })
})
```

## 2. 逻辑测试（tests/impl.spec.ts）

纯逻辑模块（`parse.ts`/`impl.ts` 等）逐边界用例。dsh-tool-csv 的 parse.spec.ts 模式：每个解析器边界（引号、转义、CRLF、BOM、空行、畸形输入、超限）一条用例。

## 3. 差分/对照测试（诊断类工具，session-health 的做法）

涉及与官方实现语义对照的（如 zstd 帧扫描），用**真实文件 + 官方实现**做差分：

```ts
// 用 ~/.dsh/sessions 的真实多帧会话文件做 fixtures
// 断言：自研 scan 结果与官方 scanZstdFrames 帧数一致
```

## 4. 运行方式

```sh
# 用 monorepo 的 vitest 直接跑（插件目录内运行，避免 monorepo 配置干扰）
cd <plugin-dir> && node <monorepo>/node_modules/vitest/vitest.mjs run tests
# 注意：路径盘符必须大写（C:/...），小写 c:/ 会报 "Tests no tests"（坑 11）
```

## 5. 安全/只读断言（诊断类工具）

- "扫描后文件字节数不变"（只读保证）；
- "深度分析解析失败时明确降级不静默"。

## 6. ReDoS/长耗时用例：必须放可终止 worker（不能同步跑）

病理正则（`(a+)+$` + 40KB）同步执行会永久挂死测试进程——用 `node:worker_threads` + `eval: true` + 预算到期 `worker.terminate()`：

```ts
const outcome = await runInKillableWorker(code, 3000) // 'completed' | 'terminated'
expect(['completed', 'terminated']).toContain(outcome) // 断言不挂死
```

## 7. 真实管道直调（tool-driver，见 tool-plugin.md）

单元测试之外，用最小 cordis + ToolRegistry 服务栈对 `ctx.tools.execute` 直调全 actions/错误路径；输出存盘（`driver-output-*.txt`）作为审查证据。

## 用例数参考（截至 2026-08-08，实际值）

time 65 / encoding 46 / json 66 / calculator 31 / csv 50 / regex 63 / markdown 71（toolkit test-all 合计 392）。

→ 下一步：[publish.md](publish.md)
