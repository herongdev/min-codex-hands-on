# Lab 02: Pause a Task Step by Step (State Machine)

[中文版](./03-lab-02-state-machine.md)

## 1. What This Lab Produces

In this lab, we add a sense of progress to the task record from the previous lab. After you finish, a run is no longer just a vague `running`. It knows which step it is on. In engineering terms, that step is called a `phase`:

```text
context -> plan -> plan_review -> diff -> diff_review -> apply -> verify -> delivery -> done
```

This lab only implements the first half of that path: read context, generate a plan, then stop and wait for your approval. Your goal is to make pause, wait for approval, and resume reliable.

On the first run, the task stops at `plan_review` until you run:

```bash
pnpm minicodex approve <runId> --yes
```

Then it advances to the next phase. The acceptance bar for Lab 02 is "can pause, can approve, can resume." It does not require real change drafts, file writes, or verification yet.

## Design Training: Teach the Task When to Stop

Learn these terms first:

```text
state machine: break a task into explicit steps and define when each step can advance.
phase: which step the task is on, such as reading files, generating a plan, or waiting for approval.
approval: an action that needs your confirmation before continuing.
status: the overall task state, such as running, awaiting approval, completed, or failed.
resume: continue from the saved position after a pause.
```

At this stage, you are turning a freely executing task into a long-running task that can pause, wait for approval, and resume.

The point: the biggest risk in a coding agent is not "cannot generate code." It is failing to pause before high-risk steps. A state machine tells the system whether it is in context, plan, diff, apply, or verify, and tells the user what they are approving.

Sketch a state flow diagram before reading the implementation. A simple one is fine:

```text
Why not go straight to apply after context?
Why separate plan_review and diff_review?
Should approval change status or phase?
If approve runs twice, will files be written twice?
```

Common pitfalls:

```text
Using one while loop from start to finish with no human intervention.
Keeping approval state only in memory, so it is lost after restart.
Mixing phase and status so the UI cannot tell running from awaiting approval.
Marking failed without explaining blockedBy.
Reading phase only once per advanceRun call, so context never advances to plan_review.
```

Common terms:

```text
state machine
phase
checkpoint
human-in-the-loop
pending approval
idempotency
resume
```

Current product reference: LangGraph emphasizes durable execution and human-in-the-loop. Codex permissions emphasize user authorization for high-risk actions. The Mini Codex state machine is a minimal version of both ideas.

Interview prompts:

```text
Why use a state machine in an agent runtime?
At which stages should human-in-the-loop appear?
How do you keep approve idempotent?
How do you avoid turning the state machine into a pile of if/else?
```

After this lab, consider: the current state machine is inferred from code rules. If you later support concurrent tools, repair branches, or multi-file verification, do you need an explicit transition table?

## 1.1 Before You Follow Along

This lab continues from the code you finished in Lab 01. Enter the practice workspace root first:

macOS / Linux:

```bash
cd ~/mini-codex-course/playground/minicodex-lab
```

Windows PowerShell:

```powershell
Set-Location "$HOME\mini-codex-course\playground\minicodex-lab"
```

## 2. Files to Add or Change

```text
packages/core/src/types.ts
packages/core/src/state-machine.ts
packages/core/src/runtime.ts
packages/cli/src/index.ts
```

## 3. Extend Types

Add the types below to the existing `packages/core/src/types.ts` from Lab 01. Do not remove `AgentRunStatus`, `AgentRunEvent`, `AgentRunRecord`, or other definitions you already wrote.

If you place `AgentRunPhase` and `AgentRunStateSnapshot` before `AgentRunStatus`, TypeScript still accepts it. The important part is that the final `types.ts` keeps all of these definitions together.

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

Add these fields to `AgentRunRecord`:

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

## 4. Write the State Machine

`packages/core/src/state-machine.ts`:

```ts
import type { AgentRunPhase, AgentRunRecord, AgentRunStateSnapshot } from "./types.js";

/** Infer the current phase from boolean flags already stored on the run. */
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

/** Build a UI/CLI-friendly snapshot from phase, status, completed steps, and blockers. */
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

/** Call after every run mutation so run.state stays in sync with the latest fields. */
export function syncState(run: AgentRunRecord): void {
  run.state = buildState(run);
}
```

## 5. Advance the Runtime One Step

In `packages/core/src/runtime.ts`, add the `advanceRun` method below inside the `MiniCodexRuntime` class. Place it after `getRun()`.

`createEvent`, `createId`, and `this.store` all come from the `runtime.ts` you wrote in Lab 01. Do not paste this block outside the class.

```ts
import { syncState } from "./state-machine.js";

/** Auto-advance safe stages; stop and persist when a human approval point is reached. */
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

This loop does not run from start to finish in one shot. It only auto-advances safe internal stages:

```text
context -> plan -> plan_review
```

Once it reaches `plan_review`, it sets status to `awaiting_approval` and stops. diff, apply, and verify wait until the user approves.

At the end of `createRun()`, change the return to:

```ts
await this.store.saveRun(run);
return this.advanceRun(run);
```

## 6. Implement approve

Add `approveRun` inside the same `MiniCodexRuntime` class. Place it after `advanceRun()`.

In this lab, `approveRun` only handles approvals with `purpose === "plan"`. After approval, it sets `planApproved` to `true` and calls `advanceRun()`. Because diff generation is not implemented yet, it advances to `phase: "diff"` and stops. It still does not modify target repository files.

```ts
/** Handle the user's approval decision; resume the state machine or fail the run. */
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

## 7. Update the CLI

In `packages/cli/src/index.ts`, improve the `run` output so it shows the current phase and approval question:

```ts
console.log(`Run created: ${run.id}`);
console.log(`Status: ${run.status}`);
console.log(`Phase: ${run.state?.phase}`);
console.log(`Events: ${run.events.length}`);
if (run.pendingApproval) {
  console.log(`Approval: ${run.pendingApproval.question}`);
}
```

Improve the `status` output to print `state` and `pendingApproval` together:

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

Then add the `approve` command:

```ts
program
  .command("approve")
  .argument("<runId>")
  .option("--yes", "approve")
  .option("--no", "reject")
  .action(async (runId: string, options: { yes?: boolean; no?: boolean }) => {
    // approve only submits the user's decision; Runtime still owns advancement.
    const approved = options.yes === true && options.no !== true;
    const run = await runtime.approveRun(runId, approved);
    console.log(`Run ${run.id}: ${run.status}`);
    console.log(`Phase: ${run.state?.phase}`);
  });
```

## 7.1 Call Chain and Data Flow

On the first `run`, the state machine walks automatically to the approval point:

```text
CLI run
-> MiniCodexRuntime.createRun()
-> advanceRun()
-> syncState()
-> inferPhase(): context
-> set contextRead
-> inferPhase(): plan
-> set planGenerated
-> inferPhase(): plan_review
-> write pendingApproval
-> status = awaiting_approval
-> RunStore.saveRun()
```

After the user confirms, approval resumes the same run:

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

The key outcome of this lab is not "finish every step." It is proving that a run can pause, persist, and continue from the same position.

## 8. Run It

```bash
pnpm run build
pnpm minicodex run "把登录按钮文案改成提交" --target fixtures/target-repo
```

Expected:

```text
Status: awaiting_approval
Phase: plan_review
Events: 5
Approval: 是否认可当前 Patch Plan？
```

Check status:

```bash
pnpm minicodex status <runId>
```

Approve:

```bash
pnpm minicodex approve <runId> --yes
```

Expected:

```text
Run run-...: running
Phase: diff
```

This means approval worked and the state machine moved past `plan_review`. Lab 02 still does not generate a diff, so seeing `Phase: diff` is enough. Do not expect a diff artifact yet.

Check status again:

```bash
pnpm minicodex status <runId>
```

You should see something like:

```json
{
  "status": "running",
  "phase": "diff",
  "state": {
    "completedSteps": ["context", "plan", "plan_approved"]
  }
}
```

Optional boundary check:

```bash
pnpm minicodex approve <runId> --no
```

If a run is waiting for approval, `--no` marks it as `failed`. If a run no longer has `pendingApproval`, a repeated approve should throw. That is intentional: approval must not silently run twice.

## 9. Acceptance Checklist

- [ ] A new run auto-advances to `plan_review`.
- [ ] Run status becomes `awaiting_approval`.
- [ ] `pendingApproval` exists.
- [ ] `status <runId>` shows `state.completedSteps` includes `context` and `plan`.
- [ ] After `approve --yes`, phase enters `diff`.
- [ ] After `approve --yes`, `state.completedSteps` includes `plan_approved`.
- [ ] Target repository files are still unchanged. This lab does not implement apply.

## 10. Next Step

Continue to [Lab 03: Restrict what AI can do](./04-lab-03-tool-registry-permission.md).
