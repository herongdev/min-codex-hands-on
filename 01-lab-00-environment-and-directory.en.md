# Lab 00: Environment and Practice Workspace

[中文版](./01-lab-00-environment-and-directory.md)

## 1. What This Lab Produces

In this lab, we do not write agent logic yet. We only set up the learning project. When you finish, you will have a fresh Mini Codex practice workspace:

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

It will not do anything "smart" yet, but this skeleton will carry the CLI, backend, desktop UI, task records, and verification logic in later labs.

Create the directory skeleton first, then review why it is split this way.

### 1.1 Enter the practice workspace

This course uses `mini-codex-course/playground/minicodex-lab` as the practice workspace. Later labs assume you continue from the `minicodex-lab` root.

macOS / Linux:

```bash
mkdir -p ~/mini-codex-course/playground/minicodex-lab
cd ~/mini-codex-course/playground/minicodex-lab
```

Windows PowerShell:

```powershell
New-Item -ItemType Directory -Force "$HOME\mini-codex-course\playground\minicodex-lab" | Out-Null
Set-Location "$HOME\mini-codex-course\playground\minicodex-lab"
```

### 1.2 Create the monorepo directory structure

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

## 2. Design Training: Why Split Directories This Way

At this stage, the directory skeleton is ready and you still have not written any agent logic. Use the structure you just created to check whether the boundaries are clear.

The point of this stage is to decide where Mini Codex capabilities live. Real Codex is not a single entry point. It can appear in the CLI, IDE, Web, and desktop surfaces. That is why we extract `core` first and let the CLI, backend, and desktop reuse it.

Take one minute and answer these questions against the directories you just created:

```text
If I only build a CLI, do I still need a core package?
If I add a desktop UI later, what logic must not stay inside the CLI?
Why should targetRoot not default to the tool's own source directory?
Should shared hold business logic, or only shared types?
```

Common pitfalls:

```text
Mixing the learning tool source code with the target repository it modifies.
Putting file I/O, state transitions, and permission checks directly in the CLI.
Starting with Electron before the core loop works.
Creating a monorepo without clear package boundaries.
```

Common terms:

```text
monorepo
workspace
package boundary
runtime core
adapter
targetRoot
```

You can think of `core` and `adapter` like this:

```text
core is the real runtime: save tasks, advance state, call tools, enforce permissions, record traces.
adapter is the entry layer: CLI parses commands, backend exposes HTTP APIs, desktop handles interaction.
```

So one rule, such as "the user approved the next step," should live in `core` once. CLI, backend, and desktop only translate user actions from different entry points into the same core call.

Current product reference: OpenAI Codex has CLI, IDE, Web, and ChatGPT entry points, so capability cannot be locked to one UI. The OpenAI Agents SDK also treats agent, tool, guardrail, and trace as composable modules instead of putting everything in one page.

Interview prompts:

```text
Why use a monorepo?
Why build core before UI?
What adapter role do CLI, backend, and desktop play?
If a user opens multiple targetRoot values at once, how should the directory design change?
```

After this lab, write a short note in `design-journal.md`: if you redesigned a Docs Agent from scratch, would you also use `core + cli + backend + desktop`, and why?

## 3. Create the pnpm workspace

Now create the project skeleton. This is not about making folders look neat. It is about putting later-changing capabilities in the right boundaries:

```text
packages/shared: shared types only, no business flow.
packages/core: Mini Codex runtime logic.
packages/cli: translate CLI input into core calls.
apps/backend: expose core over HTTP later.
apps/desktop: build the desktop UI later.
fixtures/target-repo: sample project that Mini Codex will modify.
```

Create `pnpm-workspace.yaml`:

```yaml
packages:
  - "packages/*"
  - "apps/*"
```

Create the root `package.json`:

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

Create `tsconfig.base.json`:

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

Install dependencies:

```bash
pnpm install
```

## 4. Create the shared package

`packages/shared/package.json`:

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

`packages/shared/tsconfig.json`:

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

`packages/shared/src/index.ts`:

```ts
export type JsonObject = Record<string, unknown>;
```

## 5. Create the core package

`packages/core/package.json`:

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

`packages/core/tsconfig.json`:

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

`packages/core/src/index.ts`:

```ts
export const minicodexCoreVersion = "0.1.0";
```

## 6. Create the cli package

`packages/cli/package.json`:

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

`packages/cli/tsconfig.json`:

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

`packages/cli/src/index.ts`:

```ts
#!/usr/bin/env node
import { Command } from "commander";
import { minicodexCoreVersion } from "@minicodex/core";

const program = new Command();

program
  .name("minicodex")
  .description("A local Mini Codex runtime")
  .version(minicodexCoreVersion);

/** Minimal runnable command: proves the CLI starts and reads the core package. */
program.command("hello").action(() => {
  console.log("Mini Codex is alive.");
});

program.parse(process.argv);
```

## 7. Create the target repository fixture

`fixtures/target-repo/package.json`:

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

`fixtures/target-repo/tsconfig.json`:

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

`fixtures/target-repo/src/button.ts`:

```ts
/** Sample business function: later labs will change the return text from "登录" to "提交". */
export function getLoginButtonText(): string {
  return "登录";
}
```

`fixtures/target-repo/AGENTS.md`:

```md
# AGENTS.md

This repository is used for Mini Codex course practice.

Rules:

- Prefer editing TypeScript files under src/.
- Run pnpm run check after changes.
- Do not modify package manager configuration.
```

## 8. Install workspace dependencies and verify

From the `minicodex-lab` root, run:

```bash
pnpm install
pnpm run build
pnpm minicodex hello
```

Expected output:

```text
Mini Codex is alive.
```

## 8.1 Skeleton call chain and data flow

This lab only verifies the smallest path:

```text
pnpm minicodex hello
-> packages/cli/src/index.ts
-> read minicodexCoreVersion exported by @minicodex/core
-> print Mini Codex is alive.
```

The sample target repository is not modified yet. It only becomes input for later labs:

```text
fixtures/target-repo/src/button.ts
-> getLoginButtonText()
-> returns "登录"
```

After this lab, you still do not have agent capability, but you already have the places where later code can grow naturally: CLI for entry, core for rules, and fixture for the project being modified.

## 9. Acceptance checklist

- [ ] The `minicodex-lab` directory exists.
- [ ] `packages/core`, `packages/cli`, and `packages/shared` exist.
- [ ] `apps/backend` and `apps/desktop` exist.
- [ ] `fixtures/target-repo` has `package.json`, `AGENTS.md`, and `src/button.ts`.
- [ ] `pnpm run build` passes.
- [ ] `pnpm minicodex hello` prints `Mini Codex is alive.`.

## 10. Next step

Continue to [Lab 01: Save the first task](./02-lab-01-cli-first-run.md).
