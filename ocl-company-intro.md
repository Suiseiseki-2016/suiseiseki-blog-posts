---
title: ocl-company：用 AI 智能体构建一支虚拟公司团队
slug: ocl-company-intro
summary: 本文介绍 ocl-company —— 一个用 Rust 编写的命令行工具，它将 OpenClaw 智能体封装为一支具备角色分工的虚拟公司，让你可以用一行命令将复杂任务分配给合适的 AI 员工并获得经过董事会评审的最终结果。
category: Project, AI, Agent
published_at: 2026-03-11
---

# ocl-company：用 AI 智能体构建一支虚拟公司团队

如果你曾经思考过「如何让多个 AI 智能体协作完成一项任务」，ocl-company 给出了一个有趣的答案：把它们组织成一家公司。

## 项目是什么

[ocl-company](https://github.com/openclaw/ocl-company) 是一个单二进制 Rust CLI 工具。它在本地维护一支「虚拟员工」团队，每位员工拥有独立的身份（soul）、记忆（memory）与技能（skills），底层由 [OpenClaw](https://openclaw.dev) 智能体运行时驱动。

当你把一项任务交给公司时，整个流程是这样的：

```
你 → Leader（选人）→ 员工（执行）→ 董事会（评审）→ Leader（综合）→ 最终结果
```

这条七步流水线不只是简单的「调用一次 LLM」，而是一套具备分工、评审与记忆沉淀的协作机制。

## 团队结构

公司默认由 `team.yaml` 定义，包含三类角色：

| 角色 | 职责 |
|------|------|
| **Leader** | 理解任务、选定执行员工、综合最终结果 |
| **Employee** | 按自身技能专注执行具体子任务 |
| **Board** | 独立评审结果，提出挑战、改进建议与新技能 |

每位员工在 `~/.local/share/com/openclaw/agents/{id}/` 目录下拥有三个文件：

- `soul.md` —— 身份与性格设定
- `memory.md` —— 任务完成后自动追加的经验记忆
- `skills/` —— 技能文件夹，董事会建议的新技能会持续写入

随着任务的积累，员工的能力会真正「成长」。

## 快速上手

### 安装

```bash
git clone https://github.com/openclaw/ocl-company
cd ocl-company
cargo build --release
```

确保已安装并配置好 `openclaw`（或 `ocl`）命令行工具。

### 初始化项目

```bash
./ocl-company init
```

这会在当前目录生成 `team.yaml`，并在本地 agents 目录为每位员工初始化 soul / memory / skills 文件。

### 分配任务

```bash
./ocl-company run --task "调研当前主流的 Rust 异步运行时，输出对比报告"
```

终端会实时打印每一步的进展：Leader 选人 → 员工执行 → 董事会评审 → 最终综合结果。任务完成后，员工的 `memory.md` 和 `skills/` 会自动更新。

### 也可以从文件读取任务

```bash
./ocl-company run --task-file task.txt
```

### 查看团队状态

```bash
./ocl-company status
```

输出当前所有员工的姓名、角色以及记忆文件路径，方便随时了解团队现状。

### 动态调整团队

```bash
./ocl-company hire "Elena" --description "前端专家，擅长 React 与性能优化"
./ocl-company fire "Elena"
```

`hire` 命令会为新员工生成完整的 soul / memory / skills 初始文件；`fire` 则将其从 team.yaml 中移除。

## 任务流水线详解

完整的七步流程如下：

1. **Leader 选人** —— Leader 根据任务描述与员工 soul，用 LLM 推理出最合适的执行者
2. **员工执行** —— 以 soul + skills + 任务消息 拼接为 prompt，调用 `openclaw agent --agent main` 执行
3. **董事会评审** —— 每位 Board 成员独立输出 `CHALLENGES`、`IMPROVEMENTS`、`NEW_SKILL` 三类反馈
4. **Leader 综合** —— 汇总员工结果与董事会意见，产出最终交付物
5. **写入记忆** —— 本次任务摘要追加到员工 `memory.md`
6. **写入技能** —— 董事会建议的 `NEW_SKILL` 写入员工 `skills/` 目录
7. **返回结果** —— 最终结果输出到终端或写入 result 文件

这个设计保证了：单次任务有质量把关（董事会），长期使用有能力积累（memory + skills）。

## 与 OpenClaw 的关系

ocl-company 是 OpenClaw 生态的上层应用。它不实现自己的 LLM 调用逻辑，而是将每个子任务委托给 `openclaw agent --agent main` 执行。这意味着：

- 你可以自由切换 OpenClaw 背后使用的模型
- 所有 OpenClaw 支持的 MCP 工具、技能系统对员工同样生效
- 多智能体协作的复杂度由 ocl-company 管理，单个智能体的能力由 OpenClaw 保证

## 小结

ocl-company 把「多智能体协作」这个抽象概念落地成了一套直觉上非常易懂的隐喻：公司、员工、董事会、晋升成长。你不需要手动编排 prompt 链，只需要维护好 `team.yaml` 和员工的 soul 文件，剩下的交给流水线。

如果你在探索如何让 AI 智能体在真实项目中持续发挥作用，不妨从 `ocl-company init` 开始试试。
