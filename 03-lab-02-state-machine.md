# Lab 02：让任务按步骤暂停（状态机）

## 1. 本 Lab 最终效果

这一节我们给上一节的任务记录加上“进度感”。做完后，它不再只是一个模糊的 `running`，而是知道自己走到了哪一步。工程里把这种步骤叫做 `phase`：

```text
context -> plan -> plan_review -> diff -> diff_review -> apply -> verify -> delivery -> done
```

本 Lab 只实现这条路线的前半段：先读取上下文，生成计划，然后停下来等你确认。你现在要把“能停、能等审批、能继续”这件事跑稳。

第一次运行时，任务会停在 `plan_review`，等待你执行：

```bash
pnpm minicodex approve <runId> --yes
```

然后继续往后推进到下一个 phase。当前 Lab 02 的验收标准是“能暂停、能审批、能恢复推进”；不要求真正生成改动草稿、写文件或运行检查。

## 设计能力训练：让任务知道什么时候该停

本节先认识这些名字：

```text
状态机：把任务拆成一组明确步骤，并规定每一步什么时候能进入下一步。
phase：任务当前走到哪一步，例如读文件、生成计划、等待确认。
approval：需要你确认后才能继续的操作。
status：任务整体状态，例如运行中、等待确认、完成或失败。
resume：任务暂停后，从保存的位置继续执行。
```

当前阶段：你正在把一个自由执行的任务，改造成可暂停、可审批、可恢复的长任务。

这一阶段的意义是：Coding agent 最大的风险不是“不会生成代码”，而是它在高风险步骤不暂停。状态机让系统知道当前处于 context、plan、diff、apply、verify 的哪一步，也让用户知道自己正在审批什么。

先自己画一版状态流转图，再看实现。画得简单一点也没关系：

```text
context 之后为什么不是直接 apply？
plan_review 和 diff_review 为什么要分开？
approval 应该改变 status，还是改变 phase？
approve 命令重复执行会不会重复写文件？
```

常见坑：

```text
用一个 while 循环从头跑到尾，无法人工介入。
审批状态只存在内存里，进程重启后丢失。
phase 和 status 混用，导致 UI 不知道任务是运行中还是等待审批。
失败时只标 failed，不说明 blockedBy。
每次 advanceRun 只读一次 phase，导致 context 后没有继续推进到 plan_review。
```

常见概念：

```text
state machine
phase
checkpoint
human-in-the-loop
pending approval
idempotency
resume
```

最新技术对照：LangGraph 强调 durable execution 和 human-in-the-loop，Codex permissions 强调高风险动作需要用户授权。Mini Codex 的状态机就是这两类思想的最小实现。

面试常问：

```text
为什么 agent runtime 要用状态机？
Human-in-the-loop 应该放在哪些阶段？
如何保证 approve 是幂等的？
如何避免状态机变成一堆 if else？
```

做完本 Lab 后，可以继续想：现在的状态机是代码规则推导出来的，后面如果支持并发工具、分支 repair、多文件验证，是否需要显式 transition table？

## 1.1 跟练前确认

本 Lab 从 Lab 01 完成后的代码继续。先进入练习项目根目录：

macOS / Linux：

```bash
cd ~/mini-codex-course/playground/minicodex-lab
```

Windows PowerShell：

```powershell
Set-Location "$HOME\mini-codex-course\playground\minicodex-lab"
```

## 2. 新增和修改文件

```text
packages/core/src/types.ts
packages/core/src/state-machine.ts
packages/core/src/runtime.ts
packages/cli/src/index.ts
```

## 3. 扩展类型

在 Lab 01 已有的 `packages/core/src/types.ts` 基础上追加下面这些类型。不要删除 Lab 01 已经写好的 `AgentRunStatus`、`AgentRunEvent`、`AgentRunRecord` 等定义。

如果你把 `AgentRunPhase` 和 `AgentRunStateSnapshot` 放在 `AgentRunStatus` 前面，TypeScript 也能识别；关键是最终同一个 `types.ts` 里同时保留这些定义。

```ts
export type AgentRunPhase =
  | "context"
  | "plan"
  | "plan_review"
  | "diff"
  | "diff_review"
  | "apply"
  | "verify"
  | "repair"
  | "delivery"
  | "done"
  | "failed";

export interface PendingApproval {
  id: string;
  purpose: "plan" | "diff" | "tool" | "repair";
  question: string;
  reason: string;
}

export interface AgentRunStateSnapshot {
  phase: AgentRunPhase;
  status: AgentRunStatus;
  awaitingApproval: boolean;
  completedSteps: string[];
  blockedBy?: string;
  updatedAt: string;
}
```

给 `AgentRunRecord` 加字段：

```ts
state?: AgentRunStateSnapshot;
pendingApproval?: PendingApproval;
contextRead?: boolean;
planGenerated?: boolean;
planApproved?: boolean;
diffGenerated?: boolean;
diffApproved?: boolean;
patchApplied?: boolean;
verificationAttempted?: boolean;
verificationPassed?: boolean;
deliveryRecorded?: boolean;
```

## 4. 写状态机

`packages/core/src/state-machine.ts`：

```ts
import type { AgentRunPhase, AgentRunRecord, AgentRunStateSnapshot } from "./types.js";

/** 根据 run 上已经完成的布尔标记，推导任务当前应该处在哪个 phase。 */
export function inferPhase(run: AgentRunRecord): AgentRunPhase {
  if (run.status === "failed") return "failed";
  if (run.status === "completed") return "done";
  if (!run.contextRead) return "context";
  if (!run.planGenerated) return "plan";
  if (!run.planApproved) return "plan_review";
  if (!run.diffGenerated) return "diff";
  if (!run.diffApproved) return "diff_review";
  if (!run.patchApplied) return "apply";
  if (!run.verificationAttempted) return "verify";
  if (!run.deliveryRecorded) return "delivery";
  return "done";
}

/** 把 phase、status、已完成步骤和阻塞原因整理成 UI/CLI 更容易展示的状态快照。 */
export function buildState(run: AgentRunRecord): AgentRunStateSnapshot {
  const phase = inferPhase(run);
  return {
    phase,
    status: run.status,
    awaitingApproval: run.status === "awaiting_approval",
    completedSteps: [
      run.contextRead ? "context" : "",
      run.planGenerated ? "plan" : "",
      run.planApproved ? "plan_approved" : "",
      run.diffGenerated ? "diff" : "",
      run.diffApproved ? "diff_approved" : "",
      run.patchApplied ? "apply" : "",
      run.verificationAttempted ? "verify" : "",
      run.deliveryRecorded ? "delivery" : "",
    ].filter(Boolean),
    blockedBy: run.pendingApproval?.question,
    updatedAt: run.updatedAt,
  };
}

/** 每次修改 run 后调用，保证 run.state 和最新字段保持一致。 */
export function syncState(run: AgentRunRecord): void {
  run.state = buildState(run);
}
```

## 5. 让 Runtime 推进一步

在 `packages/core/src/runtime.ts` 里，把下面的 `advanceRun` 方法加到 `MiniCodexRuntime` 类内部。推荐放在 `getRun()` 后面、`approveRun()` 前面。

这里用到的 `createEvent`、`createId` 和 `this.store` 都来自 Lab 01 已经写好的 `runtime.ts`，不要把这段代码贴到类外面。

```ts
import { syncState } from "./state-machine.js";

/** 自动推进安全阶段；遇到人工审批点就保存状态并停住。 */
async advanceRun(run: AgentRunRecord): Promise<AgentRunRecord> {
  while (run.status === "running") {
    syncState(run);
    const phase = run.state?.phase;

    if (phase === "context") {
      run.contextRead = true;
      run.events.push(createEvent("assistant_message", "上下文已读取", "Lab 02 先用占位上下文。"));
      continue;
    }

    if (phase === "plan") {
      run.planGenerated = true;
      run.events.push(createEvent("artifact", "Patch Plan", "计划：把 src/button.ts 中的登录文案改成提交。"));
      continue;
    }

    if (phase === "plan_review") {
      run.status = "awaiting_approval";
      run.pendingApproval = {
        id: createId("approval"),
        purpose: "plan",
        question: "是否认可当前 Patch Plan？",
        reason: "确认后才会生成 diff。",
      };
      run.events.push(createEvent("approval_request", "等待确认", run.pendingApproval.question));
      break;
    }

    break;
  }

  run.updatedAt = new Date().toISOString();
  syncState(run);
  await this.store.saveRun(run);
  return run;
}
```

这个循环不是“从头跑到尾”。它只自动推进安全的内部阶段：

```text
context -> plan -> plan_review
```

一旦进入 `plan_review`，就把状态改成 `awaiting_approval` 并停住。后面的 diff、apply、verify 要等用户 approve 后才继续。

在 `createRun()` 末尾改为：

```ts
await this.store.saveRun(run);
return this.advanceRun(run);
```

## 6. 实现 approve

在同一个 `MiniCodexRuntime` 类内部加入 `approveRun`。推荐放在 `advanceRun()` 后面。

本节的 `approveRun` 只处理 `purpose === "plan"` 的审批。审批通过后，它把 `planApproved` 设为 `true`，再调用 `advanceRun()`。由于后续 diff 生成还没有实现，所以它会推进到 `phase: "diff"` 后停住，仍然不会修改目标仓库文件。

```ts
/** 处理用户的审批结果；通过后恢复状态机，拒绝后结束 run。 */
async approveRun(runId: string, approved: boolean): Promise<AgentRunRecord> {
  const run = await this.store.readRun(runId);
  if (!run.pendingApproval) {
    throw new Error(`Run ${runId} has no pending approval.`);
  }

  if (!approved) {
    run.status = "failed";
    run.events.push(createEvent("error", "审批拒绝", "用户拒绝继续执行。"));
  } else if (run.pendingApproval.purpose === "plan") {
    run.planApproved = true;
    run.status = "running";
    run.pendingApproval = undefined;
    run.events.push(createEvent("permission_decision", "计划已确认", "用户确认 patch plan。"));
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

## 7. 改 CLI

`packages/cli/src/index.ts` 先补强 `run` 输出，让它显示当前 phase 和审批问题：

```ts
console.log(`Run created: ${run.id}`);
console.log(`Status: ${run.status}`);
console.log(`Phase: ${run.state?.phase}`);
console.log(`Events: ${run.events.length}`);
if (run.pendingApproval) {
  console.log(`Approval: ${run.pendingApproval.question}`);
}
```

再补强 `status` 输出，把 `state` 和 `pendingApproval` 一起打印出来：

```ts
console.log(
  JSON.stringify(
    {
      id: run.id,
      status: run.status,
      phase: run.state?.phase,
      targetRoot: run.targetRoot,
      eventCount: run.events.length,
      pendingApproval: run.pendingApproval,
      state: run.state,
      updatedAt: run.updatedAt,
    },
    null,
    2,
  ),
);
```

然后增加 `approve` 命令：

```ts
program
  .command("approve")
  .argument("<runId>")
  .option("--yes", "approve")
  .option("--no", "reject")
  .action(async (runId: string, options: { yes?: boolean; no?: boolean }) => {
    // approve 命令只提交用户决定，真正推进仍由 Runtime 完成。
    const approved = options.yes === true && options.no !== true;
    const run = await runtime.approveRun(runId, approved);
    console.log(`Run ${run.id}: ${run.status}`);
    console.log(`Phase: ${run.state?.phase}`);
  });
```

## 7.1 调用链和数据流

第一次 `run` 时，状态机会自动走到审批点：

```text
CLI run
-> MiniCodexRuntime.createRun()
-> advanceRun()
-> syncState()
-> inferPhase(): context
-> 标记 contextRead
-> inferPhase(): plan
-> 标记 planGenerated
-> inferPhase(): plan_review
-> 写入 pendingApproval
-> status = awaiting_approval
-> RunStore.saveRun()
```

用户确认后，审批结果会恢复到同一条 run：

```text
CLI approve <runId> --yes
-> MiniCodexRuntime.approveRun()
-> RunStore.readRun()
-> planApproved = true
-> advanceRun()
-> syncState()
-> phase = diff
-> RunStore.saveRun()
```

本 Lab 的关键不是“跑完所有步骤”，而是证明 run 可以暂停、保存、再从同一个位置继续。

## 8. 运行

```bash
pnpm run build
pnpm minicodex run "把登录按钮文案改成提交" --target fixtures/target-repo
```

期望：

```text
Status: awaiting_approval
Phase: plan_review
Events: 5
Approval: 是否认可当前 Patch Plan？
```

查看状态：

```bash
pnpm minicodex status <runId>
```

确认：

```bash
pnpm minicodex approve <runId> --yes
```

期望：

```text
Run run-...: running
Phase: diff
```

这表示审批已经生效，状态机已经越过 `plan_review`。但当前 Lab 02 还没有实现 diff 生成，所以这里看到 `Phase: diff` 就够了，不要期待出现 diff artifact。

再次查看状态：

```bash
pnpm minicodex status <runId>
```

你应该能看到类似：

```json
{
  "status": "running",
  "phase": "diff",
  "state": {
    "completedSteps": ["context", "plan", "plan_approved"]
  }
}
```

可选边界检查：

```bash
pnpm minicodex approve <runId> --no
```

如果一个 run 正在等待审批，`--no` 会把它标记为 `failed`。如果一个 run 已经没有 `pendingApproval`，重复 approve 应该报错。这是有意设计：审批动作不能静默重复执行。

## 9. 本 Lab 验收清单

- [ ] 新 run 会自动推进到 `plan_review`。
- [ ] run 状态变成 `awaiting_approval`。
- [ ] `pendingApproval` 存在。
- [ ] `status <runId>` 能看到 `state.completedSteps` 包含 `context` 和 `plan`。
- [ ] `approve --yes` 后 phase 进入 `diff`。
- [ ] `approve --yes` 后 `state.completedSteps` 包含 `plan_approved`。
- [ ] 目标仓库文件仍未被修改，本 Lab 不实现 apply。

## 10. 下一步

进入 [Lab 03：限制 AI 能做哪些动作](./04-lab-03-tool-registry-permission.md)。
