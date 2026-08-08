# 05.5 · 桌面前端代码走读

包：`@memmy/frontend-desktop`（`App/frontend/desktop/）。React 19 + Vite 8，`useReducer`+Context 状态（无 Redux/Zustand），无 UI 组件库（自定义 CSS），Markdown 用 `react-markdown`+`remark-gfm`/`remark-math`/`rehype-katex`，DAG 用 `@xyflow/react`，图标 `lucide-react`，校验 `zod`。

## 5.5.1 入口与路由

- `src/main.tsx` — Vite 入口，`createRoot(...).render(<App />)`（含 dev-only `?preview=` 路由：startup/nickname/conflict/skills）。
- `src/app.tsx` — 顶层 `App` 包 `AppProviders`+`RuntimeApp`；启动 API 客户端、SSE（`createEventsConnection`）、Agent 任务协调器。
- `src/app/providers.tsx` — 组合 `AppStateProvider`→`ApiClientsProvider`→`TaskBusProvider`→`I18nProvider`/`ThemeProvider`。
- `src/app/router.tsx`+`routes.ts` — `AppRouter`；`routes.ts` 含 `resolveInitialView`/`resolveMainWindowRouteTarget`/`FOCUSED_AGENT_CHAT_STORAGE_KEY`。

## 5.5.2 状态管理

`useReducer` + Context：
- `src/state/app-state.tsx` `AppStateProvider` 经 `useReducer(appReducer,...,createInitialAppState)` 暴露 `{state,dispatch}`。
- `src/state/app-reducer.ts`/`app-actions.ts` — `AppAction` 联合、`agentActions`/`appActions`。
- 切片：`agent-chat-slice.ts`(`AgentState`)、`agent-composer-state.ts`、`agent-tool-traces.ts`、`tools-slice.ts`、`update-notification.ts`、`task-done-notification.ts`。

## 5.5.3 与本地后端的通信

**没有固定本地 API 端口**——运行时动态解析：
- `src/api/runtime-config.ts` `getRuntimeConfig()` 优先试 Electron 预加载桥 `window.memmy.getRuntimeConfig()`；回退 Vite env（`VITE_MEMMY_LOCAL_API_URL`+`VITE_MEMMY_LOCAL_TOKEN`）；再回退浏览器调试 `GET /__memmy_runtime_config`。返回 `RuntimeConfig{baseUrl,localToken,memory?,agentGateway?}`（schema 来自 `@memmy/local-api-contracts`）。
- `src/api/http.ts` `requestJson<T>()` 做 `fetch(new URL(path,config.baseUrl),{headers:{"x-memmy-local-token":config.localToken,...}})` 并用 zod 解析。**本地后端绑 `127.0.0.1:0`（临时端口）**，固定 `19000` 只是 Vite 渲染开发服务器。
- API 客户端模块（`src/api/`）：`account/agent-source/asr/bootstrap/byok-token-usage/channels/config/integrations/local-data/memmy-agent/memory-runtime/token-quota-client.ts`、`client-types.ts`(`createAppClients`)、`events.ts`(SSE)、`integration-errors.ts`。
- `src/global.d.ts` 声明预加载桥 `window.memmy?`。

## 5.5.4 主要页面（`src/pages/`）

- **工作台**：`home-page.tsx`、`agent-thread-messages.tsx`、`agent-message-content.tsx`、`agent-command-palette.tsx`、`agent-file-attachment-chip.tsx`、`history-dag-panel.tsx`(`@xyflow/react` DAG)、`pet-page.tsx`(桌宠)。
- **记忆**：`memory-page.tsx`、`home-memory.tsx`、`memory-sources-page.tsx`、`memory-source-scan.ts`、`memory-plugin-conflict-modal.tsx`，子页 `pages/memory/`：`overview/memories/sources/tasks/skills/policies/world-model/analytics/logs-sub-page.tsx`。
- **工具/集成**：`tools-page.tsx`；集成逻辑在 `src/integrations/`(`toolkit-catalog.ts`/`integration-meta.tsx`/`connection-state.ts`)。
- **设置**：`settings-page.tsx`、`model-page.tsx`(+`model-config.ts`)、`token-detail-page.tsx`、`token-exhausted-modal.tsx`、`api-key-page.tsx`、`api-key-optional-page.tsx`、`api-key-form-fields.tsx`。
- **引导**：`onboarding-page.tsx`、`welcome-page.tsx`、`startup-screen.tsx`、`first-encounter-*.tsx`、`product-tour.tsx`、`onboarding-scan-animation.tsx`。
- **账户/登录**：`login-page.tsx`；app 级 `app/account-channel.ts`、`app/login-mode.ts`、`app/token-exhausted-apply-more.ts`。
- **外壳**：`app-frame.tsx`、`sidebar-resize.tsx`。

## 5.5.5 其他目录

`analytics/`(gtag,`use-analytics.ts`)、`i18n/`(`i18n-provider.tsx`,`use-translation.ts`)、`theme/`(`theme-provider.tsx`)、`components/`(modal/button/`search-palette.tsx`/`connect-channel-modal.tsx`/`mascot/`)、`lib/`(`task-bus.ts`,`nickname.ts`)、`workers/`、`feedback/`、`community/`、`legal/`、`assets/`(brand/agent-logos/channel-logos/mascot/fonts)。

> 设计评价：**优点**：运行时配置三重回退稳健、zod 契约与后端一致、记忆页分组清晰（工作/洞察/系统）、DAG 可视化提升会话可读性。**潜在改进**：无 UI 组件库导致样式散落 CSS 文件、可维护性偏弱；状态切片较多但缺统一选择器层，随功能增长可能需要更结构化的状态方案。

---

> ← [走读索引](./index.md) ｜ 上一个 → [04 Agent 来源](./04-agent-sources.md) ｜ 下一个 → [06 Electron 外壳](./06-electron-shell.md)

---

☕️ 制作不易，请我喝咖啡☕️关注我➕