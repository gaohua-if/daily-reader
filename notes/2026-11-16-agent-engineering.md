# Agent 工程阅读笔记｜从按需加载到持续运行

整理于 2026-11-16。8 个链接去重后共 7 个来源，包含 3 个 GitHub 仓库、2 篇英文文章、1 篇中文解读和 1 份官方文档。Anthropic 的常驻 Agent 指南原清单中出现了两次。

## 摘要

Anthropic 的常驻 Agent 指南把系统拆成事件、定时器、记忆、权限和消息投递。Letta 具体展示了怎样用 Git 管理跨会话记忆。`skills-router` 让 Agent 按需发现工作说明，Code Mode 让 Agent 用一段 JavaScript 串联工具并筛选返回结果，两者分别减少技能加载和工具执行时的上下文开销。AgentRun 展示了运行恢复和运行后复用经验的工程实现，Future Pruning 提议在定时器到期时结合当前状态重新判断。多 Agent 研究发现，当多个用户的 Agent 争用同一资源时，团队得分可能掩盖未经授权的操作。

## 推荐阅读顺序

1. **首读《Building always-on agents》**。事件如何唤醒 Agent、运行状态如何保存、权限如何等待人来确认，这篇文章把整个循环讲得最完整。重点读事件收件箱和延后工具调用两节。如果只读一篇，选它。[Anthropic 指南](https://claude.com/resources/articles/building-always-on-agents)
2. **接着读《Introducing Context Repositories》**。最容易借鉴的是它对记忆加载范围的划分。看哪些内容放进 `system/`，哪些只保留路径和描述，以及编辑记忆的子 Agent 如何使用 Git worktree。[Letta 原文](https://www.letta.com/blog/context-repositories/)
3. **设计多人共用资源时，读《Worse Together》**。先看四种模拟环境，再看为什么共享预算适合指定协调者，而团体点餐更需要提交前的系统检查。乔木版适合中文速读，实验条件和数字可对照研究团队原文。[乔木解读](https://blog.qiaomu.ai/worse-together-multi-agent) · [研究团队原文](https://alignment.anthropic.com/2026/worse-together/)

## 逐条笔记

### 1. Building always-on agents

William Hu 为 Anthropic 写的架构指南。Agent 平时休眠，由用户消息、Webhook 或定时器事件唤醒。持久化收件箱记录事件状态并按来源 ID 去重，worker 从中读取待处理事件，延续已有任务时可以恢复旧会话。定时器独立持久化，到期后将事件送入同一个收件箱。运行时读取长期记忆并按权限行动，需要用户确认时可以挂起工具调用，等答复作为新事件进入后恢复。需要控制投递节奏或处理投递失败时，可以加入发件箱。[Anthropic 指南](https://claude.com/resources/articles/building-always-on-agents)

### 2. Context Repositories

Letta Code 将 Agent 的记忆同步到本地文件系统并用 Git 记录改动。目录树始终进入系统提示，文件的 frontmatter 用来描述内容，`system/` 下的文件全文常驻，其余按需读取。多个记忆子 Agent 可以各自在独立 worktree 处理历史对话或整理文件，再把结果合并回主记忆。文章主要展示 Letta 的存储与加载设计。[Letta 原文](https://www.letta.com/blog/context-repositories/)

### 3. 多 agent 团队越帮越忙

乔木解读了论文 *Worse Together*。论文比较了一个协调 Agent 服务所有用户，以及每个用户各自委托 Agent 的方式。在共享算力预算、诊所排期、团体点餐或订票、代码合并队列这四类模拟环境中，后者的团队得分较低。共享预算实验中，明确优先级并赋予负责人分配权有帮助。团体点餐实验中，要求提交者先读完待处理消息，挽回了 Opus 5 和 Sonnet 5 失败回放中的 73% 和 62%。这些比例对应特定场景及失败样本。研究重点是多个用户带着不同目标争用资源，同一目标下的子 Agent 分工需要另行评估。[乔木解读](https://blog.qiaomu.ai/worse-together-multi-agent) · [研究团队原文](https://alignment.anthropic.com/2026/worse-together/) · [arXiv 论文](https://arxiv.org/abs/2610.00583)

### 4. Claude Code 的 Code Mode

Claude 写一段 JavaScript 来并行执行工具、把中间结果传给下一步，并只把需要的结果返回给模型。适合跨文件的重复操作、批量搜索后的过滤，以及可同时运行的检查。官方文档将其标为默认关闭的实验功能。工具调用仍经过权限规则和钩子，Auto Mode 的分类器审查生成的脚本。直接使用 Claude Code 做大量工具调用时，这一篇最值得查阅。[Claude Code 官方文档](https://code.claude.com/docs/en/code-mode)

### 5. Parcha-ai/AgentRun

开源的 Agent 运行与控制平台，支持 Claude Code、Codex 等执行环境。服务重启后可以接续运行，提供工具审批、费用上限和运行时间线。开启学习功能后，Agent 在一次运行结束时提出可复用的笔记、技能或工作流，经过程序检查和模型评审，风险规则需要人工确认，通过的内容进入该 Agent 的 Git 目录，供后续运行使用。想看持久执行如何落地，可从 README 的恢复机制读起。效率改善来自项目方公开的案例。[AgentRun README](https://github.com/Parcha-ai/agentrun)

### 6. ShenQingchuan/skills

面向个人工作流的技能管理框架。默认只暴露 `skills-router`，Agent 通过 `boss search-skills` 搜少量候选，或用 `boss list-skills` 查看分类，然后读取选中的 `SKILL.md` 和相关参考文件。仓库还能用 Profile 管理共享或本机私有的技能来源，并锁定上游同步版本。手头技能多、需要管理发现和更新时，优先看这个仓库的路由与同步部分。[skills README](https://github.com/ShenQingchuan/skills)

### 7. Future Pruning

Joe Wilson 的设计草案将定时点定义为一次复核机会。安排时记下目的、受益者、触发时间和依据。到期后开启新会话，读取当前进展、权限和停止条件，决定执行、修改或放弃，并留下理由。作者报告的单用户样本中，28 个到期决策里有 9 个执行、6 个修改、13 个放弃。这是早期观察，参考实现仍为私有。适合与 Anthropic 的定时器章节一起读，一个解决如何唤醒，另一个细化醒来后怎样复核旧计划。[Future Pruning 草案](https://github.com/joewilsonai/future-pruning)
