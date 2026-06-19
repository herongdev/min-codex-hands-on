# Lab 04：完成第一次真实代码修改

## 1. 本 Lab 最终效果

这一节是一个小里程碑：命令行版 Mini Codex 会第一次真的修改目标仓库。做完后，它能对 `fixtures/target-repo` 完成最小真实修改：

```text
读取上下文
-> 生成 plan
-> 审批 plan
-> 生成 diff
-> 审批 diff
-> apply patch
-> verify
-> delivery
```

本 Lab 的验收目标是先把工程闭环跑稳，所以计划和改动草稿先用规则生成。以后接入模型时，只替换 `generatePlan` 和 `generateDiff`，其它安全边界不变。

## 设计能力训练：真正的 coding agent 闭环

本节会出现一组代码修改流程里的名字：

```text
context：系统读到的项目信息，比如相关文件、项目规则、检查命令。
plan：修改计划，先说明准备改哪里、为什么改。
diff：改动草稿，用来展示哪些行会被增加、删除或修改。
apply：把确认过的改动真正写入文件。
verify：运行检查命令，证明修改后项目还能通过检查。
repair：检查失败后的修复步骤。
delivery：最终交付说明，告诉你完成了什么、检查结果如何。
```

当前阶段：你正在让 Mini Codex 第一次修改真实文件，并用验证命令证明它没有只是“嘴上说成功”。

这一阶段的意义是：从这里开始，它才从任务管理器变成 coding agent。核心闭环不是“模型回答怎么改”，而是 `context -> plan -> diff -> approve -> apply -> verify -> repair -> delivery`。

先自己设计这个闭环，再看实现。你可以先用文字画一遍：

```text
为什么要先生成 plan，而不是直接生成 diff？
为什么 diff 也要审批？
apply 前要不要检查文件是否被用户改过？
verify 失败后应该马上 repair，还是先停下来让用户看？
```

这里的 `context` 先用规则实现，不代表真实 AI 编程助手只需要“读取一个目录”。成熟版要升级成“可解释的项目资料检索”，工程里常叫 Context Retrieval / RAG：

```text
候选文件收集
关键词检索或符号检索
chunk 切分
scoring / rerank
token budget 裁剪
selected / skipped 记录
source / trust level 标记
context report 输出
```

先不要急着接向量数据库。第一步是让系统说清楚：为什么选这些文件、为什么跳过那些文件、哪些内容只是 untrusted repo content、哪些事实来自 runtime。这样模型拿到上下文时，系统仍然知道边界在哪里。

常见坑：

```text
直接覆盖文件，不生成 diff。
生成 diff 后不检查目标文件是否还是原始内容。
验证失败还输出成功交付。
repair 没有最大次数，可能无限循环。
把规则 demo 误以为真实模型接入已经完成。
```

常见概念：

```text
context engineering
context retrieval
patch plan
unified diff
stale snapshot
apply
verify command
repair loop
delivery summary
```

最新技术对照：主流 coding agent 都在围绕“读取仓库上下文、提出计划、修改代码、运行验证、失败修复”优化。本 Lab 先用规则实现 plan/diff，目标是把工程闭环做稳定；真实模型接入放到进阶课。

面试常问：

```text
如何安全地让 agent 修改代码？
Context engineering 和 prompt engineering 有什么区别？
为什么 unified diff 比直接输出完整文件更适合 review？
verify 失败后如何定位是 plan 错、diff 错，还是环境错？
```

做完本 Lab 后，可以继续问自己：本 Lab 用规则生成 plan/diff 是为了教学闭环。接入大模型时，哪些边界不能变？哪些模块可以替换？

## 2. 新增文件

```text
packages/core/src/coding/plan.ts
packages/core/src/coding/diff.ts
packages/core/src/coding/apply.ts
packages/core/src/coding/verify.ts
packages/core/src/coding/delivery.ts
```

## 2.1 跟练前确认

本 Lab 从 Lab 03 完成后的代码继续，工作目录固定为：

macOS / Linux：
```bash
cd ~/mini-codex-course/playground/minicodex-lab
```

Windows PowerShell：

```powershell
Set-Location "$HOME\mini-codex-course\playground\minicodex-lab"
```

这是课程里第一次会真实写入目标仓库文件的 Lab。开始前先看 fixture 当前内容：

```bash
sed -n '1,80p' fixtures/target-repo/src/button.ts
```

第一次跟练时应看到：

```ts
export function getLoginButtonText(): string {
  return "登录";
}
```

如果已经是 `"提交"`，后面的 diff 可能会报“没有生成任何修改”，这不是状态机坏了，而是目标文件已经处在修改后状态。为了体验完整闭环，可以先手动把它改回 `"登录"`，再开始本 Lab。

本 Lab 只实现严格的单文件规则替换：`src/button.ts` 里的 `"登录"` 到 `"提交"`。验收时先别顺手做多文件 diff、checkpoint、rollback 或真实模型生成；这些能力放到进阶课逐个实现。

本节会用到 `diff` 包里的 `createTwoFilesPatch`。当前 `diff@5` 在这个工程里没有可用的 TypeScript 声明，直接 import 会导致 `TS7016`。为了让 Lab 自包含，先加一个最小声明文件。

`packages/core/src/diff.d.ts`：

```ts
declare module "diff" {
  export function createTwoFilesPatch(
    oldFileName: string,
    newFileName: string,
    oldStr: string,
    newStr: string,
  ): string;
}
```

## 3. 生成 Patch Plan

`packages/core/src/coding/plan.ts`：

```ts
import type { AgentRunRecord } from "../types.js";
import type { ToolResult } from "../tools/types.js";

/** 根据用户任务生成可审查的修改计划；本 Lab 先用规则版本代替模型。 */
export async function generatePlan(run: AgentRunRecord): Promise<ToolResult> {
  const content = [
    "# Patch Plan",
    "",
    "## Task",
    run.userMessage,
    "",
    "## Candidate Files",
    "",
    "- src/button.ts",
    "",
    "## Proposed Change",
    "",
    "把 getLoginButtonText 返回值从 登录 改成 提交。",
    "",
    "## Verification",
    "",
    "- pnpm run check",
  ].join("\n");

  return {
    title: "Patch Plan 已生成",
    content: "等待人工确认计划。",
    observation: {
      planGenerated: true,
      candidateFiles: ["src/button.ts"],
      verificationCommand: "pnpm run check",
    },
    artifact: {
      title: "Patch Plan",
      content,
    },
  };
}
```

在 Runtime 的 `plan` phase 调用它，并停在 `plan_review`。

## 4. 生成 Diff

`packages/core/src/coding/diff.ts`：

```ts
import { readFile } from "node:fs/promises";
import path from "node:path";
import { createTwoFilesPatch } from "diff";
import type { AgentRunRecord } from "../types.js";
import type { ToolResult } from "../tools/types.js";

/** 读取目标文件并生成 Unified Diff 草稿；此时还不写入文件。 */
export async function generateDiff(run: AgentRunRecord): Promise<ToolResult> {
  if (!run.targetRoot) throw new Error("targetRoot is required.");
  const relativePath = "src/button.ts";
  const absolutePath = path.join(run.targetRoot, relativePath);
  const oldContent = await readFile(absolutePath, "utf-8");
  const newContent = oldContent.replace('"登录"', '"提交"');

  if (oldContent === newContent) {
    throw new Error("没有生成任何修改，请检查目标文件内容。");
  }

  const patch = createTwoFilesPatch(relativePath, relativePath, oldContent, newContent);

  return {
    title: "Diff 草稿已生成",
    content: "等待人工确认 diff。",
    observation: {
      diffGenerated: true,
      changedFiles: [relativePath],
      patch,
    },
    artifact: {
      title: "Unified Diff",
      content: patch,
    },
  };
}
```

## 5. Apply Patch

本 Lab 先做严格的单文件 apply：只处理当前示例里的确定替换。完整 unified diff parser 放到进阶课。

`packages/core/src/coding/apply.ts`：

```ts
import { readFile, writeFile } from "node:fs/promises";
import path from "node:path";
import type { AgentRunRecord } from "../types.js";
import type { ToolResult } from "../tools/types.js";

/** 在用户批准后，把本 Lab 的确定修改真正写入目标文件。 */
export async function applyPatch(run: AgentRunRecord): Promise<ToolResult> {
  if (!run.targetRoot) throw new Error("targetRoot is required.");
  const relativePath = "src/button.ts";
  const absolutePath = path.join(run.targetRoot, relativePath);
  const oldContent = await readFile(absolutePath, "utf-8");
  const newContent = oldContent.replace('"登录"', '"提交"');

  if (oldContent === newContent) {
    return {
      title: "Patch 已无需应用",
      content: "目标文件已经是期望内容。",
      observation: { patchApplied: true, changedFiles: [] },
    };
  }

  await writeFile(absolutePath, newContent, "utf-8");

  return {
    title: "Patch 已应用",
    content: `已修改 ${relativePath}`,
    observation: {
      patchApplied: true,
      changedFiles: [relativePath],
    },
  };
}
```

真实项目里需要完整路径检查和 stale snapshot，本教程下一轮增强再补。

## 6. Verify

`packages/core/src/coding/verify.ts`：

```ts
import { execFile } from "node:child_process";
import { promisify } from "node:util";
import type { AgentRunRecord } from "../types.js";
import type { ToolResult } from "../tools/types.js";

const execFileAsync = promisify(execFile);

/** 在目标仓库里运行检查命令，证明修改后项目仍然可用。 */
export async function verifyTarget(run: AgentRunRecord): Promise<ToolResult> {
  if (!run.targetRoot) throw new Error("targetRoot is required.");

  try {
    const { stdout, stderr } = await execFileAsync("pnpm", ["run", "check"], {
      cwd: run.targetRoot,
      timeout: 120_000,
    });
    return {
      title: "验证通过",
      content: stdout || stderr || "pnpm run check passed",
      observation: {
        verificationAttempted: true,
        verificationPassed: true,
      },
    };
  } catch (error) {
    const anyError = error as { stdout?: string; stderr?: string; message?: string };
    return {
      title: "验证失败",
      content: [anyError.stdout, anyError.stderr, anyError.message].filter(Boolean).join("\n"),
      observation: {
        verificationAttempted: true,
        verificationPassed: false,
      },
    };
  }
}
```

## 7. Delivery

`packages/core/src/coding/delivery.ts`：

```ts
import type { AgentRunRecord } from "../types.js";
import type { ToolResult } from "../tools/types.js";

/** 汇总本轮任务的最终交付说明，作为 Delivery artifact 保存。 */
export async function recordDelivery(run: AgentRunRecord): Promise<ToolResult> {
  const content = [
    "# Delivery",
    "",
    `Run: ${run.id}`,
    `Task: ${run.userMessage}`,
    `Status: ${run.status}`,
    "",
    "## Completed Steps",
    "",
    ...(run.state?.completedSteps ?? []).map((step) => `- ${step}`),
  ].join("\n");

  return {
    title: "交付记录已生成",
    content: "本轮任务已归档。",
    observation: {
      deliveryRecorded: true,
    },
    artifact: {
      title: "Delivery",
      content,
    },
  };
}
```

## 8. 接入 phase

本节分三步跟练，顺序不要跳：

```text
8.1  Tool Registry 补 handler
8.2  advanceRun 按 phase 接线
8.3  审批恢复（types + runTool + approveRun）
```

最终 phase 映射：

```text
context      -> read_target_context（Lab 03 已有）
plan         -> generate_plan
plan_review  -> 人工审批 artifact（purpose: plan）
diff         -> generate_diff
diff_review  -> 人工审批 artifact（purpose: diff）
apply        -> apply_patch（高风险，purpose: tool）
verify       -> verify（高风险，purpose: tool）
delivery     -> record_delivery
done         -> status = completed
```

### 8.1 补齐 Tool Registry handler

文件：`packages/core/src/tools/registry.ts`

在文件顶部 import 五个 coding 函数：

```ts
import { generatePlan } from "../coding/plan.js";
import { generateDiff } from "../coding/diff.js";
import { applyPatch } from "../coding/apply.js";
import { verifyTarget } from "../coding/verify.js";
import { recordDelivery } from "../coding/delivery.js";
```

在 `execute()` 的 `handlers` 对象里追加（只读工具保持不动）：

```ts
generate_plan: async ({ run }) => generatePlan(run),
generate_diff: async ({ run }) => generateDiff(run),
apply_patch: async ({ run }) => applyPatch(run),
verify: async ({ run }) => verifyTarget(run),
record_delivery: async ({ run }) => recordDelivery(run),
```

改完后 `pnpm run build`，确认没有 `Tool not implemented yet` 的编译期遗漏。

### 8.2 改 `advanceRun` 各 phase 分支

文件：`packages/core/src/runtime.ts`

把 Lab 02 里 `plan` / `plan_review` 的占位实现，扩展成完整链路。`context` 分支保持 Lab 03 不变。

先在 `createEvent` 后面加一个小 helper，避免 TypeScript 因为 `while (run.status === "running")` 把 `run.status` 窄化后，不相信 `runTool()` 会把它改成 `awaiting_approval`：

```ts
/** 判断 runTool 是否把 run 暂停在工具审批点。 */
function isAwaitingApproval(run: AgentRunRecord): boolean {
  return run.status === "awaiting_approval";
}
```

**通用规则**：每次 `await this.runTool(...)` 之后，如果 `isAwaitingApproval(run)`，立刻 `break` 跳出 `while`，不要 `continue`。否则会在用户还没批工具时继续往下跑。

各 phase 建议写法：

```ts
if (phase === "plan") {
  await this.runTool(run, "generate_plan", { targetRoot: run.targetRoot });
  if (isAwaitingApproval(run)) break;
  run.planGenerated = true;
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

if (phase === "diff") {
  await this.runTool(run, "generate_diff", { targetRoot: run.targetRoot });
  if (isAwaitingApproval(run)) break;
  run.diffGenerated = true;
  continue;
}

if (phase === "diff_review") {
  run.status = "awaiting_approval";
  run.pendingApproval = {
    id: createId("approval"),
    purpose: "diff",
    question: "是否认可当前 Unified Diff？",
    reason: "确认后才会请求 apply_patch。",
  };
  run.events.push(createEvent("approval_request", "等待确认", run.pendingApproval.question));
  break;
}

if (phase === "apply") {
  await this.runTool(run, "apply_patch", { targetRoot: run.targetRoot });
  break; // 第一次会停在 tool 审批；审批通过后由 approveRun 设 patchApplied 再继续
}

if (phase === "verify") {
  await this.runTool(run, "verify", { targetRoot: run.targetRoot });
  break; // 同上，审批通过后再设 verificationAttempted
}

if (phase === "delivery") {
  run.deliveryRecorded = true;
  run.status = "completed";
  syncState(run);
  await this.runTool(run, "record_delivery", { targetRoot: run.targetRoot });
  if (isAwaitingApproval(run)) break;
  break;
}
```

这一段代码的阅读顺序是：

```text
phase === "plan"      -> 生成计划 artifact
phase === "plan_review" -> 等你确认计划
phase === "diff"      -> 生成 diff artifact
phase === "diff_review" -> 等你确认 diff
phase === "apply"     -> 请求高风险工具审批，批准后写文件
phase === "verify"    -> 请求高风险工具审批，批准后运行检查
phase === "delivery"  -> 记录交付结果并完成 run
```

`plan_review` / `diff_review` 不调用工具，只创建 `pendingApproval` 并暂停。`apply` / `verify` 通过 `runTool` 触发高风险审批，真正写文件和跑命令发生在用户 `approve` 之后（见 §8.3）。

### 8.3 审批恢复：三种 purpose 各走各的路

这是本 Lab **必须完成** 的部分。不加 `toolName` / `toolInput`，`apply_patch` 和 `verify` 会在用户 `approve` 后再次触发 `request_approval`，形成死循环。

两种审批不要混：

```text
artifact 审批（plan / diff）
  用户批的是「计划/差异内容」
  恢复方式：设 planApproved / diffApproved，再 advanceRun

工具审批（tool）
  用户批的是「是否执行 apply_patch / verify」
  恢复方式：用已保存的 toolName + toolInput 直接执行工具，再 advanceRun
```

#### 步骤 1：扩展 `PendingApproval` 类型

文件：`packages/core/src/types.ts`

在 `PendingApproval` 接口里追加两个可选字段：

```ts
export interface PendingApproval {
  id: string;
  purpose: "plan" | "diff" | "tool" | "repair";
  question: string;
  reason: string;
  toolName?: string;
  toolInput?: Record<string, unknown>;
}
```

只有 `purpose === "tool"` 时才会写入后两个字段；`plan` / `diff` 审批不需要。

#### 步骤 2：`runTool` 暂停时记住工具计划

文件：`packages/core/src/runtime.ts`，`runTool` 方法里 `decision.kind === "request_approval"` 分支。

把 Lab 03 的：

```ts
run.pendingApproval = {
  id: createId("approval"),
  purpose: "tool",
  question: decision.question,
  reason: decision.reason,
};
```

改成：

```ts
run.pendingApproval = {
  id: createId("approval"),
  purpose: "tool",
  question: decision.question,
  reason: decision.reason,
  toolName: plan.toolName,
  toolInput: plan.input,
};
```

同时补一条 `approval_request` 事件，和 `plan_review` 保持一致，方便在 `run.json` 里看出当前在等什么：

```ts
run.events.push(
  createEvent("approval_request", "等待工具确认", run.pendingApproval.question),
);
```

#### 步骤 3：`approveRun` 处理三种 purpose

文件：`packages/core/src/runtime.ts`，`approveRun` 方法。

在现有 `purpose === "plan"` 分支旁边，补 `diff` 和 `tool`：

```ts
} else if (run.pendingApproval.purpose === "diff") {
  run.diffApproved = true;
  run.status = "running";
  run.pendingApproval = undefined;
  run.events.push(createEvent("permission_decision", "Diff 已确认", "用户确认 unified diff。"));
} else if (run.pendingApproval.purpose === "tool") {
  const { toolName, toolInput } = run.pendingApproval;
  if (!toolName) {
    throw new Error("tool approval missing toolName.");
  }

  // 已批准：跳过 PermissionPolicy，直接执行
  const plan = this.tools.buildPlan(run, toolName as ToolName, toolInput ?? {});
  const result = await this.tools.execute(run, plan);
  run.events.push(createEvent("tool_result", result.title, result.content));
  if (result.artifact) {
    run.events.push(createEvent("artifact", result.artifact.title, result.artifact.content));
  }

  // 按工具写回 run 标志位，供 state-machine 推进 phase
  if (toolName === "apply_patch") {
    run.patchApplied = Boolean(result.observation?.patchApplied);
  }
  if (toolName === "verify") {
    run.verificationAttempted = Boolean(result.observation?.verificationAttempted);
    run.verificationPassed = Boolean(result.observation?.verificationPassed);
  }

  run.pendingApproval = undefined;
  run.events.push(createEvent("permission_decision", "工具已批准", `用户批准执行 ${toolName}。`));
  if (toolName === "verify" && !run.verificationPassed) {
    run.status = "failed";
    run.events.push(createEvent("error", "验证失败", "验证未通过，本 Lab 先停止交付。"));
  } else {
    run.status = "running";
  }
}
```

`approveRun` 末尾已有「`status === "running"` 时调用 `advanceRun`」的逻辑，三种 purpose 批完后都会自动进入下一阶段。

#### 自检：四种审批是否都能走通

跑完 §9 全流程后，在 `run.json` 里应能看到四次人工停顿，且 `pendingApproval.purpose` 依次为：

```text
plan -> diff -> tool(apply_patch) -> tool(verify)
```

若 `approve` 后 phase 不变、又出现同样的 `是否允许执行工具：apply_patch？`，说明 §8.3 没做对——优先检查 `toolName` / `toolInput` 是否写入，以及 `approveRun` 是否走了「直接 execute」而不是再次 `runTool`。

## 8.4 完整调用链和数据流

本 Lab 跑通以后，一条任务的数据会这样流动：

```text
CLI run
-> createRun()
-> advanceRun()
-> context: read_target_context
-> plan: generate_plan
-> plan_review: pendingApproval(purpose = plan)
```

第一次人工确认后：

```text
CLI approve --yes
-> approveRun()
-> planApproved = true
-> advanceRun()
-> diff: generate_diff
-> diff_review: pendingApproval(purpose = diff)
```

第二次人工确认后：

```text
CLI approve --yes
-> diffApproved = true
-> advanceRun()
-> apply: runTool("apply_patch")
-> pendingApproval(purpose = tool, toolName = apply_patch)
```

工具审批通过后：

```text
CLI approve --yes
-> approveRun() 直接 execute apply_patch
-> patchApplied = true
-> advanceRun()
-> verify: runTool("verify")
-> pendingApproval(purpose = tool, toolName = verify)
```

最后一次工具审批通过后：

```text
CLI approve --yes
-> execute verify
-> verificationAttempted = true
-> advanceRun()
-> delivery: record_delivery
-> status = completed
```

这条链路里有两个不同的“确认”：

```text
plan / diff 审批：确认内容是否合理。
tool 审批：确认是否允许执行会写文件或跑命令的动作。
```

## 9. 运行

在 `minicodex-lab` 根目录下执行（§2.1 已进入该目录则跳过）：

macOS / Linux：
```bash
cd ~/mini-codex-course/playground/minicodex-lab
```

Windows PowerShell：

```powershell
Set-Location "$HOME\mini-codex-course\playground\minicodex-lab"
```

先安装 fixture 依赖（路径相对 `minicodex-lab` 根）：

```bash
pnpm --dir fixtures/target-repo install
```

创建任务：

```bash
pnpm minicodex run "把登录按钮文案改成提交" --target fixtures/target-repo
```

确认 plan：

```bash
pnpm minicodex approve <runId> --yes
```

确认 diff：

```bash
pnpm minicodex approve <runId> --yes
```

确认 apply 工具：

```bash
pnpm minicodex approve <runId> --yes
```

确认 verify 工具：

```bash
pnpm minicodex approve <runId> --yes
```

查看结果：

```bash
sed -n '1,80p' fixtures/target-repo/src/button.ts
pnpm minicodex status <runId>
```

期望文件内容变成：

```ts
export function getLoginButtonText(): string {
  return "提交";
}
```

`status` 里应能看到：

```text
status: completed
phase: done
completedSteps 包含 context、plan、plan_approved、diff、diff_approved、apply、verify、delivery
```

如果某一步停在 `awaiting_approval`，先看 `pendingApproval.purpose`：

```text
plan: 你正在确认 Patch Plan
diff: 你正在确认 Unified Diff
tool: 你正在确认 apply_patch 或 verify
```

继续执行对应的 `approve <runId> --yes`，直到 delivery 完成。

## 10. 本 Lab 验收清单

- [ ] 能生成 Patch Plan。
- [ ] plan 必须审批。
- [ ] 能生成 Unified Diff。
- [ ] diff 必须审批。
- [ ] apply 和 verify 必须审批。
- [ ] `fixtures/target-repo/src/button.ts` 被修改为 `提交`。
- [ ] verify 通过后生成 delivery。
- [ ] 如果重复运行同一任务，能解释“目标文件已经是期望内容”和真实失败的区别。
- [ ] 能说清 plan/diff 审批和 tool 审批的恢复方式为什么不同。

## 11. 下一步

进入 [Lab 05：保存过程并复盘](./06-lab-05-trace-replay-eval.md)。
