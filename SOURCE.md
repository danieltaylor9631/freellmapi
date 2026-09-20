# FreeLLMAPI 源代码说明

> 文档编号：SOURCE.md  
> 统计口径：仓库内 `*.ts` / `*.tsx` / `*.js` / `*.mjs`，排除 `node_modules`、`.git`、构建产物  
> 统计时刻：对当前工作区静态扫描  

## 1. 源代码整体介绍

### 1.1 项目是什么

FreeLLMAPI（npm 工作区名 `@freellmapi/monorepo`）是自托管 LLM 网关。源码的职责是：在一台机器上跑 Express 代理，把多家免费提供方与自定义 OpenAI 兼容端点聚合成 `/v1`，并用 React 仪表盘管理密钥与路由。它不是模型训练代码，不包含 CUDA 内核，几乎全部是 TypeScript 编写的网络、路由、记账与 UI。

### 1.2 使用的语言

| 语言 | 角色 | 说明 |
|---|---|---|
| TypeScript | 主体 | server、client、cli、desktop、shared 几乎全部 |
| TSX | 仪表盘 | React 19 组件与页面 |
| JavaScript / ESM `.mjs` | 胶水 | 桌面打包脚本、bootstrap 测试、i18n 检查 |
| SQL | 嵌入在迁移 | SQLite DDL/DML 写在 `*.ts` 字符串中 |
| JSON | 配置与 i18n | locale、tools.json、package.json |
| CSS | 仪表盘 | `client/src/index.css` + Tailwind 4 |
| YAML | Compose 与生成器 | docker-compose、部分 Agent 配置输出 |
| Bash / PowerShell | 安装 | `docs/install.sh`、`docs/install.ps1`、`docker-entrypoint.sh` |
| HTML | 文档站点与入口 | `client/index.html`、`docs/*.html` |
| Dockerfile | 镜像 | 多阶段构建 Node 服务 |

**没有** Python 运行时、没有 Go、没有 Rust。桌面层用 Electron（Chromium + Node）包一层 TypeScript 主进程。

TypeScript 版本：server 与 desktop 约 5.8，client 与 cli 约 6.0（`typescript` / `~6.0.2`）。模块系统统一 `"type": "module"`，导入带 `.js` 后缀以符合 Node ESM 解析。

### 1.3 开发工具与运行时

| 工具 | 用途 |
|---|---|
| Node.js `>=20.18.0 <25.0.0` | 运行时，`.nvmrc` 锁定大版本 |
| npm `>=10` workspaces | 包管理，不用 pnpm/yarn |
| tsx | server 开发热加载 `tsx watch src/index.ts` |
| tsc | server/cli 生产编译 |
| Vite 8 | client 开发与打包 |
| Vitest 3 | 单测与组件测，server 使用 `pool=forks` 且 `fileParallelism=false` |
| ESLint 9 + typescript-eslint | server/cli/client lint |
| Express 5 | HTTP |
| better-sqlite3 12（可选依赖） | 默认 SQLite；Android 走 Node 内置 |
| Helmet / cors / compression / multer / sharp | 安全、上传、图像 |
| undici + socks-proxy-agent | 出站 HTTP 与 SOCKS |
| zod / ajv | 校验 |
| React 19 + react-router 7 + TanStack Query 5 | 仪表盘 |
| Tailwind 4 + Base UI + lucide + recharts | UI |
| highlight.js / react-markdown | 试验台 |
| Electron 38 + electron-builder 25 + esbuild | 桌面 |
| concurrently | 同时起 server 与 client |
| Docker / Compose | 生产分发 |
| GitHub Actions | CI（见 `.github/workflows`） |
| Vitest coverage v8 | server 可选覆盖率 |

IDE 无强制。仓库含 `.claude/hooks` 做贡献检查，说明维护者使用 Claude Code 工作流，但不作为编译依赖。

### 1.4 源代码文件数与行数（扫描结果）

纳入扫描的 TS/JS 源文件共 **719** 个，合计约 **146312** 行（含测试、含空行与注释，不含 `package-lock.json` 与 locale JSON）。

按扩展名：

- `.ts`：603 个文件，125362 行
- `.tsx`：103 个文件，20056 行
- `.mjs`：9 个文件，651 行
- `.js`：4 个文件，243 行

按顶层目录：

| 目录 | 文件数 | 行数 | 占比（行） | 职责 |
|---|---:|---:|---:|---|
| `server/` | 502 | 112183 | 76.7% | 网关、路由、提供方、数据库、测试主体 |
| `client/` | 174 | 26574 | 18.2% | 管理仪表盘与前端单测 |
| `cli/` | 12 | 4122 | 2.8% | 智能体配置生成器与 doctor |
| `desktop/` | 28 | 2525 | 1.7% | Electron 壳与打包脚本测试 |
| `shared/` | 1 | 659 | 0.5% | 共享类型单一模块 |
| `examples/` | 1 | 159 | 0.1% | 示例程序 |
| `scripts/` | 1 | 90 | 0.1% | 仓库级 node 测试脚本 |


实现文件（非测试）**385** 个；测试文件 **334** 个。测试文件中解析到 **3479** 条 `it`/`test`。函数/类声明约 **2514** 个，`export` 符号约 **1730** 个。

这些数字说明：这是一个**测试密度很高**的 TypeScript 单体。server 独占约四分之三行数，其中 `routes/proxy.ts`、`services/router.ts`、`db/migrations/legacy_baseline.ts`、`lib/fallback-loop.ts`、`services/ratelimit.ts` 是最重的实现文件。client 的重量集中在 `PlaygroundPage.tsx` 与 `AnalyticsPage.tsx`。cli 虽小但是独立 npm 包 `freellmapi`，要单独版本与发布。

### 1.5 构建产物不计入源码

下列出现在磁盘或 CI 中但不计入上文：`node_modules/`、`dist/`、`build/`、`client/dist`、Electron 打包目录、SQLite `*.db`、`.encryption-key`。阅读源码请认 `src/` 与 `shared/types.ts`。

### 1.6 包版本锚点（来自 package.json）

- 根私有包 `@freellmapi/monorepo`
- `@freellmapi/server` version 字段 0.2.1（内部）
- 公开 CLI `freellmapi` 0.6.0
- 桌面 `freellmapi-desktop` 0.11.0
- client 0.0.0（随 server 分发，不单独发 npm）

版本号跨包不同步是有意的：CLI 与桌面有独立发布节奏，server 随 Docker `:latest` 走。

### 1.7 源码阅读顺序建议

1. `shared/types.ts` — 领域词汇
2. `server/src/index.ts` 与 `app.ts` — 进程与 HTTP 装配
3. `server/src/routes/proxy.ts` + `services/router.ts` + `lib/fallback-loop.ts` — 数据面
4. `server/src/services/scoring.ts` + `ratelimit.ts` — 决策与账本
5. `server/src/providers/index.ts` + `base.ts` + `openai-compat.ts` — 上游
6. `server/src/db/index.ts` + `migrations/20260101_000000_legacy_baseline.ts` — 状态
7. `client/src/App.tsx` + `pages/KeysPage.tsx` — 控制面
8. `cli/src/index.ts` + `tools.ts` — 智能体接入
9. `desktop/src/main.ts` + `server-host.ts` — 桌面

### 1.8 编码风格与不变量（从源码体现）

- ESM only，相对导入带 `.js` 扩展名。
- 密钥路径必须经 `encrypt`/`decrypt`，测试也不走捷径。
- 错误信息经 `sanitizeProviderErrorMessage` 与日志红线。
- 时间相关函数接受 `now` 参数便于测。
- SQLite 测试串行。
- 注释大量引用 GitHub issue 号（#1230、#1222、#899…），issue 即设计讨论档案。

### 1.9 与文档树的关系

`docs/en` 与 `docs/zh-cn` 是用户文档，不是生成代码。本 SOURCE.md 描述的是可执行源。OpenAPI 则由 `server/src/docs/openapi.ts` 内嵌，运行时在 `/v1/docs` 提供。

---

## 2. 各包结构鸟瞰

### 2.1 server/src

```
server/src/
  index.ts          进程入口
  app.ts            Express 装配
  env.ts            dotenv
  routes/           HTTP 路由（数据面+控制面）
  services/         领域服务（路由、限流、Fusion、压缩…）
  providers/        上游适配器
  lib/              无状态算法与基础设施
  db/               连接、迁移、类型
  middleware/       鉴权与限流
  docs/             OpenAPI
  scripts/          目录导出、密钥轮换、探测
  __tests__/        与上面镜像的测试树
```

### 2.2 client/src

```
client/src/
  main.tsx          Vite 入口
  App.tsx           路由与壳
  pages/            一页一文件
  components/       含 keys/、playground/、ui/
  lib/              api client 与纯函数
  hooks/            如 use-premium
  i18n/             约 60 份 locale JSON
  theme             明暗主题
```

locale JSON 未计入本节 TS 行数，但属于产品源。改文案要跑 `npm run check:i18n -w client`。

### 2.3 cli/src

`index.ts` 命令分发，`tools.ts` 各智能体生成器，`config-files.ts` 安全写入，`models.ts` 拉目录，`doctor.ts` 诊断，`types.ts` 类型。`tools.json` 随包发布。

### 2.4 desktop/src

`main.ts`、`server-host.ts`、`tray.ts`、`window.ts`、`popover.ts`、`preload.ts`、`logger.ts`、`config.ts`、`i18n.ts`、`stats.ts`。`scripts/*.mjs` 负责 bundle server、stage client、图标、mac 更新元数据、linux sandbox。

### 2.5 shared

单一 `types.ts`，被 server 与 client 引用。禁止在此文件 import server 内部模块，以免仪表盘打进 Node 内置。

---

## 3. 源代码文件清单

下表列出扫描到的每一个 TS/JS 文件：路径、所在目录、行数、识别到的函数/类数量、主要功能简介。测试文件在简介中标明其锁定的契约。

共 **719** 条。按路径排序。

| # | 文件名 | 所在目录 | 行数 | 函数/类 | 主要功能简介 |
|---|---|---|---:|---:|---|
| 1 | `eslint.config.js` | `cli` | 28 | 0 | 该文件承担工程源文件职责。 这是一份实现文件，约 28 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 2 | `config-files.test.ts` | `cli/src` | 472 | 0 | 该文件承担编码智能体配置 CLI职责。 这是一份测试文件，约 472 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 3 | `config-files.ts` | `cli/src` | 390 | 25 | 该文件承担编码智能体配置 CLI职责。 这是一份实现文件，约 390 行，识别到 25 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 4 | `doctor.test.ts` | `cli/src` | 393 | 4 | 该文件承担编码智能体配置 CLI职责。 这是一份测试文件，约 393 行，识别到 4 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 5 | `doctor.ts` | `cli/src` | 508 | 14 | 该文件承担编码智能体配置 CLI职责。 这是一份实现文件，约 508 行，识别到 14 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 6 | `index.test.ts` | `cli/src` | 198 | 0 | 该文件承担编码智能体配置 CLI职责。 这是一份测试文件，约 198 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 7 | `index.ts` | `cli/src` | 475 | 18 | 该文件承担编码智能体配置 CLI职责。 这是一份实现文件，约 475 行，识别到 18 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 8 | `models.test.ts` | `cli/src` | 81 | 0 | 该文件承担编码智能体配置 CLI职责。 这是一份测试文件，约 81 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 9 | `models.ts` | `cli/src` | 81 | 4 | 该文件承担编码智能体配置 CLI职责。 这是一份实现文件，约 81 行，识别到 4 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 10 | `tools.test.ts` | `cli/src` | 508 | 0 | 该文件承担编码智能体配置 CLI职责。 这是一份测试文件，约 508 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 11 | `tools.ts` | `cli/src` | 943 | 27 | 为 Claude Code、Codex、Cline、Aider 等十几个智能体生成配置文件。 这是一份实现文件，约 943 行，识别到 27 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 12 | `types.ts` | `cli/src` | 45 | 0 | 该文件承担编码智能体配置 CLI职责。 这是一份实现文件，约 45 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 13 | `mockApi.ts` | `client/dev` | 152 | 5 | 该文件承担工程源文件职责。 这是一份实现文件，约 152 行，识别到 5 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 14 | `eslint.config.js` | `client` | 23 | 0 | 该文件承担工程源文件职责。 这是一份实现文件，约 23 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 15 | `check-i18n.mjs` | `client/scripts` | 88 | 2 | 该文件承担工程源文件职责。 这是一份实现文件，约 88 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 16 | `App.test.ts` | `client/src` | 38 | 0 | 该文件承担工程源文件职责。 这是一份测试文件，约 38 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 17 | `App.tsx` | `client/src` | 492 | 8 | 仪表盘壳：路由、导航、鉴权门、命令面板、主题与 React Query。 这是一份实现文件，约 492 行，识别到 8 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 18 | `agent-icons.tsx` | `client/src/components` | 339 | 3 | 该文件承担仪表盘共享组件职责。 这是一份实现文件，约 339 行，识别到 3 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 19 | `api-usage.tsx` | `client/src/components` | 26 | 2 | 该文件承担仪表盘共享组件职责。 这是一份实现文件，约 26 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 20 | `auth-gate.desktop.test.tsx` | `client/src/components` | 182 | 3 | 该文件承担仪表盘共享组件职责。 这是一份测试文件，约 182 行，识别到 3 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 21 | `auth-gate.login-identifier.test.tsx` | `client/src/components` | 121 | 5 | 该文件承担仪表盘共享组件职责。 这是一份测试文件，约 121 行，识别到 5 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 22 | `auth-gate.tsx` | `client/src/components` | 525 | 6 | 该文件承担仪表盘共享组件职责。 这是一份实现文件，约 525 行，识别到 6 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 23 | `chain-manager.test.ts` | `client/src/components` | 103 | 2 | 该文件承担仪表盘共享组件职责。 这是一份测试文件，约 103 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 24 | `chain-manager.tsx` | `client/src/components` | 254 | 2 | 该文件承担仪表盘共享组件职责。 这是一份实现文件，约 254 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 25 | `code-block.tsx` | `client/src/components` | 24 | 1 | 该文件承担仪表盘共享组件职责。 这是一份实现文件，约 24 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 26 | `command-palette-state.ts` | `client/src/components` | 5 | 1 | 该文件承担仪表盘共享组件职责。 这是一份实现文件，约 5 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 27 | `command-palette.tsx` | `client/src/components` | 292 | 2 | 该文件承担仪表盘共享组件职责。 这是一份实现文件，约 292 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 28 | `confirm-button.tsx` | `client/src/components` | 81 | 1 | 该文件承担仪表盘共享组件职责。 这是一份实现文件，约 81 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 29 | `copy-button.tsx` | `client/src/components` | 44 | 1 | 该文件承担仪表盘共享组件职责。 这是一份实现文件，约 44 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 30 | `custom-weights-popover.tsx` | `client/src/components` | 114 | 1 | 该文件承担仪表盘共享组件职责。 这是一份实现文件，约 114 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 31 | `empty-state.tsx` | `client/src/components` | 28 | 1 | 该文件承担仪表盘共享组件职责。 这是一份实现文件，约 28 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 32 | `error-boundary.tsx` | `client/src/components` | 56 | 2 | 该文件承担仪表盘共享组件职责。 这是一份实现文件，约 56 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 33 | `floating-bar.tsx` | `client/src/components` | 30 | 1 | 该文件承担仪表盘共享组件职责。 这是一份实现文件，约 30 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 34 | `getting-started.tsx` | `client/src/components` | 178 | 4 | 该文件承担仪表盘共享组件职责。 这是一份实现文件，约 178 行，识别到 4 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 35 | `add-endpoint-key-dialog.tsx` | `client/src/components/keys` | 102 | 1 | 该文件承担密钥管理界面组件职责。 这是一份实现文件，约 102 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 36 | `add-key-dialog.tsx` | `client/src/components/keys` | 92 | 1 | 该文件承担密钥管理界面组件职责。 这是一份实现文件，约 92 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 37 | `add-key-form.tsx` | `client/src/components/keys` | 290 | 1 | 该文件承担密钥管理界面组件职责。 这是一份实现文件，约 290 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 38 | `add-model-dialog.test.tsx` | `client/src/components/keys` | 67 | 5 | 该文件承担密钥管理界面组件职责。 这是一份测试文件，约 67 行，识别到 5 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 39 | `add-model-dialog.tsx` | `client/src/components/keys` | 344 | 2 | 该文件承担密钥管理界面组件职责。 这是一份实现文件，约 344 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 40 | `agent-compatibility-section.tsx` | `client/src/components/keys` | 254 | 1 | 该文件承担密钥管理界面组件职责。 这是一份实现文件，约 254 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 41 | `anthropic-section.tsx` | `client/src/components/keys` | 101 | 1 | 该文件承担密钥管理界面组件职责。 这是一份实现文件，约 101 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 42 | `backups-section.tsx` | `client/src/components/keys` | 367 | 5 | 该文件承担密钥管理界面组件职责。 这是一份实现文件，约 367 行，识别到 5 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 43 | `client-profiles-section.tsx` | `client/src/components/keys` | 194 | 1 | 该文件承担密钥管理界面组件职责。 这是一份实现文件，约 194 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 44 | `copy-key-dialog.tsx` | `client/src/components/keys` | 120 | 1 | 该文件承担密钥管理界面组件职责。 这是一份实现文件，约 120 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 45 | `custom-provider-section.tsx` | `client/src/components/keys` | 272 | 2 | 该文件承担密钥管理界面组件职责。 这是一份实现文件，约 272 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 46 | `discover-models-dialog.tsx` | `client/src/components/keys` | 233 | 1 | 该文件承担密钥管理界面组件职责。 这是一份实现文件，约 233 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 47 | `edit-key-dialog.test.tsx` | `client/src/components/keys` | 91 | 4 | 该文件承担密钥管理界面组件职责。 这是一份测试文件，约 91 行，识别到 4 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 48 | `edit-key-dialog.tsx` | `client/src/components/keys` | 164 | 1 | 该文件承担密钥管理界面组件职责。 这是一份实现文件，约 164 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 49 | `edit-models-dialog.tsx` | `client/src/components/keys` | 200 | 1 | 该文件承担密钥管理界面组件职责。 这是一份实现文件，约 200 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 50 | `export-keys-dialog.tsx` | `client/src/components/keys` | 213 | 2 | 该文件承担密钥管理界面组件职责。 这是一份实现文件，约 213 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 51 | `import-keys-section.tsx` | `client/src/components/keys` | 285 | 1 | 该文件承担密钥管理界面组件职责。 这是一份实现文件，约 285 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 52 | `model-scope-dialog.tsx` | `client/src/components/keys` | 142 | 1 | 该文件承担密钥管理界面组件职责。 这是一份实现文件，约 142 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 53 | `model-select-dialog.tsx` | `client/src/components/keys` | 162 | 1 | 该文件承担密钥管理界面组件职责。 这是一份实现文件，约 162 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 54 | `provider-checklist-section.tsx` | `client/src/components/keys` | 102 | 1 | 该文件承担密钥管理界面组件职责。 这是一份实现文件，约 102 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 55 | `provider-list.tsx` | `client/src/components/keys` | 838 | 1 | 该文件承担密钥管理界面组件职责。 这是一份实现文件，约 838 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 56 | `proxy-settings-section.tsx` | `client/src/components/keys` | 257 | 2 | 该文件承担密钥管理界面组件职责。 这是一份实现文件，约 257 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 57 | `quota-outlook-section.tsx` | `client/src/components/keys` | 101 | 2 | 该文件承担密钥管理界面组件职责。 这是一份实现文件，约 101 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 58 | `quota-signals-section.test.tsx` | `client/src/components/keys` | 124 | 3 | 该文件承担密钥管理界面组件职责。 这是一份测试文件，约 124 行，识别到 3 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 59 | `quota-signals-section.tsx` | `client/src/components/keys` | 61 | 3 | 该文件承担密钥管理界面组件职责。 这是一份实现文件，约 61 行，识别到 3 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 60 | `shared.tsx` | `client/src/components/keys` | 148 | 3 | 该文件承担密钥管理界面组件职责。 这是一份实现文件，约 148 行，识别到 3 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 61 | `test-models-dialog.test.tsx` | `client/src/components/keys` | 125 | 1 | 该文件承担密钥管理界面组件职责。 这是一份测试文件，约 125 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 62 | `test-models-dialog.tsx` | `client/src/components/keys` | 133 | 1 | 该文件承担密钥管理界面组件职责。 这是一份实现文件，约 133 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 63 | `unified-key-section.tsx` | `client/src/components/keys` | 112 | 1 | 该文件承担密钥管理界面组件职责。 这是一份实现文件，约 112 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 64 | `markdown.tsx` | `client/src/components` | 182 | 5 | 该文件承担仪表盘共享组件职责。 这是一份实现文件，约 182 行，识别到 5 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 65 | `media-models.tsx` | `client/src/components` | 278 | 3 | 该文件承担仪表盘共享组件职责。 这是一份实现文件，约 278 行，识别到 3 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 66 | `model-combobox.tsx` | `client/src/components` | 164 | 1 | 该文件承担仪表盘共享组件职责。 这是一份实现文件，约 164 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 67 | `model-maker-icon.tsx` | `client/src/components` | 52 | 1 | 该文件承担仪表盘共享组件职责。 这是一份实现文件，约 52 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 68 | `model-table.tsx` | `client/src/components` | 420 | 9 | 该文件承担仪表盘共享组件职责。 这是一份实现文件，约 420 行，识别到 9 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 69 | `models-tabs.tsx` | `client/src/components` | 25 | 1 | 该文件承担仪表盘共享组件职责。 这是一份实现文件，约 25 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 70 | `page-header.tsx` | `client/src/components` | 25 | 1 | 该文件承担仪表盘共享组件职责。 这是一份实现文件，约 25 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 71 | `peak-hours-controls.tsx` | `client/src/components` | 126 | 3 | 该文件承担仪表盘共享组件职责。 这是一份实现文件，约 126 行，识别到 3 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 72 | `penalty-inspector.tsx` | `client/src/components` | 251 | 5 | 该文件承担仪表盘共享组件职责。 这是一份实现文件，约 251 行，识别到 5 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 73 | `artifact-panel.tsx` | `client/src/components/playground` | 185 | 1 | 该文件承担试验台界面组件职责。 这是一份实现文件，约 185 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 74 | `conversation-sidebar.tsx` | `client/src/components/playground` | 230 | 1 | 该文件承担试验台界面组件职责。 这是一份实现文件，约 230 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 75 | `dictation-button.tsx` | `client/src/components/playground` | 137 | 1 | 该文件承担试验台界面组件职责。 这是一份实现文件，约 137 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 76 | `dictation-wave.tsx` | `client/src/components/playground` | 66 | 1 | 该文件承担试验台界面组件职责。 这是一份实现文件，约 66 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 77 | `liquid-composer.tsx` | `client/src/components/playground` | 162 | 1 | 该文件承担试验台界面组件职责。 这是一份实现文件，约 162 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 78 | `settings-rail.tsx` | `client/src/components/playground` | 277 | 2 | 该文件承担试验台界面组件职责。 这是一份实现文件，约 277 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 79 | `settings-dialog.test.ts` | `client/src/components` | 52 | 1 | 该文件承担仪表盘共享组件职责。 这是一份测试文件，约 52 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 80 | `settings-dialog.tsx` | `client/src/components` | 1090 | 13 | 该文件承担仪表盘共享组件职责。 这是一份实现文件，约 1090 行，识别到 13 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 81 | `sortable-header.tsx` | `client/src/components` | 37 | 1 | 该文件承担仪表盘共享组件职责。 这是一份实现文件，约 37 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 82 | `toaster.tsx` | `client/src/components` | 83 | 2 | 该文件承担仪表盘共享组件职责。 这是一份实现文件，约 83 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 83 | `token-usage-bar.tsx` | `client/src/components` | 129 | 1 | 该文件承担仪表盘共享组件职责。 这是一份实现文件，约 129 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 84 | `tooltip.tsx` | `client/src/components` | 91 | 1 | 该文件承担仪表盘共享组件职责。 这是一份实现文件，约 91 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 85 | `badge.tsx` | `client/src/components/ui` | 52 | 1 | 该文件承担基础 UI 原语职责。 这是一份实现文件，约 52 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 86 | `button.tsx` | `client/src/components/ui` | 58 | 1 | 该文件承担基础 UI 原语职责。 这是一份实现文件，约 58 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 87 | `card.tsx` | `client/src/components/ui` | 103 | 7 | 该文件承担基础 UI 原语职责。 这是一份实现文件，约 103 行，识别到 7 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 88 | `dialog.tsx` | `client/src/components/ui` | 71 | 6 | 该文件承担基础 UI 原语职责。 这是一份实现文件，约 71 行，识别到 6 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 89 | `dropdown-menu.tsx` | `client/src/components/ui` | 266 | 15 | 该文件承担基础 UI 原语职责。 这是一份实现文件，约 266 行，识别到 15 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 90 | `field-error.tsx` | `client/src/components/ui` | 13 | 1 | 该文件承担基础 UI 原语职责。 这是一份实现文件，约 13 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 91 | `input.tsx` | `client/src/components/ui` | 20 | 1 | 该文件承担基础 UI 原语职责。 这是一份实现文件，约 20 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 92 | `label.tsx` | `client/src/components/ui` | 18 | 1 | 该文件承担基础 UI 原语职责。 这是一份实现文件，约 18 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 93 | `popover.tsx` | `client/src/components/ui` | 43 | 3 | 该文件承担基础 UI 原语职责。 这是一份实现文件，约 43 行，识别到 3 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 94 | `segmented-control.tsx` | `client/src/components/ui` | 44 | 1 | 该文件承担基础 UI 原语职责。 这是一份实现文件，约 44 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 95 | `select.tsx` | `client/src/components/ui` | 201 | 9 | 该文件承担基础 UI 原语职责。 这是一份实现文件，约 201 行，识别到 9 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 96 | `separator.tsx` | `client/src/components/ui` | 23 | 1 | 该文件承担基础 UI 原语职责。 这是一份实现文件，约 23 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 97 | `skeleton.tsx` | `client/src/components/ui` | 47 | 3 | 该文件承担基础 UI 原语职责。 这是一份实现文件，约 47 行，识别到 3 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 98 | `switch.tsx` | `client/src/components/ui` | 30 | 1 | 该文件承担基础 UI 原语职责。 这是一份实现文件，约 30 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 99 | `table.tsx` | `client/src/components/ui` | 116 | 8 | 该文件承担基础 UI 原语职责。 这是一份实现文件，约 116 行，识别到 8 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 100 | `textarea.tsx` | `client/src/components/ui` | 18 | 1 | 该文件承担基础 UI 原语职责。 这是一份实现文件，约 18 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 101 | `update-reminder.tsx` | `client/src/components` | 277 | 4 | 该文件承担仪表盘共享组件职责。 这是一份实现文件，约 277 行，识别到 4 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 102 | `usage-summary-card.tsx` | `client/src/components` | 96 | 1 | 该文件承担仪表盘共享组件职责。 这是一份实现文件，约 96 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 103 | `use-premium.ts` | `client/src/hooks` | 40 | 1 | 该文件承担React 钩子职责。 这是一份实现文件，约 40 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 104 | `I18nProvider.tsx` | `client/src/i18n` | 165 | 5 | 该文件承担国际化职责。 这是一份实现文件，约 165 行，识别到 5 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 105 | `context.ts` | `client/src/i18n` | 22 | 1 | 该文件承担国际化职责。 这是一份实现文件，约 22 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 106 | `index.ts` | `client/src/i18n` | 10 | 0 | 该文件承担国际化职责。 这是一份实现文件，约 10 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 107 | `locale-config.ts` | `client/src/i18n` | 66 | 0 | 该文件承担国际化职责。 这是一份实现文件，约 66 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 108 | `alias-merge.test.ts` | `client/src/lib` | 81 | 0 | 该文件承担前端工具与业务辅助职责。 这是一份测试文件，约 81 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 109 | `alias-merge.ts` | `client/src/lib` | 52 | 4 | 该文件承担前端工具与业务辅助职责。 这是一份实现文件，约 52 行，识别到 4 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 110 | `api.test.ts` | `client/src/lib` | 79 | 2 | 该文件承担前端工具与业务辅助职责。 这是一份测试文件，约 79 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 111 | `api.ts` | `client/src/lib` | 84 | 5 | 该文件承担前端工具与业务辅助职责。 这是一份实现文件，约 84 行，识别到 5 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 112 | `artifact-host.ts` | `client/src/lib` | 13 | 0 | 该文件承担前端工具与业务辅助职责。 这是一份实现文件，约 13 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 113 | `artifact-panel-size.test.ts` | `client/src/lib` | 44 | 0 | 该文件承担前端工具与业务辅助职责。 这是一份测试文件，约 44 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 114 | `artifact-panel-size.ts` | `client/src/lib` | 40 | 5 | 该文件承担前端工具与业务辅助职责。 这是一份实现文件，约 40 行，识别到 5 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 115 | `artifacts.test.ts` | `client/src/lib` | 62 | 0 | 该文件承担前端工具与业务辅助职责。 这是一份测试文件，约 62 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 116 | `artifacts.ts` | `client/src/lib` | 80 | 5 | 该文件承担前端工具与业务辅助职责。 这是一份实现文件，约 80 行，识别到 5 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 117 | `attachments.test.ts` | `client/src/lib` | 123 | 0 | 该文件承担前端工具与业务辅助职责。 这是一份测试文件，约 123 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 118 | `attachments.ts` | `client/src/lib` | 200 | 14 | 该文件承担前端工具与业务辅助职责。 这是一份实现文件，约 200 行，识别到 14 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 119 | `budget-legend.test.ts` | `client/src/lib` | 76 | 1 | 该文件承担前端工具与业务辅助职责。 这是一份测试文件，约 76 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 120 | `budget-legend.ts` | `client/src/lib` | 59 | 2 | 该文件承担前端工具与业务辅助职责。 这是一份实现文件，约 59 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 121 | `chart-axis.render.test.tsx` | `client/src/lib` | 162 | 1 | 该文件承担前端工具与业务辅助职责。 这是一份测试文件，约 162 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 122 | `chart-axis.test.ts` | `client/src/lib` | 131 | 0 | 该文件承担前端工具与业务辅助职责。 这是一份测试文件，约 131 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 123 | `chart-axis.ts` | `client/src/lib` | 150 | 6 | 该文件承担前端工具与业务辅助职责。 这是一份实现文件，约 150 行，识别到 6 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 124 | `clipboard.test.ts` | `client/src/lib` | 83 | 1 | 该文件承担前端工具与业务辅助职责。 这是一份测试文件，约 83 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 125 | `clipboard.ts` | `client/src/lib` | 55 | 2 | 该文件承担前端工具与业务辅助职责。 这是一份实现文件，约 55 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 126 | `composer-geometry.test.ts` | `client/src/lib` | 43 | 0 | 该文件承担前端工具与业务辅助职责。 这是一份测试文件，约 43 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 127 | `composer-geometry.ts` | `client/src/lib` | 44 | 4 | 该文件承担前端工具与业务辅助职责。 这是一份实现文件，约 44 行，识别到 4 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 128 | `fusion-filter.test.ts` | `client/src/lib` | 56 | 0 | 该文件承担前端工具与业务辅助职责。 这是一份测试文件，约 56 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 129 | `fusion-filter.ts` | `client/src/lib` | 38 | 2 | 该文件承担前端工具与业务辅助职责。 这是一份实现文件，约 38 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 130 | `highlight.test.ts` | `client/src/lib` | 50 | 0 | 该文件承担前端工具与业务辅助职责。 这是一份测试文件，约 50 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 131 | `highlight.ts` | `client/src/lib` | 83 | 3 | 该文件承担前端工具与业务辅助职责。 这是一份实现文件，约 83 行，识别到 3 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 132 | `logs.test.ts` | `client/src/lib` | 178 | 1 | 该文件承担前端工具与业务辅助职责。 这是一份测试文件，约 178 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 133 | `logs.ts` | `client/src/lib` | 188 | 10 | 该文件承担前端工具与业务辅助职责。 这是一份实现文件，约 188 行，识别到 10 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 134 | `model-groups.ts` | `client/src/lib` | 74 | 2 | 该文件承担前端工具与业务辅助职责。 这是一份实现文件，约 74 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 135 | `model-maker.test.ts` | `client/src/lib` | 52 | 0 | 该文件承担前端工具与业务辅助职责。 这是一份测试文件，约 52 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 136 | `model-maker.ts` | `client/src/lib` | 92 | 1 | 该文件承担前端工具与业务辅助职责。 这是一份实现文件，约 92 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 137 | `model-scope-selection.test.ts` | `client/src/lib` | 140 | 2 | 该文件承担前端工具与业务辅助职责。 这是一份测试文件，约 140 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 138 | `model-scope-selection.ts` | `client/src/lib` | 102 | 3 | 该文件承担前端工具与业务辅助职责。 这是一份实现文件，约 102 行，识别到 3 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 139 | `model-settings.test.ts` | `client/src/lib` | 95 | 0 | 该文件承担前端工具与业务辅助职责。 这是一份测试文件，约 95 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 140 | `model-settings.ts` | `client/src/lib` | 180 | 6 | 该文件承担前端工具与业务辅助职责。 这是一份实现文件，约 180 行，识别到 6 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 141 | `playground-conversations.test.ts` | `client/src/lib` | 175 | 1 | 该文件承担前端工具与业务辅助职责。 这是一份测试文件，约 175 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 142 | `playground-conversations.ts` | `client/src/lib` | 194 | 10 | 该文件承担前端工具与业务辅助职责。 这是一份实现文件，约 194 行，识别到 10 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 143 | `playground-sampling.test.ts` | `client/src/lib` | 150 | 0 | 该文件承担前端工具与业务辅助职责。 这是一份测试文件，约 150 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 144 | `playground-sampling.ts` | `client/src/lib` | 206 | 13 | 该文件承担前端工具与业务辅助职责。 这是一份实现文件，约 206 行，识别到 13 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 145 | `playground-stream.test.ts` | `client/src/lib` | 224 | 1 | 该文件承担前端工具与业务辅助职责。 这是一份测试文件，约 224 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 146 | `playground-stream.ts` | `client/src/lib` | 187 | 2 | 该文件承担前端工具与业务辅助职责。 这是一份实现文件，约 187 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 147 | `routing.test.ts` | `client/src/lib` | 403 | 1 | 该文件承担前端工具与业务辅助职责。 这是一份测试文件，约 403 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 148 | `routing.ts` | `client/src/lib` | 531 | 21 | 该文件承担前端工具与业务辅助职责。 这是一份实现文件，约 531 行，识别到 21 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 149 | `table-sort.ts` | `client/src/lib` | 79 | 6 | 该文件承担前端工具与业务辅助职责。 这是一份实现文件，约 79 行，识别到 6 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 150 | `test-models.ts` | `client/src/lib` | 19 | 1 | 该文件承担前端工具与业务辅助职责。 这是一份实现文件，约 19 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 151 | `toast-timer.test.ts` | `client/src/lib` | 73 | 0 | 该文件承担前端工具与业务辅助职责。 这是一份测试文件，约 73 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 152 | `toast-timer.ts` | `client/src/lib` | 62 | 1 | 该文件承担前端工具与业务辅助职责。 这是一份实现文件，约 62 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 153 | `toast.test.ts` | `client/src/lib` | 91 | 1 | 该文件承担前端工具与业务辅助职责。 这是一份测试文件，约 91 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 154 | `toast.ts` | `client/src/lib` | 82 | 5 | 该文件承担前端工具与业务辅助职责。 这是一份实现文件，约 82 行，识别到 5 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 155 | `transcription.test.ts` | `client/src/lib` | 75 | 1 | 该文件承担前端工具与业务辅助职责。 这是一份测试文件，约 75 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 156 | `transcription.ts` | `client/src/lib` | 80 | 7 | 该文件承担前端工具与业务辅助职责。 这是一份实现文件，约 80 行，识别到 7 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 157 | `utils.ts` | `client/src/lib` | 29 | 3 | 该文件承担前端工具与业务辅助职责。 这是一份实现文件，约 29 行，识别到 3 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 158 | `validate.ts` | `client/src/lib` | 16 | 2 | 该文件承担前端工具与业务辅助职责。 这是一份实现文件，约 16 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 159 | `waveform.test.ts` | `client/src/lib` | 33 | 0 | 该文件承担前端工具与业务辅助职责。 这是一份测试文件，约 33 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 160 | `waveform.ts` | `client/src/lib` | 41 | 1 | 该文件承担前端工具与业务辅助职责。 这是一份实现文件，约 41 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 161 | `main.tsx` | `client/src` | 10 | 0 | 该文件承担工程源文件职责。 这是一份实现文件，约 10 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 162 | `AgentsPage.tsx` | `client/src/pages` | 185 | 1 | 该文件承担仪表盘页面职责。 这是一份实现文件，约 185 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 163 | `AnalyticsPage.tsx` | `client/src/pages` | 1188 | 12 | 该文件承担仪表盘页面职责。 这是一份实现文件，约 1188 行，识别到 12 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 164 | `AudioPage.tsx` | `client/src/pages` | 5 | 0 | 该文件承担仪表盘页面职责。 这是一份实现文件，约 5 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 165 | `EmbeddingDetailPage.tsx` | `client/src/pages` | 103 | 0 | 该文件承担仪表盘页面职责。 这是一份实现文件，约 103 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 166 | `EmbeddingsPage.tsx` | `client/src/pages` | 363 | 2 | 该文件承担仪表盘页面职责。 这是一份实现文件，约 363 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 167 | `FallbackPage.tsx` | `client/src/pages` | 650 | 1 | 该文件承担仪表盘页面职责。 这是一份实现文件，约 650 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 168 | `FusionPage.tsx` | `client/src/pages` | 333 | 0 | 该文件承担仪表盘页面职责。 这是一份实现文件，约 333 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 169 | `ImagePage.tsx` | `client/src/pages` | 5 | 0 | 该文件承担仪表盘页面职责。 这是一份实现文件，约 5 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 170 | `KeysPage.tsx` | `client/src/pages` | 133 | 0 | 该文件承担仪表盘页面职责。 这是一份实现文件，约 133 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 171 | `LogsPage.tsx` | `client/src/pages` | 432 | 2 | 该文件承担仪表盘页面职责。 这是一份实现文件，约 432 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 172 | `MediaDetailPage.tsx` | `client/src/pages` | 124 | 0 | 该文件承担仪表盘页面职责。 这是一份实现文件，约 124 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 173 | `ModelDetailPage.tsx` | `client/src/pages` | 644 | 3 | 该文件承担仪表盘页面职责。 这是一份实现文件，约 644 行，识别到 3 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 174 | `NotFoundPage.tsx` | `client/src/pages` | 24 | 0 | 该文件承担仪表盘页面职责。 这是一份实现文件，约 24 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 175 | `PlaygroundPage.tsx` | `client/src/pages` | 1235 | 4 | 该文件承担仪表盘页面职责。 这是一份实现文件，约 1235 行，识别到 4 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 176 | `PremiumPage.tsx` | `client/src/pages` | 248 | 2 | 该文件承担仪表盘页面职责。 这是一份实现文件，约 248 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 177 | `VideoPage.tsx` | `client/src/pages` | 5 | 0 | 该文件承担仪表盘页面职责。 这是一份实现文件，约 5 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 178 | `agent-descriptions.test.ts` | `client/src/pages` | 52 | 1 | 该文件承担仪表盘页面职责。 这是一份测试文件，约 52 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 179 | `capability-tier-hint.test.ts` | `client/src/pages` | 79 | 1 | 该文件承担仪表盘页面职责。 这是一份测试文件，约 79 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 180 | `model-limit-hints.test.ts` | `client/src/pages` | 51 | 1 | 该文件承担仪表盘页面职责。 这是一份测试文件，约 51 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 181 | `routing-more-options.test.ts` | `client/src/pages` | 65 | 0 | 该文件承担仪表盘页面职责。 这是一份测试文件，约 65 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 182 | `theme-context.ts` | `client/src` | 19 | 1 | 该文件承担工程源文件职责。 这是一份实现文件，约 19 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 183 | `theme.tsx` | `client/src` | 56 | 3 | 该文件承担工程源文件职责。 这是一份实现文件，约 56 行，识别到 3 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 184 | `vite-env.d.ts` | `client/src` | 3 | 0 | 该文件承担工程源文件职责。 这是一份实现文件，约 3 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 185 | `vite.config.ts` | `client` | 31 | 0 | 该文件承担工程源文件职责。 这是一份实现文件，约 31 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 186 | `vitest.config.ts` | `client` | 42 | 0 | 该文件承担工程源文件职责。 这是一份实现文件，约 42 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 187 | `after-pack-linux-sandbox.mjs` | `desktop/scripts` | 21 | 1 | 该文件承担工程源文件职责。 这是一份实现文件，约 21 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 188 | `bundle-server.mjs` | `desktop/scripts` | 31 | 0 | 该文件承担工程源文件职责。 这是一份实现文件，约 31 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 189 | `gen-icons.mjs` | `desktop/scripts` | 155 | 5 | 该文件承担工程源文件职责。 这是一份实现文件，约 155 行，识别到 5 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 190 | `merge-mac-update-metadata.mjs` | `desktop/scripts` | 60 | 1 | 该文件承担工程源文件职责。 这是一份实现文件，约 60 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 191 | `refresh-mac-update-metadata.mjs` | `desktop/scripts` | 110 | 4 | 该文件承担工程源文件职责。 这是一份实现文件，约 110 行，识别到 4 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 192 | `smoke-mac-package.mjs` | `desktop/scripts` | 71 | 0 | 该文件承担工程源文件职责。 这是一份实现文件，约 71 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 193 | `stage-client.mjs` | `desktop/scripts` | 25 | 0 | 该文件承担工程源文件职责。 这是一份实现文件，约 25 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 194 | `linux-chrome-sandbox-packaging.test.ts` | `desktop/src/__tests__` | 77 | 0 | 该文件承担Electron 桌面壳职责。 这是一份测试文件，约 77 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 195 | `logger.test.ts` | `desktop/src/__tests__` | 104 | 0 | 该文件承担Electron 桌面壳职责。 这是一份测试文件，约 104 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 196 | `merge-mac-update-metadata.test.ts` | `desktop/src/__tests__` | 110 | 0 | 该文件承担Electron 桌面壳职责。 这是一份测试文件，约 110 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 197 | `refresh-mac-update-metadata.test.ts` | `desktop/src/__tests__` | 108 | 0 | 该文件承担Electron 桌面壳职责。 这是一份测试文件，约 108 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 198 | `server-host-boot.test.ts` | `desktop/src/__tests__` | 209 | 1 | 该文件承担Electron 桌面壳职责。 这是一份测试文件，约 209 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 199 | `tray-visibility.test.ts` | `desktop/src/__tests__` | 34 | 0 | 该文件承担Electron 桌面壳职责。 这是一份测试文件，约 34 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 200 | `version.test.ts` | `desktop/src/__tests__` | 41 | 0 | 该文件承担Electron 桌面壳职责。 这是一份测试文件，约 41 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 201 | `window-chrome.test.ts` | `desktop/src/__tests__` | 31 | 0 | 该文件承担Electron 桌面壳职责。 这是一份测试文件，约 31 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 202 | `config.ts` | `desktop/src` | 43 | 3 | 该文件承担Electron 桌面壳职责。 这是一份实现文件，约 43 行，识别到 3 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 203 | `i18n.ts` | `desktop/src` | 178 | 3 | 该文件承担Electron 桌面壳职责。 这是一份实现文件，约 178 行，识别到 3 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 204 | `logger.ts` | `desktop/src` | 140 | 6 | 该文件承担Electron 桌面壳职责。 这是一份实现文件，约 140 行，识别到 6 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 205 | `main.ts` | `desktop/src` | 342 | 0 | Electron 主进程入口，拉起托盘、悬浮窗与内嵌 Node 服务器。 这是一份实现文件，约 342 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 206 | `popover.ts` | `desktop/src` | 100 | 3 | 该文件承担Electron 桌面壳职责。 这是一份实现文件，约 100 行，识别到 3 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 207 | `preload-popover.ts` | `desktop/src` | 13 | 0 | 该文件承担Electron 桌面壳职责。 这是一份实现文件，约 13 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 208 | `preload.ts` | `desktop/src` | 91 | 3 | 该文件承担Electron 桌面壳职责。 这是一份实现文件，约 91 行，识别到 3 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 209 | `server-host.ts` | `desktop/src` | 166 | 4 | 该文件承担Electron 桌面壳职责。 这是一份实现文件，约 166 行，识别到 4 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 210 | `stats.ts` | `desktop/src` | 71 | 4 | 该文件承担Electron 桌面壳职责。 这是一份实现文件，约 71 行，识别到 4 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 211 | `tray-visibility.ts` | `desktop/src` | 35 | 1 | 该文件承担Electron 桌面壳职责。 这是一份实现文件，约 35 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 212 | `tray.ts` | `desktop/src` | 66 | 2 | 该文件承担Electron 桌面壳职责。 这是一份实现文件，约 66 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 213 | `window-chrome.ts` | `desktop/src` | 46 | 1 | 该文件承担Electron 桌面壳职责。 这是一份实现文件，约 46 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 214 | `window.ts` | `desktop/src` | 47 | 2 | 该文件承担Electron 桌面壳职责。 这是一份实现文件，约 47 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 215 | `index.js` | `examples/fetch-relay-worker/src` | 159 | 7 | 该文件承担示例职责。 这是一份实现文件，约 159 行，识别到 7 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 216 | `dev-bootstrap.test.mjs` | `scripts` | 90 | 3 | 该文件承担仓库级脚本职责。 这是一份测试文件，约 90 行，识别到 3 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 217 | `eslint.config.js` | `server` | 33 | 0 | 该文件承担工程源文件职责。 这是一份实现文件，约 33 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 218 | `factory-injection.test.ts` | `server/src/__tests__/db` | 37 | 1 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 37 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 219 | `hardening.test.ts` | `server/src/__tests__/db` | 287 | 3 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 287 行，识别到 3 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 220 | `idempotency.test.ts` | `server/src/__tests__/db` | 313 | 0 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 313 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 221 | `init-db-encryption.test.ts` | `server/src/__tests__/db` | 59 | 1 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 59 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 222 | `init-db-options.test.ts` | `server/src/__tests__/db` | 42 | 0 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 42 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 223 | `intelligence-tiers.test.ts` | `server/src/__tests__/db` | 85 | 0 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 85 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 224 | `custom-endpoint-host-labels.test.ts` | `server/src/__tests__/db/migrate` | 72 | 2 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 72 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 225 | `custom-model-tombstones.test.ts` | `server/src/__tests__/db/migrate` | 63 | 1 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 63 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 226 | `custom-model-tool-support.test.ts` | `server/src/__tests__/db/migrate` | 73 | 2 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 73 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 227 | `endpoint-identity.test.ts` | `server/src/__tests__/db/migrate` | 223 | 3 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 223 行，识别到 3 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 228 | `model-source-provenance.test.ts` | `server/src/__tests__/db/migrate` | 133 | 4 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 133 行，识别到 4 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 229 | `registry-drift.test.ts` | `server/src/__tests__/db/migrate` | 30 | 0 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 30 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 230 | `roundtrip.test.ts` | `server/src/__tests__/db/migrate` | 294 | 11 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 294 行，识别到 11 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 231 | `runner.test.ts` | `server/src/__tests__/db/migrate` | 258 | 5 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 258 行，识别到 5 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 232 | `unified-key-output.test.ts` | `server/src/__tests__/db/migrate` | 118 | 3 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 118 行，识别到 3 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 233 | `node-sqlite.test.ts` | `server/src/__tests__/db` | 89 | 0 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 89 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 234 | `quota-snapshot-freshness.test.ts` | `server/src/__tests__/db` | 30 | 0 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 30 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 235 | `acl.ts` | `server/src/__tests__/helpers` | 26 | 1 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 26 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 236 | `auth.ts` | `server/src/__tests__/helpers` | 14 | 2 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 14 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 237 | `chain.ts` | `server/src/__tests__/helpers` | 22 | 1 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 22 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 238 | `full-flow.test.ts` | `server/src/__tests__/integration` | 186 | 2 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 186 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 239 | `anthropic-documents.test.ts` | `server/src/__tests__/lib` | 137 | 0 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 137 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 240 | `app-version.test.ts` | `server/src/__tests__/lib` | 114 | 0 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 114 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 241 | `budget.test.ts` | `server/src/__tests__/lib` | 29 | 0 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 29 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 242 | `client-classifier.test.ts` | `server/src/__tests__/lib` | 106 | 1 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 106 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 243 | `client-context.test.ts` | `server/src/__tests__/lib` | 70 | 2 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 70 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 244 | `config.test.ts` | `server/src/__tests__/lib` | 96 | 0 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 96 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 245 | `content.test.ts` | `server/src/__tests__/lib` | 272 | 0 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 272 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 246 | `crypto-init.test.ts` | `server/src/__tests__/lib` | 107 | 2 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 107 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 247 | `crypto-keyfile.test.ts` | `server/src/__tests__/lib` | 128 | 2 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 128 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 248 | `crypto.test.ts` | `server/src/__tests__/lib` | 74 | 0 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 74 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 249 | `db-backup.test.ts` | `server/src/__tests__/lib` | 306 | 1 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 306 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 250 | `degraded-error.test.ts` | `server/src/__tests__/lib` | 78 | 2 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 78 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 251 | `endpoint-scope.test.ts` | `server/src/__tests__/lib` | 54 | 0 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 54 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 252 | `env-drift.test.ts` | `server/src/__tests__/lib` | 103 | 0 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 103 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 253 | `error-classify-transport.test.ts` | `server/src/__tests__/lib` | 147 | 2 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 147 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 254 | `error-redaction.test.ts` | `server/src/__tests__/lib` | 66 | 0 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 66 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 255 | `exhaustion-statuses.test.ts` | `server/src/__tests__/lib` | 453 | 2 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 453 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 256 | `fallback-loop-account-suspended.test.ts` | `server/src/__tests__/lib` | 221 | 2 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 221 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 257 | `fallback-loop-client-abort.test.ts` | `server/src/__tests__/lib` | 169 | 2 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 169 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 258 | `fallback-loop-hedge.test.ts` | `server/src/__tests__/lib` | 281 | 3 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 281 行，识别到 3 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 259 | `fallback-loop-issue-1230.test.ts` | `server/src/__tests__/lib` | 202 | 3 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 202 行，识别到 3 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 260 | `fallback-loop-issue-1239.test.ts` | `server/src/__tests__/lib` | 319 | 2 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 319 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 261 | `fallback-loop-lease.test.ts` | `server/src/__tests__/lib` | 131 | 2 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 131 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 262 | `fallback-loop-model-bench.test.ts` | `server/src/__tests__/lib` | 157 | 4 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 157 行，识别到 4 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 263 | `fallback-loop-size-skip.test.ts` | `server/src/__tests__/lib` | 238 | 2 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 238 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 264 | `fallback-loop.test.ts` | `server/src/__tests__/lib` | 905 | 2 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 905 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 265 | `file-permissions.test.ts` | `server/src/__tests__/lib` | 296 | 3 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 296 行，识别到 3 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 266 | `gemini-schema-compat.test.ts` | `server/src/__tests__/lib` | 254 | 0 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 254 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 267 | `gemini-wire.test.ts` | `server/src/__tests__/lib` | 185 | 0 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 185 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 268 | `guardrails.test.ts` | `server/src/__tests__/lib` | 146 | 0 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 146 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 269 | `header-value.test.ts` | `server/src/__tests__/lib` | 77 | 0 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 77 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 270 | `image-normalize-no-sharp.test.ts` | `server/src/__tests__/lib` | 71 | 1 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 71 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 271 | `image-normalize.test.ts` | `server/src/__tests__/lib` | 227 | 4 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 227 行，识别到 4 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 272 | `key-parser-models.test.ts` | `server/src/__tests__/lib` | 127 | 0 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 127 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 273 | `key-parser.test.ts` | `server/src/__tests__/lib` | 225 | 0 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 225 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 274 | `log-redaction.test.ts` | `server/src/__tests__/lib` | 218 | 1 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 218 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 275 | `module-purity.test.ts` | `server/src/__tests__/lib` | 196 | 2 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 196 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 276 | `process-safety-net.test.ts` | `server/src/__tests__/lib` | 81 | 0 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 81 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 277 | `provider-identity.test.ts` | `server/src/__tests__/lib` | 102 | 0 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 102 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 278 | `provider-size-parser.test.ts` | `server/src/__tests__/lib` | 97 | 0 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 97 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 279 | `provider-timeout.test.ts` | `server/src/__tests__/lib` | 106 | 0 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 106 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 280 | `proxy-fetch-relay-integration.test.ts` | `server/src/__tests__/lib` | 131 | 2 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 131 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 281 | `proxy-fetch-relay.test.ts` | `server/src/__tests__/lib` | 232 | 0 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 232 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 282 | `proxy-restore.test.ts` | `server/src/__tests__/lib` | 305 | 1 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 305 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 283 | `proxy.test.ts` | `server/src/__tests__/lib` | 1101 | 3 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 1101 行，识别到 3 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 284 | `request-attempts.test.ts` | `server/src/__tests__/lib` | 260 | 3 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 260 行，识别到 3 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 285 | `request-caller.test.ts` | `server/src/__tests__/lib` | 49 | 1 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 49 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 286 | `sampling-params.test.ts` | `server/src/__tests__/lib` | 242 | 0 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 242 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 287 | `served-model.test.ts` | `server/src/__tests__/lib` | 99 | 0 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 99 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 288 | `structured-output.test.ts` | `server/src/__tests__/lib` | 70 | 0 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 70 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 289 | `system-prompt.test.ts` | `server/src/__tests__/lib` | 92 | 1 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 92 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 290 | `task-type.test.ts` | `server/src/__tests__/lib` | 115 | 1 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 115 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 291 | `think-tags.test.ts` | `server/src/__tests__/lib` | 189 | 2 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 189 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 292 | `tool-args.test.ts` | `server/src/__tests__/lib` | 238 | 0 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 238 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 293 | `tool-call-rescue.test.ts` | `server/src/__tests__/lib` | 170 | 0 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 170 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 294 | `tool-validate.test.ts` | `server/src/__tests__/lib` | 175 | 0 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 175 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 295 | `unified-max-tokens.test.ts` | `server/src/__tests__/lib` | 130 | 0 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 130 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 296 | `upstream-classification.test.ts` | `server/src/__tests__/lib` | 55 | 0 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 55 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 297 | `url-guard.test.ts` | `server/src/__tests__/lib` | 203 | 0 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 203 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 298 | `wake-detect.test.ts` | `server/src/__tests__/lib` | 107 | 0 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 107 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 299 | `abort-signal.test.ts` | `server/src/__tests__/providers` | 167 | 2 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 167 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 300 | `aihorde.test.ts` | `server/src/__tests__/providers` | 129 | 0 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 129 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 301 | `clod-speechify-blaze.test.ts` | `server/src/__tests__/providers` | 80 | 0 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 80 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 302 | `cloudflare.test.ts` | `server/src/__tests__/providers` | 165 | 0 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 165 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 303 | `cn-providers.test.ts` | `server/src/__tests__/providers` | 69 | 0 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 69 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 304 | `cohere.test.ts` | `server/src/__tests__/providers` | 125 | 0 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 125 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 305 | `default-max-tokens.test.ts` | `server/src/__tests__/providers` | 123 | 3 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 123 行，识别到 3 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 306 | `google-auth-header.test.ts` | `server/src/__tests__/providers` | 77 | 1 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 77 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 307 | `google-params.test.ts` | `server/src/__tests__/providers` | 96 | 0 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 96 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 308 | `google-schema.test.ts` | `server/src/__tests__/providers` | 275 | 0 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 275 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 309 | `google.test.ts` | `server/src/__tests__/providers` | 812 | 0 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 812 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 310 | `lucidity-airforce-dreamprompting-waterfall-logfare.test.ts` | `server/src/__tests__/providers` | 117 | 0 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 117 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 311 | `modelscope.test.ts` | `server/src/__tests__/providers` | 202 | 0 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 202 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 312 | `openai-compat.test.ts` | `server/src/__tests__/providers` | 1132 | 0 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 1132 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 313 | `pollinations.test.ts` | `server/src/__tests__/providers` | 132 | 0 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 132 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 314 | `radeon.test.ts` | `server/src/__tests__/providers` | 98 | 0 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 98 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 315 | `read-sse-frames.test.ts` | `server/src/__tests__/providers` | 112 | 3 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 112 行，识别到 3 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 316 | `reasoning-timeouts.test.ts` | `server/src/__tests__/providers` | 67 | 0 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 67 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 317 | `recurring-credit-gateways.test.ts` | `server/src/__tests__/providers` | 148 | 2 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 148 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 318 | `router9-septor.test.ts` | `server/src/__tests__/providers` | 153 | 4 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 153 行，识别到 4 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 319 | `sail.test.ts` | `server/src/__tests__/providers` | 170 | 2 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 170 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 320 | `stated-retry.test.ts` | `server/src/__tests__/providers` | 145 | 0 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 145 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 321 | `stream-first-byte.test.ts` | `server/src/__tests__/providers` | 275 | 3 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 275 行，识别到 3 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 322 | `unified-max-tokens-cap.test.ts` | `server/src/__tests__/providers` | 191 | 4 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 191 行，识别到 4 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 323 | `zhipu.test.ts` | `server/src/__tests__/providers` | 178 | 1 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 178 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 324 | `admin-rate-limit.test.ts` | `server/src/__tests__/routes` | 91 | 1 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 91 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 325 | `analytics-client.test.ts` | `server/src/__tests__/routes` | 57 | 1 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 57 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 326 | `analytics-request-attempts.test.ts` | `server/src/__tests__/routes` | 122 | 4 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 122 行，识别到 4 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 327 | `analytics-requests.test.ts` | `server/src/__tests__/routes` | 155 | 3 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 155 行，识别到 3 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 328 | `analytics.test.ts` | `server/src/__tests__/routes` | 826 | 4 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 826 行，识别到 4 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 329 | `anthropic-documents.test.ts` | `server/src/__tests__/routes` | 190 | 3 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 190 行，识别到 3 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 330 | `anthropic-fallback-convergence.test.ts` | `server/src/__tests__/routes` | 234 | 5 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 234 行，识别到 5 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 331 | `anthropic.test.ts` | `server/src/__tests__/routes` | 662 | 7 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 662 行，识别到 7 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 332 | `audio-transcriptions.test.ts` | `server/src/__tests__/routes` | 275 | 4 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 275 行，识别到 4 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 333 | `auth-login-identifier.test.ts` | `server/src/__tests__/routes` | 70 | 2 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 70 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 334 | `auth-reset-password.test.ts` | `server/src/__tests__/routes` | 164 | 4 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 164 行，识别到 4 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 335 | `auth-setup-code.test.ts` | `server/src/__tests__/routes` | 100 | 2 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 100 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 336 | `auth.test.ts` | `server/src/__tests__/routes` | 102 | 1 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 102 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 337 | `backups.test.ts` | `server/src/__tests__/routes` | 255 | 4 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 255 行，识别到 4 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 338 | `body-limit.test.ts` | `server/src/__tests__/routes` | 99 | 1 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 99 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 339 | `client-profiles.test.ts` | `server/src/__tests__/routes` | 240 | 2 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 240 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 340 | `compression.test.ts` | `server/src/__tests__/routes` | 166 | 1 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 166 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 341 | `conversations.test.ts` | `server/src/__tests__/routes` | 259 | 1 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 259 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 342 | `csp-inline-bootstrap.test.ts` | `server/src/__tests__/routes` | 80 | 2 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 80 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 343 | `csp-security.test.ts` | `server/src/__tests__/routes` | 171 | 1 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 171 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 344 | `custom-endpoint-identity.test.ts` | `server/src/__tests__/routes` | 393 | 5 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 393 行，识别到 5 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 345 | `custom-modalities.test.ts` | `server/src/__tests__/routes` | 416 | 4 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 416 行，识别到 4 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 346 | `custom-model-discovery.test.ts` | `server/src/__tests__/routes` | 637 | 5 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 637 行，识别到 5 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 347 | `custom-provider-multikey.test.ts` | `server/src/__tests__/routes` | 339 | 4 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 339 行，识别到 4 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 348 | `custom-provider.test.ts` | `server/src/__tests__/routes` | 547 | 3 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 547 行，识别到 3 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 349 | `custom-transcription.test.ts` | `server/src/__tests__/routes` | 221 | 6 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 221 行，识别到 6 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 350 | `disabled-model-routing.test.ts` | `server/src/__tests__/routes` | 255 | 7 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 255 行，识别到 7 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 351 | `docs.test.ts` | `server/src/__tests__/routes` | 105 | 1 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 105 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 352 | `embeddings-usage.test.ts` | `server/src/__tests__/routes` | 104 | 2 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 104 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 353 | `error-handler-security.test.ts` | `server/src/__tests__/routes` | 46 | 1 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 46 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 354 | `fallback-detail-header.test.ts` | `server/src/__tests__/routes` | 300 | 2 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 300 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 355 | `fallback-hardening.test.ts` | `server/src/__tests__/routes` | 271 | 3 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 271 行，识别到 3 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 356 | `fallback-keycount-parity.test.ts` | `server/src/__tests__/routes` | 71 | 1 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 71 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 357 | `fallback-pressure.test.ts` | `server/src/__tests__/routes` | 114 | 1 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 114 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 358 | `fallback-rate-limit-usage.test.ts` | `server/src/__tests__/routes` | 155 | 4 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 155 行，识别到 4 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 359 | `fallback.test.ts` | `server/src/__tests__/routes` | 418 | 1 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 418 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 360 | `free-tier.test.ts` | `server/src/__tests__/routes` | 196 | 4 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 196 行，识别到 4 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 361 | `fusion.test.ts` | `server/src/__tests__/routes` | 843 | 4 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 843 行，识别到 4 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 362 | `gemini.test.ts` | `server/src/__tests__/routes` | 239 | 2 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 239 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 363 | `key-monthly-budget.test.ts` | `server/src/__tests__/routes` | 93 | 0 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 93 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 364 | `keys-cooldowns.test.ts` | `server/src/__tests__/routes` | 117 | 2 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 117 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 365 | `keys-desktop-reauth.test.ts` | `server/src/__tests__/routes` | 147 | 3 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 147 行，识别到 3 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 366 | `keys-export-csv.test.ts` | `server/src/__tests__/routes` | 93 | 2 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 93 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 367 | `keys-export-custom-roundtrip.test.ts` | `server/src/__tests__/routes` | 158 | 6 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 158 行，识别到 6 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 368 | `keys-export-password.test.ts` | `server/src/__tests__/routes` | 67 | 2 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 67 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 369 | `keys-import-models.test.ts` | `server/src/__tests__/routes` | 224 | 5 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 224 行，识别到 5 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 370 | `keys-import-selected-custom.test.ts` | `server/src/__tests__/routes` | 161 | 3 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 161 行，识别到 3 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 371 | `keys-model-scope.test.ts` | `server/src/__tests__/routes` | 119 | 1 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 119 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 372 | `keys-providers-checklist.test.ts` | `server/src/__tests__/routes` | 72 | 2 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 72 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 373 | `keys-proxy.test.ts` | `server/src/__tests__/routes` | 198 | 1 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 198 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 374 | `keys-reveal.test.ts` | `server/src/__tests__/routes` | 89 | 2 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 89 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 375 | `keys-ssrf-guard.test.ts` | `server/src/__tests__/routes` | 93 | 1 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 93 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 376 | `keys.test.ts` | `server/src/__tests__/routes` | 457 | 2 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 457 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 377 | `logs.test.ts` | `server/src/__tests__/routes` | 448 | 2 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 448 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 378 | `mcp.test.ts` | `server/src/__tests__/routes` | 204 | 2 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 204 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 379 | `media-transcription-dashboard.test.ts` | `server/src/__tests__/routes` | 119 | 3 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 119 行，识别到 3 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 380 | `media-usage.test.ts` | `server/src/__tests__/routes` | 141 | 3 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 141 行，识别到 3 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 381 | `models-management.test.ts` | `server/src/__tests__/routes` | 596 | 1 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 596 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 382 | `models-provider-actions.test.ts` | `server/src/__tests__/routes` | 335 | 2 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 335 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 383 | `models-test-budget.test.ts` | `server/src/__tests__/routes` | 78 | 0 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 78 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 384 | `ollama.test.ts` | `server/src/__tests__/routes` | 256 | 2 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 256 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 385 | `profiles-chains.test.ts` | `server/src/__tests__/routes` | 242 | 6 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 242 行，识别到 6 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 386 | `provider-reported-size.test.ts` | `server/src/__tests__/routes` | 37 | 0 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 37 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 387 | `proxy-array-content.test.ts` | `server/src/__tests__/routes` | 403 | 2 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 403 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 388 | `proxy-auth-cors.test.ts` | `server/src/__tests__/routes` | 85 | 1 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 85 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 389 | `proxy-auth-rotation.test.ts` | `server/src/__tests__/routes` | 248 | 2 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 248 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 390 | `proxy-auto-model.test.ts` | `server/src/__tests__/routes` | 245 | 2 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 245 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 391 | `proxy-cache.test.ts` | `server/src/__tests__/routes` | 315 | 3 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 315 行，识别到 3 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 392 | `proxy-completions.test.ts` | `server/src/__tests__/routes` | 236 | 3 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 236 行，识别到 3 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 393 | `proxy-empty-completion.test.ts` | `server/src/__tests__/routes` | 205 | 1 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 205 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 394 | `proxy-error-redaction.test.ts` | `server/src/__tests__/routes` | 137 | 2 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 137 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 395 | `proxy-execution-id.test.ts` | `server/src/__tests__/routes` | 148 | 3 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 148 行，识别到 3 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 396 | `proxy-github-limits.test.ts` | `server/src/__tests__/routes` | 131 | 1 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 131 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 397 | `proxy-max-tokens-routing.test.ts` | `server/src/__tests__/routes` | 123 | 1 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 123 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 398 | `proxy-model-groups.test.ts` | `server/src/__tests__/routes` | 282 | 5 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 282 行，识别到 5 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 399 | `proxy-moonshot-partial.test.ts` | `server/src/__tests__/routes` | 148 | 3 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 148 行，识别到 3 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 400 | `proxy-multikey-penalty.test.ts` | `server/src/__tests__/routes` | 149 | 2 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 149 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 401 | `proxy-pinned-model.test.ts` | `server/src/__tests__/routes` | 113 | 2 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 113 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 402 | `proxy-provider-level-skip.test.ts` | `server/src/__tests__/routes` | 154 | 2 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 154 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 403 | `proxy-rate-limit.test.ts` | `server/src/__tests__/routes` | 69 | 1 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 69 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 404 | `proxy-retry.test.ts` | `server/src/__tests__/routes` | 271 | 0 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 271 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 405 | `proxy-served-model.test.ts` | `server/src/__tests__/routes` | 221 | 6 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 221 行，识别到 6 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 406 | `proxy-stream-chunk.test.ts` | `server/src/__tests__/routes` | 32 | 0 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 32 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 407 | `proxy-stream-integrity.test.ts` | `server/src/__tests__/routes` | 544 | 3 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 544 行，识别到 3 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 408 | `proxy-stream-usage.test.ts` | `server/src/__tests__/routes` | 356 | 7 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 356 行，识别到 7 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 409 | `proxy-tools-routing.test.ts` | `server/src/__tests__/routes` | 172 | 1 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 172 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 410 | `proxy-tools.test.ts` | `server/src/__tests__/routes` | 378 | 2 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 378 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 411 | `proxy-vision.test.ts` | `server/src/__tests__/routes` | 96 | 1 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 96 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 412 | `quota-outlook.test.ts` | `server/src/__tests__/routes` | 41 | 0 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 41 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 413 | `reasoning-control.test.ts` | `server/src/__tests__/routes` | 331 | 7 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 331 行，识别到 7 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 414 | `reasoning-only-failover.test.ts` | `server/src/__tests__/routes` | 167 | 3 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 167 行，识别到 3 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 415 | `rescue-wants-tools.test.ts` | `server/src/__tests__/routes` | 201 | 3 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 201 行，识别到 3 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 416 | `responses-fallback-convergence.test.ts` | `server/src/__tests__/routes` | 143 | 4 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 143 行，识别到 4 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 417 | `responses-fusion.test.ts` | `server/src/__tests__/routes` | 151 | 3 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 151 行，识别到 3 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 418 | `responses-tool-args-repair.test.ts` | `server/src/__tests__/routes` | 136 | 1 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 136 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 419 | `responses-translate.test.ts` | `server/src/__tests__/routes` | 305 | 0 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 305 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 420 | `responses.test.ts` | `server/src/__tests__/routes` | 600 | 2 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 600 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 421 | `routed-via-header.test.ts` | `server/src/__tests__/routes` | 206 | 4 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 206 行，识别到 4 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 422 | `routing-semantics.test.ts` | `server/src/__tests__/routes` | 357 | 7 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 357 行，识别到 7 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 423 | `settings-headroom.test.ts` | `server/src/__tests__/routes` | 86 | 1 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 86 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 424 | `settings-mcp.test.ts` | `server/src/__tests__/routes` | 166 | 1 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 166 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 425 | `settings-output-limit.test.ts` | `server/src/__tests__/routes` | 102 | 1 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 102 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 426 | `settings-proxy.test.ts` | `server/src/__tests__/routes` | 198 | 1 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 198 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 427 | `settings-task-weight-share.test.ts` | `server/src/__tests__/routes` | 115 | 1 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 115 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 428 | `static-assets-caching.test.ts` | `server/src/__tests__/routes` | 116 | 1 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 116 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 429 | `static-client.test.ts` | `server/src/__tests__/routes` | 40 | 0 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 40 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 430 | `status.test.ts` | `server/src/__tests__/routes` | 107 | 2 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 107 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 431 | `tool-rejection-e2e-1230.test.ts` | `server/src/__tests__/routes` | 132 | 1 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 132 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 432 | `tool-validate-surfaces.test.ts` | `server/src/__tests__/routes` | 258 | 3 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 258 行，识别到 3 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 433 | `update-release.test.ts` | `server/src/__tests__/routes` | 231 | 4 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 231 行，识别到 4 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 434 | `update.test.ts` | `server/src/__tests__/routes` | 602 | 5 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 602 行，识别到 5 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 435 | `url-tokens.test.ts` | `server/src/__tests__/routes` | 119 | 1 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 119 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 436 | `video-generations.test.ts` | `server/src/__tests__/routes` | 87 | 1 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 87 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 437 | `rotate-encryption-key-cli.test.ts` | `server/src/__tests__/scripts` | 148 | 3 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 148 行，识别到 3 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 438 | `rotate-encryption-key.test.ts` | `server/src/__tests__/scripts` | 84 | 1 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 84 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 439 | `anthropic-map.test.ts` | `server/src/__tests__/services` | 31 | 0 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 31 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 440 | `cache.test.ts` | `server/src/__tests__/services` | 697 | 1 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 697 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 441 | `catalog-sync-scheduler.test.ts` | `server/src/__tests__/services` | 81 | 1 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 81 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 442 | `catalog-sync-source.test.ts` | `server/src/__tests__/services` | 209 | 4 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 209 行，识别到 4 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 443 | `catalog-sync.test.ts` | `server/src/__tests__/services` | 671 | 4 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 671 行，识别到 4 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 444 | `compression.test.ts` | `server/src/__tests__/services` | 451 | 2 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 451 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 445 | `context-handoff.test.ts` | `server/src/__tests__/services` | 370 | 0 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 370 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 446 | `cooldown-probe.test.ts` | `server/src/__tests__/services` | 347 | 1 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 347 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 447 | `custom-model-seed.test.ts` | `server/src/__tests__/services` | 77 | 1 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 77 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 448 | `custom-model-sync.test.ts` | `server/src/__tests__/services` | 265 | 3 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 265 行，识别到 3 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 449 | `declarative-config-profiles.test.ts` | `server/src/__tests__/services` | 112 | 0 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 112 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 450 | `declarative-config.test.ts` | `server/src/__tests__/services` | 282 | 1 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 282 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 451 | `degradation.test.ts` | `server/src/__tests__/services` | 166 | 1 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 166 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 452 | `embeddings.test.ts` | `server/src/__tests__/services` | 382 | 4 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 382 行，识别到 4 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 453 | `exhaustion-summary.test.ts` | `server/src/__tests__/services` | 112 | 0 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 112 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 454 | `fusion.test.ts` | `server/src/__tests__/services` | 112 | 1 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 112 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 455 | `health-error.test.ts` | `server/src/__tests__/services` | 82 | 0 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 82 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 456 | `health-key-proxy.test.ts` | `server/src/__tests__/services` | 114 | 1 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 114 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 457 | `health-log.test.ts` | `server/src/__tests__/services` | 104 | 0 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 104 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 458 | `health-probe-pacing.test.ts` | `server/src/__tests__/services` | 203 | 2 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 203 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 459 | `health-scheduler.test.ts` | `server/src/__tests__/services` | 73 | 1 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 73 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 460 | `health-transport-preserve.test.ts` | `server/src/__tests__/services` | 135 | 0 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 135 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 461 | `idempotency.test.ts` | `server/src/__tests__/services` | 126 | 1 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 126 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 462 | `key-budget.test.ts` | `server/src/__tests__/services` | 240 | 5 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 240 行，识别到 5 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 463 | `key-concurrency.test.ts` | `server/src/__tests__/services` | 135 | 0 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 135 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 464 | `learn-limits.test.ts` | `server/src/__tests__/services` | 89 | 0 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 89 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 465 | `match-tier-routing.test.ts` | `server/src/__tests__/services` | 99 | 4 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 99 行，识别到 4 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 466 | `media.test.ts` | `server/src/__tests__/services` | 603 | 4 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 603 行，识别到 4 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 467 | `model-discovery.test.ts` | `server/src/__tests__/services` | 323 | 0 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 323 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 468 | `model-groups.test.ts` | `server/src/__tests__/services` | 248 | 2 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 248 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 469 | `model-listing-execution-status.test.ts` | `server/src/__tests__/services` | 107 | 3 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 107 行，识别到 3 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 470 | `model-retirement.test.ts` | `server/src/__tests__/services` | 253 | 3 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 253 行，识别到 3 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 471 | `model-state-overrides.test.ts` | `server/src/__tests__/services` | 57 | 1 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 57 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 472 | `model-weight-overrides.test.ts` | `server/src/__tests__/services` | 165 | 0 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 165 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 473 | `provider-minute-cap.test.ts` | `server/src/__tests__/services` | 92 | 1 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 92 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 474 | `provider-quota.test.ts` | `server/src/__tests__/services` | 418 | 2 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 418 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 475 | `provider-reported-routing.test.ts` | `server/src/__tests__/services` | 29 | 0 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 29 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 476 | `provisional-usage.test.ts` | `server/src/__tests__/services` | 107 | 0 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 107 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 477 | `quirks.test.ts` | `server/src/__tests__/services` | 71 | 0 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 71 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 478 | `quota-forecast.test.ts` | `server/src/__tests__/services` | 211 | 3 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 211 行，识别到 3 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 479 | `quota-outlook.test.ts` | `server/src/__tests__/services` | 151 | 2 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 151 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 480 | `ratelimit-cooldown-ceiling.test.ts` | `server/src/__tests__/services` | 104 | 1 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 104 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 481 | `ratelimit-cooldown-error-kind.test.ts` | `server/src/__tests__/services` | 111 | 1 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 111 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 482 | `ratelimit-daily-boundary.test.ts` | `server/src/__tests__/services` | 155 | 2 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 155 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 483 | `ratelimit-local-endpoint.test.ts` | `server/src/__tests__/services` | 96 | 2 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 96 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 484 | `ratelimit-retention.test.ts` | `server/src/__tests__/services` | 217 | 3 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 217 行，识别到 3 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 485 | `ratelimit.test.ts` | `server/src/__tests__/services` | 441 | 1 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 441 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 486 | `request-retention.test.ts` | `server/src/__tests__/services` | 126 | 2 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 126 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 487 | `router-bandit.test.ts` | `server/src/__tests__/services` | 552 | 3 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 552 行，识别到 3 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 488 | `router-key-bandit.test.ts` | `server/src/__tests__/services` | 165 | 4 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 165 行，识别到 4 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 489 | `router-key-proxy.test.ts` | `server/src/__tests__/services` | 60 | 0 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 60 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 490 | `router-keycount-budget.test.ts` | `server/src/__tests__/services` | 74 | 0 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 74 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 491 | `router-model-scope.test.ts` | `server/src/__tests__/services` | 106 | 0 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 106 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 492 | `router-priority-penalty.test.ts` | `server/src/__tests__/services` | 131 | 2 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 131 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 493 | `router-speed-scoring.test.ts` | `server/src/__tests__/services` | 266 | 4 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 266 行，识别到 4 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 494 | `router-task-type.test.ts` | `server/src/__tests__/services` | 157 | 4 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 157 行，识别到 4 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 495 | `router-window-headroom.test.ts` | `server/src/__tests__/services` | 188 | 0 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 188 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 496 | `router.test.ts` | `server/src/__tests__/services` | 330 | 0 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 330 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 497 | `routing-exhaustion.test.ts` | `server/src/__tests__/services` | 162 | 1 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 162 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 498 | `routing-keyless-filter.test.ts` | `server/src/__tests__/services` | 144 | 2 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 144 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 499 | `scoring.test.ts` | `server/src/__tests__/services` | 417 | 0 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 417 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 500 | `transcription.test.ts` | `server/src/__tests__/services` | 354 | 4 | 该文件承担服务端自动化测试职责。 这是一份测试文件，约 354 行，识别到 4 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 501 | `app.ts` | `server/src` | 368 | 4 | 该文件承担工程源文件职责。 这是一份实现文件，约 368 行，识别到 4 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 502 | `index.ts` | `server/src/db` | 251 | 14 | 该文件承担数据库与迁移运行时职责。 这是一份实现文件，约 251 行，识别到 14 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 503 | `TEMPLATE.ts` | `server/src/db/migrate` | 22 | 2 | 该文件承担数据库与迁移运行时职责。 这是一份实现文件，约 22 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 504 | `cli.ts` | `server/src/db/migrate` | 219 | 14 | 该文件承担数据库与迁移运行时职责。 这是一份实现文件，约 219 行，识别到 14 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 505 | `defaults.ts` | `server/src/db/migrate` | 126 | 0 | 该文件承担数据库与迁移运行时职责。 这是一份实现文件，约 126 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 506 | `runner.ts` | `server/src/db/migrate` | 265 | 17 | 该文件承担数据库与迁移运行时职责。 这是一份实现文件，约 265 行，识别到 17 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 507 | `20260101_000000_legacy_baseline.ts` | `server/src/db/migrations` | 2311 | 41 | 该文件承担SQLite 迁移脚本职责。 这是一份实现文件，约 2311 行，识别到 41 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 508 | `20260627_000001_custom_provider_modalities.ts` | `server/src/db/migrations` | 32 | 5 | 该文件承担SQLite 迁移脚本职责。 这是一份实现文件，约 32 行，识别到 5 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 509 | `20260627_000002_catalog_model_state.ts` | `server/src/db/migrations` | 32 | 2 | 该文件承担SQLite 迁移脚本职责。 这是一份实现文件，约 32 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 510 | `20260628_120000_request_aggregates.ts` | `server/src/db/migrations` | 120 | 4 | 该文件承担SQLite 迁移脚本职责。 这是一份实现文件，约 120 行，识别到 4 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 511 | `20260630_000001_github_gpt41_context.ts` | `server/src/db/migrations` | 24 | 2 | 该文件承担SQLite 迁移脚本职责。 这是一份实现文件，约 24 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 512 | `20260706_000001_request_client_info.ts` | `server/src/db/migrations` | 33 | 3 | 该文件承担SQLite 迁移脚本职责。 这是一份实现文件，约 33 行，识别到 3 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 513 | `20260706_000002_custom_model_tool_support.ts` | `server/src/db/migrations` | 22 | 2 | 该文件承担SQLite 迁移脚本职责。 这是一份实现文件，约 22 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 514 | `20260714_000001_profile_chain_backfill.ts` | `server/src/db/migrations` | 49 | 2 | 该文件承担SQLite 迁移脚本职责。 这是一份实现文件，约 49 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 515 | `20260720_000001_key_health_error.ts` | `server/src/db/migrations` | 19 | 3 | 该文件承担SQLite 迁移脚本职责。 这是一份实现文件，约 19 行，识别到 3 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 516 | `20260726_000001_cooldown_probe_provenance.ts` | `server/src/db/migrations` | 42 | 3 | 该文件承担SQLite 迁移脚本职责。 这是一份实现文件，约 42 行，识别到 3 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 517 | `20260726_000002_request_attempts.ts` | `server/src/db/migrations` | 46 | 2 | 该文件承担SQLite 迁移脚本职责。 这是一份实现文件，约 46 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 518 | `20260726_000003_model_source_provenance.ts` | `server/src/db/migrations` | 82 | 3 | 该文件承担SQLite 迁移脚本职责。 这是一份实现文件，约 82 行，识别到 3 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 519 | `20260726_000004_media_model_meta.ts` | `server/src/db/migrations` | 38 | 3 | 该文件承担SQLite 迁移脚本职责。 这是一份实现文件，约 38 行，识别到 3 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 520 | `20260726_000005_request_served_model.ts` | `server/src/db/migrations` | 35 | 3 | 该文件承担SQLite 迁移脚本职责。 这是一份实现文件，约 35 行，识别到 3 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 521 | `20260726_000006_attempt_error_summary.ts` | `server/src/db/migrations` | 33 | 3 | 该文件承担SQLite 迁移脚本职责。 这是一份实现文件，约 33 行，识别到 3 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 522 | `20260727_000001_agent_compatibility.ts` | `server/src/db/migrations` | 52 | 3 | 该文件承担SQLite 迁移脚本职责。 这是一份实现文件，约 52 行，识别到 3 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 523 | `20260728_000001_tombstone_provenance.ts` | `server/src/db/migrations` | 41 | 3 | 该文件承担SQLite 迁移脚本职责。 这是一份实现文件，约 41 行，识别到 3 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 524 | `20260729_000001_custom_model_endpoint_identity.ts` | `server/src/db/migrations` | 165 | 4 | 该文件承担SQLite 迁移脚本职责。 这是一份实现文件，约 165 行，识别到 4 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 525 | `20260802_000001_custom_endpoint_host_labels.ts` | `server/src/db/migrations` | 58 | 3 | 该文件承担SQLite 迁移脚本职责。 这是一份实现文件，约 58 行，识别到 3 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 526 | `20260805_000001_key_model_scope.ts` | `server/src/db/migrations` | 20 | 3 | 该文件承担SQLite 迁移脚本职责。 这是一份实现文件，约 20 行，识别到 3 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 527 | `20260805_000002_client_profiles.ts` | `server/src/db/migrations` | 36 | 2 | 该文件承担SQLite 迁移脚本职责。 这是一份实现文件，约 36 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 528 | `20260810_000001_api_key_proxy.ts` | `server/src/db/migrations` | 37 | 3 | 该文件承担SQLite 迁移脚本职责。 这是一份实现文件，约 37 行，识别到 3 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 529 | `20260819_000001_custom_model_tombstones.ts` | `server/src/db/migrations` | 22 | 2 | 该文件承担SQLite 迁移脚本职责。 这是一份实现文件，约 22 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 530 | `20260820_000001_playground_conversations.ts` | `server/src/db/migrations` | 50 | 2 | 该文件承担SQLite 迁移脚本职责。 这是一份实现文件，约 50 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 531 | `20260823_000001_server_logs.ts` | `server/src/db/migrations` | 56 | 2 | 该文件承担SQLite 迁移脚本职责。 这是一份实现文件，约 56 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 532 | `20260823_000002_backups_table.ts` | `server/src/db/migrations` | 24 | 2 | 该文件承担SQLite 迁移脚本职责。 这是一份实现文件，约 24 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 533 | `20260823_000003_attempt_key_label.ts` | `server/src/db/migrations` | 30 | 2 | 该文件承担SQLite 迁移脚本职责。 这是一份实现文件，约 30 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 534 | `20260823_000004_profile_auto_include.ts` | `server/src/db/migrations` | 28 | 3 | 该文件承担SQLite 迁移脚本职责。 这是一份实现文件，约 28 行，识别到 3 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 535 | `20260901_000001_idempotency_claims.ts` | `server/src/db/migrations` | 49 | 2 | 该文件承担SQLite 迁移脚本职责。 这是一份实现文件，约 49 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 536 | `20260901_000002_quota_observation_lookup.ts` | `server/src/db/migrations` | 40 | 2 | 该文件承担SQLite 迁移脚本职责。 这是一份实现文件，约 40 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 537 | `20260901_000003_request_caller.ts` | `server/src/db/migrations` | 31 | 3 | 该文件承担SQLite 迁移脚本职责。 这是一份实现文件，约 31 行，识别到 3 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 538 | `20260902_000001_analytics_latency_percentile_index.ts` | `server/src/db/migrations` | 38 | 2 | 该文件承担SQLite 迁移脚本职责。 这是一份实现文件，约 38 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 539 | `20260903_000001_mcp_enabled_default.ts` | `server/src/db/migrations` | 42 | 2 | 该文件承担SQLite 迁移脚本职责。 这是一份实现文件，约 42 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 540 | `20260903_000002_response_cache.ts` | `server/src/db/migrations` | 56 | 2 | 该文件承担SQLite 迁移脚本职责。 这是一份实现文件，约 56 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 541 | `20260904_000001_key_monthly_budget.ts` | `server/src/db/migrations` | 37 | 3 | 该文件承担SQLite 迁移脚本职责。 这是一份实现文件，约 37 行，识别到 3 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 542 | `20260914_000001_key_monthly_usage.ts` | `server/src/db/migrations` | 39 | 2 | 该文件承担SQLite 迁移脚本职责。 这是一份实现文件，约 39 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 543 | `20260915_000001_quota_snapshot_freshness.ts` | `server/src/db/migrations` | 37 | 2 | 该文件承担SQLite 迁移脚本职责。 这是一份实现文件，约 37 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 544 | `model-pricing.ts` | `server/src/db` | 223 | 1 | 该文件承担数据库与迁移运行时职责。 这是一份实现文件，约 223 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 545 | `node-sqlite.ts` | `server/src/db` | 115 | 3 | 该文件承担数据库与迁移运行时职责。 这是一份实现文件，约 115 行，识别到 3 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 546 | `types.ts` | `server/src/db` | 27 | 0 | 该文件承担数据库与迁移运行时职责。 这是一份实现文件，约 27 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 547 | `docs-page.ts` | `server/src/docs` | 319 | 0 | 该文件承担内嵌 OpenAPI 文档职责。 这是一份实现文件，约 319 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 548 | `openapi.ts` | `server/src/docs` | 1110 | 0 | 该文件承担内嵌 OpenAPI 文档职责。 这是一份实现文件，约 1110 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 549 | `env.ts` | `server/src` | 10 | 0 | 该文件承担工程源文件职责。 这是一份实现文件，约 10 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 550 | `index.ts` | `server/src` | 147 | 1 | 该文件承担工程源文件职责。 这是一份实现文件，约 147 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 551 | `anthropic-documents.ts` | `server/src/lib` | 154 | 5 | 该文件承担服务端共享算法库职责。 这是一份实现文件，约 154 行，识别到 5 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 552 | `app-version.ts` | `server/src/lib` | 61 | 4 | 该文件承担服务端共享算法库职责。 这是一份实现文件，约 61 行，识别到 4 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 553 | `attempt-trace.ts` | `server/src/lib` | 79 | 4 | 该文件承担服务端共享算法库职责。 这是一份实现文件，约 79 行，识别到 4 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 554 | `budget.ts` | `server/src/lib` | 19 | 1 | 该文件承担服务端共享算法库职责。 这是一份实现文件，约 19 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 555 | `client-classifier.ts` | `server/src/lib` | 101 | 2 | 该文件承担服务端共享算法库职责。 这是一份实现文件，约 101 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 556 | `client-context.ts` | `server/src/lib` | 50 | 4 | 该文件承担服务端共享算法库职责。 这是一份实现文件，约 50 行，识别到 4 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 557 | `config.ts` | `server/src/lib` | 121 | 5 | 该文件承担服务端共享算法库职责。 这是一份实现文件，约 121 行，识别到 5 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 558 | `content.ts` | `server/src/lib` | 195 | 10 | 该文件承担服务端共享算法库职责。 这是一份实现文件，约 195 行，识别到 10 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 559 | `crypto.ts` | `server/src/lib` | 231 | 12 | 提供方密钥 AES-256-GCM 与主密钥生命周期。 这是一份实现文件，约 231 行，识别到 12 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 560 | `custom-provider-cleanup.ts` | `server/src/lib` | 47 | 3 | 该文件承担服务端共享算法库职责。 这是一份实现文件，约 47 行，识别到 3 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 561 | `db-backup.ts` | `server/src/lib` | 272 | 15 | 该文件承担服务端共享算法库职责。 这是一份实现文件，约 272 行，识别到 15 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 562 | `endpoint-scope.ts` | `server/src/lib` | 109 | 7 | 该文件承担服务端共享算法库职责。 这是一份实现文件，约 109 行，识别到 7 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 563 | `env-drift.ts` | `server/src/lib` | 130 | 9 | 该文件承担服务端共享算法库职责。 这是一份实现文件，约 130 行，识别到 9 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 564 | `error-classify.ts` | `server/src/lib` | 679 | 23 | 该文件承担服务端共享算法库职责。 这是一份实现文件，约 679 行，识别到 23 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 565 | `error-redaction.ts` | `server/src/lib` | 51 | 2 | 该文件承担服务端共享算法库职责。 这是一份实现文件，约 51 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 566 | `fallback-loop.ts` | `server/src/lib` | 1562 | 32 | 跨模型故障转移循环，分类上游错误并写追踪头。 这是一份实现文件，约 1562 行，识别到 32 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 567 | `file-permissions.ts` | `server/src/lib` | 232 | 7 | 该文件承担服务端共享算法库职责。 这是一份实现文件，约 232 行，识别到 7 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 568 | `gemini-wire.ts` | `server/src/lib` | 432 | 16 | 该文件承担服务端共享算法库职责。 这是一份实现文件，约 432 行，识别到 16 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 569 | `guardrails.ts` | `server/src/lib` | 129 | 7 | 该文件承担服务端共享算法库职责。 这是一份实现文件，约 129 行，识别到 7 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 570 | `header-value.ts` | `server/src/lib` | 55 | 3 | 该文件承担服务端共享算法库职责。 这是一份实现文件，约 55 行，识别到 3 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 571 | `image-normalize.ts` | `server/src/lib` | 259 | 7 | 该文件承担服务端共享算法库职责。 这是一份实现文件，约 259 行，识别到 7 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 572 | `inbound-chat.ts` | `server/src/lib` | 606 | 5 | 该文件承担服务端共享算法库职责。 这是一份实现文件，约 606 行，识别到 5 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 573 | `key-parser.ts` | `server/src/lib` | 701 | 15 | 该文件承担服务端共享算法库职责。 这是一份实现文件，约 701 行，识别到 15 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 574 | `key-proxy.ts` | `server/src/lib` | 104 | 4 | 该文件承担服务端共享算法库职责。 这是一份实现文件，约 104 行，识别到 4 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 575 | `log-redaction.ts` | `server/src/lib` | 185 | 3 | 该文件承担服务端共享算法库职责。 这是一份实现文件，约 185 行，识别到 3 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 576 | `model-scope.ts` | `server/src/lib` | 27 | 2 | 该文件承担服务端共享算法库职责。 这是一份实现文件，约 27 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 577 | `one-time-code.ts` | `server/src/lib` | 59 | 1 | 该文件承担服务端共享算法库职责。 这是一份实现文件，约 59 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 578 | `password.ts` | `server/src/lib` | 27 | 2 | 该文件承担服务端共享算法库职责。 这是一份实现文件，约 27 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 579 | `process-safety-net.ts` | `server/src/lib` | 110 | 6 | 该文件承担服务端共享算法库职责。 这是一份实现文件，约 110 行，识别到 6 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 580 | `provider-identity.ts` | `server/src/lib` | 107 | 4 | 该文件承担服务端共享算法库职责。 这是一份实现文件，约 107 行，识别到 4 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 581 | `provider-size-parser.ts` | `server/src/lib` | 73 | 2 | 该文件承担服务端共享算法库职责。 这是一份实现文件，约 73 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 582 | `provider-timeout.ts` | `server/src/lib` | 75 | 7 | 该文件承担服务端共享算法库职责。 这是一份实现文件，约 75 行，识别到 7 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 583 | `proxy.ts` | `server/src/lib` | 1065 | 45 | 该文件承担服务端共享算法库职责。 这是一份实现文件，约 1065 行，识别到 45 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 584 | `request-log.ts` | `server/src/lib` | 155 | 5 | 该文件承担服务端共享算法库职责。 这是一份实现文件，约 155 行，识别到 5 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 585 | `reset-code.ts` | `server/src/lib` | 43 | 4 | 该文件承担服务端共享算法库职责。 这是一份实现文件，约 43 行，识别到 4 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 586 | `sampling-params.ts` | `server/src/lib` | 451 | 11 | 该文件承担服务端共享算法库职责。 这是一份实现文件，约 451 行，识别到 11 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 587 | `scheduler.ts` | `server/src/lib` | 16 | 1 | 该文件承担服务端共享算法库职责。 这是一份实现文件，约 16 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 588 | `served-model.ts` | `server/src/lib` | 82 | 3 | 该文件承担服务端共享算法库职责。 这是一份实现文件，约 82 行，识别到 3 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 589 | `server-logs.ts` | `server/src/lib` | 501 | 24 | 该文件承担服务端共享算法库职责。 这是一份实现文件，约 501 行，识别到 24 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 590 | `setup-code.ts` | `server/src/lib` | 40 | 4 | 该文件承担服务端共享算法库职责。 这是一份实现文件，约 40 行，识别到 4 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 591 | `structured-output.ts` | `server/src/lib` | 77 | 3 | 该文件承担服务端共享算法库职责。 这是一份实现文件，约 77 行，识别到 3 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 592 | `system-prompt.ts` | `server/src/lib` | 82 | 5 | 该文件承担服务端共享算法库职责。 这是一份实现文件，约 82 行，识别到 5 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 593 | `task-type.ts` | `server/src/lib` | 104 | 4 | 该文件承担服务端共享算法库职责。 这是一份实现文件，约 104 行，识别到 4 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 594 | `think-tags.ts` | `server/src/lib` | 211 | 4 | 该文件承担服务端共享算法库职责。 这是一份实现文件，约 211 行，识别到 4 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 595 | `tool-args.ts` | `server/src/lib` | 202 | 5 | 该文件承担服务端共享算法库职责。 这是一份实现文件，约 202 行，识别到 5 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 596 | `tool-call-rescue.ts` | `server/src/lib` | 246 | 9 | 该文件承担服务端共享算法库职责。 这是一份实现文件，约 246 行，识别到 9 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 597 | `tool-capability.ts` | `server/src/lib` | 73 | 6 | 该文件承担服务端共享算法库职责。 这是一份实现文件，约 73 行，识别到 6 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 598 | `tool-validate.ts` | `server/src/lib` | 179 | 7 | 该文件承担服务端共享算法库职责。 这是一份实现文件，约 179 行，识别到 7 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 599 | `url-guard.ts` | `server/src/lib` | 266 | 10 | 该文件承担服务端共享算法库职责。 这是一份实现文件，约 266 行，识别到 10 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 600 | `wake-detect.ts` | `server/src/lib` | 109 | 6 | 该文件承担服务端共享算法库职责。 这是一份实现文件，约 109 行，识别到 6 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 601 | `errorHandler.ts` | `server/src/middleware` | 50 | 1 | 该文件承担Express 中间件职责。 这是一份实现文件，约 50 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 602 | `rateLimit.ts` | `server/src/middleware` | 145 | 4 | 该文件承担Express 中间件职责。 这是一份实现文件，约 145 行，识别到 4 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 603 | `requireAuth.ts` | `server/src/middleware` | 18 | 1 | 该文件承担Express 中间件职责。 这是一份实现文件，约 18 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 604 | `aihorde.ts` | `server/src/providers` | 222 | 2 | 该文件承担上游提供方适配器职责。 这是一份实现文件，约 222 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 605 | `airforce.ts` | `server/src/providers` | 44 | 2 | 该文件承担上游提供方适配器职责。 这是一份实现文件，约 44 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 606 | `base.ts` | `server/src/providers` | 474 | 7 | 该文件承担上游提供方适配器职责。 这是一份实现文件，约 474 行，识别到 7 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 607 | `blaze.ts` | `server/src/providers` | 45 | 2 | 该文件承担上游提供方适配器职责。 这是一份实现文件，约 45 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 608 | `clod.ts` | `server/src/providers` | 46 | 2 | 该文件承担上游提供方适配器职责。 这是一份实现文件，约 46 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 609 | `cloudflare.ts` | `server/src/providers` | 214 | 1 | 该文件承担上游提供方适配器职责。 这是一份实现文件，约 214 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 610 | `cohere.ts` | `server/src/providers` | 148 | 2 | 该文件承担上游提供方适配器职责。 这是一份实现文件，约 148 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 611 | `dreamprompting.ts` | `server/src/providers` | 54 | 2 | 该文件承担上游提供方适配器职责。 这是一份实现文件，约 54 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 612 | `electronhub.ts` | `server/src/providers` | 73 | 2 | 该文件承担上游提供方适配器职责。 这是一份实现文件，约 73 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 613 | `experiential.ts` | `server/src/providers` | 36 | 1 | 该文件承担上游提供方适配器职责。 这是一份实现文件，约 36 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 614 | `google.ts` | `server/src/providers` | 877 | 26 | 该文件承担上游提供方适配器职责。 这是一份实现文件，约 877 行，识别到 26 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 615 | `index.ts` | `server/src/providers` | 593 | 5 | 该文件承担上游提供方适配器职责。 这是一份实现文件，约 593 行，识别到 5 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 616 | `logfare.ts` | `server/src/providers` | 45 | 2 | 该文件承担上游提供方适配器职责。 这是一份实现文件，约 45 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 617 | `lucidity.ts` | `server/src/providers` | 51 | 2 | 该文件承担上游提供方适配器职责。 这是一份实现文件，约 51 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 618 | `modelscope.ts` | `server/src/providers` | 153 | 2 | 该文件承担上游提供方适配器职责。 这是一份实现文件，约 153 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 619 | `openai-compat.ts` | `server/src/providers` | 589 | 4 | 该文件承担上游提供方适配器职责。 这是一份实现文件，约 589 行，识别到 4 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 620 | `pollinations.ts` | `server/src/providers` | 71 | 1 | 该文件承担上游提供方适配器职责。 这是一份实现文件，约 71 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 621 | `router9.ts` | `server/src/providers` | 46 | 1 | 该文件承担上游提供方适配器职责。 这是一份实现文件，约 46 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 622 | `sail.ts` | `server/src/providers` | 403 | 1 | 该文件承担上游提供方适配器职责。 这是一份实现文件，约 403 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 623 | `septor.ts` | `server/src/providers` | 42 | 2 | 该文件承担上游提供方适配器职责。 这是一份实现文件，约 42 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 624 | `speechify.ts` | `server/src/providers` | 33 | 1 | 该文件承担上游提供方适配器职责。 这是一份实现文件，约 33 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 625 | `waterfall.ts` | `server/src/providers` | 45 | 2 | 该文件承担上游提供方适配器职责。 这是一份实现文件，约 45 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 626 | `zhipu.ts` | `server/src/providers` | 109 | 1 | 该文件承担上游提供方适配器职责。 这是一份实现文件，约 109 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 627 | `analytics.ts` | `server/src/routes` | 784 | 3 | 该文件承担服务端 HTTP 路由职责。 这是一份实现文件，约 784 行，识别到 3 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 628 | `anthropic.ts` | `server/src/routes` | 1157 | 20 | 该文件承担服务端 HTTP 路由职责。 这是一份实现文件，约 1157 行，识别到 20 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 629 | `auth.ts` | `server/src/routes` | 290 | 5 | 该文件承担服务端 HTTP 路由职责。 这是一份实现文件，约 290 行，识别到 5 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 630 | `backups.ts` | `server/src/routes` | 138 | 3 | 该文件承担服务端 HTTP 路由职责。 这是一份实现文件，约 138 行，识别到 3 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 631 | `cache.ts` | `server/src/routes` | 50 | 0 | 该文件承担服务端 HTTP 路由职责。 这是一份实现文件，约 50 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 632 | `client-profiles.ts` | `server/src/routes` | 155 | 5 | 该文件承担服务端 HTTP 路由职责。 这是一份实现文件，约 155 行，识别到 5 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 633 | `compression.ts` | `server/src/routes` | 85 | 1 | 该文件承担服务端 HTTP 路由职责。 这是一份实现文件，约 85 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 634 | `conversations.ts` | `server/src/routes` | 263 | 6 | 该文件承担服务端 HTTP 路由职责。 这是一份实现文件，约 263 行，识别到 6 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 635 | `docs.ts` | `server/src/routes` | 23 | 0 | 该文件承担服务端 HTTP 路由职责。 这是一份实现文件，约 23 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 636 | `embeddings.ts` | `server/src/routes` | 286 | 1 | 该文件承担服务端 HTTP 路由职责。 这是一份实现文件，约 286 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 637 | `fallback.ts` | `server/src/routes` | 714 | 5 | 该文件承担服务端 HTTP 路由职责。 这是一份实现文件，约 714 行，识别到 5 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 638 | `free-tier.ts` | `server/src/routes` | 205 | 1 | 该文件承担服务端 HTTP 路由职责。 这是一份实现文件，约 205 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 639 | `gemini.ts` | `server/src/routes` | 233 | 7 | 该文件承担服务端 HTTP 路由职责。 这是一份实现文件，约 233 行，识别到 7 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 640 | `health.ts` | `server/src/routes` | 80 | 0 | 该文件承担服务端 HTTP 路由职责。 这是一份实现文件，约 80 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 641 | `keys.ts` | `server/src/routes` | 1616 | 12 | 该文件承担服务端 HTTP 路由职责。 这是一份实现文件，约 1616 行，识别到 12 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 642 | `logs.ts` | `server/src/routes` | 106 | 3 | 该文件承担服务端 HTTP 路由职责。 这是一份实现文件，约 106 行，识别到 3 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 643 | `mcp.ts` | `server/src/routes` | 376 | 13 | 该文件承担服务端 HTTP 路由职责。 这是一份实现文件，约 376 行，识别到 13 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 644 | `media.ts` | `server/src/routes` | 207 | 0 | 该文件承担服务端 HTTP 路由职责。 这是一份实现文件，约 207 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 645 | `models.ts` | `server/src/routes` | 502 | 2 | 该文件承担服务端 HTTP 路由职责。 这是一份实现文件，约 502 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 646 | `ollama.ts` | `server/src/routes` | 611 | 13 | 该文件承担服务端 HTTP 路由职责。 这是一份实现文件，约 611 行，识别到 13 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 647 | `premium.ts` | `server/src/routes` | 126 | 2 | 该文件承担服务端 HTTP 路由职责。 这是一份实现文件，约 126 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 648 | `profiles.ts` | `server/src/routes` | 518 | 5 | 该文件承担服务端 HTTP 路由职责。 这是一份实现文件，约 518 行，识别到 5 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 649 | `proxy.ts` | `server/src/routes` | 2898 | 26 | OpenAI 兼容 `/v1/chat/completions` 与 `/v1/completions` 主入口，含压缩、缓存、工具救援与流式管道。 这是一份实现文件，约 2898 行，识别到 26 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 650 | `responses.ts` | `server/src/routes` | 1619 | 17 | 该文件承担服务端 HTTP 路由职责。 这是一份实现文件，约 1619 行，识别到 17 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 651 | `settings.ts` | `server/src/routes` | 575 | 3 | 该文件承担服务端 HTTP 路由职责。 这是一份实现文件，约 575 行，识别到 3 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 652 | `status.ts` | `server/src/routes` | 239 | 1 | 该文件承担服务端 HTTP 路由职责。 这是一份实现文件，约 239 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 653 | `update.ts` | `server/src/routes` | 618 | 13 | 该文件承担服务端 HTTP 路由职责。 这是一份实现文件，约 618 行，识别到 13 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 654 | `url-tokens.ts` | `server/src/routes` | 35 | 0 | 该文件承担服务端 HTTP 路由职责。 这是一份实现文件，约 35 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 655 | `export-catalog.ts` | `server/src/scripts` | 149 | 4 | 该文件承担运维与目录脚本职责。 这是一份实现文件，约 149 行，识别到 4 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 656 | `rotate-encryption-key.ts` | `server/src/scripts` | 326 | 8 | 该文件承担运维与目录脚本职责。 这是一份实现文件，约 326 行，识别到 8 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 657 | `routing-sim.ts` | `server/src/scripts` | 149 | 6 | 该文件承担运维与目录脚本职责。 这是一份实现文件，约 149 行，识别到 6 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 658 | `test-all-models.ts` | `server/src/scripts` | 68 | 0 | 该文件承担运维与目录脚本职责。 这是一份实现文件，约 68 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 659 | `anthropic-map.ts` | `server/src/services` | 157 | 5 | 该文件承担服务端领域服务职责。 这是一份实现文件，约 157 行，识别到 5 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 660 | `auth.ts` | `server/src/services` | 128 | 11 | 该文件承担服务端领域服务职责。 这是一份实现文件，约 128 行，识别到 11 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 661 | `backups.ts` | `server/src/services` | 554 | 30 | 该文件承担服务端领域服务职责。 这是一份实现文件，约 554 行，识别到 30 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 662 | `cache.ts` | `server/src/services` | 769 | 29 | 该文件承担服务端领域服务职责。 这是一份实现文件，约 769 行，识别到 29 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 663 | `catalog-sync.ts` | `server/src/services` | 866 | 12 | 该文件承担服务端领域服务职责。 这是一份实现文件，约 866 行，识别到 12 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 664 | `config.ts` | `server/src/services/compression` | 169 | 9 | 该文件承担提示词压缩流水线职责。 这是一份实现文件，约 169 行，识别到 9 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 665 | `aging.ts` | `server/src/services/compression/engines` | 58 | 1 | 该文件承担提示词压缩流水线职责。 这是一份实现文件，约 58 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 666 | `custom-filters.ts` | `server/src/services/compression/engines` | 49 | 3 | 该文件承担提示词压缩流水线职责。 这是一份实现文件，约 49 行，识别到 3 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 667 | `dedup.ts` | `server/src/services/compression/engines` | 67 | 2 | 该文件承担提示词压缩流水线职责。 这是一份实现文件，约 67 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 668 | `filter-definitions.ts` | `server/src/services/compression/engines` | 49 | 0 | 该文件承担提示词压缩流水线职责。 这是一份实现文件，约 49 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 669 | `hard-budget.ts` | `server/src/services/compression/engines` | 50 | 0 | 该文件承担提示词压缩流水线职责。 这是一份实现文件，约 50 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 670 | `index.ts` | `server/src/services/compression/engines` | 8 | 0 | 该文件承担提示词压缩流水线职责。 这是一份实现文件，约 8 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 671 | `jsoncompact.ts` | `server/src/services/compression/engines` | 146 | 5 | 该文件承担提示词压缩流水线职责。 这是一份实现文件，约 146 行，识别到 5 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 672 | `lite.ts` | `server/src/services/compression/engines` | 35 | 0 | 该文件承担提示词压缩流水线职责。 这是一份实现文件，约 35 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 673 | `read-lifecycle.ts` | `server/src/services/compression/engines` | 69 | 2 | 该文件承担提示词压缩流水线职责。 这是一份实现文件，约 69 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 674 | `relevance.ts` | `server/src/services/compression/engines` | 74 | 2 | 该文件承担提示词压缩流水线职责。 这是一份实现文件，约 74 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 675 | `toolfilter.ts` | `server/src/services/compression/engines` | 197 | 8 | 该文件承担提示词压缩流水线职责。 这是一份实现文件，约 197 行，识别到 8 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 676 | `fidelity-gate.ts` | `server/src/services/compression` | 109 | 4 | 该文件承担提示词压缩流水线职责。 这是一份实现文件，约 109 行，识别到 4 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 677 | `helpers.ts` | `server/src/services/compression` | 52 | 6 | 该文件承担提示词压缩流水线职责。 这是一份实现文件，约 52 行，识别到 6 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 678 | `pipeline.ts` | `server/src/services/compression` | 268 | 5 | 该文件承担提示词压缩流水线职责。 这是一份实现文件，约 268 行，识别到 5 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 679 | `preservation.ts` | `server/src/services/compression` | 133 | 7 | 该文件承担提示词压缩流水线职责。 这是一份实现文件，约 133 行，识别到 7 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 680 | `registry.ts` | `server/src/services/compression` | 20 | 4 | 该文件承担提示词压缩流水线职责。 这是一份实现文件，约 20 行，识别到 4 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 681 | `stats.ts` | `server/src/services/compression` | 67 | 3 | 该文件承担提示词压缩流水线职责。 这是一份实现文件，约 67 行，识别到 3 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 682 | `types.ts` | `server/src/services/compression` | 101 | 0 | 该文件承担提示词压缩流水线职责。 这是一份实现文件，约 101 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 683 | `context-handoff.ts` | `server/src/services` | 188 | 10 | 该文件承担服务端领域服务职责。 这是一份实现文件，约 188 行，识别到 10 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 684 | `cooldown-probe.ts` | `server/src/services` | 227 | 7 | 该文件承担服务端领域服务职责。 这是一份实现文件，约 227 行，识别到 7 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 685 | `custom-endpoint.ts` | `server/src/services` | 213 | 10 | 该文件承担服务端领域服务职责。 这是一份实现文件，约 213 行，识别到 10 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 686 | `custom-media-register.ts` | `server/src/services` | 76 | 1 | 该文件承担服务端领域服务职责。 这是一份实现文件，约 76 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 687 | `custom-model-register.ts` | `server/src/services` | 159 | 2 | 该文件承担服务端领域服务职责。 这是一份实现文件，约 159 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 688 | `custom-model-seed.ts` | `server/src/services` | 79 | 2 | 该文件承担服务端领域服务职责。 这是一份实现文件，约 79 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 689 | `custom-model-sync.ts` | `server/src/services` | 145 | 5 | 该文件承担服务端领域服务职责。 这是一份实现文件，约 145 行，识别到 5 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 690 | `custom-model-tombstone.ts` | `server/src/services` | 48 | 3 | 该文件承担服务端领域服务职责。 这是一份实现文件，约 48 行，识别到 3 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 691 | `declarative-config.ts` | `server/src/services` | 504 | 13 | 该文件承担服务端领域服务职责。 这是一份实现文件，约 504 行，识别到 13 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 692 | `degradation.ts` | `server/src/services` | 173 | 6 | 该文件承担服务端领域服务职责。 这是一份实现文件，约 173 行，识别到 6 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 693 | `embeddings.ts` | `server/src/services` | 371 | 11 | 该文件承担服务端领域服务职责。 这是一份实现文件，约 371 行，识别到 11 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 694 | `fusion.ts` | `server/src/services` | 892 | 21 | 该文件承担服务端领域服务职责。 这是一份实现文件，约 892 行，识别到 21 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 695 | `gemini-map.ts` | `server/src/services` | 81 | 4 | 该文件承担服务端领域服务职责。 这是一份实现文件，约 81 行，识别到 4 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 696 | `health.ts` | `server/src/services` | 423 | 14 | 该文件承担服务端领域服务职责。 这是一份实现文件，约 423 行，识别到 14 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 697 | `idempotency.ts` | `server/src/services` | 182 | 8 | 该文件承担服务端领域服务职责。 这是一份实现文件，约 182 行，识别到 8 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 698 | `key-budget.ts` | `server/src/services` | 136 | 9 | 该文件承担服务端领域服务职责。 这是一份实现文件，约 136 行，识别到 9 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 699 | `media.ts` | `server/src/services` | 1116 | 31 | 该文件承担服务端领域服务职责。 这是一份实现文件，约 1116 行，识别到 31 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 700 | `model-discovery.ts` | `server/src/services` | 715 | 23 | 该文件承担服务端领域服务职责。 这是一份实现文件，约 715 行，识别到 23 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 701 | `model-groups.ts` | `server/src/services` | 386 | 17 | 该文件承担服务端领域服务职责。 这是一份实现文件，约 386 行，识别到 17 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 702 | `model-listing.ts` | `server/src/services` | 137 | 2 | 该文件承担服务端领域服务职责。 这是一份实现文件，约 137 行，识别到 2 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 703 | `model-retirement.ts` | `server/src/services` | 119 | 3 | 该文件承担服务端领域服务职责。 这是一份实现文件，约 119 行，识别到 3 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 704 | `model-state.ts` | `server/src/services` | 430 | 22 | 该文件承担服务端领域服务职责。 这是一份实现文件，约 430 行，识别到 22 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 705 | `model-weight-overrides.ts` | `server/src/services` | 185 | 5 | 该文件承担服务端领域服务职责。 这是一份实现文件，约 185 行，识别到 5 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 706 | `penalty-inspector.ts` | `server/src/services` | 272 | 6 | 该文件承担服务端领域服务职责。 这是一份实现文件，约 272 行，识别到 6 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 707 | `profile-models.ts` | `server/src/services` | 64 | 3 | 该文件承担服务端领域服务职责。 这是一份实现文件，约 64 行，识别到 3 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 708 | `provider-credential.ts` | `server/src/services` | 57 | 1 | 该文件承担服务端领域服务职责。 这是一份实现文件，约 57 行，识别到 1 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 709 | `provider-quota.ts` | `server/src/services` | 592 | 17 | 该文件承担服务端领域服务职责。 这是一份实现文件，约 592 行，识别到 17 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 710 | `quirks.ts` | `server/src/services` | 73 | 3 | 该文件承担服务端领域服务职责。 这是一份实现文件，约 73 行，识别到 3 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 711 | `quota-forecast.ts` | `server/src/services` | 203 | 5 | 该文件承担服务端领域服务职责。 这是一份实现文件，约 203 行，识别到 5 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 712 | `quota-outlook.ts` | `server/src/services` | 88 | 3 | 该文件承担服务端领域服务职责。 这是一份实现文件，约 88 行，识别到 3 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 713 | `ratelimit.ts` | `server/src/services` | 1511 | 77 | 限流账本。内存滑动窗口 + SQLite 持久化 RPM/RPD/TPM/TPD、冷却、租约与提供方级日限额。 这是一份实现文件，约 1511 行，识别到 77 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 714 | `request-retention.ts` | `server/src/services` | 273 | 8 | 该文件承担服务端领域服务职责。 这是一份实现文件，约 273 行，识别到 8 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 715 | `router.ts` | `server/src/services` | 2326 | 67 | 路由器核心。为每次推理请求挑选模型与密钥，管理粘性会话、统一模型分组、Fusion 候选链、惩罚与社区先验。 这是一份实现文件，约 2326 行，识别到 67 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 716 | `scoring.ts` | `server/src/services` | 536 | 26 | 老虎机评分引擎。把可靠性、速度、智能做成 [0,1] 凸组合，再乘以额度与 429 护栏。 这是一份实现文件，约 536 行，识别到 26 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 717 | `url-tokens.ts` | `server/src/services` | 76 | 5 | 该文件承担服务端领域服务职责。 这是一份实现文件，约 76 行，识别到 5 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 718 | `vitest.config.ts` | `server` | 16 | 0 | 该文件承担工程源文件职责。 这是一份实现文件，约 16 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |
| 719 | `types.ts` | `shared` | 659 | 0 | 前后端共享的 Platform、Model、Chat、Analytics、Quota 类型定义。 这是一份实现文件，约 659 行，识别到 0 个函数或类声明。阅读时请同时对照同名测试与 `docs/zh-cn` 中对应主题，避免只看实现忽略不变量。 |


## 4. 超大文件导读（行数 Top）

这些文件是认知负荷最高的地方，改动前应先读对应测试。

| 行数 | 路径 | 为什么大 |
|---:|---|---|
| 2898 | `server/src/routes/proxy.ts` | OpenAI chat/completions 全功能：压缩、缓存、流、工具救援、鉴权轮换。 |
| 2326 | `server/src/services/router.ts` | 选模型与选密钥的全部策略、粘性、Fusion 链、惩罚、社区先验。 |
| 2311 | `server/src/db/migrations/20260101_000000_legacy_baseline.ts` | 历史 schema 与种子的不可逆基线，含大量 ensure* 列补丁。 |
| 1619 | `server/src/routes/responses.ts` | Codex 所需 Responses 协议双向转译。 |
| 1616 | `server/src/routes/keys.ts` | 密钥 CRUD、导入导出、探测、范围、代理覆盖。 |
| 1562 | `server/src/lib/fallback-loop.ts` | 故障转移状态机与追踪头。 |
| 1511 | `server/src/services/ratelimit.ts` | 窗口、租约、冷却、提供方日限额、从错误学习限额。 |
| 1235 | `client/src/pages/PlaygroundPage.tsx` | 试验台会话、流式、附件、采样。 |
| 1188 | `client/src/pages/AnalyticsPage.tsx` | 图表与多时间窗聚合展示。 |
| 1157 | `server/src/routes/anthropic.ts` | Anthropic Messages 表面。 |
| 1132 | `server/src/__tests__/providers/openai-compat.test.ts` | 服务端自动化测试 |
| 1116 | `server/src/services/media.ts` | 图像/音频/视频路由。 |
| 1110 | `server/src/docs/openapi.ts` | 运行时 OpenAPI 文档树。 |
| 1101 | `server/src/__tests__/lib/proxy.test.ts` | 服务端自动化测试 |
| 1090 | `client/src/components/settings-dialog.tsx` | 几乎全部热更新设置的 UI。 |
| 1065 | `server/src/lib/proxy.ts` | 出站代理、Fetch Relay、SSRF 策略。 |
| 943 | `cli/src/tools.ts` | 十几个智能体的配置生成。 |
| 905 | `server/src/__tests__/lib/fallback-loop.test.ts` | 服务端自动化测试 |
| 892 | `server/src/services/fusion.ts` | 多模型面板与评审。 |
| 877 | `server/src/providers/google.ts` | 原生 Gemini 差异最大的适配器。 |
| 866 | `server/src/services/catalog-sync.ts` | 签名目录拉取与应用。 |
| 843 | `server/src/__tests__/routes/fusion.test.ts` | 服务端自动化测试 |
| 838 | `client/src/components/keys/provider-list.tsx` | 密钥管理界面组件 |
| 826 | `server/src/__tests__/routes/analytics.test.ts` | 服务端自动化测试 |
| 812 | `server/src/__tests__/providers/google.test.ts` | 服务端自动化测试 |


## 5. 测试文件与实现的镜像关系

server 测试目录镜像 `routes/`、`services/`、`providers/`、`lib/`、`db/`。命名 `proxy-retry.test.ts` 这种「路由名-场景」表示同一路由文件有多份场景切分，避免单文件 2000 行测不下去。client 测试多与 `lib/` 纯函数放在一起（`foo.ts` 旁 `foo.test.ts`），页面级较少。cli 用快照锁生成物。desktop 测打包与托盘而不是测 React。

新增功能的默认位置：实现放在对应层，测试放在镜像路径，不要只在 `__tests__/integration` 堆巨型脚本（已有 integration 目录给跨层用例）。

## 6. 非 TS 但仍属「源」的重要文件

虽不在行数表，修改时同等谨慎：

- `package.json` / 各包 package.json / lockfile
- `Dockerfile` / `docker-compose.yml` / `docker-entrypoint.sh`
- `.env.example`（配置契约）
- `shared` 以外的 JSON locale
- `cli/tools.json`
- GitHub workflows
- `docs/` 用户文档
- `LICENSE` MIT

## 7. 生成代码与手写代码

OpenAPI 是手写 TS 对象而非从注释生成。i18n JSON 手写。目录模型来自外部签名 feed，不是源码生成。CLI 对各 Agent 的 JSON/YAML 是运行时生成写到用户家目录，生成器本身手写在 `cli/src/tools.ts`。

## 8. 许可与第三方

LICENSE 为 MIT。运行时依赖见各 package.json。better-sqlite3 含原生代码，Electron 重建用 `@electron/rebuild`。sharp 用于图像规范化。贡献时注意不要把提供方 SDK 大面积引入——项目倾向「一种 OpenAI 兼容客户端打天下」，例外才专用适配器。

## 9. 统计方法说明

扫描脚本遍历工作区，跳过 `node_modules`、`.git`、隐藏目录名以点开头的文件夹（但会统计到已列出的 `desktop` 等）。行数按 `\n` 计数。函数识别基于顶层 `function`/`class`/`export function` 正则，**不含**类方法与箭头函数赋值，因此实际函数数量高于表中「函数/类」列。该列用于相对比较文件复杂度，不是精确 AST。

## 10. 结语

FreeLLMAPI 源码是典型的 2020 年代 TypeScript 全栈单体：一个 Node 网关、一个 SPA、一个 CLI、一个 Electron 壳、一份共享类型。复杂度不在语言花样，而在「免费层真实世界」：限流、冷却、协议方言、密钥安全与 Agent 兼容。阅读与修改时请以 `router.ts` / `ratelimit.ts` / `fallback-loop.ts` 为心脏，以三千加测试为心电图。

完整文件级清单见第 3 章表格，可按目录过滤作为代码地图使用。
