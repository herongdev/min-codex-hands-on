# Lab 01：保存第一条任务（Agent Run）

## 1. 本 Lab 最终效果

这一节我们会让 Mini Codex 第一次“接住”一个用户任务。做完后，你可以运行：

```bash
pnpm minicodex run "把登录按钮文案改成提交" --target fixtures/target-repo
pnpm minicodex status <runId>
```

并在 `.minicodex/runs/<runId>/run.json` 看到一次保存下来的任务记录。工程里我们把这条任务记录叫做 `Agent Run`。

这一节还不会调用模型，也不会改文件。先别急，我们今天只做一件很重要的事：建立任务容器。

## 设计能力训练：把一句话任务保存成可追踪记录

本节第一次出现几个名字，先用白话理解就够：

```text
Agent Run：一次 AI 编程任务，从用户输入到最终结果都算在这一条任务里。
runId：这条任务的编号，后面查询状态、审批、回放都靠它找到任务。
event：任务过程中发生的一条记录，比如用户输入、系统回复、等待确认。
artifact：任务生成的重要成果，比如计划、改动草稿、最终说明。
status：任务当前是运行中、等待你确认、完成，还是失败。
```

当前阶段：你正在把“用户一句话”，变成一个可以保存、追踪、恢复的工程对象。

这一阶段的意义是：AI 编程助手不是一次聊天回复，而是一段任务执行过程。后面生成计划、等待确认、改文件、运行检查、保存过程，都必须挂在同一个 `runId` 上。如果没有 Agent Run，后面的所有能力都会变成散落的函数调用。

看代码前，先自己设计一版 `AgentRunRecord`。不用完美，先回答：

```text
一个 run 必须保存哪些字段？
event 和 status 有什么区别？
targetRoot 应该存在 run 上，还是每个工具调用单独传？
为什么现在不急着接大模型？
```

常见坑：

```text
只保存最终回答，不保存过程事件。
runId 只在内存里，进程重启后任务消失。
没有 createdAt / updatedAt，后面无法排序和排查问题。
没有 artifact 概念，plan 和 diff 只能塞进普通消息。
```

常见概念：

```text
run
thread
event
artifact
status
local persistence
event timeline
```

最新技术对照：Codex CLI、Claude Code 这类工具都不是“一问一答”的聊天，而是围绕一次代码任务持续读取上下文、执行动作和反馈结果。Mini Codex 先做 Agent Run，就是在建立这个任务容器。

面试常问：

```text
Agent Run 和普通 Chat Message 有什么区别？
为什么 agent 系统需要 event timeline？
如果 run 执行到一半崩溃，如何恢复？
哪些数据应该持久化，哪些可以临时计算？
```

做完本 Lab 后，可以反过来问自己：当前用 JSON 文件保存 run 很简单，但并发写、索引查询、历史归档会有什么问题？什么时候需要换成 SQLite 或服务端数据库？

## 1.1 跟练前确认

本 Lab 从 Lab 00 完成后的目录继续。先进入练习项目根目录：

macOS / Linux：

```bash
cd ~/mini-codex-course/playground/minicodex-lab
```

Windows PowerShell：

```powershell
Set-Location "$HOME\mini-codex-course\playground\minicodex-lab"
```

## 2. 新增文件

```text
packages/core/src/types.ts
packages/core/src/workspace-root.ts
packages/core/src/run-store.ts
packages/core/src/runtime.ts
packages/core/src/index.ts
packages/cli/src/index.ts
```

## 3. 定义核心类型

`packages/core/src/types.ts`：

```ts
export type AgentRunStatus = "running" | "awaiting_approval" | "completed" | "failed";

export type AgentRunEventType =
  | "user_message"
  | "assistant_message"
  | "tool_call"
  | "tool_result"
  | "approval_request"
  | "permission_decision"
  | "artifact"
  | "error";

export interface AgentRunEvent {
  id: string;
  type: AgentRunEventType;
  title: string;
  content: string;
  data?: Record<string, unknown>;
  createdAt: string;
}

export interface AgentRunRecord {
  id: string;
  userMessage: string;
  targetRoot?: string;
  status: AgentRunStatus;
  events: AgentRunEvent[];
  createdAt: string;
  updatedAt: string;
}

export interface CreateRunInput {
  message: string;
  targetRoot?: string;
}
```

## 4. 写本地 run store

现在 `run` 不再只是一次命令输出，它必须能被后面的 `status`、`approve`、`trace` 找回来。如果把保存逻辑写在 CLI 里，后端和桌面端就不能复用；如果每个地方自己拼路径，后面很容易把数据写到不同目录。

所以这里新增两个文件：

```text
workspace-root.ts：统一判断 minicodex-lab 根目录，解决“命令从哪里启动”的问题。
run-store.ts：统一保存和读取 run.json，解决“任务记录放在哪里”的问题。
```

`packages/core/src/workspace-root.ts`（从当前目录向上找 `minicodex-lab` 的 monorepo 根；`pnpm minicodex` 实际在 `packages/cli` 下跑，不能直接用 `process.cwd()`，也不能只靠子目录里的 `.minicodex`）：

```ts
import { existsSync, readFileSync } from "node:fs";
import path from "node:path";

/** 取得用户执行 pnpm 命令时所在的目录，而不是 CLI 包自己的目录。 */
export function resolveInvocationCwd(): string {
  return path.resolve(process.env.INIT_CWD || process.cwd());
}

/** 从当前目录向上寻找 minicodex-lab 的 workspace 根目录。 */
export function resolveWorkspaceRoot(startDir = resolveInvocationCwd()): string {
  let dir = path.resolve(startDir);
  while (true) {
    const workspaceFile = path.join(dir, "pnpm-workspace.yaml");
    if (existsSync(workspaceFile)) {
      const pkgFile = path.join(dir, "package.json");
      if (existsSync(pkgFile)) {
        try {
          const pkg = JSON.parse(readFileSync(pkgFile, "utf-8")) as { name?: string };
          if (pkg.name === "minicodex-lab") return dir;
        } catch { /* continue */ }
      }
    }
    const parent = path.dirname(dir);
    if (parent === dir) break;
    dir = parent;
  }
  return path.resolve(startDir);
}

/** 把用户输入的相对路径解析到 workspace 根目录下。 */
export function resolveWorkspacePath(inputPath: string, baseDir = resolveWorkspaceRoot()): string {
  return path.isAbsolute(inputPath)
    ? path.normalize(inputPath)
    : path.resolve(baseDir, inputPath);
}
```

`packages/core/src/run-store.ts`：

```ts
import { mkdir, readFile, writeFile } from "node:fs/promises";
import path from "node:path";
import type { AgentRunRecord } from "./types.js";
import { resolveWorkspaceRoot } from "./workspace-root.js";

/** 负责把每一次 Agent Run 保存到 .minicodex/runs/<runId>/run.json。 */
export class RunStore {
  constructor(private readonly rootDir = resolveWorkspaceRoot()) {}

  private runsDir(): string {
    return path.join(this.rootDir, ".minicodex", "runs");
  }

  runDir(runId: string): string {
    return path.join(this.runsDir(), runId);
  }

  runFile(runId: string): string {
    return path.join(this.runDir(runId), "run.json");
  }

  /** 写入或覆盖某个 run 的最新状态。 */
  async saveRun(run: AgentRunRecord): Promise<void> {
    await mkdir(this.runDir(run.id), { recursive: true });
    await writeFile(this.runFile(run.id), JSON.stringify(run, null, 2), "utf-8");
  }

  /** 按 runId 读取之前保存的任务记录。 */
  async readRun(runId: string): Promise<AgentRunRecord> {
    const raw = await readFile(this.runFile(runId), "utf-8");
    return JSON.parse(raw) as AgentRunRecord;
  }
}
```

## 5. 写 Runtime

现在已经有了保存层，下一步需要一个“任务入口”。CLI 不应该自己创建 run 文件，它只负责接收命令；真正把用户输入变成 `AgentRunRecord` 的逻辑放进 Runtime。

这个文件负责：

```text
接收用户任务
解析 targetRoot
创建初始事件
调用 RunStore 保存 run
提供 getRun 给 status 查询
```

`packages/core/src/runtime.ts`：

```ts
import { RunStore } from "./run-store.js";
import type { AgentRunEvent, AgentRunRecord, CreateRunInput } from "./types.js";
import {
  resolveInvocationCwd,
  resolveWorkspacePath,
  resolveWorkspaceRoot,
} from "./workspace-root.js";

/** 生成带前缀的本地 id，用来区分 run 和 event。 */
function createId(prefix: string): string {
  return `${prefix}-${Date.now()}-${Math.random().toString(36).slice(2, 8)}`;
}

/** 把一次任务过程中的动作包装成可保存的事件。 */
function createEvent(type: AgentRunEvent["type"], title: string, content: string): AgentRunEvent {
  return {
    id: createId("event"),
    type,
    title,
    content,
    createdAt: new Date().toISOString(),
  };
}

export class MiniCodexRuntime {
  private readonly workspaceRoot: string;
  private readonly store: RunStore;

  constructor(options: { workspaceRoot?: string; invocationCwd?: string } = {}) {
    const invocationCwd = options.invocationCwd ?? resolveInvocationCwd();
    this.workspaceRoot = options.workspaceRoot ?? resolveWorkspaceRoot(invocationCwd);
    this.store = new RunStore(this.workspaceRoot);
  }

  /** 创建一条新的任务记录，并写入初始事件。 */
  async createRun(input: CreateRunInput): Promise<AgentRunRecord> {
    const now = new Date().toISOString();
    const targetRoot = input.targetRoot
      ? resolveWorkspacePath(input.targetRoot, this.workspaceRoot)
      : undefined;
    const run: AgentRunRecord = {
      id: createId("run"),
      userMessage: input.message,
      targetRoot,
      status: "running",
      events: [
        createEvent("user_message", "用户任务", input.message),
        createEvent(
          "assistant_message",
          "Runtime 已创建任务",
          targetRoot ? `目标仓库：${targetRoot}` : "未选择目标仓库",
        ),
      ],
      createdAt: now,
      updatedAt: now,
    };

    await this.store.saveRun(run);
    return run;
  }

  /** 按 runId 读取当前任务状态，供 status 命令使用。 */
  async getRun(runId: string): Promise<AgentRunRecord> {
    return this.store.readRun(runId);
  }
}
```

## 6. 导出 core

`packages/core/src/index.ts`：

```ts
export * from "./types.js";
export * from "./workspace-root.js";
export * from "./run-store.js";
export * from "./runtime.js";

export const minicodexCoreVersion = "0.1.0";
```

## 7. 改 CLI

`packages/cli/src/index.ts`：

```ts
#!/usr/bin/env node
import { Command } from "commander";
import { MiniCodexRuntime, minicodexCoreVersion } from "@minicodex/core";

const program = new Command();
const runtime = new MiniCodexRuntime();

program
  .name("minicodex")
  .description("A local Mini Codex runtime")
  .version(minicodexCoreVersion);

program
  .command("run")
  .argument("<message>")
  .option("--target <path>", "target repository root")
  .action(async (message: string, options: { target?: string }) => {
    // 把用户输入转交给 Runtime，CLI 只负责输入输出。
    const run = await runtime.createRun({
      message,
      targetRoot: options.target,
    });
    console.log(`Run created: ${run.id}`);
    console.log(`Status: ${run.status}`);
    console.log(`Events: ${run.events.length}`);
  });

program
  .command("status")
  .argument("<runId>")
  .action(async (runId: string) => {
    // status 不重新执行任务，只读取已经保存的 run.json。
    const run = await runtime.getRun(runId);
    console.log(JSON.stringify({
      id: run.id,
      status: run.status,
      targetRoot: run.targetRoot,
      eventCount: run.events.length,
      updatedAt: run.updatedAt,
    }, null, 2));
  });

program.parse(process.argv);
```

## 7.1 调用链和数据流

这一节的代码跑起来时，数据会这样走：

```text
pnpm minicodex run "任务" --target fixtures/target-repo
-> CLI 解析 message / target
-> MiniCodexRuntime.createRun()
-> resolveWorkspacePath() 把 targetRoot 变成绝对路径
-> createEvent() 生成初始事件
-> RunStore.saveRun()
-> .minicodex/runs/<runId>/run.json
```

查询状态时不会重新执行任务，只读文件：

```text
pnpm minicodex status <runId>
-> MiniCodexRuntime.getRun()
-> RunStore.readRun()
-> 打印 run.json 的摘要
```

## 8. 运行

```bash
pnpm install
pnpm run build
pnpm minicodex run "把登录按钮文案改成提交" --target fixtures/target-repo
```

如果你刚修改过 `packages/core/src`，一定要先重新执行 `pnpm run build`。当前 `packages/cli` 虽然用 `tsx src/index.ts` 启动，但它依赖的 `@minicodex/core` 仍然会按 `packages/core/package.json` 的 `main: dist/index.js` 读取构建产物；不重新 build，就可能继续运行旧逻辑。

你会看到：

```text
Run created: run-...
Status: running
Events: 2
```

查看文件：

```bash
find .minicodex/runs -maxdepth 2 -type f
```

注意这里必须出现在 lab 根目录的 `.minicodex/runs`，不能出现在 `packages/cli/.minicodex`。因为根脚本通过 `pnpm --dir packages/cli run dev` 启动 CLI，运行时 `process.cwd()` 可能变成 `packages/cli`。

同理，`--target fixtures/target-repo` 也必须按 lab 根目录解析。这里不能依赖你手动创建 `.env`，也不能假设一定有 `INIT_CWD`。当前实现先解析 workspace root，再用 workspace root 解析相对 target，所以最终 run 里的 `targetRoot` 应该类似：

```text
.../minicodex-lab/fixtures/target-repo
```

而不是：

```text
.../minicodex-lab/packages/cli/fixtures/target-repo
```

再查状态：

```bash
pnpm minicodex status run-你的-id
```

## 9. 本 Lab 验收清单

- [ ] `pnpm minicodex run ...` 能创建 run。
- [ ] `.minicodex/runs/<runId>/run.json` 存在。
- [ ] run 里有 `user_message` 和 `assistant_message`。
- [ ] `pnpm minicodex status <runId>` 能读取 run。

## 10. 下一步

进入 [Lab 02：让任务按步骤暂停](./03-lab-02-state-machine.md)。
