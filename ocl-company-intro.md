---
title: ocl-company：把一个 GitHub 仓库变成一支 AI 公司
slug: ocl-company-intro
summary: ocl-company 是一个 Rust 编写的命令行框架，它把 OpenClaw 智能体组织成一家真正意义上的公司——有 Leader、有专职员工、有独立董事会，还有一套让员工越用越好的经验沉淀机制。clone 这个 repo，你就拥有了这支团队。
category: Project, AI, Agent
published_at: 2026-03-11
---

# ocl-company：把一个 GitHub 仓库变成一支 AI 公司

最近我在思考一个问题：如果 AI 智能体真的能承担复杂工作，那「分享经验」这件事应该怎么做？

不是分享 prompt，而是分享一位经过训练的「员工」——它有自己的价值观，有积累了多次任务后的记忆，有被验证过的工作方法论。

ocl-company 是我给这个问题的回答。它的核心理念很简单：**这个 repo 就是这家公司。** clone 它，你就拥有了这支团队；commit 它，你的员工经验随代码一起传递给任何人。

## 这家公司长什么样

公司由三类角色构成：

**Leader**：唯一对外窗口。理解任务，决定由谁来做（或者直接招一个合适的人），最后把员工结果与董事会意见综合成最终交付物。

**Employee**：专职执行者。每人有明确的认知模式和方法论——研究员系统地收集信息，开发者写能跑起来的代码，写作员有真正的观点而不只是格式化输出。

**Board（董事会）**：独立监督者。每次任务完成后，各位董事独立评审结果，不是为了赞美，而是为了挑战：哪里有问题、怎么改、这个过程里有没有值得沉淀的方法论。

任务流水线：

```
你 → Leader（路由 or 招人）→ 员工（执行）→ 董事会（独立评审）→ Leader（综合）→ 最终结果
```

## 每位员工的文件结构

每个员工在 `agents/{id}/` 目录下拥有五个东西：

```
agents/researcher/
├── soul.md        ← 价值观与性格——他们是谁
├── agents.md      ← 岗位 JD——他们做什么、不做什么、怎么做
├── memory/        ← 每次任务后自动写入的经验笔记
├── playbooks/     ← 被验证过的、可复用的工作流程
└── skills/        ← OpenClaw 原生技能（由 openclaw 在运行时调用）
```

这里有一个关键设计决定：**skills 不会被注入到 prompt 里。** 把技能当文本塞进消息是一种诱人的捷径，但那样做的员工是在阅读一份 SOP 手册，而不是真正掌握一种能力。Skills 由 OpenClaw 在运行时原生加载和调用——员工不读技能描述，他们使用技能工具。

真正被注入为上下文的，是 `agents.md`（角色定义）、`memory/`（过往经验）和 `playbooks/`（验证过的流程）。这三样东西是员工「知道自己是谁、做过什么、怎么做事」的来源。

## 经验沉淀机制

这是我认为最重要的设计。

每次董事会评审结束，董事可以发出三种升级指令：

```
MEMORY: <一句话，值得员工记住的观察>
PLAYBOOK: <标题> | <可复用的步骤流程>
NEW_SKILL: <需要自动化的能力>
```

对应三个层级：

| 层级 | 位置 | 触发条件 |
|---|---|---|
| 记忆 | `memory/` | 有值得记录的观察 |
| 剧本 | `playbooks/` | 有可复用的工作流程 |
| 技能 | `skills/` | 有需要自动化的能力 |

idea.md 里有一句话我很喜欢：「不要急于升级。一份有效的记忆笔记胜过一份过早制定的剧本，一份有效的剧本胜过一个有 bug 的技能。」

这不是官僚主义，是对经验成熟度的尊重。员工不是从第一天起就拥有完整的 playbook 库——他们是在一次次任务中，把「我上次怎么做的」变成「我们团队的标准做法」，再变成「我掌握的一种工具」。

## Leader 可以自主招人

不需要你手动决定团队规模。

当 Leader 判断现有团队里没有人适合这个任务时，它会返回：

```
HIRE: 数据分析师 | 统计分析、数据可视化、量化推理
```

流水线捕获这个指令，自动创建员工、初始化文件、更新 `team.yaml`，然后把任务路由给这位新成员。当然，你也可以手动干预：

```bash
ocl-company hire "数据分析师" --description "统计分析、数据可视化、量化推理"
ocl-company fire "数据分析师"
```

## 快速上手

### 方式一：让你的 OpenClaw 一键安装

如果你已经在用 OpenClaw，最简单的方式是直接告诉它：

```
fetch https://github.com/Suiseiseki-2016/ocl-company and follow COMPANY.md
```

OpenClaw 会自己 clone 仓库、阅读 `COMPANY.md` 里的说明、完成 build 和初始化。全程不需要你手动敲一行命令。这也是这套框架设计的初衷——当你的 AI 智能体本身就是用户时，安装流程也应该是智能体可以自主完成的。

### 方式二：手动安装

```bash
git clone https://github.com/Suiseiseki-2016/ocl-company
cd ocl-company
cargo build --release
cp target/release/ocl-company ~/.local/bin/

# 在你的项目目录里初始化团队
ocl-company init

# 分配一个任务
ocl-company run --task "调研当前主流的 Rust 异步运行时，输出对比报告"

# 查看任务状态
ocl-company status
```

`init` 会生成 `team.yaml` 和完整的 `agents/` 目录，每位默认员工（Leader、两位董事、Researcher、Developer、Writer）都已经配好了完整的 soul 和岗位定义，可以直接开始工作，也可以随时编辑。

## 共享一支训练好的团队

`agents/` 目录是版本控制的。这意味着：

- 员工积累的 memory 随 git 历史保留
- 被验证过的 playbooks 可以 commit 和 push
- fork 这个 repo，你 fork 的不只是代码，还有这支团队的经验

当你对某位员工的表现满意时，push 一次就完成了分享。对方 clone 下来，运行 `ocl-company init`，就拥有了你训练过的团队。

## 与 OpenClaw 的关系

ocl-company 是 OpenClaw 生态的编排层。它不实现自己的 LLM 调用，每个子任务都委托给 `openclaw agent --agent main` 执行。OpenClaw 负责单个智能体的能力边界，ocl-company 负责多个智能体的协作结构。

这个分层很重要：你切换模型、配置工具、调整 OpenClaw 的技能系统，这些变化对所有员工自动生效——ocl-company 不需要知道这些细节。

## 一句话总结

ocl-company 把「如何让多个 AI 智能体协作」这个问题，变成了一个你已经很熟悉的问题：如何管理一支团队。

如果你想试试，从 `ocl-company init` 开始。
