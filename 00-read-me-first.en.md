# Build a Working Mini Codex

[中文版](./00-read-me-first.md)

If you already write business code and use tools such as Codex, Cursor, or Claude Code, but still cannot clearly explain how an agent completes an engineering task, this course is for you.

You will not only learn a few more prompts. You will start from an empty folder, build a local AI coding assistant, and gain a transferable agent-programming skill set:

```text
enter one coding request
-> save it as a task that can continue later
-> read files from a sample project
-> propose a change plan
-> wait for your approval
-> modify files
-> run checks
-> save the whole process
-> show it in a desktop workbench
```

## 1. What You Will Get

### 1.1 A Project You Can Show, Explain, and Upgrade

By the end of the foundation course, you will build a small but complete Mini Codex:

```text
enter one coding request
-> save a task
-> read project files
-> generate a plan
-> wait for your approval
-> generate a change draft
-> modify target files
-> run checks
-> save process records
-> review the result
-> observe everything in a desktop workbench
```

It is not a full replacement for the official Codex product. It is a strong portfolio, interview, and extension starting point because you can show more than "I called a model":

```text
I know how a Coding Agent stores tasks
I know why it should plan first and wait for approval
I know why a model should not freely modify files
I know what risks diff, verify, trace, and eval reduce
I know how to wrap CLI capability into an API and a desktop workbench
```

### 1.2 A Transferable Agent-Programming Skill Set

Mini Codex is the training vehicle. The real skill is the structure you can reuse:

```text
turn one request into a trackable task
make AI propose a plan before changing code
control which actions require approval
review a change draft before writing files
run checks after modifications
save the process for replay and review
turn CLI capability into a desktop workbench
```

After this course, you can transfer the same structure to other tools:

```text
code review assistant: read changes, then produce risk notes and verification suggestions
document assistant: collect sources, then generate an outline, draft, and review record
data checking assistant: read files, then produce a cleaning plan and anomaly report
operations assistant: collect logs, then propose investigation steps and evidence
workflow assistant: break down work, execute with permissions, record, and review
```

### 1.3 Evidence for Job Search, Career Transition, and Self-Built Tools

This course cannot promise that you will get an offer, and it does not reduce "AI application engineer" to a few buzzwords. It does help you build evidence for questions real teams often ask:

```text
Can you turn an AI tool into a controlled engineering system?
Can you explain Agent Run, tool registry, permission gates, verification, and replay?
Can you explain the difference between prompts and runtime policy?
Can you make AI output reviewable, recoverable, and measurable?
Can you extend a minimum system toward real models, RAG, MCP, memory, or a workbench?
```

If you are aiming for AI application engineering, agent engineering, AI productivity tooling, or internal platform work, this project can become part of your explanation material. If you are not changing jobs, you can still adapt it into your own development assistant, a team tool, or a vertical agent workbench.

## 2. Learning Path

```text
00 goals and orientation
01-10 build the Mini Codex foundation loop from scratch
11 compare the boundary with real AI coding tools
12-13 review the result with demo cases and an eval ledger
```

Files:

1. [Lab 00: Create the practice workspace](./01-lab-00-environment-and-directory.en.md)
2. [Lab 01: Save the first task](./02-lab-01-cli-first-run.md)
3. [Lab 02: Pause a task for approval](./03-lab-02-state-machine.md)
4. [Lab 03: Restrict what AI can do](./04-lab-03-tool-registry-permission.md)
5. [Lab 04: Complete the first real code change](./05-lab-04-coding-loop.md)
6. [Lab 05: Save, replay, and review the process](./06-lab-05-trace-replay-eval.md)
7. [Lab 06: Expose CLI capability to the UI](./07-lab-06-backend-api-sse.md)
8. [Lab 07: Build a desktop workbench](./08-lab-07-desktop-workbench.md)
9. [Lab 07b: UI polish and design system](./09-lab-07b-ui-polish-design-system.md)
10. [Lab 08: Compare capability boundaries with real AI coding tools](./10-lab-08-codex-feature-parity.md)
11. [Capability upgrade pack](./11-real-codex-upgrade-pack.md)
12. [Demo cases](./12-demo-cases.md)
13. [Eval ledger](./13-eval-ledger.md)

## 3. How To Use Each Lab

Each lab follows this rhythm:

```text
see what the lab will produce
-> create files and write code
-> run commands and inspect results
-> check the acceptance list
-> review the design skill trained by the lab
```

Keep a `design-journal.md` and answer three questions:

```text
What engineering problem did this lab solve?
Where would the system fail without this design?
If I built an advanced version, what would I improve first?
```

## 4. What Not To Rush

```text
do not add external tools, long-term memory, or automation before the foundation loop works
do not copy every feature from mature tools into the foundation course
do not treat one successful demo as real mastery
do not skip the acceptance checks
```

The foundation course is about making the smallest loop reliable. Advanced capabilities such as richer diffs, rollback, project retrieval, long-term memory, external tools, Git workflows, and quality dashboards can come after that.

## 5. Next Step

Start here:

[Lab 00: Environment and practice workspace](./01-lab-00-environment-and-directory.en.md)
