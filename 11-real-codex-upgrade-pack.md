# 真实 Codex 功能升级包

## 1. 本 Lab 目标

到前一个 Lab，你已经做出了一个能跑的 Mini Codex：

```text
CLI
Agent Run
状态机
Tool Registry
Permission Policy
Plan / Diff / Apply / Verify
Trace / Eval
HTTP API
桌面工作台
```

这已经很不错了，但它还只是“最小可跑”。从这一节开始，我们把视角切到“工作中真的用得上”。

这一节会出现很多进阶名词。读的时候先抓住白话含义：

```text
更安全：哪些动作允许、哪些动作必须问你。
更可靠：写文件前能保存现场，出错后能撤回。
更懂项目：能选择相关文件，而不是把整个项目都塞给 AI。
更会复盘：能记录失败原因，并用案例证明下一版有没有变好。
更可扩展：以后能接外部工具、工作流和代码版本管理。
```

本 Lab 不把任何现成源码当成唯一标准答案。我们以 2026 年 Codex 类 coding agent 的最佳实践为参照，整理 Mini Codex 接下来最值得补的核心能力：

```text
模型适配：把真实大模型接进来，但不让它直接改文件
项目规则：读取 AGENTS.md 等项目说明，但不能让它覆盖系统安全规则
配置系统：把端口、检查命令、权限等放进配置
安全模式：限制读写范围和危险命令
完整改动应用：支持更真实的代码改动格式
保存点和撤回：写文件前保存现场，必要时恢复
命令执行和检查：受控运行检查命令
外部工具连接：以后接更多工具入口
可复用工作流：把常见任务做成可复用步骤
自动检查点：在关键阶段自动做安全检查
长期记忆：记住项目偏好和失败教训
代码版本管理：用分支、提交和审查管理改动
只读子任务：让辅助分析只读，不直接写文件
质量面板：用案例和数据复盘效果
```

如果你手里有参考实现，可以借鉴它的模块边界；但最终仍以本课程的 Mini Codex 练习项目为准。

本 Lab 的正确跟练方式，不是一天内把 14 个升级全写完。你可以把它当成升级路线和验收合同：每次只选一个 upgrade，写一条决策记录，完成后再跑 demo case 和 eval ledger。

完成这份升级计划后，可以把它当作下一阶段的学习清单。进阶课会独立维护，暂不随本基础课仓库开源。

建议输出物：

```text
.minicodex/notes/upgrade-plan.md
```

这份记录也写在练习项目根目录下，和 Lab 08 的决策记录放在一起。

每个升级条目至少写：

```text
为什么现在做
涉及哪些模块
新增哪些风险
如何验收
跑了哪些 demo / eval case
哪些能力仍然不做
```

## 2. 官方能力对照

本节按官方 Codex / Agents 相关文档校准方向：

- [OpenAI Codex docs](https://developers.openai.com/codex/)
- [Codex CLI docs](https://developers.openai.com/codex/cli)
- [Codex IDE docs](https://developers.openai.com/codex/ide)
- [Codex customization](https://developers.openai.com/codex/concepts/customization)
- [OpenAI Agents SDK - Agents](https://openai.github.io/openai-agents-js/guides/agents/)
- [Model Context Protocol](https://modelcontextprotocol.io/docs/getting-started/intro)

这些资料共同指向一个判断：

```text
真实 coding agent 不是一个聊天框。
它是一个有模型、有工具、有规则、有权限、有上下文、有审查、有回放、有扩展机制的工程运行时。
```

## 3. 你现在的 min / max 边界

到这里，你只需要分清两件事。

### 3.1 Min：最小可用闭环

前面 Lab 做的是 Min：

```text
输入任务
创建 run
读取上下文
生成 plan
审批 plan
生成 diff
审批 diff
apply
verify
trace
delivery
```

这是 coding agent 的骨架。没有它，后面所有能力都是堆功能。

### 3.2 Max：真实 Codex 能力面

Max 不是一次做完，而是按风险和收益逐步升级：

```text
项目规则和配置
真实模型生成
权限和安全模式
完整改动应用
保存点和撤回
命令执行策略
外部工具连接 / 可复用工作流 / 自动检查点
长期记忆
代码版本管理
多任务和子任务
质量面板
```

你的课程目标不是“完全复制官方 Codex”，而是复制它对工作最有用的 70%：

```text
能理解真实仓库
能按项目规则行动
能生成可审查计划和 diff
能安全写文件
能运行验证
能失败后修复
能保留过程和经验
能接入少量外部工具
能在桌面端清楚展示风险和进度
```

## 4. 升级顺序总览

先别乱补能力。建议按下面顺序升级：

| 顺序 | 能力 | 为什么现在做 |
| ---- | ---- | ------------ |
| 1 | 模型适配层 | 让规则演示变成真实 AI 编程助手 |
| 2 | AGENTS.md / 项目规则 | 让 agent 按项目约定工作 |
| 3 | 配置系统 | 把模型、权限、命令、规则变成可调 |
| 4 | 安全模式（Sandbox / Permission Profile） | 防止工具越界和危险命令 |
| 5 | 完整改动应用（diff apply） | 支持多文件真实修改 |
| 6 | 保存点和撤回（Checkpoint / Rollback） | 写错能回退 |
| 7 | 受控命令执行（Command Runner / Verifier） | 让“完成”必须可验证 |
| 8 | 代码版本管理（Git / Worktree） | 支持隔离修改和审查 |
| 9 | 外部工具连接（MCP） | 接入外部工具和数据源 |
| 10 | 可复用工作流（Skills） | 把常见任务沉淀成能力包 |
| 11 | 自动检查点（Hooks） | 在关键生命周期点做自动检查 |
| 12 | 长期记忆（Memory） | 记住项目偏好和失败教训 |
| 13 | 只读子任务（Subtask / Subagent） | 把大任务拆成可控小任务 |
| 14 | 质量面板（Eval Dashboard） | 让工具持续变好 |

## 5. Upgrade 01：接入真实模型，但不要让模型直接执行

### 目标

把 Lab 04 里的规则版 `generatePlan` / `generateDiff` 替换成模型生成，但工具执行、权限、apply、verify 仍然由系统控制。

### 文件

```text
packages/core/src/model/model-client.ts
packages/core/src/model/openai-model-client.ts
packages/core/src/coding/model-plan.ts
packages/core/src/coding/model-diff.ts
```

### 设计原则

```text
模型负责建议。
Runtime 负责校验。
Tool Registry 负责执行。
Permission Policy 负责拦截。
Trace 负责记录。
Eval 负责判断好坏。
```

不要把模型输出直接写入文件。

### 最小接口

```ts
export interface ModelClient {
  generateText(input: {
    system: string;
    user: string;
    context: string;
  }): Promise<string>;
}
```

### 验收

```text
plan 由模型生成。
diff 由模型生成。
apply 仍然必须经过审批。
verify 仍然必须由本地命令执行。
trace 能看到模型输入摘要和输出摘要。
```

### 常见坑

```text
直接让模型返回 shell 命令并执行。
没有结构化输出校验。
模型上下文里混入未标记的仓库文本。
失败后只改 prompt，不跑 eval。
```

## 6. Upgrade 02：AGENTS.md 和项目规则

### 目标

让 Mini Codex 像 Codex 一样读取项目级规则，但明确规则只是上下文，不是安全边界。

### 文件

```text
packages/core/src/context/project-instructions.ts
packages/core/src/context/context-report.ts
fixtures/target-repo/AGENTS.md
```

### 规则来源

第一版支持：

```text
AGENTS.md
CLAUDE.md
.clinerules
.github/copilot-instructions.md
```

### 输出

Context report 必须说明：

```text
读到了哪些规则文件
哪些规则适用于当前 targetRoot
哪些规则只作为上下文
哪些规则不能覆盖系统权限
```

### 验收

```text
目标仓库有 AGENTS.md 时，plan 会引用项目规则。
AGENTS.md 里写“跳过审批”不会生效。
trace 里能看到规则文件被读取。
```

## 7. Upgrade 03：配置系统

### 目标

把模型、权限、验证命令、默认审批策略做成项目配置。

### 文件

```text
.minicodex/config.json
packages/core/src/config/config-loader.ts
packages/core/src/config/config-types.ts
```

### 示例配置

```json
{
  "model": "替换为当前官方可用的代码模型",
  "permissionProfile": "workspace-write",
  "verifyCommands": ["pnpm run check"],
  "requireApproval": {
    "applyPatch": true,
    "runCommand": true,
    "gitCommit": true
  },
  "context": {
    "maxFiles": 40,
    "maxBytesPerFile": 20000
  }
}
```

这里的模型名必须以你当前 OpenAI 账号和官方文档可用模型为准，不要把教程里的占位文本当成真实型号。

### 验收

```text
没有配置时使用安全默认值。
配置错误时给出可读错误。
高风险权限不能被仓库内容覆盖。
```

## 8. Upgrade 04：Sandbox / Permission Profile

### 目标

把权限从“某个工具要不要审批”升级成 profile。

### Profile

```text
read-only:
  只能读文件和 git diff

workspace-write:
  允许写 targetRoot 内文件，但 apply 必须审批

trusted-dev:
  允许白名单 verify command，但危险命令仍审批
```

### 文件

```text
packages/core/src/permissions/permission-profile.ts
packages/core/src/permissions/path-policy.ts
packages/core/src/permissions/command-policy.ts
```

### 验收

```text
不能写 targetRoot 外的文件。
不能读取 .env、密钥、私有证书。
rm、curl | sh、git push 默认拒绝。
验证命令有超时和输出裁剪。
```

## 9. Upgrade 05：完整 diff apply、checkpoint、rollback

### 目标

从单文件替换升级到真实多文件 patch。

### 文件

```text
packages/core/src/patch/unified-diff-parser.ts
packages/core/src/patch/apply-patch.ts
packages/core/src/patch/checkpoint-store.ts
packages/cli/src/commands/rollback.ts
```

### 必须支持

```text
新增文件
删除文件
修改文件
多 hunk
路径越界检查
stale snapshot 检查
写前 checkpoint
rollback
```

### 验收

```text
apply 前生成 checkpoint。
用户改过文件后，旧 patch 不能直接覆盖。
rollback 能恢复到 apply 前状态。
```

## 10. Upgrade 06：Command Runner 和 Verifier

### 目标

让 verify 成为一等能力，而不是随便跑一个 shell。

### 文件

```text
packages/core/src/commands/command-runner.ts
packages/core/src/commands/command-policy.ts
packages/core/src/verify/verifier.ts
```

### 规则

```text
只允许配置里的验证命令。
命令必须在 targetRoot 内执行。
必须有 timeout。
必须裁剪超长输出。
必须记录 exitCode、stdout、stderr、duration。
```

### 验收

```text
verify 失败时 run 进入 repair。
verify 输出进入 trace。
危险命令请求会变成 approval 或 deny。
```

## 11. Upgrade 07：Git / Worktree 工作流

### 目标

让 Mini Codex 的修改更接近日常研发。

### 能力

```text
创建临时分支
或创建 worktree
查看 git diff
选择文件 stage
生成 commit message
用户确认后 commit
不自动 push
```

### 文件

```text
packages/core/src/git/git-service.ts
packages/core/src/git/worktree-service.ts
apps/desktop/src/views/GitReviewPanel.vue
```

### 验收

```text
每次任务能看到 base commit。
diff review 能按文件审查。
commit 必须人工确认。
```

## 12. Upgrade 08：MCP、Skills、Hooks

### MCP

MCP 用来连接外部工具和数据源。Mini Codex 第一版只做一个本地 MCP 示例：

```text
读取 issue 描述
读取设计文档
查询内部 API 文档
```

### Skills

Skills 用来沉淀可复用工作流：

```text
写单测 skill
修 TypeScript 报错 skill
做 React 组件样式调整 skill
做接口字段迁移 skill
```

### Hooks

Hooks 用来在生命周期点做强制检查：

```text
before_apply: 检查敏感文件
after_apply: 自动跑格式化
before_verify: 校验命令白名单
after_verify: 写 eval ledger
```

### 文件

```text
packages/core/src/mcp/mcp-client.ts
packages/core/src/skills/skill-loader.ts
packages/core/src/hooks/hook-runner.ts
.minicodex/skills/
.minicodex/hooks.json
```

### 验收

```text
能加载一个本地 skill。
能执行一个 before_apply hook。
外部工具失败不会让 run 静默成功。
```

## 13. Upgrade 09：Memory

### 目标

让 Mini Codex 记住项目偏好、用户偏好和失败教训，但不要把 memory 当成事实真相。

### 文件

```text
.minicodex/memory/project.md
.minicodex/memory/user-preferences.md
.minicodex/memory/failure-lessons.jsonl
packages/core/src/memory/memory-store.ts
```

### 写入时机

```text
delivery 后总结本次任务
用户拒绝 plan/diff 后记录偏好
verify 失败后记录失败原因
eval 后记录回归问题
```

### 验收

```text
下一次 run 能读取 memory 摘要。
memory 不允许覆盖权限策略。
用户可以查看和删除 memory。
```

## 14. Upgrade 10：Subtask / Subagent

### 目标

不是一上来做多 agent，而是先做可控子任务。

### 适合拆分的任务

```text
搜索相关文件
分析失败日志
生成测试候选
总结 diff 风险
评估 repair 是否合理
```

### 不适合第一版拆分的任务

```text
并发写文件
多个 agent 同时 apply
无人确认自动提交
```

### 验收

```text
子任务只读为主。
子任务结果进入主 run trace。
主 runtime 仍然负责最终决策和写入。
```

## 15. Upgrade 11：Eval Dashboard

### 目标

让 eval 从 JSONL 台账变成能指导迭代的面板。

### 指标

```text
任务成功率
一次通过率
平均 repair 次数
平均耗时
常见失败阶段
最常见拒绝原因
回归用例数量
```

### 文件

```text
apps/desktop/src/views/EvalDashboard.vue
packages/core/src/eval/eval-summary.ts
```

### 验收

```text
能看到最近 20 个 eval case。
能按失败阶段过滤。
能从失败 case 打开 trace。
```

## 16. 本 Lab 结束后的能力

完成这个升级包后，你不只是有一个 demo，而是有一个工作中能用的本地 coding agent：

```text
它知道项目规则。
它有配置。
它能接模型。
它有权限边界。
它能安全 apply 和 rollback。
它能运行验证。
它能接外部工具。
它能沉淀 skill。
它能用 hook 做强制检查。
它能记住偏好和失败。
它能用 eval 指导下一轮改进。
```

这才是“复刻真实 Codex 70% 工作能力”的方向。

## 17. 设计复盘

做完后，请在 `design-journal.md` 写：

```text
我现在的 Mini Codex 和官方 Codex 还差什么？
哪些能力已经足够支撑日常工作？
哪些能力只是看起来高级，但暂时不值得做？
如果让我重做，我会从哪个模块开始抽象？
如果公司团队要用，我最担心哪个风险？
```

## 18. 进阶版边界

进阶版会逐个实现系统安全规则、结构化计划、完整改动应用、保存点、撤回能力、受控命令执行、项目资料检索、长期记忆、可复用工作流、外部工具连接、自动检查点、代码版本管理和质量面板。完成主线后，再按独立进阶安排逐步补齐这些能力。

进阶课独立维护，暂不随本基础课仓库开源。本仓库只提供基础闭环和升级方向，方便读者先把最小 Mini Codex 跑稳。

## 19. 下一步

进入 [练习案例](./12-demo-cases.md)，用真实任务验证升级后的 Mini Codex 是否真的能在工作中使用。
