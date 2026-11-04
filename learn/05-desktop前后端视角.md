# apps/desktop 用「前端 / 后端」视角来理解

> 来源：扫描 `apps/desktop/`、`apps/desktop-host/`、`apps/desktop/README.md`，以及 `src/` 下的 `main.ts`、`host-process.ts`、`backend-controller.ts`、`web-document.ts`、`apps/desktop-host/src/index.ts`。

## 结论先给：可以，但要加一个限定

**可以**用「前端/后端」来理解，而且在 Desktop 这个应用里这个区分比在仓库别处更成立。但有一个**关键限定**：

> Desktop 自己**几乎不写业务**。它是一个 **Electron 薄壳（thin wrapper）**，把完整的 dsh **Web 应用** 装进一个原生窗口里。真正的「前端」和「后端」都是 Web 应用的那一套，Desktop 只是在外面套了个壳并做适配。

README 的技术决策表里原话就把它描述为 thin-wrapper：Electron 加载打包好的 Web 资源，Host 提供 boot 注入和带鉴权的 API。

## 三个角色，而不是两个

如果非要套传统架构，Desktop 场景下其实是 **三层**：

```
┌─────────────────────────────────────────────────────────┐
│  Electron 主进程（apps/desktop/src/*.ts）                  │
│  —— 原生外壳 / 宿主控制器，不是业务后端                      │
│  窗口、菜单、托盘、更新、单实例锁、快捷键、目录选择、          │
│  子进程生命周期、HTTP 转发 + 鉴权                           │
└───────────────┬───────────────────────┬───────────────────┘
                │ 加载页面                │ Node IPC + 派生子进程
                ▼                        ▼
┌───────────────────────────┐  ┌────────────────────────────┐
│  前端 = dsh Web 客户端      │  │  后端 = dsh Web Host（子进程）│
│  (packages/client, apps/web)│  │  (apps/desktop-host + Host) │
│  React UI，跑在窗口文档里    │◄─┤  profile 运行器，拥有会话、    │
│  通过 RPC/WebSocket 连后端   │HTTP│  Agent 循环、工具、持久化     │
└───────────────────────────┘ WS └────────────────────────────┘
```

## 「前端」是什么

**前端 = dsh 的 Web 客户端**，不是 Desktop 独有的东西。

- 代码主体在 `packages/client/`（shell、wire、对象服务、slot、各种 `ui-*` 插件）和 `apps/web`。
- Electron 启动后，窗口**立刻**加载打包好的 Web 入口（协议 `dsh-app://app/`）。
- 它的加载页等待 Host 的 boot 注入，然后启动 client，**不跳转到另一个文档**。
- `apps/desktop/src/web-document.ts` 的 `serveWebDocument()` 就是在本地读这些打包好的静态资源（html/js/css），并往 `index.html` 的 `<head>` 注入一段 `__DSH_BOOT_READY__` 的启动脚本。

Desktop 自己 `renderer/` 下那几个 html（`welcome.html`、`update-dialog.html`、`mandatory-update.html` 等）是**外壳自己的辅助界面**（欢迎页、更新对话框），不是主产品前端。主产品前端完全是 Web 那套 React 应用。

## 「后端」是什么

**后端 = dsh 的 Web Host 进程**，由 Electron 作为一个 **RunAsNode 子进程** 启动。

- 入口在 `apps/desktop-host/src/index.ts`：它调用 `runProfile({ profile: 'desktop', ... })` 启动 `desktop` profile，参数是 `--no-open --host 127.0.0.1 --port 0`（端口交给操作系统随机分配，避免和 Web 的 3080 冲突）。
- 这个 Host 才是真正干活的：**会话日志、Agent 主循环、工具执行、持久化、鉴权** 全在这里（也就是前面笔记里 `packages/core/*` 那套东西）。
- 它和 Electron 主进程通过 **Node IPC** 通信，传递 boot 注入、ready、shutdown、以及 platform-session / quit-inspection / update-tasks 等控制消息（见 `host-process.ts` 里的 `DesktopHostEvent` 联合类型）。
- 主进程拿到 Host 报告的 ready URL 后，用 `authenticateWebHost()` 换一个绑定 authority 的 cookie，之后窗口发出的应用 HTTP 请求由主进程转发给这个带鉴权的 Host，WebSocket 流也连到它。

所以后端是一个**独立的本地进程**，Desktop 主进程只是它的「生命周期管理者 + 鉴权代理」。

## Electron 主进程算什么

`apps/desktop/src/` 下几十个 `.ts` 文件，职责是**原生宿主外壳**，不是业务后端：

- `main.ts` — Electron 应用入口
- `host-process.ts` — 管理 Host 子进程的生命周期（spawn、IPC、ready/fatal 事件）
- `backend-controller.ts` — `DesktopBackendController`，串行化后端的启动/重试/退出，防止子进程活过已关闭的窗口
- `web-document.ts` — 本地 Web 静态资源服务 + 向 Host 转发带鉴权的 HTTP
- 其余大量文件：窗口/托盘/菜单、自动更新（`update-*`）、单实例锁、快捷键、目录选择器、崩溃上报、`preload-*`（各类 Electron preload 脚本）、平台内嵌浏览器视图等

可以把它理解成传统 Electron 应用里的「main process」—— 负责操作系统集成和把前端 webview 接到后端，本身不承载产品业务逻辑。

## 一个重要提醒：这不是普通的 client/server 分离

虽然套上「前端/后端」能快速理解 Desktop，但别忘了整个 dsh 的底座是「一切皆插件」（见 `01-Cordis框架.md`）：

- 所谓「后端」其实是一棵 **Cordis 插件树**，由 `desktop` profile 组合而成。会话、工具、模型适配器、Agent 循环都是插件，可从配置替换。
- 所谓「前端」也是插件化的（`packages/client` 下一堆 `ui-*` 插件 + slot 机制）。
- Desktop 和 Web 共享**同一套** RPC 和流实现（README 技术决策表「Transport」行：Web serving 和鉴权共用一套实现）。Desktop 没有另写一套后端，只是把 Web 那套用 Electron 包起来。

一句话收尾：**前端 = dsh Web 客户端（React/插件化 UI），后端 = dsh Web Host 子进程（Cordis 插件树，承载 Agent 循环与会话），Electron 主进程 = 把两者接起来并做 OS 集成的原生薄壳。**

## 想继续深入可以看

- `.agents/notes/implemented/architecture/2026-09-10-desktop-web-wrapper.md` — thin-wrapper 决策（前端共享行为 + Desktop 适配器）
- `.agents/notes/implemented/architecture/2026-08-25-electron-desktop-packaging-and-updates.md` — 打包/更新/发布身份
- `packages/client/README.md` — Web 前端（浏览器侧）
- `packages/host/README.md` — Web GUI host 服务
- `docs/architecture.md` 的「Desktop application」一节
