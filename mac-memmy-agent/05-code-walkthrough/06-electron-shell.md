# 05.6 · Electron 外壳代码走读

包：`@memmy/desktop`（`App/shell/desktop/）。Electron 38.4 + electron-builder ^26，`main: ./dist/main/main.js`。依赖 `@memmy/backend`、`@memmy/desktop-interface`、`@memmy/local-api-contracts`、`dotenv`、`yaml`。

## 5.6.1 主进程入口 `src/main/main.ts`

模块级全局：`mainWindow`/`petWindow`(桌宠)/`localBackend`/`menuBarTray`/`runtimeServices`/`runtimeConfig`/`memoryServiceControl`/`packagedRendererServer`。

`boot()`（启动时调）顺序：
1. `set MEMMY_APP_EDITION` → `initLogger()`
2. 可选预备更新安装
3. `showSplashWindow()`
4. `registerIpcHandlers()`
5. `installBundledCliIfNeeded()`
6. `startPackagedRendererServerIfNeeded()`
7. **`app.isPackaged`** → `startPackagedRuntimeServices(...)`（spawn memory + agent 网关），开发态为 `null`
8. `startLocalApi(runtimeServices)`
9. `createInitialWindow()`
10. 同步菜单栏 Tray + 后台更新检查

### 后端编排（三层）
- `src/main/runtime-services.ts` `startPackagedRuntimeServices()`→`PackagedRuntimeServices{memory,agentGateway,restartMemory(),close(),terminateSync()}`。常量：`DEFAULT_MEMORY_URL="http://127.0.0.1:18960"`、`DEFAULT_AGENT_GATEWAY_HEALTH_PORT=18970`、`DEFAULT_AGENT_WEBSOCKET_PORT=18980`、`LOCAL_HOST="127.0.0.1"`、`STARTUP_TIMEOUT_MS=30_000`。`preparePackagedRuntimeConfig()` 写 `~/.memmy/config.yaml`(storage/sqlitePath、websocket host/port/token、gateway host/port、agents.defaults workspace/model/provider)，确保 `~/.memmy/workspace`、`~/.memmy/memory-service/memory.sqlite`；spawn memory + `AgentGatewaySupervisor`；跑 `startPackagedBrowserPreparation`(托管 Chromium) 与 `syncBundledAgentSkills`。
- `startLocalApi()`(main.ts 640 行)：设 env（`MEMMY_CONFIG`/`MEMMY_MEMORY_LAYER_URL`/`MEMMY_MEMORY_LAYER_TOKEN`/`MEMMY_MEMORY_DB_PATH`），`localBackend = await createLocalBackend({databasePath:<userData>/app.sqlite, bootstrapScenario, desktopInstallFingerprint, memmyConfigPath, memoryBaseUrl, runtimeConfigPath})`，`agentGateway` 取 `services?.agentGateway ?? resolveAgentGatewayRuntimeConfig()`，返回交给渲染进程的 `DesktopRuntimeConfig`。
- 重启：`restartMemoryService()`→`runtimeServices.restartMemory()`(打包) 或 `restartExternalMemoryService()`(开发)。

## 5.6.2 窗口管理

全窗口 + 桌宠窗口 + macOS 菜单栏 `Tray`。逻辑在 `src/main/window-mode.ts`：`fullWindowOptions`/`petWindowOptions`/`resolveBootWindowMode`/`resolvePetWindowBounds`/`resolveRendererUrl`/`DesktopWindowMode`/`RendererRouteTarget`/`PetWindowLayout`/`PetWindowPointer`。`main.ts` 实现拖动、退出全屏处理、`PendingMainWindowAction` 缓冲、退出清理（`APP_QUIT_CLEANUP_FORCE_EXIT_DELAY_MS=5000`）。打包态渲染由回环静态服务提供：`src/main/renderer-static-server.ts`(`startPackagedRendererStaticServer`，`dist/renderer` 由前端构建拷入)。

## 5.6.3 IPC

`registerIpcHandlers()`(main.ts 732) 注册 `memmy:*` 的 `ipcMain.handle`：`get-runtime-config`、`get-app-info`、`check-for-updates`、`download-update`、`open-update-installer`、`openExternal`、`openAgentTool`、`openMailto`、`copy-image-to-clipboard`、`save-image`、`notify-task-done`、`notify-update-available`、`export-memory-database`、`install-cli-tools`、`restart-memory-service`、`open-logs-directory`、`export-diagnostics-report`、`get/set-log-level`、`get/request-microphone-access`、`select-project-directory`、`select-empty-project-directory`、`set-pet-window`、`hide-pet-window`、`set-menu-bar-icon`、`get-main-window-fullscreen`；外加推送通道 `memmy:route-target-request`、`memmy:update-download-progress`。

## 5.6.4 托管 Chromium 与外部 Agent 工具

托管 Chromium 浏览器工具由 `runtime-services.ts`(`startPackagedBrowserPreparation`，env `MEMMY_BROWSER_PREPARATION_ATTEMPT_ID`) 预备；Agent 侧用 `node dist/main.js internal browser-prepare` 启动（见 dev-start）。Agent 请求**永不下载浏览器**；托管可执行不可用时浏览器工具被省略而其余 Agent 能力照常。外部 Agent 工具启动：`src/main/agent-tool-terminal.ts`(终端脚本常量 `claudeCodeBinaryCandidates`/`cursorAgentBinaryCandidates` 等)、`agent-tool-deeplink.ts`。

## 5.6.5 打包配置

`electron-builder.yml`(+`.unsigned.yml`/`.win.yml`/`.win.unsigned.yml`)：`appId: cn.memtensor.memmy`、`productName: Memmy`、`asar:true` 且 `asarUnpack` 解包原生 `.node`/`.dylib`/`onnxruntime-node`/`sharp-libvips`/`node-pty`/`sqlite-vec`/`@memmy/migrations`/renderer。`extraResources` 把 `dist/runtime/bin` 作为 `cli`(memmy/memmy-memory 二进制) 并拷根 `.env`。Mac 目标 `dmg`(hardenedRuntime, forceCodeSigning, entitlements `build/entitlements.mac.plist`, `notarize:false`——公证由发布脚本做)。构建资源在 `App/shell/desktop/build/`(图标、DMG 背景、`MemmyUpdatePrompt.ps1`、字体 Nunito)。

## 5.6.6 其他主进程模块

`logger.ts`/`log-level.ts`/`rotating-log-file.ts`、`desktop-edition.ts`(从 `desktop-edition.json` manifest + `MEMMY_ACCOUNT_CHANNEL` 解析 cn/intl)、`sqlite-backup.ts`(`backupSqliteDatabase`)、`mailto-url.ts`、`renderer-context-menu.ts`、`renderer-shortcuts.ts`、`project-directory-picker.ts`、`analytics-client-id-store.ts`。测试在 `App/shell/desktop/tests/`。

> 设计评价：**优点**：`boot()` 编排清晰、打包/开发态分支明确、托管 Chromium 失败优雅降级、原生模块 `asarUnpack` 处理到位、IPC 通道命名统一。**潜在改进**：`main.ts` 承担编排+IPC+窗口多职责偏重；更新/公证逻辑分散在脚本，可抽取为独立模块提升可测试性。

---

> ← [走读索引](./index.md) ｜ 上一个 → [05 桌面前端](./05-frontend.md) ｜ 返回 → [06 数据模型](../06-data-model.md)

---

# ❤️ 制作不易，请我喝咖啡☕️关注我➕

![promotion](../../res/promotion.jpg)