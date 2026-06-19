# Lab 06：把命令行能力提供给界面使用（HTTP API / SSE）

## 1. 本 Lab 最终效果

这一节我们把已经能跑的 CLI Runtime 包成 HTTP 服务。做完后，桌面端就可以通过 API 和事件流观察它：

```text
POST /agent-runs
GET /agent-runs/:id
POST /agent-runs/:id/approval
GET /agent-runs/:id/trace
SSE /agent-runs/:id/events
```

桌面端不会直接读写 `.minicodex` 文件，而是通过后端 API 和 SSE 观察任务变化。

## 设计能力训练：把本地 runtime 变成可被产品使用的服务

本节先认识这些名字：

```text
后端：给界面提供能力的一层服务，界面通过它创建任务、查询状态、提交确认。
API：界面和后端约定好的调用入口，例如创建任务、查看任务详情。
HTTP：最常见的网页/接口通信方式。
SSE：一种持续推送消息的方式，后端可以把任务进度实时推给界面。
事件流：按顺序推送出来的一条条任务变化。
```

当前阶段：你正在把 CLI 已经跑通的能力，整理成产品可以调用的 HTTP API 和事件流。

这一阶段的意义是：桌面端不应该知道 `.minicodex/runs` 怎么存，也不应该直接推进状态机。UI 只表达用户意图，真正的任务创建、审批、trace 读取都由后端转给 core runtime。

先自己设计一版 API，再看实现：

```text
创建任务应该是 POST 还是 GET？
审批应该是单独接口，还是更新整个 run？
实时进度用轮询、SSE 还是 WebSocket？
后端如何把 runtime 错误变成前端能理解的事件？
```

常见坑：

```text
前端直接读写本地 run.json。
后端只返回最终结果，无法观察长任务过程。
SSE 断开后没有重连和补偿读取。
API 直接暴露内部数据结构，后面难以演进。
```

常见概念：

```text
agent server
REST API
SSE
event stream
async job
long-running task
backpressure
```

最新技术对照：Coding agent 的任务通常不是同步请求，而是持续运行的异步任务。Codex 类产品需要让用户看到执行进度和等待审批点，所以这里用 API + SSE 把 runtime 服务化。

面试常问：

```text
为什么 agent 任务通常是长任务 API？
SSE 和 WebSocket 怎么选？
后端如何支持多个桌面端窗口观察同一个 run？
如何设计 approval API 才不会误操作？
```

做完本 Lab 后，可以继续想：当前后端和 CLI 共用同一个本地 rootDir，如果未来支持多用户、多 workspace、多进程，存储和锁该怎么改？

## 2. 安装依赖

先确认目录存在：

```bash
mkdir -p apps/backend/src
```

本 Lab 开始会用 `.env.local` 管理本机端口，避免 3000 和其它项目冲突。先在根目录安装 dotenv 启动器：

```bash
pnpm add -Dw dotenv-cli
```

在项目根目录的 `.env.example` 写入可提交的示例配置：

```env
MINICODEX_BACKEND_PORT=3100
VITE_MINICODEX_BACKEND_URL=http://localhost:3100
MINICODEX_DESKTOP_URL=http://localhost:5173
```

在项目根目录的 `.env.local` 写入你本机实际配置：

```env
MINICODEX_BACKEND_PORT=3100
VITE_MINICODEX_BACKEND_URL=http://localhost:3100
MINICODEX_DESKTOP_URL=http://localhost:5173
```

`.env.local` 是本机偏好，不应该提交。根目录 `.gitignore` 至少加：

```gitignore
.env.local
```

然后把根目录 `package.json` 的脚本改成统一加载 `.env.local`：

```json
{
  "scripts": {
    "build": "pnpm -r run build",
    "check": "pnpm -r run check",
    "minicodex": "pnpm --dir packages/cli run dev",
    "dev:backend": "dotenv -e .env.local -- pnpm --dir apps/backend run dev",
    "dev:desktop": "dotenv -e .env.local -- pnpm --dir apps/desktop run dev",
    "dev:electron": "dotenv -e .env.local -- pnpm --dir apps/desktop run electron"
  }
}
```

这样后面启动时就不用每次手写端口环境变量。`.env.example` 记录团队默认值，`.env.local` 记录你本机实际值。

在 `apps/backend/package.json`：

```json
{
  "name": "@minicodex/backend",
  "version": "0.1.0",
  "type": "module",
  "scripts": {
    "dev": "tsx src/main.ts",
    "build": "tsc -p tsconfig.json",
    "check": "tsc -p tsconfig.json --noEmit"
  },
  "dependencies": {
    "@minicodex/core": "workspace:*",
    "cors": "^2.8.5",
    "express": "^4.19.2"
  },
  "devDependencies": {
    "@types/cors": "^2.8.17",
    "@types/express": "^4.17.21"
  }
}
```

`apps/backend/tsconfig.json`：

```json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "rootDir": "src",
    "outDir": "dist"
  },
  "include": ["src"]
}
```

安装：

```bash
pnpm install
```

此时 **`apps/backend/src/` 里还没有 `.ts` 文件**（只有占位目录），若立刻执行 `pnpm --dir apps/backend run build`，会报：

```text
error TS18003: No inputs were found in config file ...
Specified 'include' paths were '["src"]'
```

这是正常的：**先完成 §3 写出 `apps/backend/src/main.ts`，再 build**。§4 若加了 SSE 路由，改完后再 build 一次即可。

```bash
# §3 写完 main.ts 之后
pnpm --dir apps/backend run build
```

## 2.1 跟练前确认

本 Lab 从 Lab 05 完成后的代码继续，工作目录固定为：

macOS / Linux：
```bash
cd ~/mini-codex-course/playground/minicodex-lab
```

Windows PowerShell：

```powershell
Set-Location "$HOME\mini-codex-course\playground\minicodex-lab"
```

后端是 CLI runtime 的 HTTP 包装，不应该重新实现一套状态机。开始前先确认 CLI 还能跑：

```bash
pnpm run build
pnpm minicodex status <任意已有 runId>
```

本 Lab 至少需要两个终端：

```text
终端 A：启动 backend dev server
终端 B：用 curl 创建、查询、审批 run
```

`targetRoot` 继续传 `fixtures/target-repo`。后端会从 workspace root 解析这个相对路径，不要把它理解成 `apps/backend` 下面的目录。

## 3. 写后端入口

到这里，CLI 已经能创建、审批、回放 run；桌面端还不能直接调用 TypeScript 类，所以需要一个后端入口把 HTTP 请求翻译成 Runtime 调用。

这个文件负责：

```text
接收前端 HTTP 请求
调用 MiniCodexRuntime
把 run / trace 结果返回给前端
提供 SSE 事件流，让界面看到状态变化
```

这个文件不负责：

```text
自己推进 phase
自己判断权限
自己写 .minicodex/runs
```

`apps/backend/src/main.ts`：

```ts
import cors from "cors";
import express from "express";
import { MiniCodexRuntime } from "@minicodex/core";

const app = express();
const runtime = new MiniCodexRuntime({ invocationCwd: process.cwd() });

app.use(cors());
app.use(express.json());

/**
 * 创建 run：HTTP 只接收请求，真正逻辑交给 Runtime。
 */
app.post("/agent-runs", async (req, res, next) => {
  try {
    const run = await runtime.createRun({
      message: String(req.body.message ?? ""),
      targetRoot: req.body.targetRoot ? String(req.body.targetRoot) : undefined,
    });
    res.json(run);
  } catch (error) {
    next(error);
  }
});

/**
 * 查询 run：供桌面端刷新当前状态。
 */
app.get("/agent-runs/:id", async (req, res, next) => {
  try {
    res.json(await runtime.getRun(req.params.id));
  } catch (error) {
    next(error);
  }
});

/**
 * 提交审批：plan、diff、tool 的恢复逻辑仍由 Runtime 处理。
 */
app.post("/agent-runs/:id/approval", async (req, res, next) => {
  try {
    res.json(
      await runtime.approveRun(req.params.id, req.body.approved === true),
    );
  } catch (error) {
    next(error);
  }
});

/**
 * 回放 trace：把本地 trace.jsonl 暴露给桌面端查看。
 */
app.get("/agent-runs/:id/trace", async (req, res, next) => {
  try {
    res.json({
      runId: req.params.id,
      events: await runtime.replayTrace(req.params.id),
    });
  } catch (error) {
    next(error);
  }
});

app.use(
  (
    error: unknown,
    _req: express.Request,
    res: express.Response,
    _next: express.NextFunction,
  ) => {
    const message = error instanceof Error ? error.message : "Unknown error";
    res.status(500).json({ message });
  },
);

const port = Number(process.env.MINICODEX_BACKEND_PORT ?? process.env.PORT ?? 3000);

app.listen(port, () => {
  console.log(`Mini Codex backend listening on http://localhost:${port}`);
});
```

这里默认仍用 3000，但端口不是写死的。你本机如果经常同时跑多个项目，可以这样启动：

```bash
MINICODEX_BACKEND_PORT=3100 pnpm --dir apps/backend run dev
```

`PORT` 也支持，是为了兼容很多部署平台和工具习惯；本课程自己的名字用 `MINICODEX_BACKEND_PORT`，语义更清楚。

## 4. SSE 事件流

最小版 SSE 可以轮询 run 文件并推送变化。后续再做真正的事件总线。

在 `main.ts` 加：

```ts
/**
 * 事件流：轮询 run 状态，有变化就推送给前端。
 */
app.get("/agent-runs/:id/events", async (req, res) => {
  res.writeHead(200, {
    "Content-Type": "text/event-stream",
    "Cache-Control": "no-cache",
    Connection: "keep-alive",
  });

  let lastPayload = "";

  const send = async () => {
    try {
      const run = await runtime.getRun(req.params.id);
      const payload = JSON.stringify(run);
      if (payload !== lastPayload) {
        lastPayload = payload;
        res.write(`data: ${payload}\n\n`);
      }
      if (run.status === "completed" || run.status === "failed") {
        clearInterval(timer);
        res.end();
      }
    } catch {
      clearInterval(timer);
      res.end();
    }
  };

  const timer = setInterval(send, 800);
  await send();

  req.on("close", () => {
    clearInterval(timer);
  });
});
```

## 4.1 调用链和数据流

本 Lab 的后端不要重新实现 agent 逻辑。它只是把 HTTP 请求翻译成 Runtime 调用：

```text
POST /agent-runs
-> runtime.createRun()
-> .minicodex/runs/<runId>/run.json
-> 返回 run 给桌面端
```

审批请求也是同一个原则：

```text
POST /agent-runs/:id/approval
-> runtime.approveRun(runId, approved)
-> Runtime 决定继续、暂停、完成或失败
-> 返回最新 run
```

桌面端实时更新依赖 SSE：

```text
GET /agent-runs/:id/events
-> backend 定时 runtime.getRun(runId)
-> run 有变化就 res.write("data: ...")
-> 前端 EventSource 收到最新 run
```

如果后端自己修改 phase、pendingApproval 或 events，就会绕过 Runtime 的状态机和权限边界；本课程里所有状态变化都应该从 Runtime 出来。

## 5. 运行和调试

启动后端：

```bash
pnpm dev:backend
```

如果你临时想绕过根脚本，也可以直接传环境变量：

```bash
MINICODEX_BACKEND_PORT=3100 pnpm --dir apps/backend run dev
```

为了后面的 curl 不到处改 URL，可以在当前终端先设一个变量：

```bash
export MINICODEX_BACKEND_URL=http://localhost:3100
```

创建 run：

```bash
curl -X POST "$MINICODEX_BACKEND_URL/agent-runs" \
  -H 'Content-Type: application/json' \
  -d '{"message":"把登录按钮文案改成提交","targetRoot":"fixtures/target-repo"}'
```

期望返回 JSON，里面至少有：

```text
id
status
state.phase
events
pendingApproval
```

如果 `id` 有值但 `targetRoot` 指向了 `apps/backend/fixtures/...`，说明 workspace root 解析错了，回到 `MiniCodexRuntime({ invocationCwd: process.cwd() })` 这一行检查。

审批：

```bash
curl -X POST "$MINICODEX_BACKEND_URL/agent-runs/<runId>/approval" \
  -H 'Content-Type: application/json' \
  -d '{"approved":true}'
```

SSE：

```bash
curl "$MINICODEX_BACKEND_URL/agent-runs/<runId>/events"
```

SSE 命令会持续占住终端，这是正常的。它应该输出类似：

```text
data: {"id":"run-...","status":"awaiting_approval",...}
```

如果 run 已经 `completed` 或 `failed`，连接会自动结束。调试过程中想手动停止，用 `Ctrl+C`。

## 6. 本 Lab 验收清单

- [ ] `POST /agent-runs` 能创建 run。
- [ ] `GET /agent-runs/:id` 能读取 run。
- [ ] `POST /agent-runs/:id/approval` 能推进任务。
- [ ] `GET /agent-runs/:id/trace` 能回放 trace。
- [ ] SSE 能推送当前 run。
- [ ] 后端和 CLI 看到的是同一个 `.minicodex/runs` 目录。
- [ ] curl 创建的 run 仍遵守 plan/diff/tool 审批，不会因为走 HTTP 就跳过权限。

## 7. 可选参考实现

如果你手里有自己的参考项目，可以对照这些后端和前端分层。它们只是参考方向，不是本课程的唯一标准答案：

```text
backend/src/agent-runs/agent-runs.controller.ts
backend/src/agent-runs/agent-runs.service.ts
frontend/src/services/agentRunService.ts
frontend/src/stores/agentRunWorkbench.ts
```

## 8. 下一步

进入 [Lab 07：做一个桌面工作台](./08-lab-07-desktop-workbench.md)。
