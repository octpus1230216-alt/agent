# 🤖 Stage 4 学习手册：多智能体是协调，不是魔法

> 来源：datawhalechina/Agent-Learning-Hub · Stage 4「Multi-Agent Is Coordination, Not Magic」
>
> 阅读时长 ≈ 45 分钟 · 配套可运行 Demo · 5 个课堂练习 · 覆盖 Stage 4 全部 5 条要求

## 📚 本手册目录

1. [你要达成什么（学习目标清单）](#1-你要达成什么学习目标清单)
2. [核心心智模型：把多智能体想成一家公司](#2-核心心智模型把多智能体想成一家公司-)
3. [五类常见角色（你要认识的"工种"）](#3-五类常见角色你要认识的工种)
4. [两种管理模式：supervisor vs graph](#4-两种管理模式supervisor-vs-graph)
5. [定义边界：职责 / 输入输出 schema / 停止条件](#5-定义边界职责--输入输出-schema--停止条件第条)
6. [四大工程难题与应对](#6-四大工程难题与应对第条)
7. [何时单 agent 反而更好](#7-何时单-agent-反而更好第条)
8. [五大学习素材精读](#8-五大学习素材精读把推荐阅读嚼碎了喂给你)
9. [实战项目：research → write → review → revise](#9-实战项目research--write--review--revise)
10. [课堂练习（动手 ≥ 60%）](#10-课堂练习动手--60)
11. [自测清单 + 下一步](#11-自测清单--下一步)

---

## 1. 你要达成什么（学习目标清单）

Stage 4 的原话只有五条要求，但每一条都对应一套可落地的工程能力。先把它变成一张「通关清单」，后面每一节都在帮你打勾。

| # | Stage 4 原话要求 | 翻译成人话 | 对应章节 |
|---|---|---|---|
| ① | 理解 planner / executor / reviewer / critic / router 等常见角色 | 知道一个 AI 团队里通常有哪些"工种"，各自干嘛 | [§3](#3-五类常见角色你要认识的工种) |
| ② | 学会用 supervisor 或 graph 管理多 agent，而不是让 agent 随便聊天 | 多智能体要有"组织者/流程图"，不能群聊 | [§4](#4-两种管理模式supervisor-vs-graph) |
| ③ | 会定义每个 agent 的职责边界、输入输出 schema、停止条件 | 给每个角色写"岗位说明书 + 工作单格式 + 下班标准" | [§5](#5-定义边界职责--输入输出-schema--停止条件第条) |
| ④ | 会处理循环、争论、任务漂移、上下文膨胀 | 能修好多智能体常见的四种"翻车" | [§6](#6-四大工程难题与应对第条) |
| ⑤ | 会判断什么时候单 agent 更好 | 不盲目上多智能体，该单干就单干 | [§7](#7-何时单-agent-反而更好第条) |

> **产出（Project Output）：** 一个小型多智能体系统，例如 `research → write → review → revise`。
> 本手册 [§9](#9-实战项目research--write--review--revise) 会带你从零设计它，并附一份**可直接运行的 Python Demo**（无需 API key，用"假 LLM"模拟，换成真模型骨架不变）。

---

## 2. 核心心智模型：把多智能体想成一家公司 🏢

先建立一个贯穿全篇的类比，后面所有概念都挂在这个框架上：

**🧍 单智能体（Stage 0~3）**
就像**一个人开公司**：老板兼会计兼销售兼客服。什么都会一点，但容易忙乱、上下文塞爆、顾此失彼。适合任务清晰、流程稳定的小活。

**👥 多智能体（Stage 4）**
就像**一个团队开公司**：有产品经理（planner）、工程师（executor）、质检（reviewer）……每人只干自己那一摊，靠"交接单"协作。关键是**协调**，不是每个人多聪明。

> 💡 **Stage 4 的标题为什么叫"不是魔法"？**
>
> 很多人以为"多智能体 = 把几个 LLM 放一起聊天，它们自己就变强了"。这是误区。多智能体的价值来自**明确的分工 + 清晰的接口 + 可控的流程**，本质是**软件工程里的"模块化 + 流水线"**，只不过模块换成了会说话的 AI。没有设计，多智能体只会把错误放大、把成本翻倍。

---

## 3. 五类常见角色（你要认识的"工种"）

这些角色名字你会在几乎所有多智能体框架里反复见到。先记住它们各自**干什么、产出什么**。

### 🧭 Planner（规划者） = 项目经理
把大目标拆成有序的子任务清单；决定先做什么后做什么；常常只产出"计划"，不直接干活。

### ⚙️ Executor（执行者） = 工程师
真正调用工具、写代码、搜资料、生成内容的角色；拿计划去落地，产出草稿/结果。

### 🔍 Reviewer（评审者） = 质检 / 代码审查
检查执行结果对不对、全不全、合不合规；产出"评审意见"（通过 / 不通过 + 修改点）。

### 🧠 Critic（批评者） = 挑刺顾问
比 Reviewer 更"战略"：质疑假设、指出逻辑漏洞、评估方案优劣；不一定给修改，给判断。

### 🔀 Router（路由器） = 前台 / 调度
看清用户意图，把任务分发给最合适的下游 agent；自己不干活，只做"分发 + 汇总"。

### 👔 Supervisor（主管，管理角色）
不属于"工种"而是"管理者"：决定下一步调用谁、何时停止、把谁的结果交给谁（见 [§4](#4-两种管理模式supervisor-vs-graph)）。

> ⚠️ **易混点：Reviewer vs Critic。** Reviewer 对着"产出物"挑毛病（这行代码有 bug、这段少引用）；Critic 对着"思路/方案"挑毛病（这个方向本身不对、前提假设不成立）。实战里经常合并成一个"评审+批评"角色，但概念要分清。

| 角色 | 典型输入 | 典型输出 | 何时需要 |
|---|---|---|---|
| Planner | 用户目标 | 任务清单 / 步骤图 | 任务复杂、需多步 |
| Executor | 单个子任务 | 草稿 / 结果 / 工具返回值 | 几乎总是需要 |
| Reviewer | 执行结果 | 通过/不通过 + 意见 | 对质量有要求时 |
| Critic | 方案/中间结论 | 风险评估 / 改进建议 | 高风险、需论证时 |
| Router | 用户请求 | 路由决策 | 入口有多种任务类型 |

---

## 4. 两种管理模式：supervisor vs graph

### 4.1 为什么"让 agent 随便聊天"是错的

最朴素的想法：把 3 个 LLM 放进一个群聊，说"你们自己商量把活干完"。问题：

- **没有谁负责收尾** —— 聊到什么时候算完？谁拍板？
- **上下文互相污染** —— A 的废话灌进 B 的窗口，B 开始跑偏。
- **容易抬杠死循环** —— A 说不对，B 反驳，A 再反驳……谁也停不下来（[§6](#6-四大工程难题与应对第条) 的"争论""循环"）。
- **无法追责** —— 出错了不知道是哪个 agent 的锅。

> 🚨 **结论：** 多智能体必须有一个"组织者"（supervisor）或一张"流程图"（graph）来约束谁在什么时候、对谁说话。这就是 Stage 4 第②条要求的本质。

### 4.2 模式一：Supervisor（主管 / 中心调度）

**结构：** 一个"主管" agent 在中间，所有 worker 都只跟主管对话，worker 之间不直接聊。

```text
用户 → [Supervisor] → 派活给 Worker A
                 ← Worker A 回报结果
       [Supervisor] → 派活给 Worker B（带着 A 的结果）
                 ← Worker B 回报
       [Supervisor] → 判断：够了 → 回复用户 / 不够 → 继续派活
```

**优点：** 中心清晰、易调试、易加停止条件。**缺点：** 主管可能成为瓶颈；主管自身上下文会变大。

### 4.3 模式二：Graph（状态图 / 流程图）

**结构：** 把流程画成"节点（node）+ 边（edge）"。每个节点是一个 agent/函数，边决定流转；**条件边**决定是往下走还是绕回去（循环）。状态（state）在节点间传递。

```text
start → researcher → writer → reviewer
                              │
                   条件边：没通过？──是──→ reviser ─┐
                              │否                │
                              ↓                  ┘（回到 reviewer）
                           finish
```

**优点：** 流程显式、可可视化、天然支持循环/并行、状态隔离清晰。**缺点：** 图要提前设计，灵活性略低。

| 维度 | Supervisor（主管式） | Graph（图式） |
|---|---|---|
| 谁决定下一步 | 主管 LLM 动态判断 | 预定义的边 / 条件边 |
| 灵活性 | 高（可临场决策） | 中（结构固定） |
| 可预测性 | 中 | 高（流程可视） |
| 循环怎么表达 | 主管判断是否再派活 | 条件边指回上游节点 |
| 典型实现 | Claude Code 子代理委派、ADK 父 agent 委派 | LangGraph 状态图、ADK 2.0 Graph Workflows |

> 💡 实际项目里两者常结合：用 **graph** 搭主干流程，在某一节点用 **supervisor** 动态选子 agent。Stage 4 要求你"至少会其中一种并能讲清区别"。

---

## 5. 定义边界：职责 / 输入输出 schema / 停止条件（第③条）

这是多智能体能不能跑起来的"工程地基"。给每个 agent 写三样东西：

### 5.1 职责边界（它只管什么）

用一句话 + 一个"不做什么"清单。例：

```text
# reviewer 的岗位说明书
职责：检查 draft 的事实准确性、结构完整性、是否遗漏用户要求。
不做什么：不自己重写（重写交给 reviser）；不查外部资料（那是 researcher 的活）。
```

### 5.2 输入输出 schema（工作单格式）

强制每个 agent 的输入输出是**结构化**的（JSON / 固定字段），而不是自由文本。好处：下游能稳定解析，不会"看不懂上游说了啥"。

```json
{
  "role": "reviewer",
  "input":  { "draft": "string", "requirements": ["string"] },
  "output": {
    "passed": true,
    "issues": [ { "location": "第2段", "problem": "缺数据来源", "severity": "high" } ],
    "score": 0.82
  }
}
```

### 5.3 停止条件（termination）

必须有明确的"到此为止"规则，否则就会无限循环（[§6](#6-四大工程难题与应对第条)）。常见停止条件：

- ✅ Reviewer 返回 `passed = true`
- 🔢 修订次数达到上限（如最多 3 轮）
- ⏱️ 总步数 / 总 token 超过预算
- 🎯 达到目标指标（如评分 ≥ 0.9）

> ⚠️ **关键纪律：** 停止条件必须"双保险"——既要有"成功即停"，也要有"兜底上限"。单靠"评审通过"很容易因为评审永远不通过而死循环（这正是我们 Demo 里加 `revision < 2` 的原因）。

---

## 6. 四大工程难题与应对（第④条）

| 难题 | 它长什么样 | 怎么发现 | 怎么治 |
|---|---|---|---|
| **🔁 循环 Loop** | A 改完 B 打回，B 改完 A 又打回，反复 N 轮没完 | 步数/轮数异常高；日志里同样的话重复出现 | 加修订上限；让 Reviewer 给"是否值得再改"的二元判断；引入冷却/衰减 |
| **⚔️ 争论 Bickering** | 两个 agent 互相否定，谁也不让步，没有收敛 | 对话里出现大量"你说得不对 / 我坚持" | 让 Supervisor 当裁判拍板；或设"最多反驳 1 次就采纳多数"；给 Critic 设权威上限 |
| **🌀 任务漂移 Task Drift** | 做着做着忘了原始目标，去优化无关细节 | 最终产出和用户原始诉求对不上 | 每个 agent 输入里始终带入"原始目标"字段；Planner 定期对照目标；加目标一致性检查 |
| **📈 上下文膨胀 Context Bloat** | 所有历史都塞进一个窗口，越来越慢、越来越贵、越来越乱 | token 费用飙升；后期输出质量下降 | 子代理隔离上下文（见 [§8.1](#81--claude-code-subagents子代理)）；只传必要的 state 字段；定期 compact/摘要；用 graph 切分状态 |

---

## 7. 何时单 agent 反而更好（第⑤条）

多智能体不是银弹。**能用单 agent + 好提示词 + 几个工具解决的，就别拆。** 用下面这张决策表判断：

| 信号 | 选 单 agent | 选 多 agent |
|---|---|---|
| 任务是否可一步完成 | ✅ 是 | 需多角色协作 |
| 是否需要对同一产物反复"生成+审查" | 一次成稿即可 | ✅ 需要 write→review→revise |
| 是否需要隔离上下文/控制成本 | 单窗口够用 | ✅ 某步会产生海量中间物 |
| 是否需要不同"人设/工具权限" | 一套提示词够 | ✅ 有人只能读、有人能写 |
| 是否对可预测性/可回放要求高 | 可接受一定随机 | ✅ 要稳定可控流程 |

> 📌 **一句话原则：** 先写单 agent 原型；当你发现"它总在同一个地方出错、需要专人把关、或上下文撑爆"时，再针对痛点拆出对应角色。不要为了"看起来高级"而上多智能体。

---

## 8. 五大学习素材精读（把推荐阅读嚼碎了喂给你）

Stage 4 列的 5 篇阅读，本质是在教你四种"协调技术"。下面逐篇讲：它解决什么问题、核心概念、生活类比、最小示例、和 Stage 4 的关系。

### 8.1 🧩 Claude Code Subagents（子代理）

> **解决什么：** 如何定义"专业分工的 AI 员工"，并让它们在**各自独立的上下文**里干活，只把摘要回报给主线，从而保护主对话不被海量中间信息淹没。

| 核心概念 | 说明 |
|---|---|
| 定义方式 | 一个带 YAML 头（frontmatter）的 Markdown 文件，含 `name`、`description`（何时委派给它）、`tools`（能用哪些工具）、`model`（可用更便宜的模型）、`maxTurns` 等 |
| 上下文隔离 | 每个子代理启动时是**全新空白上下文**，看不到主对话历史；只通过"委派消息 + 返回摘要"通信 —— 这正是治"上下文膨胀"的利器 |
| 委派方式 | 自动（看 description 匹配）/ @提及 / 显式调用；可前台（阻塞）或后台（并发） |
| 典型模式 | ①隔离高容量操作（跑测试只回报失败项）②并行研究（三个模块三个代理）③链式子代理（先 reviewer 找问题，再 optimizer 修） |

```markdown
---
name: code-reviewer
description: Reviews code for quality and best practices
tools: Read, Glob, Grep
model: sonnet
---
You are a code reviewer. Analyze the code and give actionable feedback
on quality, security, and best practices.
```

> 💡 **类比：** 子代理像公司里的"外包小组"——你给 brief，他们关起门来干，最后只交一份报告，不把你卷进他们的草稿纸堆里。对应 Stage 4 的"职责边界 + 上下文隔离 + 角色分工"。

### 8.2 🪝 Claude Code Hooks（生命周期钩子）

> **解决什么：** 在 agent 运行的"特定节点"自动插一段逻辑——比如**拦截危险命令、强制格式检查、发通知**。它是多智能体系统的"纪律委员 / 闸门"。

| 生命周期事件 | 触发时机 | 能干什么 |
|---|---|---|
| `PreToolUse` | 工具调用前 | **可阻止**危险操作（如拦截 `rm -rf`） |
| `PostToolUse` | 工具调用后 | 自动 lint / 格式化 / 记录日志 |
| `Stop` | agent 准备停时 | 可阻止停止（要求补完） |
| `SessionStart` / `UserPromptSubmit` | 会话开始 / 用户提交 | 注入规则、做前置校验 |

```json
// 阻止危险命令的 PreToolUse 钩子（示意）
{
  "hooks": { "PreToolUse": [{
    "matcher": "Bash",
    "hooks": [{ "type": "command", "if": "Bash(rm *)",
                "command": "./block-rm.sh" }]
  }]}
}
```

> 💡 **和 Stage 4 的关系：** Hooks 是实现"停止条件 / 约束 / 安全"的工程手段。比如用 `Stop` 钩子强制"未通过评审不许结束"，用 `PreToolUse` 限制 reviser 只能改文件不能执行命令（职责边界的硬保障）。

### 8.3 🧠 Google ADK（Agent Development Kit）

> **解决什么：** 一个正经的"造 agent + 编排多 agent"框架（Python/TS/Go/Java）。它把"多智能体编排"做成几种**标准模式**，你直接选。

| ADK 概念 | 含义 | 对应本手册 |
|---|---|---|
| LLM Agent | 会推理、会调工具的普通智能体 | Executor / Reviewer 等角色 |
| SequentialAgent（顺序） | 子 agent 串成一列，前一个输出给下一个 | [§9](#9-实战项目research--write--review--revise) 的 research→write→review |
| ParallelAgent（并行） | 多个子 agent 同时跑，结果汇总 | 并行研究多个主题 |
| LoopAgent（循环） | 重复执行一组子 agent 直到满足条件 | review→revise 循环 |
| 父 agent 委派子 agent | 父根据情况把任务交给某个子 agent，子返回后控制权回父 | Supervisor 模式 |

```python
from google.adk import Agent

agent = Agent(
    name="researcher",
    model="gemini-flash-latest",
    instruction="You help users research topics thoroughly.",
    tools=[google_search],
)
```

> 💡 **类比：** ADK 像"乐高说明书"——它已经帮你把"顺序/并行/循环/委派"这几种团队结构预制好了，你只要把角色（Agent）插进去。这正好覆盖 Stage 4 第②条"用 supervisor 或 graph 管理"。ADK 2.0 还新增了 Graph Workflows，把"图"也做成一等公民。

### 8.4 🔗 A2A Protocol（Agent2Agent，agent 间通信标准）

> **解决什么：** 让**不同公司、不同框架、不同语言**写的 agent 能互相发现、对话、协作——即使彼此"不透明"（不暴露内部思路/工具）。

| 概念 | 说明 |
|---|---|
| Agent Card | 每个 agent 发布的"名片"（JSON）：我是谁、能干什么（skills）、在哪访问、要什么鉴权。别的 agent 先读名片再合作 |
| Task | 协作的基本工作单元，有状态：SUBMITTED→WORKING→COMPLETED / FAILED / CANCELED / INPUT_REQUIRED… |
| Message / Part / Artifact | 一轮对话=Message；最小内容单元=Part（text/raw/url/data）；最终产出=Artifact |
| 三种更新 | 轮询 / 流式（streaming）/ 推送通知（push），适配长任务 |

> 💡 **类比：** A2A 像 agent 世界的"国际邮政 + 名片交换系统"。两家公司（agent）不需要公开内部流程，只要按统一格式写"信封"（Agent Card / Task / Part），就能跨公司寄信协作。对应 Stage 4 第③条"输入输出 schema"的**跨系统版本**。

### 8.5 🔌 ACP（Agent Client Protocol，宿主与 agent 的统一接口）

> **解决什么：** 让**编辑器 / IDE / 终端**等"宿主"和"任意 agent"之间用一套标准协议对接，不用为每个 agent 单独写集成（类似 LSP 之于语言服务器）。

- **本地 agent：** 作为编辑器子进程，走 stdio 上的 JSON-RPC。
- **远程 agent：** 跑在云端，走 HTTP / WebSocket。
- 复用 MCP 的 JSON 表示，并扩展了 diff 等"编码 UX"类型；用户可读文本默认用 Markdown。

> 💡 **和 A2A 的区别（常考）：** A2A 是 **agent ↔ agent**（智能体之间）；ACP 是 **宿主应用 ↔ agent**（你的编辑器怎么跟 agent 说话）。一个管"横向协作"，一个管"纵向接入"。

---

## 9. 实战项目：research → write → review → revise

这是 Stage 4 的指定产出。下面从设计到代码一次讲透。**配套 Demo 文件：`stage4_multiagent_demo.py`（可直接运行）。**

### 9.1 角色设计与边界表

| 节点 | 角色 | 输入(schema) | 输出(schema) | 停止条件 |
|---|---|---|---|---|
| researcher | Executor | `{topic}` | `{research: str}` | 搜到材料即停 |
| writer | Executor | `{research}` | `{draft: str}` | 产出初稿即停 |
| reviewer | Reviewer | `{draft, requirements}` | `{passed: bool, issues: []}` | — |
| reviser | Executor | `{draft, issues}` | `{draft: str}`（更新） | — |
| supervisor / graph | 管理者 | 各节点回报 | final | `passed` 或 `revision≥上限` |

### 9.2 用 Supervisor 模式表达（伪码）

```python
s = TaskState(topic)
researcher(s); writer(s); reviewer(s)
while not s.review.passed and s.revision < MAX:   # 停止条件双保险
    reviser(s); reviewer(s)
return s.draft
```

### 9.3 用 Graph 模式表达（节点 + 条件边 + 循环）

```text
start → researcher → writer → reviewer
                │
   条件边(未通过 且 revision<2) ──是──▶ reviser ─┐
                │否                              │
                ▼                                ┘(回到 reviewer)
              finish
```

### 9.4 映射到真实框架（你学完 [§8](#8-五大学习素材精读把推荐阅读嚼碎了喂给你) 就该能看懂）

| 本 Demo 的概念 | ADK 写法 | LangGraph 写法 | Claude Code 写法 |
|---|---|---|---|
| 四个角色 | 4 个 `Agent` | 4 个 node 函数 | 4 个 `.md` 子代理 |
| 顺序流程 | `SequentialAgent` | 直线 edges | 主管依次 @委派 |
| review→revise 循环 | `LoopAgent` | 条件边指回 reviewer | 主管判断后再派 reviser |
| 状态传递 | agent 返回值 | `State` schema（共享/更新） | 委派消息 + 返回摘要 |

> 💡 **动手建议：** 先把 Demo 跑通（`python stage4_multiagent_demo.py`），把 `fake_llm` 换成真实模型调用，再把 reviser 拆成"critic + reviser"两个角色——你就亲手做完了 Stage 4 的产出。

---

## 10. 课堂练习（动手 ≥ 60%）✍️

### 练习 1 · 角色辨认

下面场景中，应该派哪个角色（planner/executor/reviewer/critic/router）？

- (a) 用户问"帮我看看这个方案有没有逻辑漏洞" → **___**
- (b) 把"写一份竞品报告"拆成"搜资料→写初稿→排版" → **___**
- (c) 用户消息进来，先判断他是来写代码还是来问天气，再分发 → **___**
- (d) 实际去调搜索工具把资料捞回来 → **___**
- (e) 检查初稿是否漏了用户要求的三个要点 → **___**

<details>
<summary>点击看答案</summary>

✅ (a) critic　(b) planner　(c) router　(d) executor　(e) reviewer

</details>

### 练习 2 · 找"群聊"的毛病

小明把三个 LLM 放进一个群："你们商量把这个需求做成产品"。请列出至少 3 个会翻车的地方，并各给一句"该怎么用 supervisor/graph 修"。

<details>
<summary>点击看参考答案</summary>

① 无人收尾→用 supervisor 判断停止；② 互相污染上下文→用 graph 显式传 state 或用子代理隔离；③ 抬杠死循环→加停止条件+主管裁判；④ 无法追责→每个角色固定职责与输出 schema。

</details>

### 练习 3 · 写岗位说明书（输入输出 schema）

给"reviewer"写一个结构化输出 schema（JSON），要求包含：是否通过、问题清单（位置+问题+严重度）、综合评分。写完对比 [§5.2](#52-输入输出-schema工作单格式) 的示例，差了什么。

<details>
<summary>点击看参考</summary>

至少要有 `passed`、`issues[]`（每项含 location/problem/severity）、`score`。对比发现 §5.2 还明确了 `input` 字段（draft + requirements），这是实习生容易漏的"输入约定"。

</details>

### 练习 4 · 修一个"循环"bug

你的 review→revise 循环跑了 50 轮还不停，日志里 reviser 一直在改但 reviewer 始终 `passed=false`。请写出至少两种修复方案，并说明哪种更稳妥。

<details>
<summary>点击看参考</summary>

方案A：加修订上限（如 ≤3 轮）。方案B：让 reviewer 输出"是否值得再改"的二元判断，没价值就强停。方案C：引入衰减/冷却。最稳妥是 **A+B 双保险**：既兜底又智能，避免无谓消耗。

</details>

### 练习 5 · 单 vs 多 决策

判断以下任务用单 agent 还是多 agent，并说理由：

1. "把这段中文翻译成英文"。
2. "调研三家云厂商，写一份对比报告，并保证数据有出处、结构清晰"。
3. "读 200 个日志文件， summary 出异常，但主对话不能被日志淹没"。

<details>
<summary>点击看参考</summary>

(1) 单 agent（一步可成）。(2) 多 agent（research→write→review 需要分工与把关）。(3) 多 agent（海量中间物→用子代理隔离上下文，只回报摘要）。

</details>

---

## 11. 自测清单 + 下一步

逐条对照，全打勾 = 你已掌握 Stage 4：

- ☐ 我能说出 planner/executor/reviewer/critic/router 各自干什么
- ☐ 我能解释为什么"让 agent 群聊"会翻车，并画出 supervisor 与 graph 两种结构
- ☐ 我能给任意 agent 写"职责边界 + 输入输出 schema + 停止条件"
- ☐ 我能识别并应对 循环 / 争论 / 任务漂移 / 上下文膨胀 这四类问题
- ☐ 我能用决策表判断"这个任务该单干还是该上多智能体"
- ☐ 我跑通了 Demo，并知道如何把 fake_llm 换成真模型、如何拆出新角色
- ☐ 我能区分 A2A（agent↔agent）与 ACP（宿主↔agent），并说清 Subagents/Hooks/ADK 各自解决什么

> 📌 **下一步建议（衔接 Stage 5）：** Stage 5 是「Skills, Protocols, And Capability Packaging」——把你在 Stage 4 做的角色，进一步封装成可复用、可分发、带协议接口的"能力包"。建议先把手册 [§9](#9-实战项目research--write--review--revise) 的 Demo 真正跑通并加上一个真实 LLM，再进入 Stage 5。

---

本手册由 Agent-Learning-Hub 的 Stage 4 要求与推荐阅读整理而成 · 配套 Demo：`stage4_multiagent_demo.py`

学习原则：多智能体是协调问题，不是魔法 ✦