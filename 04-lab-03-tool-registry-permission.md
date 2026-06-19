# Lab 03：限制 AI 能做哪些动作（工具和权限）

## 1. 本 Lab 最终效果

这一节我们给系统加一层“动作边界”。做完后，代码不会到处直接调用函数，而是先把动作登记成工具，再判断能不能执行：

```text
下一步动作
-> 登记到工具清单
-> 检查输入格式
-> 判断是否允许执行
-> 执行动作
-> 记录执行结果
```

你会得到 3 个只读工具：

```text
read_target_context
read_file
git_diff
```

和 1 个需要审批的高风险工具占位：

```text
apply_patch
```

## 设计能力训练：AI 提出动作，系统决定能不能执行

本节先认识这些名字：

```text
工具：系统允许 AI 请求的一种动作，例如读文件、查看改动、写入补丁。
Tool Registry：工具清单，记录有哪些工具、每个工具需要什么输入、风险多高。
Permission Policy：权限规则，决定某个动作是允许、拒绝，还是先问你。
schema：输入格式要求，用来检查 AI 传来的参数是不是合法。
observation：工具执行后的结果，系统会把它记录下来给后续步骤使用。
```

当前阶段：你正在把“系统能做什么”从 runtime 里抽出来，形成工具注册表和权限策略。

这一阶段的意义是：模型不应该直接拥有文件系统。模型只能提出工具调用意图，系统负责校验参数、判断风险、请求审批、执行 handler、记录 observation。

看实现前，先自己写一个工具定义草图。重点不是写得多完整，而是想清楚风险：

```text
read_file 和 apply_patch 的风险为什么不同？
工具定义里为什么要写 sideEffects？
只按工具名审批够不够，为什么还要看 input？
以后接外部工具时，会映射到当前哪一层？
```

本节代码只实现最小工具定义。与此同时，你可以先建立一个成熟版判断：真实 coding agent 的工具说明，不只是 `name + handler`，还应该回答：

```text
whenToUse：什么时候应该用这个工具？
whenNotToUse：什么时候不该用？
sideEffects：会不会读文件、写文件、运行命令、访问外部数据？
approvalRequired：是否必须人工确认？
trustBoundary：工具输入和输出属于可信系统事实，还是外部上下文？
```

以后接外部工具、可复用工作流或自动检查点时，也必须继续经过当前 Tool Registry 和 Permission Policy。判断标准只有一句：外部扩展不能绕过工具清单和权限规则。

常见坑：

```text
信任模型生成的 JSON，不做 schema 校验。
把所有工具都当成低风险。
权限策略散落在每个 handler 里。
工具失败只 throw error，不返回可观察的 observation。
```

常见概念：

```text
tool calling
tool schema
tool handler
risk level
side effects
permission policy
allow / deny / request approval
```

最新技术对照：成熟 AI 编程工具通常会接很多外部工具和数据源。工程里常用 MCP 这类协议来做连接，但 Mini Codex 先做本地 Tool Registry，是为了先掌握工具边界和权限，再去接外部协议。

面试常问：

```text
Tool calling 和 tool execution 有什么区别？
如何设计一个安全的 apply_patch 工具？
Permission Policy 应该依赖用户、工具、参数还是上下文？
MCP 解决的是工具协议问题，还是权限问题？
```

做完本 Lab 后，可以继续问自己：当前权限策略是静态规则，如果未来支持“本次会话允许 read_file，但 apply_patch 每次都审批”，数据结构该怎么扩展？

## 2. 新增文件

```text
packages/core/src/tools/types.ts
packages/core/src/tools/registry.ts
packages/core/src/tools/readonly-tools.ts
packages/core/src/permission-policy.ts
packages/core/src/index.ts
```

## 2.1 跟练前确认

本 Lab 从 Lab 02 完成后的代码继续，工作目录固定为：

macOS / Linux：
```bash
cd ~/mini-codex-course/playground/minicodex-lab
```

Windows PowerShell：

```powershell
Set-Location "$HOME\mini-codex-course\playground\minicodex-lab"
```

开始前先确认上一节的暂停点还正常：

```bash
pnpm run build
pnpm minicodex run "把登录按钮文案改成提交" --target fixtures/target-repo
```

如果输出里能看到 `Status: awaiting_approval` 和 `Phase: plan_review`，说明 Lab 02 基线正确。

本 Lab 只把 `context` 阶段改成通过 Tool Registry 调用只读工具，并定义 `apply_patch` / `verify` 这类高风险工具的权限边界。它仍然不会执行真实 apply，也不会修改 `fixtures/target-repo/src/button.ts`。

## 3. 定义工具类型

`packages/core/src/tools/types.ts`：

```ts
import type { AgentRunRecord } from "../types.js";

export type ToolRiskLevel = "low" | "medium" | "high";

export type ToolName =
  | "read_target_context"
  | "read_file"
  | "git_diff"
  | "generate_plan"
  | "generate_diff"
  | "apply_patch"
  | "verify"
  | "record_delivery";

export interface ToolDefinition {
  name: ToolName;
  description: string;
  riskLevel: ToolRiskLevel;
  requiresApproval: boolean;
  sideEffects: string[];
}

// 进阶版会继续补 whenToUse / whenNotToUse / trustBoundary。
// 本 Lab 先保留最小类型，避免基础闭环太早变复杂。

export interface ToolExecutionPlan {
  toolName: ToolName;
  definition: ToolDefinition;
  input: Record<string, unknown>;
  inputPreview: Record<string, unknown>;
}

export interface ToolResult {
  title: string;
  content: string;
  observation: Record<string, unknown>;
  artifact?: {
    title: string;
    content: string;
  };
}

export interface ToolContext {
  run: AgentRunRecord;
  plan: ToolExecutionPlan;
}

export type ToolHandler = (context: ToolContext) => Promise<ToolResult>;
```

## 4. 写只读工具

`packages/core/src/tools/readonly-tools.ts`：

```ts
import { readdir, readFile } from "node:fs/promises";
import path from "node:path";
import { execFile } from "node:child_process";
import { promisify } from "node:util";
import type { ToolResult } from "./types.js";

const execFileAsync = promisify(execFile);

/** 确保工具只能读取 targetRoot 内部的相对路径，防止路径逃逸。 */
function assertInside(root: string, relativePath: string): string {
  if (path.isAbsolute(relativePath) || relativePath.includes("..")) {
    throw new Error(`Unsafe relative path: ${relativePath}`);
  }
  const absolutePath = path.resolve(root, relativePath);
  if (!absolutePath.startsWith(path.resolve(root))) {
    throw new Error(`Path escapes target root: ${relativePath}`);
  }
  return absolutePath;
}

/** 读取目标仓库的一层文件列表，作为最小版上下文。 */
export async function readTargetContext(targetRoot: string): Promise<ToolResult> {
  const entries = await readdir(targetRoot, { withFileTypes: true });
  const files = entries
    .filter((entry) => entry.isFile())
    .map((entry) => entry.name)
    .sort();

  return {
    title: "目标仓库上下文",
    content: `读取文件：${files.join(", ") || "none"}`,
    observation: {
      contextRead: true,
      targetRoot,
      files,
    },
    artifact: {
      title: "Target Context",
      content: ["# Target Context", "", ...files.map((file) => `- ${file}`)].join("\n"),
    },
  };
}

/** 读取 targetRoot 内指定文件，并把内容作为 artifact 返回。 */
export async function readTargetFile(targetRoot: string, relativePath: string): Promise<ToolResult> {
  const absolutePath = assertInside(targetRoot, relativePath);
  const content = await readFile(absolutePath, "utf-8");

  return {
    title: "文件内容",
    content: `${relativePath} 已读取`,
    observation: {
      relativePath,
      bytes: Buffer.byteLength(content),
    },
    artifact: {
      title: relativePath,
      content,
    },
  };
}

/** 运行只读 git diff，用来观察目标仓库当前有没有未提交改动。 */
export async function readGitDiff(targetRoot: string): Promise<ToolResult> {
  const { stdout } = await execFileAsync("git", ["diff", "--no-ext-diff"], {
    cwd: targetRoot,
    timeout: 30_000,
  });

  return {
    title: "Git Diff",
    content: stdout.trim() || "当前没有 git diff。",
    observation: {
      hasDiff: stdout.trim().length > 0,
    },
    artifact: {
      title: "Git Diff",
      content: stdout,
    },
  };
}
```

## 5. 写 Tool Registry

`packages/core/src/tools/registry.ts`：

```ts
import { z } from "zod";
import type { AgentRunRecord } from "../types.js";
import { readGitDiff, readTargetContext, readTargetFile } from "./readonly-tools.js";
import type { ToolDefinition, ToolExecutionPlan, ToolHandler, ToolName, ToolResult } from "./types.js";

const schemas: Record<ToolName, z.ZodObject<any>> = {
  read_target_context: z.object({ targetRoot: z.string().optional() }).strict(),
  read_file: z.object({ targetRoot: z.string().optional(), relativePath: z.string() }).strict(),
  git_diff: z.object({ targetRoot: z.string().optional() }).strict(),
  generate_plan: z.object({ targetRoot: z.string().optional() }).strict(),
  generate_diff: z.object({ targetRoot: z.string().optional() }).strict(),
  apply_patch: z.object({ targetRoot: z.string().optional() }).strict(),
  verify: z.object({ targetRoot: z.string().optional() }).strict(),
  record_delivery: z.object({ targetRoot: z.string().optional() }).strict(),
};

export class ToolRegistry {
  private readonly definitions: Record<ToolName, ToolDefinition> = {
    read_target_context: {
      name: "read_target_context",
      description: "只读读取目标仓库上下文。",
      riskLevel: "low",
      requiresApproval: false,
      sideEffects: ["只读 targetRoot", "不写入", "不执行修改命令"],
    },
    read_file: {
      name: "read_file",
      description: "只读读取目标仓库内单个文件。",
      riskLevel: "low",
      requiresApproval: false,
      sideEffects: ["只读 targetRoot 内文件"],
    },
    git_diff: {
      name: "git_diff",
      description: "只读读取 git diff。",
      riskLevel: "low",
      requiresApproval: false,
      sideEffects: ["执行 git diff --no-ext-diff", "不写入"],
    },
    generate_plan: {
      name: "generate_plan",
      description: "生成 patch plan，不写入。",
      riskLevel: "low",
      requiresApproval: false,
      sideEffects: ["生成计划 artifact"],
    },
    generate_diff: {
      name: "generate_diff",
      description: "生成 diff 草稿，不写入。",
      riskLevel: "low",
      requiresApproval: false,
      sideEffects: ["生成 diff artifact"],
    },
    apply_patch: {
      name: "apply_patch",
      description: "把已确认 diff 写入目标仓库。",
      riskLevel: "high",
      requiresApproval: true,
      sideEffects: ["写入 targetRoot"],
    },
    verify: {
      name: "verify",
      description: "运行受控验证命令。",
      riskLevel: "high",
      requiresApproval: true,
      sideEffects: ["执行 pnpm run check 等验证命令"],
    },
    record_delivery: {
      name: "record_delivery",
      description: "记录交付结果。",
      riskLevel: "low",
      requiresApproval: false,
      sideEffects: ["写入 run artifact"],
    },
  };

  /** 校验工具输入，并把工具定义、输入预览整理成一次可执行计划。 */
  buildPlan(run: AgentRunRecord, toolName: ToolName, input: Record<string, unknown> = {}): ToolExecutionPlan {
    const schema = schemas[toolName];
    const parsed = schema.parse(input);
    const targetRoot = typeof parsed.targetRoot === "string" ? parsed.targetRoot : run.targetRoot;
    const normalized = { ...parsed, targetRoot };

    return {
      toolName,
      definition: this.definitions[toolName],
      input: normalized,
      inputPreview: normalized,
    };
  }

  /** 根据 ToolExecutionPlan 找到对应 handler 并执行真实工具。 */
  async execute(run: AgentRunRecord, plan: ToolExecutionPlan): Promise<ToolResult> {
    const handlers: Partial<Record<ToolName, ToolHandler>> = {
      read_target_context: async () => readTargetContext(String(plan.input.targetRoot)),
      read_file: async () => readTargetFile(String(plan.input.targetRoot), String(plan.input.relativePath)),
      git_diff: async () => readGitDiff(String(plan.input.targetRoot)),
    };

    const handler = handlers[plan.toolName];
    if (!handler) {
      throw new Error(`Tool not implemented yet: ${plan.toolName}`);
    }
    return handler({ run, plan });
  }
}
```

## 6. 写权限策略

`packages/core/src/permission-policy.ts`：

```ts
import path from "node:path";
import type { AgentRunRecord } from "./types.js";
import type { ToolExecutionPlan } from "./tools/types.js";

export type PermissionDecision =
  | { kind: "allow"; reason: string }
  | { kind: "request_approval"; question: string; reason: string }
  | { kind: "deny"; reason: string };

export class PermissionPolicy {
  /** 根据 run 边界、工具风险和副作用，决定允许、拒绝或请求人工审批。 */
  decide(run: AgentRunRecord, plan: ToolExecutionPlan): PermissionDecision {
    const inputTargetRoot = typeof plan.inputPreview.targetRoot === "string"
      ? path.resolve(plan.inputPreview.targetRoot)
      : undefined;
    const runTargetRoot = run.targetRoot ? path.resolve(run.targetRoot) : undefined;

    if (inputTargetRoot && (!runTargetRoot || inputTargetRoot !== runTargetRoot)) {
      return {
        kind: "deny",
        reason: `工具 targetRoot 与 run targetRoot 不一致。`,
      };
    }

    const needsApproval = plan.definition.requiresApproval || plan.definition.riskLevel === "high";
    if (needsApproval) {
      return {
        kind: "request_approval",
        question: `是否允许执行工具：${plan.toolName}？`,
        reason: plan.definition.sideEffects.join("；"),
      };
    }

    return {
      kind: "allow",
      reason: "低风险工具，允许执行。",
    };
  }
}
```

## 7. 导出和接入 Runtime

`packages/core/src/index.ts` 至少保留 runtime 导出，并追加工具和权限类型导出：

```ts
export * from "./runtime.js";
export * from "./permission-policy.js";
export * from "./tools/types.js";
```

如果你在 Lab 01 已经写过 `export * from "./runtime.js";`，这里不要删掉，只补后两行。

在 `runtime.ts` 顶部补 import：

```ts
import { PermissionPolicy } from "./permission-policy.js";
import { ToolRegistry } from "./tools/registry.js";
import type { ToolName } from "./tools/types.js";
```

在 `runtime.ts` 里新增成员：

```ts
private readonly tools = new ToolRegistry();
private readonly policy = new PermissionPolicy();
```

写一个执行工具的方法：

```ts
/** Runtime 的统一工具入口：先记录调用，再过权限策略，最后执行工具或暂停审批。 */
async runTool(run: AgentRunRecord, toolName: ToolName, input: Record<string, unknown> = {}): Promise<void> {
  const plan = this.tools.buildPlan(run, toolName, input);
  run.events.push(createEvent("tool_call", `工具调用：${toolName}`, JSON.stringify(plan.inputPreview)));

  const decision = this.policy.decide(run, plan);
  run.events.push(createEvent("permission_decision", `权限决策：${decision.kind}`, decision.reason));

  if (decision.kind === "deny") {
    run.status = "failed";
    run.events.push(createEvent("error", "工具被拒绝", decision.reason));
    return;
  }

  if (decision.kind === "request_approval") {
    run.status = "awaiting_approval";
    run.pendingApproval = {
      id: createId("approval"),
      purpose: "tool",
      question: decision.question,
      reason: decision.reason,
    };
    return;
  }

  const result = await this.tools.execute(run, plan);
  run.events.push(createEvent("tool_result", result.title, result.content));
  if (result.artifact) {
    run.events.push(createEvent("artifact", result.artifact.title, result.artifact.content));
  }
}
```

Lab 03 暂时只在 `context` 阶段调用：

```ts
if (phase === "context") {
  await this.runTool(run, "read_target_context", { targetRoot: run.targetRoot });
  run.contextRead = true;
  continue;
}
```

也就是说，你只替换 Lab 02 里 `phase === "context"` 的占位分支；`plan` 和 `plan_review` 暂时保持 Lab 02 的实现。本 Lab 的验收目标是证明工具系统能接进 runtime，不要求实现 diff/apply/verify。

为了保持本 Lab 小，`purpose: "tool"` 的审批请求只会被创建，还不会被真正恢复执行。高风险工具审批后的执行逻辑会在 Lab 04 处理。

## 7.1 调用链和数据流

本 Lab 让 `context` 阶段不再手写占位事件，而是走统一工具入口：

```text
advanceRun(): phase = context
-> runTool(run, "read_target_context")
-> ToolRegistry.buildPlan()
   - 用 zod 校验 input
   - 合并 run.targetRoot
   - 找到工具定义和风险等级
-> PermissionPolicy.decide()
   - targetRoot 不一致：deny
   - 高风险工具：request_approval
   - 低风险只读工具：allow
-> ToolRegistry.execute()
-> readTargetContext()
-> run.events 记录 tool_call / permission_decision / tool_result / artifact
```

你可以用这个判断每个模块的边界：

```text
readonly-tools.ts：真正读文件或执行只读命令。
registry.ts：知道有哪些工具、输入长什么样、怎么找到 handler。
permission-policy.ts：只判断能不能执行，不亲自执行工具。
runtime.ts：把工具调用、权限结果和 run 状态串起来。
```

## 8. 运行

```bash
pnpm run build
pnpm minicodex run "把登录按钮文案改成提交" --target fixtures/target-repo
```

查看 run：

```bash
pnpm minicodex status <runId>
```

这一次仍然应该停在计划审批点：

```text
Status: awaiting_approval
Phase: plan_review
Approval: 是否认可当前 Patch Plan？
```

事件数量会比 Lab 02 多，因为 `context` 不再是一条占位消息，而是拆成了工具调用、权限决策、工具结果和 artifact。打开 `.minicodex/runs/<runId>/run.json`，应该看到这些事件类型：

```text
tool_call
permission_decision
tool_result
artifact
approval_request
```

再确认目标仓库还没有被修改：

```bash
sed -n '1,80p' fixtures/target-repo/src/button.ts
```

此时仍应看到 `"登录"`。如果这里已经变成 `"提交"`，说明你跑过后面的 apply 逻辑或手动改过 fixture，不是 Lab 03 的结果。

## 9. 本 Lab 验收清单

- [ ] `read_target_context` 通过 ToolRegistry 执行。
- [ ] 低风险工具自动 allow。
- [ ] 工具结果进入 events。
- [ ] `apply_patch` 和 `verify` 被定义为 high risk。
- [ ] 权限策略会检查 targetRoot 一致性。
- [ ] 新 run 仍停在 `plan_review`，没有越过人工计划审批。
- [ ] `fixtures/target-repo/src/button.ts` 仍然是 `"登录"`。

## 10. 下一步

进入 [Lab 04：完成第一次真实代码修改](./05-lab-04-coding-loop.md)。
