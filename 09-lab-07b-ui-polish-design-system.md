# Lab 07B：让桌面界面更清楚、更可信

## 1. 本 Lab 最终效果

到 Lab 07，你已经有了能跑的桌面工作台：

```text
选择 targetRoot
输入任务
创建 run
订阅 SSE
展示事件
处理审批
查看 trace
```

这一节不新增 runtime 能力，我们专心把 UI / UX 做得更像一个可信的工具产品。完成后你会得到：

```text
一份 Mini Codex UI 规范文档
一套设计 token
明暗主题兼容
Tailwind CSS v4 接入方式
更像 Codex 类工作台的页面结构
更清楚的任务、状态、审批、事件、trace 层级
```

本节仍然是教程，不是让你一次性照抄一个复杂 UI。我们按步骤改，每一步都能跑起来，你也能看懂为什么这么改。

## 2. 设计能力训练：进入资深 UI / UX 设计师模式

本节先认识这些名字：

```text
UI：用户看见和操作的界面。
UX：用户完成任务时的整体体验。
设计系统：一组统一的颜色、字体、间距、按钮和状态规则。
设计 token：把颜色、字号、间距等设计值做成统一变量，方便全局复用和明暗主题切换。
Tailwind CSS：一种用 class 快速写样式的工具，本节使用 v4。
```

这一节请先切换到“资深 UI / UX 设计师模式”。意思不是做花哨，而是先想清楚用户真正需要什么：

```text
先定义用户是谁
再定义主要任务
再定义视觉层级
再定义 token
再落到组件和页面
最后用截图和可用性检查反推调整
```

Mini Codex 桌面端不是营销页，也不是普通聊天页。它是一个 coding agent 控制台。用户最在意：

```text
现在任务跑到哪一步？
我是否需要审批？
这次操作会改什么？
验证有没有通过？
失败后我该看哪里？
trace 能不能帮助我复盘？
```

所以 UI 的气质应该接近 Codex 类工作台：

```text
冷静
克制
高信息密度
低装饰
清晰状态
明确风险
可扫描
可审查
```

尽量不要做成：

```text
大面积渐变首页
营销式 hero
巨大的聊天气泡
花哨卡片堆叠
没有状态焦点的 dashboard
```

## 3. 当前 UI 的问题

Lab 07 的最小 UI 是正确的起点，但它还有明显问题：

```text
没有统一设计 token
明暗主题没有规划
按钮、输入框、卡片都靠局部 class 或 scoped CSS
任务输入、状态、审批、事件、trace 的视觉层级不够清楚
审批区域没有足够强的风险焦点
事件流和 trace 看起来像普通日志，而不是可审查证据
```

本节目标不是单纯“变漂亮”，而是让用户更信任这个工具、更容易判断风险、更愿意每天打开它。

## 4. 文件和目录

本节从 Lab 07 完成后的代码继续，工作目录固定为：

macOS / Linux：
```bash
cd ~/mini-codex-course/playground/minicodex-lab
```

Windows PowerShell：

```powershell
Set-Location "$HOME\mini-codex-course\playground\minicodex-lab"
```

你会新增或修改这些文件：

```text
docs/ui/minicodex-desktop-ui-guidelines.md
apps/desktop/src/styles/tokens.css
apps/desktop/src/style.css
apps/desktop/src/App.vue
```

如果你还没有接 Tailwind CSS v4，先确认 Lab 07 已经做过（`vite`、`@vitejs/plugin-vue` 应在 `devDependencies`，与 Tailwind 同类）：

```bash
pnpm --dir apps/desktop add -D tailwindcss @tailwindcss/vite lightningcss
```

`lightningcss` 是 Tailwind v4 走 Vite 时的可选 peer；在 pnpm 下**显式装上**可避免 IDE 报「两个 `vite` 的 `Plugin` 类型不兼容」（`defineConfig` 与 `@tailwindcss/vite` 各解析到一份 `vite`）。

并确认 `apps/desktop/vite.config.ts`（与 [Tailwind 官方 Vite 文档](https://tailwindcss.com/docs/installation/using-vite) 一致；与 `vue` 并存时用展开，因为 `tailwindcss()` 返回的是 `Plugin[]`）：

```ts
import tailwindcss from "@tailwindcss/vite";
import vue from "@vitejs/plugin-vue";
import { defineConfig } from "vite";

/** 从统一环境变量读取桌面端地址，避免端口散落在多个文件里。 */
const desktopUrl = process.env.MINICODEX_DESKTOP_URL ?? "http://localhost:5173";
const desktopPort = Number(new URL(desktopUrl).port || 5173);

/** 同时启用 Vue 和 Tailwind v4，Vite server 端口跟随 .env.local。 */
export default defineConfig({
  plugins: [vue(), ...tailwindcss()],
  server: {
    port: desktopPort,
  },
});
```

## 5. 先写 UI 规范文档

先不要急着改 `App.vue`。先创建：

```bash
mkdir -p docs/ui
```

新建 `docs/ui/minicodex-desktop-ui-guidelines.md`：

```md
# Mini Codex Desktop UI Guidelines

## Product Position

Mini Codex Desktop is a local coding-agent workbench.
It is not a marketing site and not a chat-first product.

The UI must help the user answer:

- What task is running?
- Which phase is the agent in?
- What is waiting for approval?
- What files or commands are risky?
- Did verification pass?
- Where can I inspect trace evidence?

## Design Direction

The visual style should feel:

- calm
- precise
- dense but readable
- trustworthy
- tool-like
- low decoration

Avoid:

- large decorative gradients
- random background blobs
- nested cards
- oversized hero typography
- chat bubbles as the primary structure
- single-hue purple / blue-heavy decoration

## Layout Model

Use a three-zone workbench:

1. Left rail: workspace, target root, run controls.
2. Center panel: current task, phase, approval, delivery.
3. Right / lower panel: event timeline, trace, artifacts.

The first visual focus is always the active run state.

## Interaction Rules

- Approval actions must be visually stronger than ordinary events.
- Destructive or risky actions must show reason and scope.
- Disabled buttons must look disabled and keep stable size.
- Trace is evidence, not decoration.
- Event cards should be scannable by type, title, and content.

## Theme

Support light and dark themes through semantic design tokens.
Components should use tokens, not raw hex colors.

## Token Categories

- canvas
- surface
- surface-muted
- border
- text
- text-muted
- accent
- accent-foreground
- danger
- warning
- success
- focus-ring
- radius
- shadow
```

这份文档不是摆设。后面每次改 UI，都先对照它。

## 6. 建立设计 Token

创建目录：

```bash
mkdir -p apps/desktop/src/styles
```

新建 `apps/desktop/src/styles/tokens.css`：

```css
:root {
  color-scheme: light;

  --mc-canvas: #f4f5f7;
  --mc-surface: #ffffff;
  --mc-surface-muted: #f8fafc;
  --mc-surface-raised: #ffffff;

  --mc-border: #d8dee8;
  --mc-border-strong: #b8c0cc;

  --mc-text: #111827;
  --mc-text-muted: #667085;
  --mc-text-subtle: #8a94a6;

  --mc-accent: #2563eb;
  --mc-accent-hover: #1d4ed8;
  --mc-accent-foreground: #ffffff;

  --mc-success: #15803d;
  --mc-success-soft: #ecfdf3;
  --mc-warning: #b7791f;
  --mc-warning-soft: #fffbeb;
  --mc-danger: #b42318;
  --mc-danger-soft: #fef3f2;

  --mc-focus-ring: rgba(37, 99, 235, 0.22);

  --mc-radius-sm: 4px;
  --mc-radius-md: 6px;
  --mc-radius-lg: 8px;

  --mc-shadow-sm: 0 1px 2px rgba(16, 24, 40, 0.06);
}

@media (prefers-color-scheme: dark) {
  :root {
    color-scheme: dark;

    --mc-canvas: #0f1115;
    --mc-surface: #171a21;
    --mc-surface-muted: #1f2430;
    --mc-surface-raised: #202632;

    --mc-border: #303744;
    --mc-border-strong: #465063;

    --mc-text: #f3f4f6;
    --mc-text-muted: #a7b0c0;
    --mc-text-subtle: #7c8799;

    --mc-accent: #60a5fa;
    --mc-accent-hover: #93c5fd;
    --mc-accent-foreground: #08111f;

    --mc-success: #4ade80;
    --mc-success-soft: #102318;
    --mc-warning: #facc15;
    --mc-warning-soft: #2a2108;
    --mc-danger: #f87171;
    --mc-danger-soft: #2a1111;

    --mc-focus-ring: rgba(96, 165, 250, 0.28);

    --mc-shadow-sm: none;
  }
}
```

为什么用 CSS variables？

```text
Tailwind 负责排版和布局效率。
CSS variables 负责产品语义和主题。
```

不要把所有颜色都写成 Tailwind 的 `bg-white`、`text-slate-900`。那样以后暗色主题会很痛。

## 7. 接入全局样式

修改 `apps/desktop/src/style.css`：

```css
@import "tailwindcss";
@import "./styles/tokens.css";

html,
body,
#app {
  min-height: 100%;
}

body {
  margin: 0;
  background: var(--mc-canvas);
  color: var(--mc-text);
  font-family:
    Inter,
    ui-sans-serif,
    system-ui,
    -apple-system,
    BlinkMacSystemFont,
    "Segoe UI",
    sans-serif;
}

button,
input,
textarea {
  font: inherit;
}

pre {
  white-space: pre-wrap;
  word-break: break-word;
}
```

确认 `apps/desktop/src/main.ts` 已经引入：

```ts
import "./style.css";
```

## 8. 页面结构设计

把页面改成三块：

```text
Shell
├── Left rail：项目和输入
├── Main work area：当前 run、审批、delivery
└── Evidence area：events / trace
```

第一版可以仍然用一个文件 `App.vue`，不要急着拆组件。等 UI 稳定后再拆：

```text
RunComposer.vue
RunStatusPanel.vue
ApprovalPanel.vue
EventTimeline.vue
TracePanel.vue
```

## 8.1 UI 规范、token 和页面的数据流

本节的代码不是从 `App.vue` 开始长出来的，而是从“界面不可信、状态不清楚”这个问题长出来的：

```text
当前 UI 问题
-> docs/ui/minicodex-desktop-ui-guidelines.md 定义界面判断标准
-> apps/desktop/src/styles/tokens.css 定义颜色、字体、间距、圆角、阴影
-> apps/desktop/src/style.css 引入 Tailwind 和 tokens
-> apps/desktop/src/App.vue 使用 token class 和布局结构
```

数据和样式各走各的边界：

```text
run 数据：backend -> store -> App.vue
视觉规则：UI guidelines -> tokens.css -> style.css -> App.vue
端口配置：.env.local -> vite.config.ts -> dev server
```

这样后面要改暗色主题、间距或字体时，优先改 token；要改业务状态展示时，优先改 store 和组件结构。

## 9. 替换 App.vue

下面是第一版“Codex 风格工作台”示例。它不是最终设计，但有正确的层级和 token 用法。

替换 `apps/desktop/src/App.vue`：

```vue
<script setup lang="ts">
import { computed } from "vue";
import { useWorkbenchStore } from "./stores/workbench";

const store = useWorkbenchStore();

const phase = computed(() => store.currentRun?.state?.phase ?? "idle");
const status = computed(() => store.currentRun?.status ?? "not_started");
</script>

<template>
  <main
    class="grid min-h-screen grid-cols-[var(--mc-rail-width)_1fr] bg-(--mc-canvas) text-(--mc-text)"
  >
    <aside
      class="border-r border-(--mc-border) bg-(--mc-surface) px-(--mc-space-5) py-(--mc-space-5)"
    >
      <div>
        <p
          class="text-(length:--mc-text-xs) font-medium uppercase leading-(--mc-line-xs) tracking-wide text-(--mc-text-subtle)"
        >
          Mini Codex
        </p>
        <h1
          class="mt-(--mc-space-1) text-(length:--mc-text-lg) font-semibold leading-(--mc-line-lg)"
        >
          Agent Workbench
        </h1>
      </div>

      <label
        class="mt-(--mc-space-6) block text-(length:--mc-text-sm) font-medium leading-(--mc-line-sm)"
      >
        Target Root
        <input
          v-model="store.targetRoot"
          class="mt-(--mc-space-2) w-full rounded-(--mc-radius-md) border border-(--mc-border) bg-(--mc-surface-raised) px-(--mc-space-3) py-(--mc-space-2) text-(length:--mc-text-sm) leading-(--mc-line-sm) outline-none focus:border-(--mc-accent) focus:ring-4 focus:ring-(--mc-focus-ring)"
        />
      </label>

      <button
        class="mt-(--mc-space-3) w-full rounded-(--mc-radius-md) border border-(--mc-border) bg-(--mc-surface-muted) px-(--mc-space-3) py-(--mc-space-2) text-(length:--mc-text-sm) font-medium leading-(--mc-line-sm) hover:border-(--mc-border-strong)"
        @click="store.selectTargetRoot"
      >
        选择目录
      </button>

      <div
        class="mt-(--mc-space-6) rounded-(--mc-radius-lg) border border-(--mc-border) bg-(--mc-surface-muted) p-(--mc-space-3)"
      >
        <div
          class="flex items-center justify-between text-(length:--mc-text-xs) leading-(--mc-line-xs)"
        >
          <span class="text-(--mc-text-muted)">Status</span>
          <span class="font-medium">{{ status }}</span>
        </div>
        <div
          class="mt-(--mc-space-2) flex items-center justify-between text-(length:--mc-text-xs) leading-(--mc-line-xs)"
        >
          <span class="text-(--mc-text-muted)">Phase</span>
          <span class="font-medium">{{ phase }}</span>
        </div>
      </div>
    </aside>

    <section
      class="grid min-h-screen grid-cols-[minmax(0,1fr)_var(--mc-evidence-width)"
    >
      <div class="px-(--mc-space-6) py-(--mc-space-5)">
        <section
          class="rounded-(--mc-radius-lg) border border-(--mc-border) bg-(--mc-surface) p-(--mc-space-4) shadow-(--mc-shadow-sm)"
        >
          <div class="flex items-center justify-between">
            <div>
              <h2
                class="text-(length:--mc-text-base) font-semibold leading-(--mc-line-base)"
              >
                Task
              </h2>
              <p
                class="mt-(--mc-space-1) text-(length:--mc-text-sm) leading-(--mc-line-sm) text-(--mc-text-muted)"
              >
                Describe the change. Mini Codex will plan, ask for approval,
                apply, verify, and record trace.
              </p>
            </div>
            <button
              :disabled="store.isRunning"
              class="min-h-(--mc-control-height) rounded-(--mc-radius-md) bg-(--mc-accent) px-(--mc-space-4) py-(--mc-space-2) text-(length:--mc-text-sm) font-medium leading-(--mc-line-sm) text-(--mc-accent-foreground) hover:bg-(--mc-accent-hover) disabled:cursor-not-allowed disabled:opacity-50"
              @click="store.startRun"
            >
              Run
            </button>
          </div>

          <textarea
            v-model="store.message"
            rows="4"
            class="mt-(--mc-space-4) w-full rounded-(--mc-radius-md) border border-(--mc-border) bg-(--mc-surface-raised) px-(--mc-space-3) py-(--mc-space-2) text-(length:--mc-text-sm) leading-(--mc-line-sm) outline-none focus:border-(--mc-accent) focus:ring-4 focus:ring-(--mc-focus-ring)"
          />
        </section>

        <section
          v-if="store.pendingApproval"
          class="mt-(--mc-space-4) rounded-(--mc-radius-lg) border border-(--mc-warning) bg-(--mc-warning-soft) p-(--mc-space-4)"
        >
          <div class="flex items-start justify-between gap-(--mc-space-4)">
            <div>
              <p
                class="text-(length:--mc-text-xs) font-semibold uppercase leading-(--mc-line-xs) tracking-wide text-(--mc-warning)"
              >
                Approval Required
              </p>
              <h2
                class="mt-(--mc-space-1) text-(length:--mc-text-base) font-semibold leading-(--mc-line-base)"
              >
                待确认
              </h2>
              <p
                class="mt-(--mc-space-2) text-(length:--mc-text-sm) leading-(--mc-line-sm)"
              >
                {{ store.pendingApproval.question }}
              </p>
              <p
                class="mt-(--mc-space-1) text-(length:--mc-text-sm) leading-(--mc-line-sm) text-(--mc-text-muted)"
              >
                {{ store.pendingApproval.reason }}
              </p>
            </div>
            <div class="flex shrink-0 gap-(--mc-space-2)">
              <button
                class="min-h-(--mc-control-height) rounded-(--mc-radius-md) bg-(--mc-success) px-(--mc-space-3) py-(--mc-space-2) text-(length:--mc-text-sm) font-medium leading-(--mc-line-sm) text-white"
                @click="store.approve(true)"
              >
                确认
              </button>
              <button
                class="min-h-(--mc-control-height) rounded-(--mc-radius-md) border border-(--mc-border) bg-(--mc-surface) px-(--mc-space-3) py-(--mc-space-2) text-(length:--mc-text-sm) font-medium leading-(--mc-line-sm)"
                @click="store.approve(false)"
              >
                拒绝
              </button>
            </div>
          </div>
        </section>

        <p
          v-if="store.errorMessage"
          class="mt-(--mc-space-4) rounded-(--mc-radius-md) bg-(--mc-danger-soft) px-(--mc-space-3) py-(--mc-space-2) text-(length:--mc-text-sm) leading-(--mc-line-sm) text-(--mc-danger)"
        >
          {{ store.errorMessage }}
        </p>

        <section class="mt-(--mc-space-4)">
          <div class="mb-(--mc-space-2) flex items-center justify-between">
            <h2
              class="text-(length:--mc-text-sm) font-semibold leading-(--mc-line-sm)"
            >
              Events
            </h2>
            <button
              :disabled="!store.currentRun"
              class="min-h-(--mc-control-height) rounded-(--mc-radius-md) border border-(--mc-border) bg-(--mc-surface) px-(--mc-space-3) py-(--mc-space-1) text-(length:--mc-text-xs) font-medium leading-(--mc-line-xs) disabled:opacity-50"
              @click="store.loadTrace"
            >
              Load Trace
            </button>
          </div>

          <article
            v-for="event in store.events"
            :key="event.id"
            class="mb-(--mc-space-3) rounded-(--mc-radius-lg) border border-(--mc-border) bg-(--mc-surface) p-(--mc-space-4) shadow-(--mc-shadow-sm)"
          >
            <div class="flex items-center gap-(--mc-space-2)">
              <span
                class="rounded-(--mc-radius-sm) bg-(--mc-surface-muted) px-(--mc-space-2) py-(--mc-space-1) text-(length:--mc-text-xs) leading-(--mc-line-xs) text-(--mc-text-muted)"
              >
                {{ event.type }}
              </span>
              <strong
                class="text-(length:--mc-text-sm) leading-(--mc-line-sm)"
                >{{ event.title }}</strong
              >
            </div>
            <pre
              class="mt-(--mc-space-3) text-(length:--mc-text-xs) leading-(--mc-line-xs) text-(--mc-text-muted)"
              >{{ event.content }}</pre
            >
          </article>
        </section>
      </div>

      <aside
        class="border-l border-(--mc-border) bg-(--mc-surface) px-(--mc-space-4) py-(--mc-space-5)"
      >
        <h2
          class="text-(length:--mc-text-sm) font-semibold leading-(--mc-line-sm)"
        >
          Trace Evidence
        </h2>
        <p
          class="mt-(--mc-space-1) text-(length:--mc-text-xs) leading-(--mc-line-xs) text-(--mc-text-muted)"
        >
          Use trace to review what the agent actually did.
        </p>

        <div
          v-if="!store.traceLines.length"
          class="mt-(--mc-space-4) rounded-(--mc-radius-lg) border border-dashed border-(--mc-border) p-(--mc-space-4) text-(length:--mc-text-sm) leading-(--mc-line-sm) text-(--mc-text-muted)"
        >
          No trace loaded yet.
        </div>

        <article
          v-for="line in store.traceLines"
          :key="line.eventIndex"
          class="mt-(--mc-space-3) rounded-(--mc-radius-md) border border-(--mc-border) bg-(--mc-surface-muted) p-(--mc-space-3)"
        >
          <strong class="text-(length:--mc-text-xs) leading-(--mc-line-xs)"
            >[{{ line.eventIndex }}] {{ line.type }}</strong
          >
          <pre
            class="mt-(--mc-space-2) text-(length:--mc-text-xs) leading-(--mc-line-xs) text-(--mc-text-muted)"
            >{{ line.event.title }}</pre
          >
        </article>
      </aside>
    </section>
  </main>
</template>
```

## 10. 明暗主题检查

本节用 `prefers-color-scheme` 自动适配系统主题。验证方式：

```text
macOS: System Settings -> Appearance -> Light / Dark
浏览器 DevTools: Rendering -> Emulate CSS prefers-color-scheme
```

检查：

```text
文字不发灰
边框仍然可见
审批区域仍然突出
按钮 hover / disabled 状态清楚
trace 的代码文本可读
```

不要只看浅色模式。桌面端工具通常会被长时间打开，暗色主题不是装饰，是使用舒适度。

## 11. 运行

启动三个进程：

```bash
pnpm dev:backend
pnpm dev:desktop
pnpm dev:electron
```

或使用你前面配置好的并行脚本：

```bash
pnpm dev:all
```

验证：

```text
页面能创建 run
审批卡片可见
事件列表可扫描
Load Trace 能展示 trace
浅色和暗色都可读
布局没有文字溢出
```

## 12. 常见坑

### 坑一：Tailwind class 写满，但没有 token

这会导致后面改主题很痛。规则是：

```text
布局用 Tailwind
颜色和语义用 token
```

### 坑二：把审批做成普通卡片

审批是风险控制点，不是普通信息。它要更突出。

### 坑三：深色模式只反转背景

真正的深色模式要检查：

```text
文本层级
边框强度
危险 / 警告 / 成功颜色
focus ring
code / pre 内容
```

### 坑四：像聊天页而不是工作台

Coding agent UI 的核心是状态和证据，不是气泡。

## 13. 本 Lab 验收清单

- [ ] 已创建 `docs/ui/minicodex-desktop-ui-guidelines.md`。
- [ ] 已创建 `apps/desktop/src/styles/tokens.css`。
- [ ] `style.css` 引入 Tailwind 和 tokens。
- [ ] `App.vue` 不再依赖一大段 scoped CSS。
- [ ] 页面使用 token 表达主要颜色。
- [ ] 浅色模式可读。
- [ ] 暗色模式可读。
- [ ] 审批区域比普通事件更突出。
- [ ] `pnpm --dir apps/desktop run check` 通过。
- [ ] `pnpm dev:desktop` 可以正常打开页面。

## 14. 设计复盘

在 `design-journal.md` 追加：

```text
## Lab 07B UI 复盘

1. 这个界面的第一视觉焦点是什么？
2. 哪些颜色必须做成 token？
3. 哪些状态需要比普通事件更突出？
4. 这个 UI 现在更像工作台，还是更像聊天页？
5. 如果继续迭代，我会先拆哪些组件？
6. 明暗主题里最容易出问题的元素是什么？
```

## 15. 下一步

进入 [Lab 08：对照真实 AI 编程工具的能力边界](./10-lab-08-codex-feature-parity.md)。
