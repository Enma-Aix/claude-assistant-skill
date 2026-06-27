# /assistant

> A triage-first Claude Code skill that turns one-line engineering goals into the right execution shape: one agent, a reviewed pair, parallel agents, or a real leader-led agent team.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Claude Code](https://img.shields.io/badge/Claude%20Code-skill-blue)](https://docs.claude.com/en/docs/claude-code)

`/assistant` is an orchestration skill for Claude Code.

You describe the goal. `/assistant` triages the task, writes the working prompts, routes the right agents or team, supervises evidence, and reports back.

```text
user says the goal
  -> /assistant writes the task briefs
  -> agents / team execute
  -> /assistant checks evidence and reports back
```

---

## Requirements

The default setup expects:

1. **Claude Code with skills support**
2. **ECC agents installed and available as `ecc:*` agents**
3. **superpowers-style methodologies installed**, such as planning, TDD, debugging, review, and verification workflows

The default `roster.md` routes work to agents such as:

```text
ecc:planner
ecc:tdd-guide
ecc:code-reviewer
ecc:architect
ecc:build-error-resolver
ecc:e2e-runner
ecc:security-reviewer
ecc:doc-updater
```

If ECC is not installed, the default routing will not work as intended.

You can still use the orchestration model with another agent pool, but you must update `roster.md` to point to your actual agents and team entrypoints.

---

## Install

Clone or download this repository, then copy the skill files into Claude Code’s skill directory:

```bash
mkdir -p ~/.claude/skills/assistant
cp SKILL.md roster.md supervision.md ~/.claude/skills/assistant/
```

Start a new Claude Code session so the skill can be loaded.

Then check that Claude Code can see the skill and your ECC agents.

---

## Quick start

### Simple task

```text
/assistant explain this error
```

Expected shape:

```text
assistant -> one suitable agent -> assistant -> user
```

### Small verified fix

```text
/assistant fix this bug and add a regression test
```

Expected shape:

```text
assistant -> writer + reviewer/tester -> assistant -> user
```

### Complex feature

```text
/assistant implement password reset, including API, DB, frontend, tests, and PR summary
```

Expected shape:

```text
assistant -> team leader -> planner / implementer / reviewer / tester -> assistant -> user
```

### Explicit project team

```text
/assistant use backend-team to implement the billing module
```

Expected shape:

```text
assistant -> backend-team leader -> backend-team members -> assistant -> user
```

### Full handoff

```text
/assistant 全程决策移交 — implement the notification module and prepare the PR summary
```

Full handoff reduces mid-flight interruptions. It does not remove safeguards for irreversible actions such as push, merge, deploy, publish, or permanent deletion.

---

## Register your own agents or teams

`/assistant` works best when your local agents and teams are registered in `roster.md`.

The recommended way is to let `/assistant` register them for you, because route registration is a governance change: `/assistant` should verify that the target entrypoint exists, update `roster.md`, and explicitly report what changed.

### Register a custom agent

```text
/assistant 注册一个自定义 agent 路由：
用户叫法：qa-reviewer / QA评审
实际入口：custom:qa-reviewer
适用范围：测试质量、验收用例、回归风险评审
路由说明：用户点名 QA评审 时使用；找不到则停报，不静默回落
```

Expected behavior:

1. `/assistant` checks whether `custom:qa-reviewer` really exists in the current Claude Code harness.
2. If it exists, it adds an entry to the custom agent registry in `roster.md`.
3. If it does not exist, it stops with `requested_team_or_agent_not_found` instead of writing a fake route.
4. It reports the governance change explicitly: what route was added, why, whether the entrypoint was verified, and which file changed.

The resulting `roster.md` entry should look like:

```markdown
| `qa-reviewer` / `QA评审` | `custom:qa-reviewer` | 测试质量、验收用例、回归风险评审 | 用户点名该角色时使用；不存在则停报，不静默回落。 |
```

### Register a custom team

```text
/assistant 注册一个自定义 team：
team 名：backend-team
leader / 入口：backend-team-leader
适用范围：后端新功能、API、DB、服务端重构
路由说明：用户说“用 backend-team / 交给 backend-team”时使用；assistant 只对接 leader
```

Expected behavior:

1. `/assistant` verifies that `backend-team-leader` is a real team entrypoint or routable leader.
2. If it exists, it adds the route to `roster.md`.
3. If it does not exist, it stops with `requested_team_or_agent_not_found`.
4. It reports the governance change explicitly.

The resulting `roster.md` entry should look like:

```markdown
| `backend-team` | `backend-team-leader` | 后端新功能、API、DB、服务端重构 | 用户说“用/交给/调用 backend-team”时使用；assistant 只对接 leader。 |
```

After registration:

```text
/assistant 用 backend-team 实现 billing 模块
```

should route to `backend-team-leader`, not the default ECC roster.

---

## What `/assistant` changes for the user

Without `/assistant`, users often need to write long prompts for every specialist:

```text
You are a backend engineer...
Use TDD...
Do not touch unrelated files...
Write tests...
Ask if ambiguous...
Return diff and test output...
Then ask a reviewer...
```

With `/assistant`, the user writes the goal:

```text
/assistant implement password reset, including API, DB, frontend, tests, and PR summary
```

`/assistant` writes the specialist task briefs:

- role and responsibility
- task scope and boundaries
- allowed tools and forbidden actions
- acceptance criteria
- testing and review expectations
- evidence requirements
- stop conditions for ambiguity, blockers, or irreversible actions

The user writes the goal. `/assistant` writes the working prompts.

---

## Core principle: triage first

`/assistant` does not treat every task as a team task.

| Task type | Dispatch shape | What happens |
|---|---|---|
| Simple one-off task | Solo agent | One focused agent handles the task. |
| Small code change needing verification | Writer + reviewer/tester | The assistant may lightly coordinate a writer and a reviewer/tester. |
| Independent batch/ops work | Flat parallel agents | The assistant dispatches independent agents in parallel and summarizes results. |
| Complex feature / module / migration | Leader-led team | A team leader plans, coordinates members, collects evidence, and reports back. |
| User explicitly names a team | Explicit team route | The assistant routes to that team’s leader/entrypoint instead of the default roster. |

The rule is:

> **Simple work must stay simple. Complex goal-oriented work must not degrade into one solo agent or assistant-managed flat fan-out.**

---

## Dispatch shapes

### 1. Solo agent

Used for bounded one-off work:

```text
/assistant inspect this config and tell me why it fails
/assistant check where this function is defined
```

Shape:

```text
user -> assistant -> one suitable agent -> assistant -> user
```

---

### 2. Writer + reviewer/tester

Used for small code changes where self-certification would be unsafe, but a full team would be overkill:

```text
/assistant fix this login parameter validation bug and add a regression test
```

Shape:

```text
user
  -> assistant
     -> writer
     -> reviewer/tester
  -> assistant
-> user
```

This is intentionally not a full team. The assistant may lightly coordinate the writer and reviewer, but it should not micromanage their work. It collects verdicts and raw evidence.

---

### 3. Flat parallel agents

Used when the subtasks are independent:

```text
/assistant check these 8 nodes and report which ones have broken Docker networking
/assistant inspect these 12 files and summarize duplicate patterns
```

Shape:

```text
user
  -> assistant
     -> agent for item 1
     -> agent for item 2
     -> agent for item 3
     -> ...
  -> assistant summarizes
-> user
```

Flat fan-out is correct for independent work. It is **not** a replacement for a real team when the work needs shared planning, sequencing, review, and final ownership.

---

### 4. Leader-led team

Used for real engineering goals:

- new feature
- new module
- multi-layer implementation
- migration
- integration
- PR-level delivery
- long-running goal
- work needing planning + implementation + testing + review under one owner

Example:

```text
/assistant implement password reset, including API, DB changes, frontend flow, tests, and PR summary
```

Correct shape:

```text
user
  -> assistant
     -> team leader / PM
        -> planner
        -> implementer
        -> reviewer
        -> tester
        -> docs agent
     -> team leader reports evidence
  -> assistant reports to user
```

In this mode, the assistant talks to the **team leader**, not every worker. The leader owns planning, member assignment, coordination, evidence collection, and final reporting.

Incorrect shape:

```text
assistant -> planner
assistant -> implementer
assistant -> reviewer
assistant -> tester
assistant summarizes everything itself
```

That is flat fan-out, not a team.

If native agent teams are available, `/assistant` should use them. If native teams are not available or are awkward in the current harness, `/assistant` may use a **managed-team** fallback: it dispatches a team leader/PM, and that leader coordinates worker agents. The assistant still talks only to the leader.

The leader’s first response should be lightweight, not bureaucratic:

```text
team: <team name or managed-team>
leader: <leader name / entrypoint>
members:
  - role: planner, actual_agent_type: ...
  - role: implementer, actual_agent_type: ...
  - role: reviewer, actual_agent_type: ...
assistant_contact_surface: leader_only
next_plan: <short plan>
```

This is an audit clue, not a heavy manifest protocol.

---

### 5. Explicit user team

Users can name their own project team:

```text
/assistant use backend-team to implement the billing module
/assistant 交给 深度研究 team，分析这个项目架构
```

When a team is explicitly named, user intent wins:

```text
user custom team / agent > project team skill > default ECC roster
```

The assistant should resolve the team from `roster.md`, route to its leader or entrypoint, and stop if the team cannot be found. It must not silently fall back to the default roster or pretend that a prompt mention is real routing.

---

## Routing source of truth

`roster.md` is the routing source of truth.

It contains:

- default role -> agent mapping
- user custom team registry
- user custom agent / role override registry
- explicit routing priority
- fail-closed behavior for missing named routes
- real `subagent_type` routing rules
- native team / managed-team boundaries
- the difference between flat fan-out and real teams
- leader-led team rules

Manual editing is supported, but using `/assistant` is recommended because it can check whether the referenced entrypoint actually exists before writing the route.

---

## Evidence and supervision

`/assistant` does not treat “done” as enough.

For coding work, completion should be backed by:

- changed files
- diff summary or raw diff
- test/build command and raw output
- reviewer verdict
- unresolved risks
- external integration evidence when mocks are not enough

For leader-led team work, the assistant checks that the contact surface is really leader-only and that the reported workers are real routed agents, not decorative names.

---

## Repository structure

| File | Purpose |
|---|---|
| `SKILL.md` | Defines assistant identity, boundaries, autonomy levels, task triage, and decision flow. |
| `roster.md` | Defines default role routing, custom team / agent registration, explicit routing priority, leader-led team rules, native team / managed-team boundaries, fail-closed routing, and the boundary between flat fan-out and real teams. |
| `supervision.md` | Defines team audits, heartbeat monitoring, evidence gates, blocker handling, and failure-mode defenses. |
| `LICENSE` | MIT license. |

---

## What `/assistant` should not do

- It should not write product code in the main session.
- It should not write tests or run build/test commands in the main session.
- It should not treat every request as a team task.
- It should not downgrade complex feature work into one solo agent.
- It should not call flat fan-out a team.
- It should not ignore a user-specified team.
- It should not route by prompt mention when a real `subagent_type` or team entrypoint is required.
- It should not register custom agents or teams that do not actually exist.
- It should not silently edit `roster.md`; route registration is a governance change and must be reported.
- It should not report completion without raw evidence.

---

## FAQ

### Do I need ECC?

For the default setup, yes.

The provided `roster.md` is written for ECC-style agents such as `ecc:tdd-guide`, `ecc:code-reviewer`, `ecc:planner`, and `ecc:architect`.

Without ECC, the default routes will fail or need replacement. You can still use the orchestration model with your own agents by editing `roster.md`.

### Does every request create an agent team?

No. `/assistant` is triage-first. Simple tasks stay lightweight. Only complex feature/module/migration work or explicitly named teams require leader-led team mode.

### What is the difference between flat parallel agents and a team?

Flat parallel agents are independent workers dispatched directly by the assistant. This is useful for batch checks.

A team has a leader. The leader plans, assigns work, coordinates dependencies, collects evidence, and reports back. Complex engineering work needs this ownership layer.

### Can I use my own agents?

Yes. Register them in `roster.md`, preferably by asking `/assistant` to do it so it can verify that the actual entrypoint exists.

### What happens if I name a team or agent that is not registered?

The assistant should stop with `requested_team_or_agent_not_found`. It must not silently fall back to the default roster.

---

# 中文说明

`/assistant` 是一个 Claude Code skill，用来把一句话需求分诊成合适的执行形态。

它不是“所有任务都拉 agent team”，也不是“一个 agent 从头干到尾”。它的定位是让 Claude Code 主会话变成一个**轻量编排者**：

```text
用户只描述目标
  -> /assistant 先判断任务类型
  -> 选择合适的派活形态
  -> 把实现、测试、评审、调查交给对应 agent / team
  -> assistant 审查证据并汇报结果
```

---

## 使用前提

默认配置需要：

1. 支持 skills 的 Claude Code
2. 已安装 ECC agents
3. 已安装 superpowers 风格的方法论能力，例如 planning、TDD、debugging、review、verification

默认 `roster.md` 会路由到这些 agent：

```text
ecc:planner
ecc:tdd-guide
ecc:code-reviewer
ecc:architect
ecc:build-error-resolver
ecc:e2e-runner
ecc:security-reviewer
ecc:doc-updater
```

如果你没有安装 ECC，默认路由就不能直接用。你仍然可以使用 `/assistant` 的编排模型，但需要先把 `roster.md` 里的 `ecc:*` 映射改成你自己的 agents。

---

## 安装

```bash
mkdir -p ~/.claude/skills/assistant
cp SKILL.md roster.md supervision.md ~/.claude/skills/assistant/
```

然后重新打开一个 Claude Code 会话，让 skill 重新加载。

---

## 快速使用

简单任务：

```text
/assistant 看看这个 Kubernetes 报错是什么意思
```

小修小补：

```text
/assistant 修复登录接口参数校验 bug，并补测试
```

批量检查：

```text
/assistant 检查这 8 台节点的 Docker 网络状态
```

复杂新功能：

```text
/assistant 实现 password reset 模块，包括 API、DB、前端、测试和 PR 说明
```

指定 team：

```text
/assistant 用 backend-team 实现 billing 模块
```

注册自定义 agent：

```text
/assistant 注册一个自定义 agent 路由：
用户叫法：安全评审
实际入口：custom:security-reviewer
适用范围：鉴权、权限、敏感信息、依赖风险
```

注册自定义 team：

```text
/assistant 注册一个自定义 team：
team 名：infra-team
leader / 入口：infra-team-leader
适用范围：部署、Kubernetes、CI/CD、基础设施改造
```

全程移交：

```text
/assistant 全程决策移交 — 实现通知模块，并准备 PR 说明
```

全程移交只减少中途打断，不取消红线。涉及 push、merge、deploy、发布、永久删除数据等不可逆动作时，仍然需要停下请示。

---

## 用户不用再写复杂提示词

使用 `/assistant` 后，用户不需要再为每个 agent 手写长 prompt。

用户只需要说清楚目标：

```text
/assistant 实现 password reset 模块，包括 API、DB、前端、测试和 PR 说明
```

剩下的由 `/assistant` 负责：

- 判断任务类型
- 选择单 agent、writer + reviewer/tester、flat fan-out、native team 或 managed-team
- 给 agent 或 team leader 写专业任务书
- 写清楚角色、边界、可用工具、禁止事项
- 写清楚验收标准、测试要求、review 要点
- 处理不确定、受阻、不可逆动作等停下条件
- 收集 diff、测试输出、review 结论等证据
- 最后用人话汇报结果

也就是说，使用方式从：

```text
用户自己写一堆复杂 agent prompt
```

变成：

```text
用户说目标
  -> assistant 写任务书
  -> agent / team 执行
  -> assistant 验收证据并汇报
```

这也是 `/assistant` 的核心价值之一：把多 agent 协作里的提示词工程成本，从用户身上转移到 assistant 身上。

---

## 中文核心逻辑：先分诊，再派活

```text
简单一次性任务
  -> 单 agent

小修小补，但需要测试 / review
  -> writer + reviewer/tester

N 个互不依赖对象
  -> 并行 agents

新功能 / 新模块 / 长任务 / 多模块开发 / 迁移集成
  -> leader-led agent team

用户显式指定 team
  -> 指定 team 的 leader / 入口
```

一句话原则：

> **简单任务不要上重流程；复杂目标任务不能退化成单 agent。**

---

## team 模式的正确形态

复杂任务包括：

- 新功能
- 新模块
- 多模块改造
- 前后端联动
- DB + API + UI 一起改
- 迁移
- 集成
- PR 级交付
- 长时间任务
- 需要计划、实现、测试、评审、文档一起收口的任务

示例：

```text
/assistant 实现 password reset 模块，包括 API、DB、前端流程、测试和 PR 说明
```

正确形态：

```text
用户
  -> assistant
     -> team leader / PM
        -> planner
        -> implementer
        -> reviewer
        -> tester
        -> docs agent
     -> leader 汇总证据
  -> assistant 汇报用户
```

错误形态：

```text
assistant -> planner
assistant -> coder
assistant -> reviewer
assistant -> tester
assistant 自己汇总
```

这只是扁平派发，不是真正的 team。

---

## native team 和 managed-team

如果 Claude Code 环境支持原生 agent team，就优先走 native team：

```text
assistant -> native agent team leader -> team members
```

如果当前 harness 不方便直接创建 native team，也可以退到 managed-team：

```text
assistant -> managed-team leader / PM -> worker agents
```

managed-team 的重点是仍然保持 leader-led：

- assistant 只对接 leader
- leader 负责协调 worker
- worker 不直接由 assistant 扁平管理
- leader 统一收口证据

---

## 用户指定 team

用户可以直接点名自己的 team：

```text
/assistant 用 backend-team 实现 billing 模块
```

或者：

```text
/assistant 交给 深度研究 team，分析这个项目架构
```

优先级是：

```text
用户自定义 team / agent > 项目 team skill > 默认 ECC roster
```

如果找不到指定 team，assistant 应该停报：

```text
requested_team_or_agent_not_found
```

不能静默回落到普通 agent，也不能把 team 名写进 prompt 假装已经路由。

---

## 推荐：直接让 `/assistant` 注册你的 agents 和 team

用户不需要手动改 `roster.md`。推荐直接让 `/assistant` 自己添加路由。

因为改 `roster.md` 是治理修正，不是产品代码。assistant 可以做，但必须先验证入口真实存在，并且必须明改明报，不能偷偷改。

---

### 注册自定义 agent

```text
/assistant 注册一个自定义 agent 路由：
用户叫法：qa-reviewer / QA评审
实际入口：custom:qa-reviewer
适用范围：测试质量、验收用例、回归风险评审
路由说明：用户点名 QA评审 时使用；找不到则停报，不静默回落
```

assistant 应该做：

1. 检查 `custom:qa-reviewer` 是否是当前环境里真实存在的 agent / `subagent_type`
2. 存在就写入 `roster.md` 的「用户自定义 agent / 角色覆盖」表
3. 不存在就停报，不写假路由
4. 汇报修改了哪个文件、新增了什么路由、为什么新增、是否验证了真实入口

写入后的表项类似：

```markdown
| `qa-reviewer` / `QA评审` | `custom:qa-reviewer` | 测试质量、验收用例、回归风险评审 | 用户点名该角色时使用；不存在则停报，不静默回落。 |
```

以后就可以这样用：

```text
/assistant 让 QA评审 检查这次改动的回归风险
```

---

### 注册自定义 team

```text
/assistant 注册一个自定义 team：
team 名：backend-team
leader / 入口：backend-team-leader
适用范围：后端新功能、API、DB、服务端重构
路由说明：用户说“用 backend-team / 交给 backend-team”时使用；assistant 只对接 leader
```

assistant 应该做：

1. 检查 `backend-team-leader` 是否真实存在
2. 存在就写入 `roster.md` 的「用户自定义 team」表
3. 不存在就停报
4. 后续用户点名 `backend-team` 时，优先路由到该 team
5. 汇报这次治理修正

写入后的表项类似：

```markdown
| `backend-team` | `backend-team-leader` | 后端新功能、API、DB、服务端重构 | 用户说“用/交给/调用 backend-team”时使用；assistant 只对接 leader。 |
```

以后就可以这样用：

```text
/assistant 用 backend-team 实现订单结算模块
```

---

## 为什么推荐让 assistant 自己注册

手动改 `roster.md` 也可以，但推荐让 `/assistant` 加，原因是：

1. 它可以先自查入口是否存在，避免写入假路由。
2. 它能保持表格格式一致，减少 Markdown 表格写坏。
3. 它能遵守 fail-closed：找不到就停报，不会伪装成已路由。
4. 它能把“用户说法”和“真实入口”分开。
5. 它符合 assistant 的定位：用户只下发目标，路由维护交给 assistant。

---

## `/assistant` 不应该做什么

- 不应该在主会话里亲自写产品代码
- 不应该在主会话里亲自写测试
- 不应该在主会话里亲自跑 build/test
- 不应该把所有任务都强制拉 team
- 不应该把复杂新功能降级成一个 solo agent
- 不应该把 flat fan-out 冒充成 team
- 不应该忽略用户显式指定的 team
- 不应该把 agent/team 名字只写进 prompt 假装路由
- 不应该注册不存在的 agent/team
- 不应该偷偷修改 `roster.md`
- 不应该没有证据就汇报完成

---

## License

MIT.
