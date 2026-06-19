# Lab 00：环境准备和重新创建目录

## 1. 本 Lab 最终效果

这一节我们先不写 agent 逻辑，只把学习项目搭起来。做完后，你会有一个全新的 Mini Codex 练习目录：

```text
minicodex-lab/
├── package.json
├── pnpm-workspace.yaml
├── tsconfig.base.json
├── packages/
│   ├── core/
│   ├── cli/
│   └── shared/
├── apps/
│   ├── backend/
│   └── desktop/
├── fixtures/
│   └── target-repo/
└── .minicodex/
```

它暂时还不会“智能”做事，但这个骨架会承载后面的命令行、后端、桌面界面、任务记录和检查逻辑。

先完成第 2 节的目录创建，再用下面几个问题检查分层是否清楚。

## 设计能力训练：先想清楚为什么这样分目录

当前阶段：你还没有写任何 agent 逻辑。我们先一起把产品和架构边界想清楚。

这一阶段的意义是：先决定 Mini Codex 的能力放在哪里。真实 Codex 不是只有一个入口，它可以出现在 CLI、IDE、Web、桌面端里。所以我们先把 `core` 抽出来，再让 CLI、后端、桌面端复用它。

动手前，先花一分钟自己回答：

```text
如果我只做 CLI，还需要 core 包吗？
如果以后要做桌面端，哪些逻辑不能写死在 CLI 里？
targetRoot 为什么不能默认等于工具自己的源码目录？
shared 包应该放业务逻辑，还是只放类型？
```

常见坑：

```text
把学习工具源码和被修改的目标仓库混在一起。
把文件读写、状态推进、权限判断都写进 CLI。
一开始就做 Electron，结果核心闭环没跑通。
monorepo 只是建了目录，没有形成清晰职责边界。
```

常见概念：

```text
monorepo
workspace
package boundary
runtime core
adapter
targetRoot
```

可以先把 `core` 和 `adapter` 这样理解：

```text
core 是真正的运行时：保存任务、推进状态、调用工具、判断权限、记录过程。
adapter 是入口层：CLI 负责解析命令，后端负责给界面提供接口，桌面端负责交互展示。
```

所以同一件事，比如“用户批准继续执行”，规则应该只在 `core` 里实现一次。CLI、backend、desktop 只是把不同入口的用户动作翻译成同一个 core 调用。

最新技术对照：OpenAI Codex 同时有 CLI、IDE、Web、ChatGPT 等入口，所以能力不能绑死在某个 UI。OpenAI Agents SDK 也把 agent、tool、guardrail、trace 作为可组合模块，而不是把所有逻辑写进一个页面。

面试常问：

```text
为什么用 monorepo？
为什么先 core 后 UI？
CLI、backend、desktop 分别是什么 adapter？
如果用户同时打开多个 targetRoot，目录设计要怎么调整？
```

做完本 Lab 后，在 `design-journal.md` 里写一小段：如果让你重新设计一个 Docs Agent，你会不会也用 `core + cli + backend + desktop` 这个结构，为什么？

## 2. 创建目录

本教程统一把练习项目放在 `mini-codex-course/playground/minicodex-lab`。后面的 Lab 都默认从 `minicodex-lab` 根目录继续。

macOS / Linux：
```bash
mkdir -p ~/mini-codex-course/playground/minicodex-lab
cd ~/mini-codex-course/playground/minicodex-lab
```

Windows PowerShell：

```powershell
New-Item -ItemType Directory -Force "$HOME\mini-codex-course\playground\minicodex-lab" | Out-Null
Set-Location "$HOME\mini-codex-course\playground\minicodex-lab"
```

创建 monorepo 结构：

```bash
mkdir -p packages/core/src
mkdir -p packages/cli/src
mkdir -p packages/shared/src
mkdir -p apps/backend/src
mkdir -p apps/desktop/src
mkdir -p fixtures/target-repo/src
mkdir -p .minicodex/runs
mkdir -p .minicodex/eval
```

## 3. 创建 pnpm workspace

现在开始创建项目骨架。这里不是为了“目录好看”，而是先把后面会变化的能力放到正确边界里：

```text
packages/shared：只放通用类型，不放业务流程。
packages/core：放 Mini Codex 的核心运行逻辑。
packages/cli：把命令行输入翻译成 core 调用。
apps/backend：后面把 core 暴露成 HTTP API。
apps/desktop：后面做桌面端界面。
fixtures/target-repo：被 Mini Codex 修改的示例项目。
```

新建 `pnpm-workspace.yaml`：

```yaml
packages:
  - "packages/*"
  - "apps/*"
```

新建根 `package.json`：

```json
{
  "name": "minicodex-lab",
  "private": true,
  "version": "0.1.0",
  "type": "module",
  "scripts": {
    "build": "pnpm -r run build",
    "check": "pnpm -r run check",
    "minicodex": "pnpm --dir packages/cli run dev",
    "dev:backend": "pnpm --dir apps/backend run dev",
    "dev:desktop": "pnpm --dir apps/desktop run dev"
  },
  "devDependencies": {
    "@types/node": "^20.12.7",
    "tsx": "^4.19.2",
    "typescript": "^5.4.5"
  }
}
```

新建 `tsconfig.base.json`：

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "outDir": "dist"
  }
}
```

安装依赖：

```bash
pnpm install
```

## 4. 创建 shared 包

`packages/shared/package.json`：

```json
{
  "name": "@minicodex/shared",
  "version": "0.1.0",
  "type": "module",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "scripts": {
    "build": "tsc -p tsconfig.json",
    "check": "tsc -p tsconfig.json --noEmit"
  }
}
```

`packages/shared/tsconfig.json`：

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

`packages/shared/src/index.ts`：

```ts
export type JsonObject = Record<string, unknown>;
```

## 5. 创建 core 包

`packages/core/package.json`：

```json
{
  "name": "@minicodex/core",
  "version": "0.1.0",
  "type": "module",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "scripts": {
    "build": "tsc -p tsconfig.json",
    "check": "tsc -p tsconfig.json --noEmit"
  },
  "dependencies": {
    "@minicodex/shared": "workspace:*",
    "diff": "^5.2.0",
    "zod": "^3.23.8"
  }
}
```

`packages/core/tsconfig.json`：

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

`packages/core/src/index.ts`：

```ts
export const minicodexCoreVersion = "0.1.0";
```

## 6. 创建 cli 包

`packages/cli/package.json`：

```json
{
  "name": "@minicodex/cli",
  "version": "0.1.0",
  "type": "module",
  "bin": {
    "minicodex": "dist/index.js"
  },
  "scripts": {
    "dev": "tsx src/index.ts",
    "build": "tsc -p tsconfig.json",
    "check": "tsc -p tsconfig.json --noEmit"
  },
  "dependencies": {
    "@minicodex/core": "workspace:*",
    "@minicodex/shared": "workspace:*",
    "commander": "^12.1.0"
  }
}
```

`packages/cli/tsconfig.json`：

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

`packages/cli/src/index.ts`：

```ts
#!/usr/bin/env node
import { Command } from "commander";
import { minicodexCoreVersion } from "@minicodex/core";

const program = new Command();

program
  .name("minicodex")
  .description("A local Mini Codex runtime")
  .version(minicodexCoreVersion);

/** 最小可运行命令：证明 CLI 能启动并读取 core 包。 */
program.command("hello").action(() => {
  console.log("Mini Codex is alive.");
});

program.parse(process.argv);
```

## 7. 创建目标仓库 fixture

`fixtures/target-repo/package.json`：

```json
{
  "name": "target-repo",
  "type": "module",
  "scripts": {
    "check": "tsc --noEmit"
  },
  "devDependencies": {
    "typescript": "^5.4.5"
  }
}
```

`fixtures/target-repo/tsconfig.json`：

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "strict": true,
    "noEmit": true
  },
  "include": ["src"]
}
```

`fixtures/target-repo/src/button.ts`：

```ts
/** 示例业务函数：后面的 Lab 会把返回文案从“登录”改成“提交”。 */
export function getLoginButtonText(): string {
  return "登录";
}
```

`fixtures/target-repo/AGENTS.md`：

```md
# AGENTS.md

这个仓库用于 Mini Codex 教程练习。

规则：

- 优先修改 src/ 下的 TypeScript 文件。
- 修改后运行 pnpm run check。
- 不要修改 package manager 配置。
```

## 8. 安装 workspace 依赖并验证

回到 `minicodex-lab` 根目录后执行：

```bash
pnpm install
pnpm run build
pnpm minicodex hello
```

期望输出：

```text
Mini Codex is alive.
```

## 8.1 骨架调用链和数据流

本 Lab 只验证最小链路：

```text
pnpm minicodex hello
-> packages/cli/src/index.ts
-> 读取 @minicodex/core 导出的 minicodexCoreVersion
-> 打印 Mini Codex is alive.
```

示例目标仓库暂时不会被修改，只作为后续 Lab 的输入：

```text
fixtures/target-repo/src/button.ts
-> getLoginButtonText()
-> 返回 "登录"
```

做完这一节，你还没有 agent 能力，但已经有了后续代码自然生长的位置：CLI 负责入口，core 负责规则，fixture 负责被修改的目标项目。

## 9. 本 Lab 验收清单

- [ ] `minicodex-lab` 目录存在。
- [ ] `packages/core`、`packages/cli`、`packages/shared` 存在。
- [ ] `apps/backend`、`apps/desktop` 存在。
- [ ] `fixtures/target-repo` 有 `package.json`、`AGENTS.md`、`src/button.ts`。
- [ ] `pnpm run build` 通过。
- [ ] `pnpm minicodex hello` 输出 `Mini Codex is alive.`。

## 10. 下一步

进入 [Lab 01：保存第一条任务](./02-lab-01-cli-first-run.md)。
