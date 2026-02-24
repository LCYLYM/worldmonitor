# World Monitor 代码导读（超详细学习路线）

> 目标：帮助你**完整、系统、可落地**地读懂这个仓库（前端、后端 API、桌面端、数据层、测试），并形成可迁移的工程化思维与排障能力。

## 0. 你在读的是什么

World Monitor 是一个 **TypeScript + Vite** 的单页应用（SPA），主 UI 是一个基于 **MapLibre + deck.gl** 的可交互地图/地球仪；项目同时提供：

- **Web 版**：部署在 Vercel（或类似平台）。
- **Desktop 版（Tauri）**：Rust 外壳 + Web UI + 本地 Node sidecar（本地优先的 API 服务器）。
- **三种 Variant（变体）**：World / Tech / Finance（同一代码库，通过环境变量切换）。

项目的核心特征是：大量数据源（RSS / 各类 API）→ 统一聚合与缓存 → 在 UI 中以「面板 + 图层」形式呈现；同时提供本地/云端 LLM 摘要与分析能力。

---

## 1. 快速上手（先跑起来再读）

在仓库根目录：

```bash
npm ci
npm run dev
```

切换变体：

```bash
npm run dev:tech
npm run dev:finance
```

构建：

```bash
npm run build
```

运行核心测试（建议先跑一遍建立信心）：

```bash
npm run lint:md
npm run test:data
```

> 说明：本仓库还有 Playwright E2E 与 sidecar/API 的 Node 测试脚本（详见 `package.json` 的 `test:*`）。

---

## 2. 目录地图（先建立“脑内索引”）

以下是最常读的目录与它们的职责：

- `src/`：Web 前端主工程（Vite 入口在这里）
  - `src/main.ts`：Web 入口，初始化监控/分析与挂载 React（或等价 UI）根组件
  - `src/App.ts`：应用中枢（全局状态、服务调度、定时刷新、各面板/图层的组合）
  - `src/components/`：UI 组件（地图容器、新闻面板、各类信息面板）
  - `src/services/`：数据服务层（拉取/缓存/聚合/转换，供 UI 使用）
  - `src/config/`：变体与运行参数配置（World/Tech/Finance 的差异通常在这里落地）
  - `src/utils/`：通用工具
  - `src/types/`：类型定义
- `api/`：Vercel Functions（线上 Web 环境的 API/代理/抓取器）
  - 常见用途：RSS 代理、CORS 处理、YouTube embed、第三方 API 的 server-side 转发
- `server/`：Node 后端/中间层（与 CORS、统一错误、某些路由聚合有关）
- `proto/`：proto-first 的接口契约（用于生成客户端/服务器类型与 OpenAPI 文档）
- `docs/`：文档（功能、部署、桌面端配置、添加端点流程等）
- `src-tauri/`：Desktop（Tauri）工程
  - `src-tauri/sidecar/`：本地 sidecar（Node 本地 API server，desktop 优先走本地）
- `tests/`、`e2e/`：Node 测试与 Playwright E2E

如果你只看 6 个文件/目录就想理解 60%：

1. `package.json`（scripts 入口）
2. `vite.config.ts`（构建与代理/插件）
3. `src/main.ts`（应用启动）
4. `src/App.ts`（应用调度中枢）
5. `src/config/`（变体与开关）
6. `api/`（Web 版 serverless API 面）

---

## 3. “三层架构”视角：UI / Service / API

建议用“三层”来读（每层都尽量找到：入口、输出、错误处理、缓存策略）。

### 3.1 UI 层（`src/components`）

你要回答的问题：

- 页面由哪些 Panel 组成？Panel 的数据来自哪里？
- Map 图层是怎样组织、开关、渲染的？
- UI 如何处理“先快后慢”的加载（例如先渲染新闻列表，后做聚类/分析）？

实践路径：

1. 在 `src/App.ts` 找到面板列表/布局组合的位置。
2. 找到 Map 容器组件（通常命名包含 `Map` / `Globe` / `Layer`）。
3. 观察组件 props：哪些是数据、哪些是配置、哪些是回调。

### 3.2 Service 层（`src/services`）

你要回答的问题：

- 服务是如何被“定时刷新”的？刷新间隔在哪定义？
- 缓存放在哪里（内存、Redis、localStorage、Service Worker）？
- 数据规范化在哪做（坐标、时间、去重、聚类）？

实践路径（推荐从“可测的”服务开始）：

- 直接读 `tests/*.test.mjs` 里覆盖到的服务/工具函数，顺藤摸瓜回源文件。
- 搜索 `CACHE_VERSION`、`getCacheKey`、`Redis`、`TTL` 等关键词，理解缓存层的边界与策略。

### 3.3 API 层（`api/` + `server/` + `src-tauri/sidecar`）

你要回答的问题：

- Web 版 API：哪些请求走 `api/`（Vercel Functions）？
- Desktop 版：为什么需要 sidecar？哪些请求本地优先、哪些会云端 fallback？
- CORS 与鉴权（例如 `WORLDMONITOR_API_KEY`）在哪处理？

学习线索：

- `docs/API_KEY_DEPLOYMENT.md`：桌面端云端 fallback 的 key gating 架构图与流程说明。
- `docs/DESKTOP_CONFIGURATION.md`：桌面端 secrets/feature schema 与降级策略。

---

## 4. 关键启动流程：从 `src/main.ts` 到“所有服务都跑起来”

读启动流程时，不要试图一次性理解全部功能，优先建立“骨架”：

1. **入口初始化**：错误监控/分析（例如 Sentry、analytics）是否会影响开发调试？
2. **Variant 决策**：`VITE_VARIANT`（或同类变量）如何决定：
   - 加载哪些 RSS 分类/数据层？
   - 显示哪些 panel？
   - 是否启用某些昂贵数据源？
3. **服务注册**：服务对象/函数在哪里被创建？是否统一放进某个 registry？
4. **定时刷新**：服务刷新是否依赖 tab 可见性（`document.visibilityState`）？
5. **状态下发**：服务输出以什么形式进入 UI（props / context / store）？

> 小技巧：如果你在 `tests` 里看到 `flushStaleRefreshes` 之类的测试，优先读它 —— 这类函数通常是“服务调度的核心骨架”。

---

## 5. Map 图层系统（deck.gl 思维）

地图类项目最难的是“图层体系”。建议你用以下问题拆解：

1. **图层定义**：每个 layer 的数据结构是什么（GeoJSON？自定义数组？聚类对象？）
2. **渲染适配**：数据如何转换为 deck.gl layer props（颜色、半径、拾取 pick、tooltip）？
3. **开关与状态**：layer toggle 状态在哪里维护？是否支持 URL 共享（把 layers 编码进 query）？
4. **性能策略**：
   - 聚类（supercluster 或自研）
   - progressive disclosure（缩放级别决定显示密度/透明度）
   - 虚拟列表（新闻面板）
5. **交互回路**：点击 marker → 打开 popup/panel → 触发更多数据加载（懒加载）？

实践路线：

- 先从一个“简单图层”（例如 countries labels）读到“复杂图层”（例如实时 AIS/flight）。
- 找到该图层对应的数据服务（通常同名或在同目录），建立 `service -> layer renderer -> UI` 的链路图。

---

## 6. Proto-first API：契约驱动的工程习惯

仓库强调 “Proto-first API contracts”，核心收益是：

- **接口先行**：先改 proto，再生成类型/客户端/文档，编译器帮你兜底。
- **跨端一致**：Web、serverless、sidecar 都共享同一套契约。

学习建议：

1. 先读 `docs/ADDING_ENDPOINTS.md`（这是“如何在这个项目里做正确改动”的最佳教程）。
2. 再看 `proto/` 与 `docs/api/*.openapi.*` 的产物，理解生成链。

---

## 7. Desktop（Tauri）与 sidecar：为什么要“本地优先”

一个典型的 Desktop 请求路径是：

1. UI 发起 `fetch('/api/...')`
2. Desktop runtime 尝试走 sidecar（本地 Node server）
3. sidecar 无法满足时才会考虑 cloud fallback（并可能被 API key gating 阻断）

你读 desktop 相关代码时，重点关注：

- secrets 如何存储（OS keychain，而不是前端明文）
- feature toggle 如何降级（missing key 时的 fallback 描述）
- CORS/header 允许列表是否一致（Web/server/sidecar 三处配置）

文档入口：

- `docs/DESKTOP_CONFIGURATION.md`
- `docs/API_KEY_DEPLOYMENT.md`
- `docs/RELEASE_PACKAGING.md`

---

## 8. 测试与“读代码的正确姿势”

强烈建议你把测试当成“可执行文档”来读：

- `npm run test:data`：覆盖数据完整性、缓存行为、关键函数（例如去重、TTL、可见性刷新等）
- `npm run test:sidecar`：覆盖部分 API handler / sidecar 逻辑
- `npm run test:e2e:*`：覆盖 UI 行为与视觉回归（对图层系统尤其关键）

读法建议：

1. 先选一个测试用例（例如“缓存 coalesce 并发 miss”）
2. 跳到源文件定位被测函数
3. 回到 `App.ts` 看它是如何被调度/调用
4. 最后再回到 UI 看它如何展示

---

## 9. 建议的“30 天学习计划”（可按你节奏调整）

> 如果你想要“工程与思维能力提升”，关键是 **循环：运行 → 断点/日志 → 读代码 → 改一点 → 写测试**。

- 第 1–2 天：跑起来 + 熟悉 scripts（dev/build/test）
- 第 3–7 天：读 `src/main.ts`、`src/App.ts`，画出服务调度的骨架
- 第 2 周：选 3 个图层（简单/中等/实时），打通 service→layer→UI
- 第 3 周：读 `api/` 与 `server/`，理解 CORS、代理、缓存、错误处理
- 第 4 周：读 `src-tauri/sidecar` 与桌面端配置，理解“本地优先”的产品与工程权衡

---

## 10. 你可以告诉我你想优先学什么

你可以直接回复我以下任意一个方向，我可以继续“按文件逐段带读”并给你做思维训练题：

1. 我想先搞懂 **App.ts 的服务调度与刷新机制**
2. 我想先搞懂 **deck.gl 图层系统**（如何从数据到渲染）
3. 我想先搞懂 **RSS/新闻聚合**（去重、聚类、性能）
4. 我想先搞懂 **Desktop + sidecar**（本地优先与云端 fallback）
5. 我想先搞懂 **proto-first 的 API 工程流**
