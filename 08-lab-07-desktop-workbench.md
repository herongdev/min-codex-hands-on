# Lab 07：做一个桌面工作台（Vue / Electron）

## 1. 本 Lab 最终效果

这一节我们把前面做好的后端能力，放到一个最小桌面工作台里。做完后，你会看到：

```text
选择 targetRoot
输入任务
创建 Agent Run
订阅 SSE
展示事件流
展示待审批项
点击确认 / 拒绝
查看 trace
```

这一步会让 Mini Codex 从“命令行能跑”，开始变成“日常可以操作”的工具。

## 设计能力训练：桌面端不是聊天框，而是 Agent 控制台

本节先认识这些名字：

```text
桌面工作台：一个本地应用界面，用来输入任务、看进度、确认风险和查看结果。
Vue：本节用来写界面的前端框架。
Electron：把网页界面包装成本地桌面应用的工具。
targetRoot：你希望 Mini Codex 读取和修改的目标项目目录。
timeline：按时间排列的任务过程记录。
```

当前阶段：你正在把底层 runtime 做成日常可用的工作台。用户不需要知道 run.json 在哪，只需要看清任务、审批、风险和结果。

这一阶段的意义是：用户使用 coding agent 时最需要的是掌控感，而不是一个漂亮输入框。桌面端必须让用户看见当前 phase、计划、风险、diff、审批、验证、trace。

先自己画一版界面信息架构，再看实现。可以很粗糙，只要想清楚信息顺序：

```text
首页应该优先展示输入框，还是当前任务状态？
审批区域要展示哪些风险信息？
diff、trace、event timeline 应该如何关联？
如果 verify 失败，UI 应该鼓励用户 repair 还是先阅读失败原因？
```

常见坑：

```text
把桌面端做成普通聊天 UI。
没有展示当前 phase，用户不知道任务卡在哪。
审批按钮没有展示工具名、参数、风险原因。
事件流、diff、trace 分散，无法排查问题。
```

常见概念：

```text
workbench
timeline
approval console
diff viewer
target picker
trace inspector
local bridge
```

最新技术对照：Codex、Claude Code 这类工具的核心体验不是“聊天”，而是围绕真实代码任务提供上下文、动作、审批和结果。桌面端要把这些运行时状态产品化，而不是只套一个对话界面。

面试常问：

```text
Coding agent workbench 和 chatbot UI 有什么区别？
高风险操作如何做交互设计？
为什么桌面端要展示 trace？
本地桌面应用相比纯 Web 的优势是什么？
```

做完本 Lab 后，可以继续问自己：当前桌面端是最小工作台，如果你每天真的用它写代码，最先要补的三个体验能力是什么？

## 2. 安装依赖

`apps/desktop/package.json`：

```json
{
  "name": "@minicodex/desktop",
  "version": "0.1.0",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "check": "vue-tsc --noEmit",
    "electron": "electron electron/main.cjs"
  },
  "dependencies": {
    "axios": "^1.7.7",
    "electron": "^32.1.2",
    "pinia": "^2.2.2",
    "vue": "^3.5.10"
  },
  "devDependencies": {
    "@tailwindcss/vite": "^4.3.1",
    "@vitejs/plugin-vue": "^5.1.4",
    "@vue/tsconfig": "^0.5.1",
    "tailwindcss": "^4.3.1",
    "lightningcss": "^1.32.0",
    "typescript": "^5.4.5",
    "vite": "^5.4.8",
    "vue-tsc": "^2.1.6"
  }
}
```

**依赖分类**：`vue` / `pinia` / `axios` 是渲染进程运行时依赖；`vite`、`@vitejs/plugin-vue`、`tailwindcss`、`@tailwindcss/vite`、`typescript`、`vue-tsc` 只在 dev/build 时用，放 `devDependencies`。`electron` 在本 Lab 仍放 `dependencies`，因为 `pnpm run electron` 需要它；严格桌面工程里也可改为 `-D`，本教程先保持不动。
```

如果你想快速起步，也可以用：

```bash
pnpm create vite apps/desktop --template vue-ts
pnpm install
pnpm --dir apps/desktop add -D tailwindcss @tailwindcss/vite
```

然后再补下面文件（模板已自带 `vite.config.ts` 等，可对照教程按需改）。

如果你手动创建，不走 Vite 模板，先建目录：

```bash
mkdir -p apps/desktop/src/services
mkdir -p apps/desktop/src/stores
mkdir -p apps/desktop/electron
```

**先安装依赖**（写 `vite.config.ts` 之前就要跑，否则 IDE / `vue-tsc` 会报找不到 `vite`、`@vitejs/plugin-vue`、`@tailwindcss/vite`）：

```bash
pnpm install
pnpm --dir apps/desktop add -D tailwindcss @tailwindcss/vite lightningcss
```

再补 `apps/desktop/vite.config.ts`（`tailwindcss()` 返回 `Plugin[]`，与 `vue()` 并存时写 `...tailwindcss()`）：

```ts
import tailwindcss from "@tailwindcss/vite";
import vue from "@vitejs/plugin-vue";
import { defineConfig } from "vite";

const desktopUrl = process.env.MINICODEX_DESKTOP_URL ?? "http://localhost:5173";
const desktopPort = Number(new URL(desktopUrl).port || 5173);

export default defineConfig({
  plugins: [vue(), ...tailwindcss()],
  server: {
    port: desktopPort,
  },
});
```

补 `apps/desktop/tsconfig.json`：

```json
{
  "extends": "@vue/tsconfig/tsconfig.dom.json",
  "compilerOptions": {
    "composite": true,
    "baseUrl": "."
  },
  "include": ["src/**/*.ts", "src/**/*.vue", "vite.config.ts"]
}
```

补 `apps/desktop/src/env.d.ts`：

```ts
/// <reference types="vite/client" />

export {};

declare global {
  interface ImportMetaEnv {
    readonly VITE_MINICODEX_BACKEND_URL?: string;
  }

  interface Window {
    miniCodexDesktop?: {
      selectDirectory: () => Promise<string>;
    };
  }
}
```

本节其余 desktop 文件都补完后，再运行：

```bash
pnpm --dir apps/desktop run check
```

## 2.1 跟练前确认

本 Lab 从 Lab 06 完成后的后端继续，工作目录固定为：

macOS / Linux：
```bash
cd ~/mini-codex-course/playground/minicodex-lab
```

Windows PowerShell：

```powershell
Set-Location "$HOME\mini-codex-course\playground\minicodex-lab"
```

本节至少开三个终端：

```text
终端 A：pnpm dev:backend
终端 B：pnpm dev:desktop
终端 C：pnpm dev:electron
```

也可以 **一条命令全起**（见 §7 的 `pnpm dev:all`）。拆三个终端的好处是日志分开、方便单独重启某一个进程；日常跟练用 `dev:all` 更省事。

默认后端地址来自 Lab 06 的 `.env.local`。本 Lab 会新增 `apps/desktop/src/config.ts`，统一维护桌面端请求地址。你平时项目多、3000 容易冲突时，只改 `.env.local`：

```env
MINICODEX_BACKEND_PORT=3100
VITE_MINICODEX_BACKEND_URL=http://localhost:3100
MINICODEX_DESKTOP_URL=http://localhost:5173
```

注意：`MINICODEX_BACKEND_PORT` 决定后端监听端口；`VITE_MINICODEX_BACKEND_URL` 决定桌面前端请求哪个后端地址。如果你把后端从 3000 改到 3100，前后两边要保持一致。

### 2.1.1 环境变量：为什么看起来像「两套」，其实是一份配置

根目录 `package.json` 里会有：

```json
"dev:backend": "dotenv -e .env.local -- pnpm --dir apps/backend run dev",
"dev:desktop": "dotenv -e .env.local -- pnpm --dir apps/desktop run dev",
"dev:electron": "dotenv -e .env.local -- pnpm --dir apps/desktop run electron"
```

你可能会问：Vite 不是会自动读 `.env` 吗，为什么还要 `dotenv -e .env.local`？

**结论：值只维护一份（`.env.local`）；加载方式有两种，这是浏览器和 Node 的边界，不是重复设计。**

```text
minicodex-lab/.env.local          ← 唯一配置源（复制 .env.example）
        │
        ├─ dotenv-cli ──→ process.env
        │                    ├─ backend：MINICODEX_BACKEND_PORT
        │                    └─ electron：MINICODEX_DESKTOP_URL
        │
        └─ Vite ──→ import.meta.env
                      └─ desktop：VITE_MINICODEX_BACKEND_URL（仅 VITE_ 前缀）
```

对照表：

| 变量                         | 谁读                     | 怎么读                       | 用途                        |
| ---------------------------- | ------------------------ | ---------------------------- | --------------------------- |
| `MINICODEX_BACKEND_PORT`     | backend（Express / tsx） | `dotenv-cli` → `process.env` | 后端监听端口                |
| `VITE_MINICODEX_BACKEND_URL` | desktop（浏览器 bundle） | Vite → `import.meta.env`     | axios / SSE 请求地址        |
| `MINICODEX_DESKTOP_URL`      | electron                 | `dotenv-cli` → `process.env` | Electron 加载哪个 Vite 页面 |

为什么要分开：

1. **Backend 不是 Vite 进程**。Node 不会自动读 `.env`，不用 `dotenv-cli`（或代码里 `import 'dotenv/config'`），`MINICODEX_BACKEND_PORT` 进不了 `process.env`。
2. **Vite 只暴露 `VITE_` 前缀给前端**。这是安全约定：避免把密钥误打包进浏览器。所以后端端口不能指望「Vite 自动读 env」解决。
3. **`.env.local` 放在 lab 根目录**。Vite 默认只在 `apps/desktop/` 下找 env 文件；用 `dotenv -e .env.local` 是为了让 backend / desktop / electron **共用根目录同一份配置**。

前端代码里仍保留 `config.ts` 的默认值：

```ts
import.meta.env.VITE_MINICODEX_BACKEND_URL || "http://localhost:3000";
```

含义是：**没配 env 时**跟 Lab 06 默认端口 3000 对齐；配了 `.env.local` 后以 env 为准。

本 Lab **保持现状即可**：不必为了「看起来统一」把 backend 也改成 Vite，或在 Lab 07 引入更重的 env 平台。以后要收一点脚本，可以单独做：`vite.config.ts` 设 `envDir: '../../'`，或 backend 在 `main.ts` 里内置 dotenv——那是可选优化，不是本 Lab 必做项。

桌面端只调用 HTTP API 和 SSE，不直接读写 `.minicodex` 文件。它的目标是把已有 runtime 状态展示清楚，不在前端重写 plan、diff、apply、verify。

若**不用** `pnpm create vite`，而是手动创建 `apps/desktop`，请按 **§2.2 完整文件清单** 逐项补全；只写 `App.vue` 无法启动。

### 2.2 手动创建：完整文件清单

不走 Vite 模板时，本节结束前 `apps/desktop/` 应有这些文件：

```text
apps/desktop/
├── package.json                 ← §2 已给出
├── vite.config.ts               ← §2 已给出
├── tsconfig.json                ← §2 已给出
├── index.html                   ← 本节 2.2.1
├── electron/
│   ├── main.cjs                 ← 本节 2.2.6 / §6
│   └── preload.cjs              ← 本节 2.2.6 / §6
└── src/
    ├── env.d.ts                 ← §2 已给出
    ├── config.ts                ← 本节 2.2.2
    ├── main.ts                  ← 本节 2.2.1
    ├── style.css                ← 本节 2.2.1
    ├── App.vue                  ← 本节 2.2.1 占位，2.2.5 / §5 替换为完整版
    ├── services/
    │   └── agentRunService.ts   ← 本节 2.2.3
    └── stores/
        └── workbench.ts         ← 本节 2.2.4
```

**建议顺序**：§2 的 `package.json` → `mkdir` → `pnpm install` → §2 的 `vite.config.ts` / `tsconfig.json` / `env.d.ts` → **2.2.1 启动骨架** → 2.2.2 配置 → 2.2.3–2.2.4 数据层 → 2.2.5 UI（§5）→ 2.2.6 Electron（§6，可后补）→ 2.2.7 自检。

#### 2.2.1 启动骨架（缺了无法 `vite dev`）

`apps/desktop/index.html`：

```html
<!DOCTYPE html>
<html lang="zh-CN">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Mini Codex Desktop</title>
  </head>
  <body>
    <div id="app"></div>
    <script type="module" src="/src/main.ts"></script>
  </body>
</html>
```

`apps/desktop/src/main.ts`：

```ts
import { createPinia } from "pinia";
import { createApp } from "vue";
import App from "./App.vue";
import "./style.css";

createApp(App).use(createPinia()).mount("#app");
```

`apps/desktop/src/style.css`：

```css
@import "tailwindcss";
```

`apps/desktop/src/App.vue`（**占位版**，先让 Vite 能编译；§2.2.5 / §5 再替换成完整工作台）：

```vue
<template>
  <main style="padding: 16px; font-family: system-ui, sans-serif">
    <h1>Mini Codex Desktop</h1>
    <p>Vite 已启动。接下来补 API / Store，再回来改这个文件。</p>
  </main>
</template>
```

补完后可先验证 Vite 能起（应看到标题 “Mini Codex Desktop”，而不是编译报错）：

```bash
pnpm dev:desktop
```

#### 2.2.2 前端配置

`apps/desktop/src/config.ts`：

```ts
const DEFAULT_BACKEND_URL = "http://localhost:3000";

export const MINICODEX_BACKEND_URL = (
  import.meta.env.VITE_MINICODEX_BACKEND_URL || DEFAULT_BACKEND_URL
).replace(/\/$/, "");

/** 生成当前 run 的 SSE 事件流地址。 */
export function agentRunEventsUrl(runId: string): string {
  return `${MINICODEX_BACKEND_URL}/agent-runs/${runId}/events`;
}
```

这一层只有一个职责：维护桌面端要请求的后端地址。以后端口冲突时，不要去 `agentRunService.ts` 和 store 里到处搜索 `3000`，优先改启动环境变量 `VITE_MINICODEX_BACKEND_URL`。

#### 2.2.3 API 封装

`apps/desktop/src/services/agentRunService.ts`：

```ts
import axios from "axios";
import { MINICODEX_BACKEND_URL, agentRunEventsUrl } from "../config";

const api = axios.create({
  baseURL: MINICODEX_BACKEND_URL,
});

/** 创建一条后端 Agent Run，并返回最新 run 状态。 */
export async function createAgentRun(params: {
  message: string;
  targetRoot?: string;
}) {
  const response = await api.post("/agent-runs", params);
  return response.data;
}

/** 提交人工审批结果，让后端 Runtime 决定下一步。 */
export async function submitApproval(runId: string, approved: boolean) {
  const response = await api.post(`/agent-runs/${runId}/approval`, {
    approved,
  });
  return response.data;
}

/** 读取某个 run 的 trace 事件，用于桌面端回放过程。 */
export async function replayTrace(runId: string) {
  const response = await api.get(`/agent-runs/${runId}/trace`);
  return response.data;
}

/** 给 store 暴露事件流 URL，避免 store 直接拼后端地址。 */
export function eventsUrl(runId: string): string {
  return agentRunEventsUrl(runId);
}
```

#### 2.2.4 Pinia Store

`apps/desktop/src/stores/workbench.ts`：

```ts
import { defineStore } from "pinia";
import { computed, ref } from "vue";
import {
  createAgentRun,
  eventsUrl,
  replayTrace,
  submitApproval,
} from "../services/agentRunService";

export const useWorkbenchStore = defineStore("workbench", () => {
  const message = ref("把登录按钮文案改成提交");
  const targetRoot = ref("fixtures/target-repo");
  const currentRun = ref<any>(null);
  const traceLines = ref<any[]>([]);
  const errorMessage = ref("");
  const isRunning = ref(false);
  let eventSource: EventSource | null = null;

  const events = computed(() => currentRun.value?.events ?? []);
  const pendingApproval = computed(
    () => currentRun.value?.pendingApproval ?? null,
  );

  /** 建立 SSE 订阅，让桌面端持续收到后端 run 状态变化。 */
  function subscribe(runId: string): void {
    eventSource?.close();
    eventSource = new EventSource(eventsUrl(runId));
    eventSource.onmessage = (event) => {
      currentRun.value = JSON.parse(event.data);
      if (currentRun.value.status !== "running") {
        isRunning.value = false;
      }
    };
    eventSource.onerror = () => {
      errorMessage.value = "事件流连接失败";
      isRunning.value = false;
      eventSource?.close();
    };
  }

  /** 从输入框内容创建新任务，并立即订阅它的事件流。 */
  async function startRun(): Promise<void> {
    errorMessage.value = "";
    isRunning.value = true;
    traceLines.value = [];
    try {
      currentRun.value = await createAgentRun({
        message: message.value,
        targetRoot: targetRoot.value,
      });
      subscribe(currentRun.value.id);
    } catch (error) {
      errorMessage.value =
        error instanceof Error ? error.message : "创建任务失败";
      isRunning.value = false;
    }
  }

  /** 提交当前 pending approval 的通过或拒绝结果。 */
  async function approve(approved: boolean): Promise<void> {
    if (!currentRun.value) return;
    isRunning.value = true;
    try {
      currentRun.value = await submitApproval(currentRun.value.id, approved);
      subscribe(currentRun.value.id);
    } catch (error) {
      errorMessage.value = error instanceof Error ? error.message : "审批失败";
      isRunning.value = false;
    }
  }

  /** 手动读取 trace，用于查看这次任务的完整过程。 */
  async function loadTrace(): Promise<void> {
    if (!currentRun.value) return;
    const response = await replayTrace(currentRun.value.id);
    traceLines.value = response.events ?? [];
  }

  /** 调用 Electron preload 暴露的目录选择能力，更新 targetRoot。 */
  async function selectTargetRoot(): Promise<void> {
    const selected = await window.miniCodexDesktop?.selectDirectory();
    if (selected) targetRoot.value = selected;
  }

  return {
    message,
    targetRoot,
    currentRun,
    traceLines,
    events,
    pendingApproval,
    errorMessage,
    isRunning,
    startRun,
    approve,
    loadTrace,
    selectTargetRoot,
  };
});
```

#### 2.2.5 主页面（替换占位 `App.vue`）

把 **§2.2.1** 里的占位 `App.vue` **整文件替换** 为下面 §5 的完整工作台 UI。

#### 2.2.6 Electron 目录选择（可后补）

先用手动输入 `targetRoot` 跑通 Web 工作台，再补：

- `apps/desktop/electron/main.cjs` — 见 **§6**
- `apps/desktop/electron/preload.cjs` — 见 **§6**

#### 2.2.7 自检

```bash
pnpm --dir apps/desktop run check
pnpm dev:desktop
```

浏览器打开 `http://localhost:5173`，应能看到 Mini Codex 工作台。默认会请求 `http://localhost:3000` 的 Lab 06 后端；如果你改过后端端口，用 `VITE_MINICODEX_BACKEND_URL=http://localhost:<你的端口>` 启动桌面前端。

## 3. 前端 API

> 手动创建时，`config.ts` 和 `agentRunService.ts` 已在 **§2.2.2–2.2.3** 给出；走 `pnpm create vite` 则从这里开始跟练。

先新建统一配置：

`apps/desktop/src/config.ts`：

```ts
const DEFAULT_BACKEND_URL = "http://localhost:3000";

export const MINICODEX_BACKEND_URL = (
  import.meta.env.VITE_MINICODEX_BACKEND_URL || DEFAULT_BACKEND_URL
).replace(/\/$/, "");

/** 生成当前 run 的 SSE 事件流地址。 */
export function agentRunEventsUrl(runId: string): string {
  return `${MINICODEX_BACKEND_URL}/agent-runs/${runId}/events`;
}
```

`apps/desktop/src/services/agentRunService.ts`：

```ts
import axios from "axios";
import { MINICODEX_BACKEND_URL, agentRunEventsUrl } from "../config";

const api = axios.create({
  baseURL: MINICODEX_BACKEND_URL,
});

/** 创建一条后端 Agent Run，并返回最新 run 状态。 */
export async function createAgentRun(params: {
  message: string;
  targetRoot?: string;
}) {
  const response = await api.post("/agent-runs", params);
  return response.data;
}

/** 提交人工审批结果，让后端 Runtime 决定下一步。 */
export async function submitApproval(runId: string, approved: boolean) {
  const response = await api.post(`/agent-runs/${runId}/approval`, {
    approved,
  });
  return response.data;
}

/** 读取某个 run 的 trace 事件，用于桌面端回放过程。 */
export async function replayTrace(runId: string) {
  const response = await api.get(`/agent-runs/${runId}/trace`);
  return response.data;
}

/** 给 store 暴露事件流 URL，避免 store 直接拼后端地址。 */
export function eventsUrl(runId: string): string {
  return agentRunEventsUrl(runId);
}
```

## 4. Pinia Store

> 手动创建时，`workbench.ts` 已在 **§2.2.4** 给出。

`apps/desktop/src/stores/workbench.ts`：

```ts
import { defineStore } from "pinia";
import { computed, ref } from "vue";
import {
  createAgentRun,
  eventsUrl,
  replayTrace,
  submitApproval,
} from "../services/agentRunService";

export const useWorkbenchStore = defineStore("workbench", () => {
  const message = ref("把登录按钮文案改成提交");
  const targetRoot = ref("fixtures/target-repo");
  const currentRun = ref<any>(null);
  const traceLines = ref<any[]>([]);
  const errorMessage = ref("");
  const isRunning = ref(false);
  let eventSource: EventSource | null = null;

  const events = computed(() => currentRun.value?.events ?? []);
  const pendingApproval = computed(
    () => currentRun.value?.pendingApproval ?? null,
  );

  /** 建立 SSE 订阅，让桌面端持续收到后端 run 状态变化。 */
  function subscribe(runId: string): void {
    eventSource?.close();
    eventSource = new EventSource(eventsUrl(runId));
    eventSource.onmessage = (event) => {
      currentRun.value = JSON.parse(event.data);
      if (currentRun.value.status !== "running") {
        isRunning.value = false;
      }
    };
    eventSource.onerror = () => {
      errorMessage.value = "事件流连接失败";
      isRunning.value = false;
      eventSource?.close();
    };
  }

  /** 从输入框内容创建新任务，并立即订阅它的事件流。 */
  async function startRun(): Promise<void> {
    errorMessage.value = "";
    isRunning.value = true;
    traceLines.value = [];
    try {
      currentRun.value = await createAgentRun({
        message: message.value,
        targetRoot: targetRoot.value,
      });
      subscribe(currentRun.value.id);
    } catch (error) {
      errorMessage.value =
        error instanceof Error ? error.message : "创建任务失败";
      isRunning.value = false;
    }
  }

  /** 提交当前 pending approval 的通过或拒绝结果。 */
  async function approve(approved: boolean): Promise<void> {
    if (!currentRun.value) return;
    isRunning.value = true;
    try {
      currentRun.value = await submitApproval(currentRun.value.id, approved);
      subscribe(currentRun.value.id);
    } catch (error) {
      errorMessage.value = error instanceof Error ? error.message : "审批失败";
      isRunning.value = false;
    }
  }

  /** 手动读取 trace，用于查看这次任务的完整过程。 */
  async function loadTrace(): Promise<void> {
    if (!currentRun.value) return;
    const response = await replayTrace(currentRun.value.id);
    traceLines.value = response.events ?? [];
  }

  /** 调用 Electron preload 暴露的目录选择能力，更新 targetRoot。 */
  async function selectTargetRoot(): Promise<void> {
    const selected = await window.miniCodexDesktop?.selectDirectory();
    if (selected) targetRoot.value = selected;
  }

  return {
    message,
    targetRoot,
    currentRun,
    traceLines,
    events,
    pendingApproval,
    errorMessage,
    isRunning,
    startRun,
    approve,
    loadTrace,
    selectTargetRoot,
  };
});
```

## 4.1 调用链和数据流

桌面端不要直接读写 `.minicodex` 文件，也不要自己推进状态机。它只通过后端 API 操作 Runtime：

```text
App.vue 点击“开始”
-> store.startRun()
-> createAgentRun()
-> POST /agent-runs
-> backend runtime.createRun()
-> store.subscribe(runId)
-> EventSource 接收最新 run
-> App.vue 根据 currentRun 渲染状态、事件和审批卡片
```

审批时：

```text
App.vue 点击“确认”
-> store.approve(true)
-> submitApproval()
-> POST /agent-runs/:id/approval
-> backend runtime.approveRun()
-> SSE 推送新 run
-> pendingApproval / events / status 自动刷新
```

查看 trace 时：

```text
App.vue 点击“查看 Trace”
-> store.loadTrace()
-> replayTrace()
-> GET /agent-runs/:id/trace
-> traceLines 更新
```

这就是桌面端的分层边界：

```text
config.ts：只管后端地址。
agentRunService.ts：只管 HTTP 请求。
workbench.ts：只管页面状态和用户动作。
App.vue：只管展示和触发 store 方法。
```

## 5. Vue 入口和页面

> 手动创建时，`index.html` / `main.ts` / `style.css` / 占位 `App.vue` 已在 **§2.2.1** 给出。下面用完整版**覆盖** `App.vue`。本节使用 Tailwind CSS v4 的 Vite 插件方式，不再写一大段 scoped CSS。

`apps/desktop/src/App.vue`：

```vue
<script setup lang="ts">
import { useWorkbenchStore } from "./stores/workbench";

const store = useWorkbenchStore();
</script>

<template>
  <main class="min-h-screen bg-slate-100 text-slate-950">
    <div class="grid min-h-screen grid-cols-[280px_1fr]">
      <aside class="border-r border-slate-200 bg-white px-5 py-5">
        <h2 class="text-lg font-semibold">Mini Codex</h2>

        <label class="mt-6 block text-sm font-medium text-slate-700">
          Target Root
          <input
            v-model="store.targetRoot"
            class="mt-2 w-full rounded-md border border-slate-300 bg-white px-3 py-2 text-sm outline-none focus:border-sky-500 focus:ring-2 focus:ring-sky-100"
          />
        </label>

        <button
          class="mt-3 w-full rounded-md border border-slate-300 bg-white px-3 py-2 text-sm font-medium hover:bg-slate-50"
          @click="store.selectTargetRoot"
        >
          选择目录
        </button>
      </aside>

      <section class="max-w-5xl px-6 py-5">
        <textarea
          v-model="store.message"
          rows="3"
          class="w-full rounded-md border border-slate-300 bg-white px-3 py-2 text-sm outline-none focus:border-sky-500 focus:ring-2 focus:ring-sky-100"
        />

        <div class="mt-3 flex gap-2">
          <button
            :disabled="store.isRunning"
            class="rounded-md bg-sky-600 px-4 py-2 text-sm font-medium text-white hover:bg-sky-700 disabled:cursor-not-allowed disabled:bg-slate-300"
            @click="store.startRun"
          >
            Run
          </button>
          <button
            :disabled="!store.currentRun"
            class="rounded-md border border-slate-300 bg-white px-4 py-2 text-sm font-medium hover:bg-slate-50 disabled:cursor-not-allowed disabled:text-slate-400"
            @click="store.loadTrace"
          >
            Trace
          </button>
        </div>

        <p v-if="store.errorMessage" class="mt-3 text-sm text-red-700">
          {{ store.errorMessage }}
        </p>

        <div
          v-if="store.pendingApproval"
          class="mt-4 rounded-md border border-amber-300 bg-amber-50 p-4"
        >
          <h3 class="text-sm font-semibold text-amber-950">待确认</h3>
          <p class="mt-2 text-sm text-amber-900">
            {{ store.pendingApproval.question }}
          </p>
          <p class="mt-1 text-sm text-amber-800">
            {{ store.pendingApproval.reason }}
          </p>
          <div class="mt-3 flex gap-2">
            <button
              class="rounded-md bg-emerald-600 px-3 py-2 text-sm font-medium text-white hover:bg-emerald-700"
              @click="store.approve(true)"
            >
              确认
            </button>
            <button
              class="rounded-md border border-slate-300 bg-white px-3 py-2 text-sm font-medium hover:bg-slate-50"
              @click="store.approve(false)"
            >
              拒绝
            </button>
          </div>
        </div>

        <article
          v-for="event in store.events"
          :key="event.id"
          class="mt-4 rounded-md border border-slate-200 bg-white p-4"
        >
          <strong class="text-sm text-slate-900">
            {{ event.type }} - {{ event.title }}
          </strong>
          <pre class="mt-2 whitespace-pre-wrap text-xs text-slate-700">{{ event.content }}</pre>
        </article>

        <section v-if="store.traceLines.length" class="mt-8">
          <h3 class="text-base font-semibold">Trace</h3>
          <article
            v-for="line in store.traceLines"
            :key="line.eventIndex"
            class="mt-4 rounded-md border border-slate-200 bg-white p-4"
          >
            <strong class="text-sm text-slate-900">
              [{{ line.eventIndex }}] {{ line.type }}
            </strong>
            <pre class="mt-2 whitespace-pre-wrap text-xs text-slate-700">{{ line.event.title }}</pre>
          </article>
        </section>
      </section>
    </div>
  </main>
</template>
```

## 6. Electron 选择目录

> 手动创建时，文件清单见 **§2.2.6**。

`apps/desktop/electron/main.cjs`：

```js
const { app, BrowserWindow, dialog, ipcMain } = require("electron");

function createWindow() {
  const win = new BrowserWindow({
    width: 1200,
    height: 800,
    webPreferences: {
      contextIsolation: true,
      nodeIntegration: false,
      preload: __dirname + "/preload.cjs",
    },
  });
  win.loadURL(process.env.MINICODEX_DESKTOP_URL || "http://localhost:5173");
}

ipcMain.handle("minicodex:select-directory", async () => {
  const result = await dialog.showOpenDialog({
    properties: ["openDirectory"],
  });
  return result.canceled ? "" : result.filePaths[0];
});

app.whenReady().then(createWindow);
```

`apps/desktop/electron/preload.cjs`：

```js
const { contextBridge, ipcRenderer } = require("electron");

// 只暴露桌面端需要的安全 API，供 Vue 页面调用。
contextBridge.exposeInMainWorld("miniCodexDesktop", {
  // 打开系统文件夹选择框；具体权限和对话框由主进程处理。
  selectDirectory: () => ipcRenderer.invoke("minicodex:select-directory"),
});
```

## 7. 运行

### 7.1 一条命令启动（推荐日常跟练）

在 lab 根目录：

```bash
pnpm dev:all
```

会并行启动 backend、Vite 桌面端，等 `.env.local` 里的 `MINICODEX_DESKTOP_URL` 就绪后再开 Electron。`Ctrl+C` 一次会关掉三个进程（`concurrently -k`）。

根目录 `package.json` 里对应脚本：

```json
"dev:electron:wait": "sh -c 'wait-on \"${MINICODEX_DESKTOP_URL:-http://127.0.0.1:5173}\" && pnpm --dir apps/desktop run electron'",
"dev:all": "dotenv -e .env.local -- concurrently -k -n backend,desktop,electron \"pnpm --dir apps/backend run dev\" \"pnpm --dir apps/desktop run dev\" \"pnpm dev:electron:wait\""
```

需要 devDependencies：`concurrently`、`wait-on`（与 `dotenv-cli` 一起在 lab 根安装）。

`wait-on` 读的是 **`MINICODEX_DESKTOP_URL`**（与 Electron 加载地址相同），不用在脚本里手写 `5173`。`apps/desktop/vite.config.ts` 也会从同一变量解析 `server.port`，改 `.env.local` 一处即可，例如：

```env
MINICODEX_DESKTOP_URL=http://localhost:5174
```

后端端口仍用 **`MINICODEX_BACKEND_PORT`** + **`VITE_MINICODEX_BACKEND_URL`** 配对修改（见 §2.1.1）。

### 7.2 三终端分别启动（排查问题时更清晰）

启动后端：

```bash
pnpm dev:backend
```

启动前端：

```bash
pnpm dev:desktop
```

打开 Electron：

```bash
pnpm dev:electron
```

期望界面能完成这条最小路径：

```text
输入 targetRoot 和任务
点击 Run
看到事件流
看到待确认卡片
点击确认
继续看到下一个审批点或 delivery
```

如果页面提示“事件流连接失败”，先确认后端实际地址和 `VITE_MINICODEX_BACKEND_URL` 一致，再用 Lab 06 的 curl 测一次 `/agent-runs/<runId>/events`。例如后端在 3100：

```bash
curl http://localhost:3100/agent-runs/<runId>/events
```

前端问题和后端问题要分开排查。若 curl 能连通但页面连不上，优先检查 `apps/desktop/src/config.ts`、启动命令里的 `VITE_MINICODEX_BACKEND_URL`，以及浏览器控制台里的请求地址。

## 8. 本 Lab 验收清单

- [ ] 页面能创建 Agent Run。
- [ ] SSE 能更新事件流。
- [ ] approval 卡片能确认 plan/diff/tool。
- [ ] 完整跑到 delivery。
- [ ] 能查看每个 artifact 的内容。
- [ ] 手动输入 targetRoot 能跑通；接入 directory picker 后能选择本地 targetRoot。
- [ ] 刷新页面或重开 Electron 后，不会绕过后端状态重新写文件；历史恢复可以记录为后续增强。
- [ ] UI 没有绕过后端权限，所有审批仍由 backend/runtime 推进。

## 9. 可选参考实现

如果你手里有自己的参考项目，可以对照这些桌面工作台分层。它们只是参考方向，不是本课程的唯一标准答案：

```text
frontend/src/pages/codex-workbench/index.vue
frontend/src/stores/agentRunWorkbench.ts
frontend/src/components/agent-workbench/AgentChatCenter.vue
frontend/src/components/agent-workbench/ApprovalDock.vue
frontend/electron/main.cjs
frontend/electron/preload.cjs
```

## 10. 下一步

进入 [Lab 07B：让桌面界面更清楚、更可信](./09-lab-07b-ui-polish-design-system.md)。
