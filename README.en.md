<h1 align="center">dsh-plugin-dev</h1>

<p align="center">
  [中文](README.md)
</p>

<p align="center">
  <strong>Pitfalls encountered and practices verified during DeepSeek Harness plugin development.</strong><br/>
  Plugin development practices recorded during the DSH public beta (dsh-external organization): vendor cordis dual copies, the tsconfig trio, Windows junction, multi-frame zstd API... Every pitfall includes the symptom observed when it occurred, the confirmed root cause, and the final fix. After the public beta ended, the related repositories were migrated to the [omdsh-dev](https://github.com/omdsh-dev) organization and made public.
</p>

<p align="center">
  <img src="https://badgen.net/badge/license/MIT/blue" alt="MIT" />
</p>

---

## What This Is

An **experience archive** (skill + documentation): records the plugin development workflow, the pitfalls hit, and the practices verified to work.

## How to Use

1. Put `skills/dsh-plugin-dev` into your skills directory (or reference it in an agent session);
2. Start with the workflow and pitfall quick-reference table in [SKILL.md](skills/dsh-plugin-dev/SKILL.md);
3. Before building, read [references/build-pitfalls.md](skills/dsh-plugin-dev/references/build-pitfalls.md) — the first entry is the cordis dual-copy issue.

## Archive Map

| Doc | What it records |
|------|------|
| [SKILL.md](skills/dsh-plugin-dev/SKILL.md) | Development workflow + pitfall quick-reference table + pre-delivery verification loop |
| [overview.md](skills/dsh-plugin-dev/references/overview.md) | Shape selection (bundle) + surveyed ecosystem map |
| [tool-plugin.md](skills/dsh-plugin-dev/references/tool-plugin.md) | defineTool contract, parameter schema, output modes, and the rc.1 settings namespace/keyed-slot contract |
| [build-pitfalls.md](skills/dsh-plugin-dev/references/build-pitfalls.md) | Complete pitfall collection (cordis dual copy / tsconfig / junction / multi-frame API) |
| [bundle-patch.md](skills/dsh-plugin-dev/references/bundle-patch.md) | profile/bundle mechanism, `dsh plugin add`, `dsh run` verification |
| [testing.md](skills/dsh-plugin-dev/references/testing.md) | Contract test / logic test / differential test patterns |
| [publish.md](skills/dsh-plugin-dev/references/publish.md) | description, topic, hub listing, collection packaging |

## Environment Baseline

> When troubleshooting/reproducing an issue or evaluating whether a pitfall still applies, first cross-check this table to confirm the environment matches; when reporting an issue, attach the output of `dsh --version` and `readlink ~/.dsh/source/current`.

### Runtime and Tool Versions

| Item | Version/Value | Notes |
|---|---|---|
| OS | Windows 11 Pro (build 26200), git-bash (MSYS2 3.5.7) | All Windows-specific cases in this document were verified in this environment |
| Node | **v24.18.1** (`~/node24` portable) | Used preferentially by the dsh wrapper; system node 22.15 is unusable |
| dsh (npm) | **npm @deepseek-ai/dsh@next (currently resolves to 0.1.1-rc.1, lib production mode)** | Started via `npx -p @deepseek-ai/dsh@next dsh web --no-open` (lib production mode; do not `install -g` globally) |
| TypeScript / Vitest | **Each repo's devDependencies are self-contained (typescript/vitest/@types/node + lockfile)** | An independent checkout can run `npm install` → `npm run typecheck` → `npm test` → `npm run build` → `npm pack` |
| pnpm | **11.18.0** | Forwarded internally by `dsh plugin` (within the profile directory) |
| gh CLI | **2.97.0** (2026-07-31), account whiteicey, scopes `gist, read:org, repo` | API operations and repository creation/visibility management |
| @types/node | 22.20.0 / 25.9.3 / 26.1.2 coexist under `.pnpm`, **22.20.0 used for builds** | Junction reaches directly into `.pnpm/@types+node@22.20.0/node_modules/@types/node` |

### Key Paths

| Path | Contents |
|---|---|
| `~/.dsh`（`$DSH_HOME`） | profiles / sessions / source / settings.yaml / web.log |
| `~/.dsh/source/current` | → DSH 0.1.1-rc.1 (npm) — a leftover of the snapshot-junction era (does not exist under npm mode) |
| `<monorepo>/vendor/cordis` | **The only legitimate cordis resolution source at build time** (pitfall 1) |
| `<monorepo>/packages/core/tools` | `@deepseek-ai/dsh-tools` (defineTool/tool pipeline) |
| `<monorepo>/node_modules/.pnpm/@types+node@22.20.0/...` | Real path of @types/node (pitfall 3) |
| `~/.dsh/profiles/{web,headless}` | profile directories (`dsh.profile.bundles` + cordis.yml + patch layer) |
| `~/.dsh/sessions/<cwd 编码>/<session-id>/session.jsonl.zstd` | Multi-frame zstd session files (pitfall 6) |
| `~/node24`、`~/.local/bin/dsh` | Portable Node, dsh launcher wrapper |

### Environment Variables and Startup Method

| Variable | Value | Notes |
|---|---|---|
| `DSH_PERMISSION_MODE` | `danger-full-access` | **⚠️ High-risk mode (review PD-04)**: Windows has no sandbox backend (bwrap/Landlock/Seatbelt); only this mode can start, and it **disables approval prompts** — use it only temporarily on a trusted local dev machine; **do not** put it into project templates, CI, or shared machines, nor copy it as a general recommendation |
| `DSH_TELEMETRY_DISABLED` | `1` | User chose to disable telemetry |
| `DSH_HOME` | `C:\Users\admin\.dsh` | Defaults to `~/.dsh` when not explicitly set |
| `DSH_*` special variables | Always passed in by the launch environment (wrapper/export) | Putting them in `~/.dsh/.env` causes a startup error (pitfall 7) |

Startup: `npx -p @deepseek-ai/dsh@next dsh web --no-open` (npm 0.1.1-rc.1, lib production mode; do not `install -g` globally). The old snapshot-era wrapper is deprecated: `~/.local/bin/dsh` (do not run `bin/dsh` directly — on Windows, MSYS path conversion triggers `ERR_UNSUPPORTED_ESM_URL_SCHEME`, issue #388; the wrapper launches tsx with a `file://` URL to work around it).

### Platform Behavior Differences (versus the "standard practice" docs)

| Behavior | Verified locally |
|---|---|
| Junction creation | Both `ln -s` and `cmd mklink /J` fail; **PowerShell `New-Item -ItemType Junction` works** (pitfall 1b) |
| Repository visibility | **dsh-external defaults everything to private during the public beta**; after the beta ended on 2026-08-13, the 15 repositories covered by this archive were migrated to the [omdsh-dev](https://github.com/omdsh-dev) organization and made public |
| headless one-shot tasks | #376 on 0807 (no output/exit code 1); **use `dsh run "task"` from 0808 on**, fixed |
| Web GUI | `dsh web` listens on `127.0.0.1:3080`; the GUI must be restarted after installing a plugin for new tools to load |

### Self-check Command Quick Reference

```sh
dsh --version && readlink ~/.dsh/source/current     # 快照
node -v                                              # Node
gh --version && gh auth status                       # gh 与认证
node <mono>/node_modules/typescript/bin/tsc --version  # TS（<mono> 换成 current 真实路径）
node <mono>/node_modules/vitest/vitest.mjs --version   # Vitest
```

## Maintenance

- The pitfall list is continuously extended as dsh snapshots evolve (e.g. 0808's `dsh run`, credential migration, 200ms batch persistence);
- New pitfalls are appended once recorded (including non-Windows platform experience, if any).

### Build Dependency Layering (review PD-05)

| Layer | Method | When applicable |
|---|---|---|
| Preferred (current) | Each repo's devDependencies are self-contained (typescript/vitest/@types/node + lockfile); an independent checkout can run `npm install` → `npm run typecheck` → `npm test` → `npm run build` → `npm pack` | Reproducible builds/CI |
| Legacy (out-of-tree) | `DSH_MONOREPO` points at the current snapshot, using the monorepo's tsc/vitest | Snapshot-era local plugin development (historical record) |
| Environment fallback | Internal paths under `.pnpm/@types+node@*` (**versions change**; auto-discover with `ls .pnpm/@types+node@* \| sort -V \| tail -1`) | This machine only |
