# Lab 05：保存过程并复盘（Trace / Eval）

## 1. 本 Lab 最终效果

这一节我们给 Mini Codex 加上“记忆现场”的能力。做完后，每次 run 都会留下可回放记录：

```text
.minicodex/runs/<runId>/
├── run.json
└── trace.jsonl
```

并支持命令：

```bash
pnpm minicodex trace <runId>
pnpm minicodex eval
```

## 设计能力训练：从“能跑”升级到“能复盘、能变好”

本节先认识这些名字：

```text
trace：任务执行过程的详细记录，方便以后回看每一步发生了什么。
replay：把保存下来的过程重新展示出来，帮助你排查问题。
eval：评测记录，用来判断一次改动是不是真的做好了。
case：一条练习或评测任务，例如“把按钮文案改掉”。
regression：新版本让以前能通过的任务变差了。
```

当前阶段：你正在给 agent 加可观察性和评估机制。简单说，就是让它做过什么、为什么失败、有没有变好，都能被看见。

这一阶段的意义是：没有 trace，你只能看到最终成败；有 trace，你能知道它为什么成功或失败。没有 eval，你只能凭感觉改 prompt；有 eval，你能判断修改是否真的让系统变好。

看实现前，先自己设计一版 trace 结构：

```text
一次工具调用应该记录哪些字段？
trace 和 run.events 有什么区别？
eval case 应该覆盖成功任务，还是也要覆盖失败任务？
如何判断一次 repair 是有效改进还是误打误撞？
```

基础版 eval 先记录成功、分数和原因。这里先埋下一颗种子：coding agent 的失败要能归类，否则你只会反复“改 prompt”。后面可以把失败类型扩展成：

```text
policy_missing：关键规则只写在 prompt，没有 runtime 执行。
tool_metadata_missing：工具说明不清，导致选错工具。
unsafe_command：命令策略没有拦住危险动作。
path_escape：patch 或文件读取越过 targetRoot。
context_miss：上下文检索漏掉关键文件。
source_trust_error：把仓库文本、长期记忆或外部结果当成系统规则。
external_tool_faked：外部工具失败，却用假结果继续。
memory_leak：长期记忆写入 secret 或一次性上下文。
eval_regression：新版本让旧 case 变差。
```

这些分类不要求本 Lab 全部实现，但要从这里开始理解：trace 负责还原过程，eval 负责把失败变成可迭代的产品信号。

常见坑：

```text
只打文本日志，不保留结构化字段。
trace 没有 eventIndex，回放顺序不稳定。
eval 只放一个 happy path。
优化 prompt 后不跑旧用例，导致回归。
```

常见概念：

```text
trace
span
event sourcing
replay
eval case
golden set
regression
observability
```

最新技术对照：OpenAI Agents SDK 把 tracing 放进 agent 开发模型，LangGraph 也强调 observability 和 long-running workflow 的可追踪。Mini Codex 的 trace/replay/eval 是以后持续改进的地基。

面试常问：

```text
你如何衡量 coding agent 的质量？
Trace、log、metric、eval 分别解决什么问题？
如何构造一个 coding agent 的 golden set？
为什么 eval 要在 UI 完善前先做？
```

做完本 Lab 后，可以继续问自己：当前 eval-ledger 只是本地 JSONL，它能发现哪些问题，发现不了哪些问题？如果要做团队级 eval 平台，需要补哪些字段？

## 2. 新增和修改文件

```text
packages/core/src/types.ts
packages/core/src/runtime.ts
packages/core/src/trace-store.ts
packages/core/src/eval/eval-ledger.ts
packages/cli/src/index.ts
.minicodex/eval/.gitignore
```

运行后会生成：

```text
.minicodex/runs/<runId>/trace.jsonl
.minicodex/eval/eval-ledger.jsonl
```

**建议跟练顺序**（避免 §4 import 时找不到 `EvalLedger`）：

```text
§3    trace-store.ts
§3.1  eval/eval-ledger.ts      ← 与 Trace Store 同期创建，不要拖到 §6
§4    runtime 接入 trace + appendEvent 替换
§5    CLI trace
§6    eval 接线（types、recordEvalIfTerminal、CLI eval）
§7–§8 运行验收
```

`eval/eval-ledger.ts` 在 §3.1 只建**存储类**；§6 才接 `recordEvalIfTerminal` 和 `pnpm minicodex eval`。

## 2.1 跟练前确认

本 Lab 从 Lab 04 完成后的代码继续，工作目录固定为：

macOS / Linux：
```bash
cd ~/mini-codex-course/playground/minicodex-lab
```

Windows PowerShell：

```powershell
Set-Location "$HOME\mini-codex-course\playground\minicodex-lab"
```

开始前至少要有一次 Lab 04 的完整 run，能从 `run`、多次 `approve` 跑到 `completed`。如果目标文件已经是 `"提交"`，再次跑同一任务可能会走到“无需应用”分支，这可以用于观察 trace，但第一次验收最好先把 fixture 手动改回 `"登录"`。

本 Lab 不新增 agent 能力，只新增观察和评估能力：trace 记录过程，eval ledger 记录结果。不要把 trace 当成普通日志，它是以后 replay、UI 时间线和失败归因的证据。

本 Lab 最容易漏的是 Runtime 接线：只写 `TraceStore` / `EvalLedger` 类还不够，必须让 `createRun`、`advanceRun`、`approveRun` 的事件都通过统一入口写 trace，并在 run 结束时写 eval 记录。

`eval-ledger.jsonl` 是运行产物，不建议提交到课程仓库。先补 `.minicodex/eval/.gitignore`：

```text
eval-ledger.jsonl
!.gitkeep
```

`.gitignore` 不是只能放在项目根目录。Git 允许在**任意子目录**放 `.gitignore`，规则只对该目录及子路径生效。本课程从 Lab 01 起就在 `.minicodex/runs/.gitignore` 里忽略 `run-*`，这里对 eval 目录用同样做法：把「这个目录里什么该忽略」写在目录旁边，而不是全堆到根目录 `.gitignore`。

如果你更习惯根目录统一管理，也可以在 lab 根 `.gitignore` 写 `.minicodex/eval/eval-ledger.jsonl`，效果等价；两种选一种即可，不要重复写。

## 3. Trace Store

`packages/core/src/trace-store.ts`：

```ts
import { appendFile, mkdir, readFile } from "node:fs/promises";
import path from "node:path";
import type { AgentRunEvent, AgentRunRecord } from "./types.js";

export interface TraceLine {
  runId: string;
  status: string;
  eventIndex: number;
  timestamp: string;
  type: string;
  event: AgentRunEvent;
}

/** 负责把 run 过程事件追加到 trace.jsonl，并支持按 runId 回放。 */
export class TraceStore {
  constructor(private readonly rootDir: string) {}

  /** 计算某个 run 的 trace 文件路径。 */
  private traceFile(runId: string): string {
    return path.join(this.rootDir, ".minicodex", "runs", runId, "trace.jsonl");
  }

  /** 追加一行 trace，记录事件发生时的 run 状态和事件序号。 */
  async append(
    run: AgentRunRecord,
    event: AgentRunEvent,
    eventIndex: number,
  ): Promise<void> {
    await mkdir(path.dirname(this.traceFile(run.id)), { recursive: true });
    const line: TraceLine = {
      runId: run.id,
      status: run.status,
      eventIndex,
      timestamp: new Date().toISOString(),
      type: event.type,
      event,
    };
    await appendFile(
      this.traceFile(run.id),
      JSON.stringify(line) + "\n",
      "utf-8",
    );
  }

  /** 读取 trace.jsonl，并还原成按顺序排列的事件数组。 */
  async replay(runId: string): Promise<TraceLine[]> {
    const raw = await readFile(this.traceFile(runId), "utf-8");
    return raw
      .split("\n")
      .filter(Boolean)
      .map((line) => JSON.parse(line) as TraceLine);
  }
}
```

不要在 `TraceStore` 里用默认的 `process.cwd()`。CLI 是通过 `pnpm --dir packages/cli run dev` 启动的，`process.cwd()` 很容易落到包目录。`rootDir` 必须由 Runtime 传入已经解析好的 workspace root。

### 3.1 评测台账（Eval Ledger，与 Trace Store 同期创建）

这里的 `Eval Ledger` 指“评测台账”：每次任务跑完后，把结果、分数、失败原因和下一步改进记下来。

§4 会在 `runtime.ts` 里 `import { EvalLedger } from "./eval/eval-ledger.js"`。若此文件尚未创建，`pnpm run build` 会直接报错。**请在进入 §4 之前**先完成本节。

如果文件已经创建，`pnpm run build` 也通过，但编辑器仍提示 `Cannot find module './eval/eval-ledger.js'`，通常是 TypeScript Server 还没刷新新建文件。执行 `Cmd+Shift+P -> TypeScript: Restart TS Server` 后再看诊断；判断优先级以 `pnpm run build` 为准。

文件：`packages/core/src/eval/eval-ledger.ts`（新建目录 `packages/core/src/eval/`）

```ts
import { appendFile, mkdir, readFile } from "node:fs/promises";
import path from "node:path";

export interface EvalRecord {
  id: string;
  runId: string;
  task: string;
  success: boolean;
  score: number;
  reason: string;
  createdAt: string;
}

/** 负责保存每次终态 run 的评分记录，形成可持续复盘的 eval ledger。 */
export class EvalLedger {
  constructor(private readonly rootDir: string) {}

  /** eval 台账固定写入 .minicodex/eval/eval-ledger.jsonl。 */
  private ledgerFile(): string {
    return path.join(this.rootDir, ".minicodex", "eval", "eval-ledger.jsonl");
  }

  /** 追加一条评测记录；调用方负责决定何时写入。 */
  async append(record: EvalRecord): Promise<void> {
    await mkdir(path.dirname(this.ledgerFile()), { recursive: true });
    await appendFile(this.ledgerFile(), JSON.stringify(record) + "\n", "utf-8");
  }

  /** 读取全部评测记录；文件不存在时返回空数组，方便第一次运行。 */
  async list(): Promise<EvalRecord[]> {
    try {
      const raw = await readFile(this.ledgerFile(), "utf-8");
      return raw
        .split("\n")
        .filter(Boolean)
        .map((line) => JSON.parse(line) as EvalRecord);
    } catch {
      return [];
    }
  }
}
```

本节只建类，**不要**在 §3.1 改 `runtime.ts`。写入路径在 run 结束且 §6 接线完成后才会出现：

```text
.minicodex/eval/eval-ledger.jsonl
```

### 3.2 设计说明：TraceStore 和 EvalLedger 为何分开

读完 §3 和 §3.1，你可能会觉得两个类「长得差不多」：都是 `rootDir` + `mkdir` + `appendFile` 追加一行 JSON + 读回来 `JSON.parse`。这是正常的——**底层存储形态相同**，不代表**业务职责相同**。

```text
相同：JSONL 追加写、按行读取（工程套路重复）
不同：记什么、何时写、文件放哪、给谁看（业务语义不同）
```

对照表：

| 维度 | TraceStore | EvalLedger |
|------|------------|------------|
| 回答的问题 | 这次 run **怎么一步步走过来的** | 这次任务 **最终算不算成功** |
| 粒度 | 每个 event 一条（很多行） | 每个终态 run 一条（很少行） |
| 文件布局 | 每个 run 一个文件：`runs/<id>/trace.jsonl` | 全局一个台账：`eval/eval-ledger.jsonl` |
| 写入时机 | 每次 `appendEvent`（过程里） | run 到 `completed` / `failed` 时（结束时） |
| 行内容 | `TraceLine`（含 `eventIndex`、完整 `event`） | `EvalRecord`（含 `score`、`success`、`reason`） |
| 典型用途 | replay、排错、看审批/工具顺序 | 对比多次 run、算成功率、做 regression |

可以这么记：

```text
Trace  ≈ 黑匣子（过程录像）
Eval   ≈ 飞行记录本摘要（这班到了没有、评分多少）
```

**本 Lab 为何故意不合并成一个类？**

1. 要分别讲清 **observability** 和 **evaluation** 两个面试高频概念。
2. 遵循「反过早抽象」：两个 JSONL 场景先各写一遍，比先造通用 `JsonlStore` 再套两层更容易理解。
3. 后续演化方向不同——trace 可能加 span、采样；eval 可能加 case id、模型版本、golden set。过早合并，后面反而要拆。

如果以后出现第 3、4 个 JSONL 存储，可以再抽一层共用的 I/O 小工具（只复用 `append` / `readAll` 样板），**TraceStore 和 EvalLedger 仍应保持独立接口和类型**，不要把「过程 trace」和「结果 eval」混成一个概念。

## 4. Runtime 接入 trace

> **前置**：§3 的 `trace-store.ts` 和 §3.1 的 `eval/eval-ledger.ts` 都已创建。下面在 Runtime 里同时接入两者；eval 的自动写入逻辑在 §6 再接。

先在 `packages/core/src/runtime.ts` 顶部 import：

```ts
import { TraceStore } from "./trace-store.js";
import { EvalLedger } from "./eval/eval-ledger.js";
```

在 `MiniCodexRuntime` 里新增成员。注意不要写成字段初始化里的 `new TraceStore(this.workspaceRoot)`，因为 `workspaceRoot` 是构造函数里才算出来的。

```ts
private readonly traceStore: TraceStore;
private readonly evalLedger: EvalLedger;
```

在 constructor 里，`this.workspaceRoot` 和 `this.store` 初始化之后再接上：

```ts
this.traceStore = new TraceStore(this.workspaceRoot);
this.evalLedger = new EvalLedger(this.workspaceRoot);
```

然后把所有 `run.events.push(event)` 改成统一方法：

```ts
/** Runtime 内部唯一的事件写入口：同时更新 run.events 和 trace.jsonl。 */
private async appendEvent(run: AgentRunRecord, event: AgentRunEvent): Promise<void> {
  run.events.push(event);
  await this.traceStore.append(run, event, run.events.length - 1);
}
```

这一小节只改一个文件：`packages/core/src/runtime.ts`。
`TraceStore`、`ToolRegistry`、`coding/*` 等模块**不要**自己写 trace；事件只从 Runtime 这一层进 `trace.jsonl`。

Lab 04 完成后，这个文件里通常有 **4 个方法** 在写事件。按下面清单逐个替换（把 `run.events.push(...)` 改成 `await this.appendEvent(run, ...)`，`createEvent(...)` 参数保持原样）：

| 方法         | 场景                                                   | 事件 type（Lab 04 典型）                                                                   | 大约几处 |
| ------------ | ------------------------------------------------------ | ------------------------------------------------------------------------------------------ | -------- |
| `createRun`  | 创建 run 时的初始事件                                  | `user_message`、`assistant_message`                                                        | 2        |
| `advanceRun` | 停在 plan / diff 人工审批                              | `approval_request`                                                                         | 2        |
| `approveRun` | 用户拒绝 / 确认 plan / 确认 diff / 批准工具 / 验证失败 | `error`、`permission_decision`、`tool_result`、`artifact`                                  | 最多 7   |
| `runTool`    | 每次工具调用全过程                                     | `tool_call`、`permission_decision`、`approval_request`、`tool_result`、`artifact`、`error` | 最多 6   |

下面按方法说明具体改哪里。

#### 4.1 `createRun`（2 处）

把 Lab 04 里构造 run 时的 `events: [ createEvent(...), createEvent(...) ]` 改成空数组，**在 `saveRun` 之前**用 `appendEvent` 追加：

```ts
const run: AgentRunRecord = {
  id: createId("run"),
  userMessage: input.message,
  targetRoot,
  status: "running",
  events: [],
  createdAt: now,
  updatedAt: now,
};

await this.appendEvent(
  run,
  createEvent("user_message", "用户任务", input.message),
);
await this.appendEvent(
  run,
  createEvent(
    "assistant_message",
    "Runtime 已创建任务",
    targetRoot ? `目标仓库：${targetRoot}` : "未选择目标仓库",
  ),
);

await this.store.saveRun(run);
return this.advanceRun(run);
```

否则 trace 会少开头两条，`eventIndex` 也会和 `run.events` 对不上。

#### 4.2 `advanceRun`（2 处）

只改 **`plan_review`** 和 **`diff_review`** 分支里创建 `approval_request` 事件的那两行。
`context` / `plan` / `diff` / `apply` / `verify` / `delivery` 分支本身不写事件，它们通过 `runTool` 间接写 trace，不用在这里改。

改前（`plan_review` 示例）：

```ts
run.events.push(
  createEvent("approval_request", "等待确认", run.pendingApproval.question),
);
```

改后，`plan_review` 和 `diff_review` 两个分支应长这样（只改最后一行事件写入方式，其它 Lab 04 逻辑不动）：

```ts
if (phase === "plan_review") {
  run.status = "awaiting_approval";
  run.pendingApproval = {
    id: createId("approval"),
    purpose: "plan",
    question: "是否认可当前 Patch Plan？",
    reason: "确认后才会生成 diff。",
  };
  await this.appendEvent(
    run,
    createEvent("approval_request", "等待确认", run.pendingApproval.question),
  );
  break;
}

if (phase === "diff_review") {
  run.status = "awaiting_approval";
  run.pendingApproval = {
    id: createId("approval"),
    purpose: "diff",
    question: "是否认可当前 Unified Diff？",
    reason: "确认后才会请求 apply_patch。",
  };
  await this.appendEvent(
    run,
    createEvent("approval_request", "等待确认", run.pendingApproval.question),
  );
  break;
}
```

两处改法相同：把 `run.events.push(createEvent("approval_request", ...))` 换成 `await this.appendEvent(run, createEvent("approval_request", ...))`。`purpose`、`question`、`reason` 保持 Lab 04 原样。

#### 4.3 `approveRun`（最多 7 处）

这个方法里**每一处** `run.events.push` 都要换成 `await this.appendEvent(run, ...)`。方法签名、`pendingApproval` 校验、末尾 `saveRun` / `advanceRun` 逻辑保持 Lab 04 不变，只改事件写入。

对照清单：

```text
1. !approved              -> error（审批拒绝）
2. purpose === "plan"      -> permission_decision（计划已确认）
3. purpose === "diff"      -> permission_decision（Diff 已确认）
4. purpose === "tool"      -> tool_result（工具执行结果）
5. purpose === "tool" 且有 artifact -> artifact
6. purpose === "tool"      -> permission_decision（工具已批准）
7. verify 未通过           -> error（验证失败）
```

改完后，`approveRun` 从 `if (!approved)` 到方法结束应类似下面（**7 处** `appendEvent` 已标注释）：

```ts
async approveRun(runId: string, approved: boolean): Promise<AgentRunRecord> {
  const run = await this.store.readRun(runId);
  if (!run.pendingApproval) {
    throw new Error(`Run ${runId} has no pending approval.`);
  }

  if (!approved) {
    run.status = "failed";
    // ① 审批拒绝
    await this.appendEvent(
      run,
      createEvent("error", "审批拒绝", "用户拒绝继续执行。"),
    );
  } else if (run.pendingApproval.purpose === "plan") {
    run.planApproved = true;
    run.status = "running";
    run.pendingApproval = undefined;
    // ② 确认 Patch Plan
    await this.appendEvent(
      run,
      createEvent("permission_decision", "计划已确认", "用户确认 patch plan。"),
    );
  } else if (run.pendingApproval.purpose === "diff") {
    run.diffApproved = true;
    run.status = "running";
    run.pendingApproval = undefined;
    // ③ 确认 Unified Diff
    await this.appendEvent(
      run,
      createEvent("permission_decision", "Diff 已确认", "用户确认 unified diff。"),
    );
  } else if (run.pendingApproval.purpose === "tool") {
    const { toolName, toolInput } = run.pendingApproval;
    if (!toolName) {
      throw new Error("tool approval missing toolName.");
    }

    const plan = this.tools.buildPlan(run, toolName as ToolName, toolInput ?? {});
    const result = await this.tools.execute(run, plan);

    // ④ 工具执行结果
    await this.appendEvent(
      run,
      createEvent("tool_result", result.title, result.content),
    );

    if (result.artifact) {
      // ⑤ 可选 artifact（如 delivery）
      await this.appendEvent(
        run,
        createEvent("artifact", result.artifact.title, result.artifact.content),
      );
    }

    if (toolName === "apply_patch") {
      run.patchApplied = Boolean(result.observation?.patchApplied);
    }
    if (toolName === "verify") {
      run.verificationAttempted = Boolean(result.observation?.verificationAttempted);
      run.verificationPassed = Boolean(result.observation?.verificationPassed);
    }

    run.pendingApproval = undefined;

    // ⑥ 用户已批准高风险工具
    await this.appendEvent(
      run,
      createEvent("permission_decision", "工具已批准", `用户批准执行 ${toolName}。`),
    );

    if (toolName === "verify" && !run.verificationPassed) {
      run.status = "failed";
      // ⑦ 验证失败
      await this.appendEvent(
        run,
        createEvent("error", "验证失败", "验证未通过，本 Lab 先停止交付。"),
      );
    } else {
      run.status = "running";
    }
  }

  run.updatedAt = new Date().toISOString();
  syncState(run);
  await this.store.saveRun(run);

  if (run.status === "running") {
    return this.advanceRun(run);
  }

  return run;
}
```

注意：

- 只替换 `run.events.push` → `await this.appendEvent`，**不要**改 `plan` / `diff` / `tool` 分支里的状态字段赋值顺序。
- `purpose === "tool"` 分支里 ④⑤⑥ 的顺序必须保持：先 `tool_result`，再可选 `artifact`，再 `permission_decision`，最后才可能 ⑦ `error`。
- 若你的 Lab 04 没有 verify 失败分支，对应 ⑦ 可以没有；但 ①–⑥ 在完整 Lab 04 流程里都会出现。

#### 4.4 `runTool`（6 处）

这是**事件最多**的方法。Lab 04 完整跑一轮，每个低风险工具会走 ①②⑤（有 artifact 再加 ⑥）；`apply_patch` / `verify` 第一次调用会走 ①②④ 然后 `return`，真正写文件/跑命令的事件在 `approveRun` 里。

对照清单：

```text
1. 开头                    -> tool_call
2. 权限决策后              -> permission_decision
3. deny 分支               -> error
4. request_approval 分支   -> approval_request
5. execute 成功后          -> tool_result
6. 有 artifact 时          -> artifact
```

改完后，整个 `runTool` 方法应类似下面（**6 处** `appendEvent` 已标注释；`buildPlan`、`decide`、`execute`、`pendingApproval` 赋值等 Lab 04 逻辑不动）：

```ts
async runTool(
  run: AgentRunRecord,
  toolName: ToolName,
  input: Record<string, unknown> = {},
): Promise<void> {
  const plan = this.tools.buildPlan(run, toolName, input);

  // ① 记录即将调用的工具
  await this.appendEvent(
    run,
    createEvent(
      "tool_call",
      `工具调用：${toolName}`,
      JSON.stringify(plan.inputPreview),
    ),
  );

  const decision = this.policy.decide(run, plan);

  // ② 记录权限策略结论（allow / request_approval / deny）
  await this.appendEvent(
    run,
    createEvent(
      "permission_decision",
      `权限决策：${decision.kind}`,
      decision.reason,
    ),
  );

  if (decision.kind === "deny") {
    run.status = "failed";
    // ③ 工具被拒绝（如 targetRoot 不一致）
    await this.appendEvent(
      run,
      createEvent("error", "工具被拒绝", decision.reason),
    );
    return;
  }

  if (decision.kind === "request_approval") {
    run.status = "awaiting_approval";
    run.pendingApproval = {
      id: createId("approval"),
      purpose: "tool",
      question: decision.question,
      reason: decision.reason,
      toolName: plan.toolName,
      toolInput: plan.input,
    };
    // ④ 高风险工具暂停，等用户 approve（apply_patch / verify）
    await this.appendEvent(
      run,
      createEvent(
        "approval_request",
        "等待工具确认",
        run.pendingApproval.question,
      ),
    );
    return;
  }

  // decision.kind === "allow"：低风险工具直接执行
  const result = await this.tools.execute(run, plan);

  // ⑤ 工具执行摘要
  await this.appendEvent(
    run,
    createEvent("tool_result", result.title, result.content),
  );

  if (result.artifact) {
    // ⑥ 可选交付物（generate_plan / generate_diff / record_delivery 会进这里）
    await this.appendEvent(
      run,
      createEvent("artifact", result.artifact.title, result.artifact.content),
    );
  }
}
```

各工具在本方法里实际会触发哪些事件：

```text
read_target_context / generate_plan / generate_diff / record_delivery
  -> ① tool_call
  -> ② permission_decision（allow）
  -> ⑤ tool_result
  -> ⑥ artifact（plan / diff / delivery 有；read_target_context 通常没有）

apply_patch / verify（第一次，尚未 approve）
  -> ① tool_call
  -> ② permission_decision（request_approval）
  -> ④ approval_request
  -> return（不执行 ⑤⑥；execute 在 approveRun 里完成）

apply_patch / verify（若误写成再次 runTool 且仍要审批）
  -> 会重复 ①②④，所以 approve 后必须在 approveRun 直接 execute，不能再次 runTool
```

注意：

- 只替换 `run.events.push` → `await this.appendEvent`，**不要**改 `decision.kind` 三分支的结构和 `return` 时机。
- ⑥ 和 §4.3 的 ⑤ 一样：只有 `result.artifact` 存在才写；`apply_patch` / `verify` 在 `runTool` 里走 ④ 就 return 了，不会进 ⑤⑥。
- `runTool` 已是 `async`，加 `await this.appendEvent` 不需要改方法签名。

#### 4.5 自检

替换完成后运行：

```bash
rg "events\\.push" packages/core/src
```

**期望结果**：只剩 `appendEvent` 方法内部那一处 `run.events.push`。
若 `advanceRun` / `approveRun` / `runTool` 里还能搜到，说明清单还没改完。

再跑一轮 Lab 04 完整流程，确认 trace 事件类型齐全：

```text
user_message -> assistant_message -> tool_call -> permission_decision -> tool_result -> artifact
-> approval_request -> permission_decision -> ... -> completed
```

`pnpm minicodex trace <runId>` 里 `[0]` 应是 `user_message`，且 index 连续无跳号。

## 5. CLI trace 命令

本节改两个文件：

```text
packages/core/src/runtime.ts          <- 新增 replayTrace 公开方法
packages/cli/src/index.ts             <- 新增 trace 子命令
```

### 5.1 Runtime：`replayTrace`

文件：`packages/core/src/runtime.ts`
类：`MiniCodexRuntime`

在 **`appendEvent` 方法之后、`createRun` 方法之前`** 增加公开方法（不要写在 `appendEvent` 里面）：

```ts
async replayTrace(runId: string) {
  return this.traceStore.replay(runId);
}
```

前提：§4 已在同文件顶部 import `TraceStore`，并在 constructor 里初始化 `this.traceStore`。`replayTrace` 只是薄封装，读的是：

```text
.minicodex/runs/<runId>/trace.jsonl
```

（路径由 `TraceStore` 根据 `workspaceRoot` 拼出，不要手写绝对路径。）

完整上下文示意：

```ts
export class MiniCodexRuntime {
  // ... traceStore / evalLedger / constructor ...

  private async appendEvent(
    run: AgentRunRecord,
    event: AgentRunEvent,
  ): Promise<void> {
    run.events.push(event);
    await this.traceStore.append(run, event, run.events.length - 1);
  }

  // ← 在这里新增 replayTrace
  async replayTrace(runId: string) {
    return this.traceStore.replay(runId);
  }

  async createRun(input: CreateRunInput): Promise<AgentRunRecord> {
    // ...
  }
}
```

`packages/core/src/index.ts` 已有 `export * from "./runtime.js"` 时，**不必**为 `replayTrace` 单独加 export。

### 5.2 CLI：`trace` 子命令

文件：`packages/cli/src/index.ts`

在现有 **`approve` 命令块之后**、`program.parse(process.argv)` **之前**追加：

```ts
program
  .command("trace")
  .argument("<runId>")
  .action(async (runId: string) => {
    const lines = await runtime.replayTrace(runId);
    for (const line of lines) {
      console.log(`[${line.eventIndex}] ${line.type} ${line.event.title}`);
    }
  });
```

文件顶部已有 `const runtime = new MiniCodexRuntime();`，trace 命令复用同一个实例即可。

验证：

macOS / Linux：
```bash
cd ~/mini-codex-course/playground/minicodex-lab
pnpm run build
pnpm minicodex trace <runId>
```

Windows PowerShell：

```powershell
Set-Location "$HOME\mini-codex-course\playground\minicodex-lab"
pnpm run build
pnpm minicodex trace <runId>
```

若报 `trace.jsonl` 不存在，说明 §4 的事件还没全部走 `appendEvent`，或 `<runId>` 不对。

## 6. Eval Ledger 接线

本节改三个位置（`EvalLedger` 类已在 **§3.1** 创建，此处不再重复）：

```text
packages/core/src/types.ts            <- AgentRunRecord 加 evalRecorded
packages/core/src/runtime.ts          <- recordEvalIfTerminal + listEvalRecords + 两处调用
packages/cli/src/index.ts             <- eval 子命令
```

`.minicodex/eval/.gitignore` 见 §2.1，此处不重复。

### 6.1 类型：`evalRecorded`

文件：`packages/core/src/types.ts`
接口：`AgentRunRecord`

在现有可选字段末尾追加：

```ts
evalRecorded?: boolean;
```

用于防止同一 run 在多次 `saveRun` 时重复写入 eval ledger。

### 6.2 Runtime：两个新方法 + 两处调用

文件：`packages/core/src/runtime.ts`

**import**（§4 若已加可跳过）：

```ts
import { EvalLedger } from "./eval/eval-ledger.js";
```

**constructor**（§4 若已加可跳过）：

```ts
this.evalLedger = new EvalLedger(this.workspaceRoot);
```

**新增方法**：放在 `replayTrace` 附近（`createRun` 之前或之后均可，保持在 `MiniCodexRuntime` 类体内即可）：

```ts
/** 当 run 已经 completed 或 failed 时，自动写入一条 eval 记录。 */
private async recordEvalIfTerminal(run: AgentRunRecord): Promise<void> {
  if (run.evalRecorded) return;
  if (run.status !== "completed" && run.status !== "failed") return;

  const success = run.status === "completed";
  await this.evalLedger.append({
    id: createId("eval"),
    runId: run.id,
    task: run.userMessage,
    success,
    score: success ? 8 : 2,
    reason: success ? "completed" : (run.state?.phase ?? "failed"),
    createdAt: new Date().toISOString(),
  });
  run.evalRecorded = true;
}

/** 供 CLI eval 命令读取当前 eval ledger。 */
async listEvalRecords() {
  return this.evalLedger.list();
}
```

**调用点 1 — `advanceRun`**：在方法末尾、`await this.store.saveRun(run)` **之前**插入：

```ts
run.updatedAt = new Date().toISOString();
syncState(run);
await this.recordEvalIfTerminal(run); // ← 新增
await this.store.saveRun(run);
return run;
```

**调用点 2 — `approveRun`**：同样在末尾、`await this.store.saveRun(run)` **之前**插入：

```ts
run.updatedAt = new Date().toISOString();
syncState(run);
await this.recordEvalIfTerminal(run); // ← 新增
await this.store.saveRun(run);

if (run.status === "running") {
  return this.advanceRun(run);
}
```

这样 run 变成 `completed` 或 `failed` 时会自动 append 一条 eval；停在 `awaiting_approval` 时不会写。

### 6.3 CLI：`eval` 子命令

文件：`packages/cli/src/index.ts`

在 **`trace` 命令块之后**、`program.parse(process.argv)` **之前**追加：

```ts
program.command("eval").action(async () => {
  const records = await runtime.listEvalRecords();
  console.table(
    records.map((record) => ({
      runId: record.runId,
      success: record.success,
      score: record.score,
      reason: record.reason,
    })),
  );
});
```

## 7. 第一版自动评分

先用简单规则：

```text
completed -> success true, score 8
failed -> success false, score 2
awaiting_approval -> success false, score 4
```

自动写入时先只记录 `completed` 和 `failed`，因为 `awaiting_approval` 不是终态，自动记录会导致每个审批点都写一行。`awaiting_approval -> score 4` 用于你手动分析“卡在审批点”的 run。

评分记录建议在 delivery 或 run 结束时写入 `.minicodex/eval/eval-ledger.jsonl`。`pnpm minicodex eval` 第一版只负责读取和展示台账；如果你只实现了 `list()`，但从来没有 `append()`，命令会正常运行但表格为空。

最小可接受做法：

```text
run completed 后自动追加一条 EvalRecord。
run failed 后自动追加一条 EvalRecord。
run 仍 awaiting_approval 时不要自动写入 eval-ledger；如果要分析卡住原因，可以人工记录 success=false、score=4。
```

## 7.1 调用链和数据流

从 Lab 05 开始，同一个事件会同时出现在两个地方：

```text
Runtime 产生事件
-> appendEvent(run, event)
-> run.events.push(event)
-> TraceStore.append(run, event, eventIndex)
-> .minicodex/runs/<runId>/trace.jsonl
```

run 到达终态时，再写入评测台账：

```text
advanceRun() 或 approveRun()
-> syncState(run)
-> recordEvalIfTerminal(run)
-> EvalLedger.append(record)
-> .minicodex/eval/eval-ledger.jsonl
```

读取时是两条不同路径：

```text
pnpm minicodex trace <runId>
-> runtime.replayTrace()
-> TraceStore.replay()
-> 看这次 run 的过程

pnpm minicodex eval
-> runtime.listEvalRecords()
-> EvalLedger.list()
-> 看多次 run 的评分记录
```

你可以这样记：trace 解释“这次怎么发生的”，eval 解释“结果好不好、下次改什么”。

## 8. 运行

```bash
pnpm run build
pnpm minicodex run "把登录按钮文案改成提交" --target fixtures/target-repo
pnpm minicodex approve <runId> --yes
pnpm minicodex approve <runId> --yes
pnpm minicodex approve <runId> --yes
pnpm minicodex approve <runId> --yes
pnpm minicodex trace <runId>
pnpm minicodex eval
```

期望能看到：

```text
pnpm minicodex trace <runId>
  [0] user_message ...
  [1] assistant_message ...
  ...
  tool_call / permission_decision / tool_result / approval_request 按顺序出现

pnpm minicodex eval
  表格里至少有刚才 runId 的一条记录
```

同时检查文件位置：

```bash
ls .minicodex/runs/<runId>/trace.jsonl
sed -n '1,5p' .minicodex/runs/<runId>/trace.jsonl
sed -n '1,5p' .minicodex/eval/eval-ledger.jsonl
```

## 9. 本 Lab 验收清单

- [ ] `trace.jsonl` 存在。
- [ ] trace 能看到 user_message、tool_call、permission_decision、tool_result、approval_request。
- [ ] `pnpm minicodex trace <runId>` 能回放。
- [ ] `.minicodex/eval/eval-ledger.jsonl` 存在。
- [ ] `pnpm minicodex eval` 能展示评测记录。
- [ ] trace 里的 `eventIndex` 和 run.events 顺序一致。
- [ ] 能解释 eval 表格为空时是“未写入记录”，不是 trace 失败。

## 10. 下一步

命令行版核心闭环已经有了。进入 [Lab 06：把命令行能力提供给界面使用](./07-lab-06-backend-api-sse.md)。
