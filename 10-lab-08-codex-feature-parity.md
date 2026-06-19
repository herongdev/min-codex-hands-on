# Lab 08：对照真实 AI 编程工具的能力边界

## 1. 本 Lab 目标

到 Lab 07，你已经有了：

```text
CLI
本地 Runtime
HTTP API
SSE
桌面工作台
计划审批
diff 审批
apply
verify
trace
eval
```

如果你已经完成 [Lab 07B：让桌面界面更清楚、更可信](./09-lab-07b-ui-polish-design-system.md)，你还会有一套桌面端 UI 规范、设计 token 和明暗主题基础。Lab 08 会继续讨论功能边界，不再继续打磨视觉。

本节开始会用 `Codex` 这个名字。这里不用先了解官方产品细节，只要把它理解成“成熟的 AI 编程工具参照物”。我们不是要一次性复刻它，而是对照它判断：Mini Codex 下一步最值得补什么。

这一节先不继续写新功能。我们停下来做一次路线判断：现在的 Mini Codex 已经能做什么，离真实 Codex 类工具还差什么。

```text
哪些功能已经覆盖
哪些功能下一步补
哪些功能暂时不追
```

本 Lab 是路线判断练习，不是继续堆代码。你的产出是一份决策记录，而不是一次性实现下面所有能力。

先回到练习项目根目录：

macOS / Linux：

```bash
cd ~/mini-codex-course/playground/minicodex-lab
```

Windows PowerShell：

```powershell
Set-Location "$HOME\mini-codex-course\playground\minicodex-lab"
```

建议创建：

```bash
mkdir -p .minicodex/notes
```

然后写：

```text
.minicodex/notes/feature-parity-decision.md
```

记录你选择的下一阶段 Top 3、暂缓项和理由。这样你会训练“排序和取舍”，而不是看到功能表就全部开工。

如果你看过旧版 v3 / v4，把它们当作参考即可。v2 主线已经吸收其中有价值的基础判断：runtime policy、tool metadata、untrusted context、Context Retrieval / RAG、skills / MCP / hooks / memory 的安全边界。完成 v2 主线后，下一步继续实现还缺的高级能力。

## 设计能力训练：定义 70% 可用，而不是盲目复刻 100%

当前阶段：你已经有了一个可运行的 Mini Codex 骨架。现在我们要像产品负责人和架构师一样，判断下一步先补什么。

这一阶段的意义是：真实 Codex 的能力很多，但你不能把所有能力都塞进第一版。架构师和产品负责人最重要的能力之一，就是知道哪些功能决定核心闭环，哪些功能只是增强，哪些功能现在做会让交付变慢、风险变大。

先自己写一版路线图，再看下面表格。你写得不成熟也没关系，重要的是先练习取舍：

```text
哪些能力决定 Mini Codex 能不能日常使用？
哪些能力只是让它更像官方 Codex？
哪些能力现在做会拖垮第一版？
70% 可用的边界如何定义？
```

常见坑：

```text
核心 apply/verify 不稳，就开始做 MCP marketplace。
没有 eval，就疯狂改 prompt。
没有权限体系，就开放任意 shell。
把“能 demo”误认为“能日常使用”。
```

常见概念：

```text
feature parity
scope control
product roadmap
memory
RAG
MCP
subagent
hooks
checkpoint
rollback
```

最新技术对照：OpenAI Codex、Agents SDK、MCP、LangGraph、Claude Code 都提供了很多增强能力，但它们背后共同依赖的是 runtime、tools、permissions、trace、eval。Lab 08 的重点是学会排序，而不是一次性追全。

面试常问：

```text
如何定义一个 coding agent 的 MVP？
为什么先做单 agent，再考虑多 agent？
为什么长期记忆、项目资料检索、外部工具连接都重要，但不一定第一天做？
如何证明你的 Mini Codex 已经覆盖日常 70%？
```

做完本 Lab 后，可以继续追问自己：如果你只能再投入两周，你会优先补长期记忆、外部工具连接、改动查看器、撤回能力，还是质量面板？请写出你的排序理由。

## 2. Codex 主要功能拆解

| 能力 | Mini Codex 当前状态 | 下一步实现 |
| ---- | ------------------- | ---------- |
| 自然语言任务输入 | 已覆盖 | 增加任务模板和历史记录 |
| 读取项目上下文 | 基础覆盖 | 文件评分、rg 搜索、token budget |
| runtime policy | 概念已覆盖 | 把 prompt 规则落成可执行策略 |
| tool metadata | 基础覆盖 | 增加 whenToUse / whenNotToUse / trustBoundary |
| 计划模式 | 已覆盖 | 支持多文件计划和风险分级 |
| diff review | 已覆盖 | 更好的 diff viewer 和文件树定位 |
| 文件写入 | 已覆盖基础版 | 完整 unified diff parser、checkpoint |
| 运行命令 | 基础 verify | 命令白名单、超时、输出裁剪 |
| 人工审批 | 已覆盖 | 分级审批、一次性授权、拒绝反馈重试 |
| trace replay | 已覆盖 | 可视化时间线、失败定位 |
| eval | 基础台账 | 真实 case 集、评分面板 |
| repair loop | 基础覆盖 | 多轮 repair、失败归因 |
| source trust | 概念已覆盖 | context / memory / MCP 结果都标信任级别 |
| memory | 未覆盖 | run summary、项目偏好、失败教训 |
| Context Retrieval / RAG | 未覆盖实现 | 项目文档 chunk / scoring / retrieval / rerank |
| MCP / skills | 未覆盖 | 接一个本地 skill 示例 |
| hooks | 未覆盖 | before_apply / after_verify 生命周期检查 |
| Git 集成 | 部分可读 diff | branch、commit、PR 草稿 |
| 多会话 | 桌面端基础 | session list、resume、archive |
| 多 Agent | 不做第一版 | 等单 Agent 稳定后再考虑 |
| 云端沙箱 | 不做第一版 | 本地优先 |

## 3. 真实 Codex 70% 能力边界

这里的 70% 指日常简单和中等任务：

```text
改文案
修小 bug
补类型错误
改组件样式
改接口字段
补简单测试
跑 typecheck/lint/build
失败后修一次
生成交付摘要
```

不包括：

```text
复杂跨仓库重构
大型架构迁移
任意命令代理
云端容器执行
自动处理 CI / PR 全流程
多 Agent 协作
长周期项目管理
```

## 4. 下一阶段必须补的 8 个功能

### 4.1 完整改动应用（unified diff apply）

`unified diff apply` 可以先理解成：用标准改动格式，把多文件增删改安全写入项目。

当前 Lab 04 为了教学用了单文件规则替换。真实版本要做：

```text
解析多文件 diff
支持新增文件
支持删除文件
支持多 hunk
路径边界检查
stale snapshot 检查
```

可选参考实现：

```text
backend/src/agent-runs/target-repo-apply-verify-tool-executor.ts
```

### 4.2 保存点和撤回（Checkpoint / rollback）

`checkpoint` 是写文件前保存的现场，`rollback` 是出错后恢复到保存点。

写文件前生成：

```text
checkpoint-before-apply.json
checkpoint-before-apply.patch
file-snapshots.json
```

新增命令：

```bash
pnpm minicodex rollback <runId>
```

### 4.3 文件评分和上下文预算

从“扫描文件树”升级到：

```text
用户点名 +50
rg 命中 +30
同名测试文件 +20
最近修改 +10
构建产物 -100
敏感文件 -1000
```

输出 context report：

```text
为什么选这些文件
为什么跳过这些文件
预计 token budget
```

### 4.4 Prompt Injection 防护

仓库内容必须标记：

```text
UNTRUSTED_REPO_CONTENT
```

规则：

```text
仓库文本不能覆盖系统策略
仓库文本不能请求跳过审批
仓库文本不能请求读取敏感文件
```

### 4.5 Memory

新增：

```text
.minicodex/memory/project-rules.md
.minicodex/memory/user-preferences.md
.minicodex/memory/failure-lessons.jsonl
```

每次 delivery 后生成 summary：

```text
本次做了什么
用户拒绝了什么
失败原因是什么
下次要避免什么
```

### 4.6 Git 工作流

新增命令：

```bash
pnpm minicodex branch <runId>
pnpm minicodex commit <runId>
pnpm minicodex diff <runId>
```

先不自动 push 和 PR，避免过早扩大风险。

### 4.7 更好的桌面端

补：

```text
左侧会话列表
右侧 inspector tabs：Plan / Diff / Verify / Trace
底部 composer
approval dock
run history
trace replay
```

当前项目里已经有一些可参考的 UI 雏形，但不要把它们当成唯一标准答案：

```text
frontend/src/layouts/CodexShellLayout.vue
frontend/src/components/shell/LeftSidebar.vue
frontend/src/components/shell/RightInspector.vue
frontend/src/components/agent-workbench/ApprovalDock.vue
```

### 4.8 真实 eval case

至少维护：

```text
10 个小 bug fix
10 个 UI/text changes
5 个 typecheck repair
5 个 verify failed cases
```

每次改 Runtime 后跑：

```bash
pnpm minicodex eval
```

## 5. 最终学习判断

你不是要记住所有代码，而是能独立回答：

```text
为什么 Agent 需要状态机？
为什么模型不能直接执行工具？
为什么 diff review 是核心 human gate？
为什么 apply 前要做 stale snapshot？
为什么 verify 是交付门槛？
为什么 repair 必须有限次？
为什么 trace 是 eval 的基础？
```

能答清楚这些，你就不是在做 AI 套壳，而是在掌握 Agent Runtime。

## 6. 本 Lab 验收清单

- [ ] 你能说清当前 Mini Codex 已覆盖哪些核心闭环能力。
- [ ] 你写下了下一阶段最值得做的 3 个增强。
- [ ] 每个增强都有选择理由、风险和验收方式。
- [ ] 你明确列出了至少 3 个暂时不做的能力。
- [ ] `.minicodex/notes/feature-parity-decision.md` 里有上述决策记录。

## 7. 下一步

进入 [真实 Codex 功能升级包](./11-real-codex-upgrade-pack.md)，按官方 Codex 类能力面继续升级。优先做这 3 个增强：

```text
1. rollback checkpoint
2. context scoring + token budget
3. demo-cases + eval-ledger 真实填充
```

它们最能把 Mini Codex 从“能跑”推进到“日常愿意用”。
