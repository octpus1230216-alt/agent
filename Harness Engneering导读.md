# learn-claude-code 全 17 课深度讲解（Stage 3 的深度学习素材）

> 来源：<https://github.com/shareAI-lab/learn-claude-code>  
> 定位：从零构建"类 Claude Code"的 agent harness 教学项目，共 17 节课  
> 读法：本文件是"教材讲解版"。每节课 = 大白话类比 + 核心机制 + 真实代码 + 与 Stage 3 的关系 + 课堂练习。  
> 配套：前面那份 `Stage3_Learning_Guide.md` 讲了"Stage 3 表面要你做什么"，本文件讲"learn-claude-code 全部内容"，两者配合 = 真正掌握 Stage 3。

---

## 0. 为什么 Stage 3 让你读这个项目

先记住一句话，它是理解全部 17 课的总纲：

> **"Agency（感知-推理-行动的能力）来自模型训练，不来自外部代码编排。模型是司机，harness 是车。本项目教你造车。"**

- 你之前在 Stage 0–2 造的"会调工具的循环" = 一辆能动的裸车。
- learn-claude-code 把一辆**生产级可用的车**拆成 17 个零件，每节课装一个，最后整车成型。
- **重点是"机制怎么组织"，不是"某行 API 怎么调"。** 这正好对应 Stage 3 导语里那句"重点不是框架 API 怎么调"。

你正在用的这类 AI 编程助手（Claude Code / CodeBuddy 等），内部就是这样一个 harness——所以学它，等于拆开你每天在用的东西看内脏。

---

## 1. 主线地图：一个不变的循环 + 逐层叠加

整个项目只有一条主线，请刻进脑子里：

> **在一段永远不变的 `while True` Agent Loop 之上，一层一层叠加机制。循环本身从 s01 到 s17 一行都没改过。**

| 层  | 章节                                            | 解决的本质问题               |
| -- | --------------------------------------------- | --------------------- |
| 内核 | s01 Agent Loop                                | 模型只吐文本，怎么让它"持续行动"     |
| 工具 | s02 Tool Use, s03 Permission, s04 Hooks       | 扩展能触达的边界 + 边界上的安全与扩展点 |
| 规划 | s05 TodoWrite, s06 Subagent                   | 复杂任务先想清楚、大任务隔离上下文     |
| 知识 | s07 Skill, s08 Compact, s09 Memory            | 上下文与长期知识的加载、压缩、持久化    |
| 协调 | s10 Task, s11 Background, s12 Cron, s13 Teams | 多步骤/异步/定时/多 Agent 编排  |
| 接入 | s14 MCP, s15 Integrated                       | 外挂工具与全机制合并            |
| 编排 | s16 Workflow                                  | 固定多 Agent 脚本的可恢复编排    |
| 目标 | s17 Goal Loop                                 | 模型说"停" ≠ 目标达成         |

**关键洞见**：前面学的 Stage 3「五大模块」(loop / tool / permission / session / compaction) 在这里分别对应 s01 / s02 / s03 / s09 / s08。其余 12 课是"车造好后还要装的中控、副驾、定速巡航、车队通信……"。下面逐课讲。

---

## 2. 逐课讲解（s01 → s17）

---

### s01 · Agent Loop — 一个循环就够了

**口号**：*"One loop & Bash is all you need"*（一个工具 + 一个循环 = 一个 Agent）

**大白话**：模型像一个人，你说一句话他回一句。但真要他"去改个文件"，他只会写一句"我建议运行 `sed ...`"——不会自己动手，也不会看结果。Agent 就是把"人肉复制粘贴命令→看输出→再问"的过程自动化：让程序不断问模型、执行、把结果喂回去。

**核心机制**：

- `while True`：调 LLM → 看回复里有没有 `tool_use` 块 → 有就执行并把结果追加回 `messages` → 没有就退出。
- 模型负责"决策"（调不调、调哪个），harness 负责"执行"（调工具、把结果当新消息追加）。
- 只有真实的 `tool_use` 块才进执行阶段，绝不追加空结果。

**关键代码**（节自 `s01_agent_loop/code.py`）：

```python
def agent_loop(messages: list):
    while True:
        response = client.messages.create(
            model=MODEL, system=SYSTEM, messages=messages,
            tools=TOOLS, max_tokens=8000,
        )
        messages.append({"role": "assistant", "content": response.content})
        tool_calls = [b for b in response.content if b.type == "tool_use"]
        if not tool_calls:
            return                      # 模型没调工具 → 循环结束
        results = []
        for block in tool_calls:
            output = run_bash(block.input["command"])
            results.append({"type": "tool_result", "tool_use_id": block.id, "content": output})
        messages.append({"role": "user", "content": results})  # 结果喂回 → 继续
```

**和 Stage 3 的关系**：这就是 Stage 3 五大模块里的 **agent loop**（发动机）。30 多行就能跑，是理解后面一切的地基。

**课堂练习 1.1**：把 `if not tool_calls: return` 改成 `if not tool_calls: break`（等价），再想：如果没有这个退出判断，循环会怎样？（答：永远不退出，一直空转烧 token。所以"模型不再要工具"就是天然的停车信号。）

---

### s02 · Tool Use — 多加一个工具，只加一行

**口号**：*"加一个工具，只加一个 handler"*（循环不动，新工具注册进分发表）

**大白话**：只有 bash 时，读文件要 `cat`，找文件要 `find`，既费 token 又易出错。就像公司只有"一个万能工"（啥都让他干），不如请几个专职工（读文件的、写文件的、搜文件的）。关键是：请新专职工时，**主管的排班表（循环）不用重写**，只在他的人事册上加一页。

**核心机制**：

- `TOOLS`：告诉模型"我能做什么"的 JSON 描述数组。
- `TOOL_HANDLERS`：工具名 → 处理函数的**分发字典**（dispatch map）。
- 加工具 = 在 `TOOLS` 加一条描述 + 在 `TOOL_HANDLERS` 加一行；`while True` 一行不改。
- `safe_path()`：文件类工具的"别跑出工作区"越界校验。

**关键代码**（节自 `s02_tool_use/code.py`）：

```python
TOOL_HANDLERS = {
    "bash": run_bash, "read_file": run_read,
    "write_file": run_write, "edit_file": run_edit, "glob": run_glob,
}
# 循环里只把"硬编码 run_bash"改成"查表"：
for block in tool_calls:
    handler = TOOL_HANDLERS.get(block.name)        # 查表分发
    output = handler(**block.input) if handler else f"Unknown: {block.name}"
    results.append({"type": "tool_result", "tool_use_id": block.id, "content": output})

def safe_path(p: str) -> Path:
    path = (WORKDIR / p).resolve()
    if not path.is_relative_to(WORKDIR):
        raise ValueError(f"Path escapes workspace: {p}")
    return path
```

**和 Stage 3 的关系**：这是 **tool registry**（手脚登记册）。分发字典（dispatch map）是后面权限、hook、MCP 共同的骨架。

**课堂练习 2.1**：在 `TOOL_HANDLERS` 里加一个 `"reverse": lambda s: s[::-1}`，并在 `TOOLS` 加它的描述。跑一次让模型调用它，体会"加工具不动循环"。

---

### s03 · Permission — 执行前做权限判断

**口号**：*"先划边界，再给自由"*（三道闸门：硬拒绝 → 软规则 → 问人）

**大白话**：没有刹车的车，马力越大越危险。让模型能跑 `rm -rf /` 这种命令，等于让一个不懂事的大力士随便拔电源。所以每调一个工具，先过三道安检：① 黑名单（永远禁止）② 危险规则（要审批）③ 其余放行但要不要问人。

**核心机制**（三道闸门，硬拒绝优先）：

- Gate 1 `DENY_LIST`：永远禁止（`rm -rf /`、`sudo`、`shutdown`…），字符串匹配。
- Gate 2 `PERMISSION_RULES`：上下文相关（写工作区外、破坏性命令）。
- Gate 3 `ask_user`：命中规则后暂停等用户确认。
- `check_permission(block)` 插在工具执行**之前**，循环只加一行。

**关键代码**（节自 `s03_permission/code.py`）：

```python
DENY_LIST = ["rm -rf /", "sudo", "shutdown", "reboot", "mkfs", "dd if=", "> /dev/sda"]
PERMISSION_RULES = [
    {"tools": ["read_file", "write_file", "edit_file"],
     "check": lambda a: not (WORKDIR / a.get("path","")).resolve().is_relative_to(WORKDIR),
     "message": "Writing outside workspace"},
    {"tools": ["bash"],
     "check": lambda a: contains_destructive_command(a.get("command",""))
                       or any(k in a.get("command","") for k in ["rm ", "> /etc/", "chmod 777"]),
     "message": "Potentially destructive command"},
]
def check_permission(block) -> bool:
    if block.name == "bash":
        reason = check_deny_list(block.input.get("command",""))
        if reason: return False                       # Gate 1
    reason = check_rules(block.name, block.input)
    if reason:
        decision = ask_user(block.name, block.input, reason)
        if decision == "deny": return False          # Gate 2 + 3
    return True
# 循环里：if not check_permission(block): 记"Permission denied"并 continue
```

**和 Stage 3 的关系**：这就是 **permission gate**（方向盘+刹车）。没有它，"行动能力"会变成"破坏能力"。

**课堂练习 3.1**：把 `"chmod 777"` 从规则里删掉，再让模型跑 `chmod 777 /tmp/x`，看是否不再被拦。思考：为什么"默认放行 + 规则拦截"比"默认禁止 + 白名单"更适合真实产品？（提示：白名单会漏掉无数没列出来的危险命令。）

---

### s04 · Hooks — 挂在循环上，不写进循环里

**口号**：*"挂在循环上，不写进循环里"*（hook 在工具前后注入扩展逻辑）

**大白话**：s03 每加一个检查（日志、自动 git add）都要改 `agent_loop`，循环很快膨胀到认不出原貌。就像为了装行车记录仪去改发动机。正确做法：在发动机几个固定位置留"外接接口"，要加功能就插个模块，发动机本体不动。

**核心机制**（四个事件，覆盖一次完整 agent 周期）：

- `UserPromptSubmit`：用户输入后、进 LLM 前（输入校验/注入上下文）。
- `PreToolUse`：工具执行前（权限、日志）。返回非 `None` → 阻止本次执行。
- `PostToolUse`：工具执行后（副作用如自动 git add、大输出告警）。
- `Stop`：循环即将退出时（统计、清理、可强制继续）。
- `HOOKS` 注册表 + `register_hook()` / `trigger_hooks()`；循环只调 `trigger_hooks`，逻辑全在回调里。

**关键代码**（节自 `s04_hooks/code.py` 思路）：

```python
HOOKS = {"UserPromptSubmit": [], "PreToolUse": [], "PostToolUse": [], "Stop": []}
def register_hook(event, callback): HOOKS[event].append(callback)
def trigger_hooks(event, *args):
    for cb in HOOKS[event]:
        result = cb(*args)
        if result is not None:        # 返回值 ≠ None → hook 说"停"
            return result
    return None
# 循环里 s03 的 check_permission(block) 变成：
blocked = trigger_hooks("PreToolUse", block)
if blocked:
    results.append({"type":"tool_result","tool_use_id":block.id,"content":str(blocked)}); continue
```

**和 Stage 3 的关系**：这是 harness 的"扩展点"思想。Stage 3 没明说 hook，但你读的 Claude Code Docs 里 hooks 是核心——它让内核稳定、把日志/审计/权限都变可插拔回调。

**课堂练习 4.1**：注册一个 `PostToolUse` hook，每次工具执行后 `print(f"[审计] 调用了 {block.name}")`）。观察日志，体会"没改循环就加了功能"。

---

### s05 · TodoWrite — 没有计划的 Agent 会跑偏

**口号**：*"没有计划的 Agent 随波逐流"*（先列步骤，再执行）

> ⚠️ 注：本节中文 README 在仓库返回 404，以下据英文版整理。

**大白话**：给 Agent 一个复杂任务，它做到一半会被测试失败之类的事吸走注意力，忘了最初目标（比如"重命名成 snake_case"）。长对话里，开头的 system 指令被海量工具结果稀释了。就像一个没清单的人逛超市，买着买着忘了要买啥。解法：逼它先写清单、每做完一条划掉。

**核心机制**：

- `TodoManager`：进程内清单，校验更新、渲染 `[ ]/[>]/[x]`（待办/进行中/完成）。
- `todo_write` 工具：经同一分发表调用；只更新计划状态，不增加执行能力（新增的是"规划能力"）。
- **提醒机制**：连续 3 轮没更新清单，就注入 `<reminder>Update your todos.</reminder>`。
- 约束：最多 20 项，同一时刻仅一个 `in_progress`。

**关键代码**（据 `s05_todo_write/README.md`）：

```python
class TodoManager:
    def __init__(self): self.items = []
    def update(self, todos):
        self.items = [...]   # 解析+校验后再替换当前列表
        return self.render()
TOOL_HANDLERS["todo_write"] = run_todo_write
# 提醒注入
rounds_since_todo = 0 if used_todo else rounds_since_todo + 1
if rounds_since_todo >= 3:
    results.append({"type":"text","text":"<reminder>Update your todos.</reminder>"})
    rounds_since_todo = 0
```

**和 Stage 3 的关系**：规划不在 Stage 3 五大模块里，但它是 agent "从会执行到靠得住"的关键一步，也是 s10 持久化任务系统的轻量前身。

**课堂练习 5.1**：给一个"重构整个 utils 目录"的任务，先让模型 `todo_write` 拆成 5 步，再观察它是否更少跑偏。

---

### s06 · Subagent — 给子任务一段独立上下文

**口号**：*子 agent 从全新的 `messages[]` 开始，最终文本返回父循环，中间对话不进父上下文。*

**大白话**：主 agent 为追调用链读了一堆文件，这些中间细节在子任务完成后还占着主对话的"脑子"（上下文）。就像一个主管亲自去翻 20 份档案，翻完脑子被塞满，正事反而想不清。解法：派个实习生去翻，实习生自己记笔记，最后只把"结论一页纸"交给主管。

**核心机制**：

- `run_subagent(prompt)`：同步跑一个全新 `messages[]` 的嵌套 loop，结束只把**最终文本**作为父对话的 `tool_result` 返回。
- 隔离的是消息，不是进程/文件系统：父子共享 `WORKDIR`，写文件互相可见；子 agent 用 `SUB_TOOLS`（无 `task` 工具，只允许一层委派）。
- `task` 工具经分发表调用，和普通工具一致。

**关键代码**（据 `s06_subagent/README.md`）：

```python
SUB_TOOLS = list(BASE_TOOLS)   # 无 task 工具
def run_subagent(prompt: str) -> str:
    messages = [{"role":"user","content":prompt}]
    for _ in range(30):
        r = client.messages.create(model=MODEL, system=SUB_SYSTEM, messages=messages, tools=SUB_TOOLS, max_tokens=8000)
        messages.append({"role":"assistant","content":r.content})
        calls = [b for b in r.content if b.type=="tool_use"]
        if not calls: return extract_text(r.content) or "(no summary)"
        results=[]
        for b in calls:
            results.append({"type":"tool_result","tool_use_id":b.id,"content":execute_tool(b, SUB_HANDLERS)})
        messages.append({"role":"user","content":results})
    return "Subagent stopped after 30 turns without a final answer."
TOOLS = [*BASE_TOOLS, TASK_TOOL]
TOOL_HANDLERS = {**BASE_HANDLERS, "task": run_subagent}
```

**和 Stage 3 的关系**：这是 context 管理的重要手段，也是 s13 团队、s16 workflow 的"上下文隔离"原型。**上下文隔离是并行/委派的基础**——一个 agent 的中间过程不该污染另一个 agent 的推理。

**课堂练习 6.1**：让主 agent 派子 agent 去"读 3 个文件总结它们的关系"，对比：子 agent 的 10 轮对话是否出现在父 agent 的 `messages` 里？（答：不会，父只拿到最终总结，省了上下文。）

---

### s07 · Skill Loading — 用到时再加载

**口号**：*system prompt 只存技能目录；`load_skill` 返回完整 `SKILL.md`。*

**大白话**：把 React 规范、SQL 指南、API 文档全塞进 system prompt，会和当前任务无关却仍占"脑子"（token）。像把整座图书馆搬进办公室，其实今天只查一本书。解法：书架空着，只放**书目卡片**；需要时再取下那本。

**核心机制**：

- `SkillLoader.scan()`：启动时扫描 `skills/*/SKILL.md`，解析 YAML 头部的 `name`/`description` 建目录。
- system prompt 只放**技能名称和描述目录**（`catalog()`）；模型需要时调 `load_skill(name)`。
- `load_skill` 返回完整 `SKILL.md`，作为 `tool_result` 追加（和读文件一样自然）。

**关键代码**（据 `s07_skill_loading/README.md`）：

```python
class SkillLoader:
    def scan(self):
        for manifest in sorted(self.skills_dir.glob("*/SKILL.md")):
            content = manifest.read_text(encoding="utf-8")
            metadata, body = self.parse_frontmatter(content)
            self.skills[name] = {"name":name,"description":description,"content":content}
def build_system_prompt() -> str:
    return (f"You are a coding agent at {WORKDIR}...\n\n"
            f"Skills available:\n{SKILL_LOADER.catalog()}\n\n"
            "Use load_skill to read the full instructions when a skill applies.")
def load(self, name):
    skill = self.skills.get(name)
    return skill["content"] if skill else f"Error: Unknown skill '{name}'..."
```

**和 Stage 3 的关系**：延迟加载（lazy loading）是控制上下文成本的通用范式，和 s08 压缩、s09 记忆同属"上下文预算管理"主题。你之前在 `Stage3_Learning_Guide.md` 里看到的 Skill 概念，这里就是它的实现。

**课堂练习 7.1**：在 `skills/` 下新建 `skills/reverse/SKILL.md`（描述"把字符串反转"），看 `catalog()` 是否出现它，再让模型 `load_skill("reverse")` 读全文。

---

### s08 · Context Compact — 上下文总会满，先整理，再总结

**口号**：*"上下文总会满，要有办法腾地方。"*（四步压缩，低成本操作优先）

**大白话**：工具结果（长文件、测试日志、多文件搜索）会迅速撑爆"脑子"。直接让模型总结全历史既丢细节又费一次调用。像行李箱满了：先压实衣物（无损）→ 再把旧东西寄回家只留清单（半损）→ 最后才把前三天经历写成一段话（有损）。**能用便宜手段腾地方就别用贵的。**

**核心机制**（固定顺序四步管线，只在必要时进有损摘要）：

1. `snip_compact`：消息 >50 条，归档中段到 `.transcripts/`，留前 3 + 后 46 + 归档标记（保护 tool_use/tool_result 配对不被截断）。
2. `micro_compact`：>50000 字符，把更早的 tool_result 缩短成"结果已存于 "，直到降到阈值 80%。
3. `compact_history`：仍超限 → 写 transcript + 让模型生成"只含事实"的状态摘要，用 `[Compacted]` 消息替换历史。

另含 `fit_tool_results`（未读新结果过大先留预览）、`reactive_compact`（API 报 `prompt_too_long` 兜底）、`compact` 工具（模型主动请求压缩）。

**关键代码**（节自 `s08_context_compact/code.py`）：

```python
class ContextCompactor:
    CONTEXT_CHAR_LIMIT = 50000
    TOOL_RESULT_BATCH_CHAR_LIMIT = 200000
    LARGE_RESULT_CHAR_LIMIT = 30000
    KEEP_RECENT_RESULTS = 3
    def prepare(self, messages, active_request):
        messages = self.tool_result_budget(messages)        # Step 1
        messages = self.snip_compact(messages)              # Step 2
        if self.estimate_chars(messages) > self.CONTEXT_CHAR_LIMIT:
            target = int(self.CONTEXT_CHAR_LIMIT * 0.8)
            messages = self.micro_compact(messages, target)  # Step 3a
            if self.estimate_chars(messages) > self.CONTEXT_CHAR_LIMIT:
                messages = self.fit_tool_results(messages, target)   # Step 3b
            if self.estimate_chars(messages) > self.CONTEXT_CHAR_LIMIT:
                messages = self.compact_history(messages, active_request)  # Step 3c
        return messages
```

**和 Stage 3 的关系**：这就是 **context compaction**（后备箱收纳）。它是"可恢复、确定性的压缩策略"——先无损转存，最后才摘要，是长任务能在有限窗口持续工作的工程核心。

**课堂练习 8.1**：把 `CONTEXT_CHAR_LIMIT` 调到很小（如 2000），跑一个会读大文件的任务，观察 `messages` 怎样从"原始内容"变成"路径+预览"再到"[Compacted] 摘要"。

---

### s09 · Memory — 让重要信息跨会话保留下来

**口号**：*"把以后还会用到的信息留下来。"*（文件存储 + 索引 + 相关性选择 + 按需召回）

**大白话**：新开一个会话，`messages` 是空的，上次聊的用户偏好、项目背景、排查线索全没了。全写进 system prompt 又会让无关内容一直占脑子。像人的长期记忆：不是把所有事塞进短期记忆，而是记在笔记本里，需要时才翻到相关那页。

**核心机制**（四件事：存储 / 召回 / 提取 / 整理）：

- 存储：`.memory/<slug>.md` 每记忆一文件，YAML 头部记 `name`/`description`/`type`（`user`/`feedback`/`project`/`reference`）。`MEMORY.md` 是索引。
- 召回：`select_relevant_memories()` 用轻量模型选最多 5 条相关索引（失败回退关键词）。
- 提取：`extract_memories()` 回合结束从对话提取，只有 `persistent` 才跨会话保存，`current_task` 只约束本次。
- 整理：`consolidate_memories()` 达 10 条时合并去重，带快照、失败回滚。

**关键代码**（节自 `s09_memory/code.py`）：

```python
def write_memory_file(name, mem_type, description, body) -> Path:
    path = memory_path(f"{memory_slug(name)}.md")
    path.write_text(memory_document(name, mem_type, description, body), encoding="utf-8")
    rebuild_memory_index()                      # 写后重建索引
    return path
def select_relevant_memories(messages, max_items=5) -> list:
    response = client.messages.create(model=MODEL, messages=[...], max_tokens=200)
    indices = extract_json_array(...)
    return [records[i]["filename"] for i in indices if 0 <= i < len(records)][:max_items]
def extract_memories(messages) -> int:
    candidates = [v for item in extract_json_array(...)
                  if (v := validate_memory_record(item, require_scope=True)) is not None]
    for c in candidates:
        if not should_store_memory(c, existing_records): continue   # 仅 persistent 且非重复
        write_memory_file(c["name"], c["type"], c["description"], c["body"])
def consolidate_memories() -> int:
    if len(records) < CONSOLIDATE_THRESHOLD: return 0
    snapshot = {r["filename"]: memory_path(r["filename"]).read_text() for r in records}
    try:
        for p in MEMORY_DIR.glob("*.md"):
            if p.name != MEMORY_INDEX.name: memory_path(p.name).unlink()
        for rec in consolidated: memory_path(f"{memory_slug(rec['name'])}.md").write_text(...)
        rebuild_memory_index()
    except Exception:
        for fn, content in snapshot.items(): memory_path(fn).write_text(content)  # 回滚
        raise
```

**和 Stage 3 的关系**：这就是 **session store / 记忆**（油箱+笔记本）。注意它和 s08 边界清晰：s08 管**会话内**上下文预算，s09 管**会话外**可复用知识。两者是上下文管理的两条不同路径。

**课堂练习 9.1**：在一轮对话里告诉它"我姓武，做 ESG 研究"，结束后再开新会话，看它能否通过 `select_relevant_memories` 把这条 `user` 记忆召回。

---

### s10 · Task System — 从执行清单到可协调的任务状态

**口号**：*"大目标拆成小任务，排好序，持久化"*（文件持久化的任务图，多 agent 协作的基础）

**大白话**：s05 的清单只有"内容+状态"，记不住依赖（"数据库表完成才能写 API"）和分工（"谁负责"）。像装修清单写了"刷墙、铺地板"，但没标"必须先刷墙再铺地板"，工人可能搞反顺序。解法：给每个任务加状态机 + 依赖边 + 负责人，写成存盘的文件，谁都能看、都能接手。

**核心机制**：

- `Task` dataclass：`id`/`subject`/`description`/`status`(pending|in_progress|completed)/`owner`/`blockedBy`。
- `.tasks/{id}.json` 文件持久化；`TaskStore` 校验 ID、读写、做环检测。
- 两阶段建图：先 `create_task` 拿运行时 ID，再 `update_task(addBlockedBy=...)` 加边（因为同级工具调用在产生结果前已定型，无法互相引用刚生成的 ID）。
- 状态机：`pending --claim--> in_progress --complete--> completed`；`complete_task` 对比完成前后可开始集合，输出被解锁的下游任务。

**关键代码**（节自 `s10_task_system/code.py`）：

```python
@dataclass
class Task:
    id: str; subject: str; description: str
    status: str; owner: str | None; blockedBy: list[str]
def incomplete_dependencies(task) -> list:
    return [d for d in task.blockedBy if load_task(d).status != "completed"]
def can_start(task_id) -> bool:
    return not incomplete_dependencies(load_task(task_id))
def claim_task(task_id, owner="agent") -> str:
    task = load_task(task_id)
    if task.status != "pending": return f"Task {task_id} is {task.status}"
    if incomplete_dependencies(task): return f"Blocked by: {deps}"
    task.owner = owner; task.status = "in_progress"; TASKS.save(task)
    return f"Claimed {task.id} ({task.subject})"
def complete_task(task_id, owner="agent") -> str:
    task = load_task(task_id)
    if task.status != "in_progress": return f"Task {task_id} is {task.status}"
    if task.owner != owner: return f"Task {task_id} owned by {task.owner}"
    ready_before = {t.id for t in list_tasks() if t.status=="pending" and t.blockedBy and can_start(t.id)}
    task.status = "completed"; TASKS.save(task)
    unblocked = [t.subject for t in list_tasks() if t.status=="pending" and t.blockedBy
                 and t.id not in ready_before and can_start(t.id)]
    return f"Completed {task.id} ({task.subject})" + (f"\nUnblocked: {', '.join(unblocked)}" if unblocked else "")
```

**和 Stage 3 的关系**：这是 session store 的"升级版持久化"——把任务图也落盘，是"多步骤工作可恢复进度"和"多 Agent 分工"的数据基础。

**课堂练习 10.1**：建任务 A（无依赖）、B（blockedBy A），先 `claim_task B` 看是否被拒，再 `complete_task A`，再 `claim_task B` 看是否成功并提示 "Unblocked"。

---

### s11 · Background Tasks — 慢操作放后台

**口号**：*"慢操作放后台，Agent Loop 继续运行"*（后台线程执行，后续轮次收集结果）

**大白话**：装依赖、跑完整测试可能要好几分钟。同步执行会卡住整个 agent，但这期间它本可以去看文档、整理文件。像你煮汤时不会干坐着等，而是去切菜。解法：把慢命令丢后台，agent 继续干别的，汤好了再通知它。

**核心机制**：

- `should_run_background(name, input)`：仅当 `bash` 且 `run_in_background is True` 才进后台（不再按关键词猜）。
- `BackgroundManager`：daemon 线程执行，`start()` 立即返回 `bg_id`；完成/失败入 `_ready` 队列。
- `inject_background_results()`：后续轮次收集已完成结果，以 `<task_notification>` 注入（不复用原 `tool_use_id`，保证一个 tool_use 只对应一个 tool_result）。
- 后台结果不会主动唤醒 agent，下次进循环才收集。

**关键代码**（据 `s11_background_tasks/README.md`）：

```python
def should_run_background(tool_name, tool_input) -> bool:
    return tool_name == "bash" and tool_input.get("run_in_background") is True
class BackgroundManager:
    def start(self, block) -> str: ...          # 登记并启 daemon 线程，返回 bg_id
    def _run(self, task_id, command):
        output, exit_code = _run_bash_process(command)
        status = "completed" if exit_code == 0 else "failed"
        with self._lock:
            self.tasks[task_id]["status"] = status
            self.results[task_id] = _format_bash_result(output, exit_code)
            self._ready.append(task_id)
while True:
    inject_background_results(messages)         # 先收集已完成的后台结果
    response = client.messages.create(...)
def execute_tool(block):
    if should_run_background(block.name, block.input):
        task_id = start_background_task(block); output = f"[Background task {task_id} started]"
    else: output = call_tool(block)
```

**和 Stage 3 的关系**：异步执行 + 通知注入，让 agent 不被慢操作阻塞，是"并发利用时间"的基础。这是 harness "能长跑"的关键能力之一。

**课堂练习 11.1**：让模型 `bash` 跑一个 `sleep 5 && echo done` 且 `run_in_background=True`，观察主循环是否立即拿到 `bg_id` 继续，5 秒后下一轮才收到 `<task_notification>`。

---

### s12 · Cron Scheduler — 按时间启动任务

**口号**：*"按时间启动一轮 Agent Loop"*（调度线程匹配 cron，空闲时交付）

**大白话**：s11 只管"命令开始后怎么跑"，不管"何时开始"。"每天 9 点跑测试"若只靠当前循环，你每次到点都得重发 prompt。像你不想每天手动设闹钟，而是设一个"每天 9 点自动响"的定时任务。解法：调度线程每秒看一眼 cron，到点就把任务作为新消息塞进对话。

**核心机制**：

- `CronJob` dataclass：`id`/`cron`/`prompt`/`recurring`/`durable`/`pending_delivery`/`last_fired`。
- 五段式 cron：`分 时 日 月 周`，支持 `*`/`*/N`/`N`/`N-M`/`N,M`；`validate_cron()` 预校验。
- `poll_due_jobs()`：每秒匹配，到期先存 `pending_delivery`+`last_fired` 再入内存队列（防同分钟重复）。
- `queue_processor_loop()`：用 `agent_lock` 避免与用户会话同时改会话，空闲时才起一轮，把 `[Scheduled] <prompt>` 追加为 user 消息。
- 持久化：`durable=True` → `.scheduled_tasks.json`（临时文件 + `os.replace` 原子更新）；至少一次交付语义。

**关键代码**（据 `s12_cron_scheduler/README.md`）：

```python
@dataclass
class CronJob:
    id: str; cron: str; prompt: str
    recurring: bool; durable: bool
    pending_delivery: bool = False; last_fired: str | None = None
def poll_due_jobs(moment):
    for job in list(scheduled_jobs.values()):
        if job.pending_delivery or job.last_fired == minute_marker: continue
        if cron_matches(job.cron, moment): _enqueue_due_job(job, minute_marker)
def queue_processor_loop(stop_event=RUNTIME_STOP):
    while not stop_event.wait(0.2):
        if not has_cron_queue() or not agent_lock.acquire(blocking=False): continue
        try:
            if has_cron_queue(): run_agent_turn_locked()
        finally: agent_lock.release()
fired = consume_cron_queue()
for job in fired:
    messages.append({"role":"user","content":f"[Scheduled] {job.prompt}"})
```

**和 Stage 3 的关系**：把"时间触发"作为一等公民接入 agent——让 agent 能自主在指定时刻启动工作，而非只能被动等用户输入。这是 harness "常驻/自主"能力的一环。

**课堂练习 12.1**：设一个 `*/1 * * * *`（每分钟）的 cron，prompt 为"说一句你好"，观察它是否每分钟自动触发一轮对话（空闲时）。

---

### s13 · Agent Teams — 团队运行时与协作协议

**口号**：*"一个 Agent 装不下整项工作时，就让队友分头完成。"*（持久队友、共享任务认领、可选 worktree 与协作协议）

**大白话**：重构整个后端这种活适合并行，但用户只说目标、不设计团队。像一个大项目，主管（Lead）把任务拆好，几个同事（Teammate）各认领一块并行干，干完把结果交回主管，空闲的同事主动接下一块。还要避免两人改同一份文件冲突——所以每人用独立的 Git 分支工作目录。

**核心机制**：

- **Lead**：负责用户对话、提团队方案、等确认（`spawn_teammate` 前先确认）。
- **队友（TeammateRuntime）**：持久线程，`WORK → IDLE → WORK`，跨任务保留上下文；`MessageBus` 用 `.mailboxes/<name>.jsonl` 传消息/结果/控制事件（不在模型上下文内共享 messages）。
- **task-bound worktree**：`Task.worktree` 可选字段；`create_worktree(name, task_id)` 绑独立 Git 分支+目录；所有文件工具从该 cwd 读（只分离工作目录，不是沙箱）。
- **原子认领**：`claim_task` 在锁内改 owner 并绑 cwd，多个队友同时发现同一候选只有一个能推进到 in_progress。
- **类型化协议**：`request_id` 做关机握手与计划审批闸门（`plan_gates`：未批准时锁住 bash/write/edit）。

**关键代码**（节自 `s13_agent_teams/code.py`）：

```python
class MessageBus:                       # 线程安全、破坏性读的文件邮箱
    def send(self, from_agent, to_agent, content, msg_type="message", metadata=None):
        msg = {"from":from_agent,"to":to_agent,"content":content,"type":msg_type,...}
        with self._changed:
            with self._path(to_agent).open("a",encoding="utf-8") as h:
                h.write(json.dumps(msg,ensure_ascii=True)+"\n")
            self._changed.notify_all()
    def wait_for_messages(self, agent, timeout=None):
        with self._changed:
            while not self.peek(agent): self._changed.wait(remaining)
            return self._read_unlocked(agent)
def claim_task(task_id, owner="agent") -> str:        # 原子认领 + 绑定 cwd
    with task_store_lock():
        task = load_task(task_id)
        if task.status != "pending" or task.owner: return "unavailable"
        if not can_start(task_id): return "Blocked by: ..."
        cwd, error = task_worktree_cwd(task)
        if error: return f"Cannot claim {task_id}: {error}"
        task.owner = owner; task.status = "in_progress"; save_task(task)
        teammate_assignments[owner] = {"task_id":task.id,"cwd":cwd}
        advance_assignment_version(owner)
def spawn_teammate_thread(name, role, prompt, task_id=None, require_plan=False):
    runtime = TeammateRuntime(name, role, prompt, task_id, require_plan)
    threading.Thread(target=runtime.run, daemon=True).start()
```

**和 Stage 3 的关系**：这是 Stage 4（多 Agent 协调）的雏形，但属于 Stage 3 深度学习素材里"harness 能装多强能力"的展示。身份持久化 + 消息总线隔离 + 任务原子认领 + worktree 隔离 + 类型化控制协议，是"一个 agent 装不下整项工作"的完整工程答案。

**课堂练习 13.1**：建 3 个互相独立的任务，spawn 2 个队友，观察它们如何通过 `claim_task` 抢活、用 `MessageBus` 交结果、空闲时发 `idle_notification`。

---

### s14 · MCP Tools — 发现并调用外部工具

**口号**：*"通过统一发现与调用协议，在运行时连接外部服务"*

**大白话**：基础工具都写死在代码里。每接一个外部系统（文档库、部署平台）都要手写工具定义+调用逻辑，对方一改你也要改。像每个外部公司都要你专门写一套对接程序。解法：定一个统一插头标准（MCP），对方按标准做插头，你这头只要"发现插头→插上→调用"，不用改自己的代码。

**核心机制**：

- `MCPClient`：`register(tool_defs, handlers)` 存发现结果，`call_tool(name, args)` 执行（错误返回给模型，不结束 Loop）。
- `connect_mcp(name)`：连接 server 并发现工具；模型先只看到 `connect_mcp`，连接后下一轮出现 `mcp__docs__search` 等。
- `assemble_tool_pool()`：每轮把基础工具 + 已连接 MCP 工具合并（工具名统一 `mcp__{server}__{tool}`，清洗并查冲突、限 64 字符）。
- `MCP_HOST_POLICY`：权限来自**宿主**配置（非 server 自述的 `readOnlyHint`），未配置默认 `confirm`。

**关键代码**（节自 `s14_mcp_plugin/code.py`）：

```python
MCP_HOST_POLICY = {
    ("docs","search"): "allow", ("docs","get_version"): "allow",
    ("deploy","status"): "allow", ("deploy","trigger"): "confirm",
}
_DISALLOWED_CHARS = re.compile(r"[^a-zA-Z0-9_-]")
def normalize_mcp_name(name): return _DISALLOWED_CHARS.sub("_", name)
class MCPClient:
    def register(self, tool_defs, handlers):
        self.tools = list(tool_defs); self._handlers = dict(handlers)
    def call_tool(self, tool_name, args):
        handler = self._handlers.get(tool_name)
        if not handler: return f"MCP error: unknown tool '{tool_name}'"
        try: return str(handler(**args))
        except Exception as e: return f"MCP error: {type(e).__name__}: {e}"
def assemble_tool_pool():
    tools = list(BUILTIN_TOOLS); handlers = dict(BUILTIN_HANDLERS)
    for server_name, server in mcp_clients.items():
        safe_server = normalize_mcp_name(server_name)
        for tool_def in server.tools:
            safe_tool = normalize_mcp_name(tool_def["name"])
            prefixed = f"mcp__{safe_server}__{safe_tool}"   # 前缀规则
            tools.append({"name":prefixed,"description":...,"input_schema":...})
            handlers[prefixed] = lambda *, client=server, tool=tool_def["name"], **kw: client.call_tool(tool, kw)
    return tools, handlers
```

**和 Stage 3 的关系**：MCP 是 Stage 3 推荐系统表里 Claude Code Docs 的核心能力之一。它把 s02 的分发字典**泛化为运行时发现的动态工具池**——agent 接入真实世界系统的标准方式。

**课堂练习 14.1**：起一个最小的 MCP server（暴露一个 `add` 工具），`connect_mcp` 后看工具池是否出现 `mcp__myserver__add`，并让模型调用它。

---

### s15 · Integrated Harness — 多种机制，一个循环

**口号**：*"多种机制，一个循环"*（工具、权限、记忆、任务、团队、插件都挂在同一个 `while True` 上）

**大白话**：前面 14 课各自独立演示。现在把它们装到同一台车里：踩油门（输入）后，依次经过权限检查、上下文压缩、记忆/技能/MCP 状态组装、模型决策、工具分发、后台/团队调度……所有零件围着同一个循环转，事件统一回同一段对话。这是"整车总装图"。

**核心机制**（不引入新机制，只做集成；循环仍是同一结构）：

- 单次 flow：用户输入 → `UserPromptSubmit` hooks → cron/background 通知注入 → context compact → memory+skills+MCP 状态组装 system prompt → LLM → 有 `tool_use`? → `PreToolUse` hooks+permission → 分发(内置/MCP/background) → `PostToolUse` hooks → 结果回 messages → 下一轮；无 tool_use → `Stop` hooks。
- 内置工具池 **26 个**：bash/read/write/edit/glob + todo_write/task/load_skill/compact + 6 个 task 工具 + 3 个 cron + spawn/list/send/request_shutdown/request_plan/review_plan + create_worktree + connect_mcp。
- 两层计划并存：`todo_write`（会话内轻量）+ task graph（跨会话、可依赖、可认领）。
- 两种 delegation：`task`（一次性 subagent，隔离上下文只返摘要）vs `spawn_teammate`（持久队友线程，WORK→result→IDLE）。
- 复用 s09 Memory、s08 压缩管线，包一层错误恢复（429/529 退避、`max_tokens` 升级、prompt too long → reactive compact）。
- 权限不写死在工具行，而是 `PreToolUse` hook；MCP 权限用宿主白名单，文件越界直接拒。

**关键代码**（据 `s15_integrated_harness/README.md`）：

```python
blocked = trigger_hooks("PreToolUse", block)          # 权限作为 hook
if blocked: results.append(tool_result(block.id, blocked)); continue
# 压缩管线（每轮 LLM 前）
tool_result_budget → snip_compact → micro_compact → compact_history
# 后台 bash：显式标记才进后台
should_run_background → start_background_task → placeholder tool_result
# MCP：connect_mcp 后下一轮工具池出现 mcp__docs__search
assemble_tool_pool()  # BUILTIN_TOOLS + connected MCP tools
```

**和 Stage 3 的关系**：这是前面 14 章的**总装图**，也是 Stage 3 那个"可调试 harness demo"在真实项目里的样子。读完这一节，你就看懂了 Claude Code 这类产品内部全貌。

**课堂练习 15.1**：画一张 s15 的 flow 图，标出 s01–s14 哪个机制出现在哪个位置。这就是一份现成的"harness 架构图"。

---

### s16 · Workflow Runtime — 模型决定单步，脚本决定编排

**口号**：*"一次 tool_use，跑完一整套编排"*（`Workflow` 工具启动可恢复的脚本运行时）

**大白话**：s01–s15 每轮都由模型决定下一步，适合"下一步取决于上一步发现"的任务。但有些流程是固定的（如代码审查：多维度并行检查 + 逐条验证 + 合并去重），需要并行、稳定结果结构、可恢复。像流水线工厂：产品怎么造是固定图纸（脚本），但每个工位仍由工人（agent）完成。而且图纸有存档，做到一半停电，来电能从断点接着做。

**核心机制**：

- `Workflow` 工具加入 s15 工具池：模型只提供 name/args/可选 `resume_from_run_id`，不提交代码。
- 编排原语（脚本收 `ExecutionState`，不直接读写文件/跑 shell）：`agent(prompt,{schema,label,phase})`、`parallel(thunks)`（等齐屏障）、`pipeline(items,*stages)`（每个 item 独立分阶段、不等齐）、`phase(title)`、`log(msg)`、`workflow(name,args)`（嵌套一层）。
- 结构化输出：`agent({schema})` 要求子 agent 只返匹配 schema 的 JSON，校验失败重试一次。
- 存储：`.runtime/` 下快照 + journal（`<runId>.journal.jsonl`）+ lock；`resume_from_run_id` 按**稳定调用键**（基于 kind/label/prompt/schema 的哈希）命中缓存，未改的 `agent()` 直接用 journal 结果。
- 生命周期事件流：`task_started` → `task_progress`* → `task_notification`。

**关键代码**（据 `s16_workflow_runtime/README.md`）：

```python
async def pipeline(self, items, *stages):     # pipeline：每个 item 独立跑完所有 stage，不等齐
    async def run_item(item, idx):
        value = item
        for stage in stages: value = await stage(value, item, idx)
        return value
    return await asyncio.gather(*[run_item(it,i) for i,it in enumerate(items)])
def key(self, kind, label, prompt, schema):    # 稳定调用键：不依赖并发完成顺序，用内容哈希
    basis = f"{kind}|{label}|{prompt}|{json.dumps(schema, sort_keys=True)}"
    return f"{kind}-{_stable_hash(basis) % 10**10:010d}"
# agent() 内部：journal 命中则直接用缓存，不重跑
cached = self.journal.cached(key)
if cached is not MISS:
    self.task.progress_event("workflow_agent", label=label, status="cached"); return cached
class WorkflowJournal:
    def record(self, key, value):
        self._f.write(json.dumps({"key":key,"value":value})+"\n"); self._f.flush()
```

**和 Stage 3 的关系**：把"固定多 Agent 脚本"从对话历史里抽出来写成可恢复代码，用 journal + 稳定键实现断点续跑。这是复杂可重复编排的工程范式，也对应 Stage 3 推荐表里 LangGraph 那类"状态图、可恢复执行"的思路。

**课堂练习 16.1**：写一个对 10 个文件并行做"lint + 格式化"的 workflow，中途杀掉进程，用 `resume_from_run_id` 重启，观察已完成的 `agent()` 是否从 journal 直接读缓存而不重跑。

---

### s17 · Goal Loop — 模型提出停止，独立判断器决定是否继续

**口号**：*"模型不再调用工具，只代表这一轮想停；目标是否完成，再交给一个独立判断器。"*

**大白话**：s01 以来退出条件 = 模型不再调工具就返回。但"修到测试全过"这种任务，模型可能只完成一半就觉得"行了"。像一个员工说"我做完了"，但主管得自己核对验收标准才算数。解法：设一个明确完成条件，每次模型想停，都让一个**独立、没工具的评审**对照条件打分，没达标就把它打回去继续干。

**核心机制**：

- `/goal <完成条件>`：会话级 Stop hook，保存条件并立即作为本轮任务交主模型。
- 独立判断器 `PromptGoalEvaluator`：**另一次无工具模型调用**，只读对话、判断完成条件（看到 condition + 对话 + 工具结果）。返回 `{ok, reason, impossible}`。
- `GoalController.evaluate_after_turn`：无活跃目标→`allow`（同 s01）；有后台运行→`defer`；`ok`→`achieved`；`impossible`→`failed`；否则 `block`（把 reason 追加回 `messages` 并 `continue`）。`block_cap` 限制连续阻止次数。
- 单会话仅一个活跃 Goal；`/goal` 查看/替换/清除；`GoalController.restore()` 可恢复。

**关键代码**（节自 `s17_goal_loop/code.py`）：

```python
@dataclass
class GoalState:
    condition: str; iterations: int; set_at: float
    tokens_at_start: int; last_reason: str | None = None
class PromptGoalEvaluator:           # 独立、无工具的评估模型
    def _evaluate_sync(self, condition, messages) -> GoalEvaluation:
        conversation = transcript_text(messages)
        response = self.client.messages.create(
            model=self.model,
            system="You are an independent completion evaluator. You have no tools. ...",
            messages=[{"role":"user","content":prompt}], max_tokens=self.max_tokens)
        return GoalEvaluation(**_parse_json_object(_extract_text(response.content)))
async def evaluate_after_turn(self, messages, background_running=False) -> StopDecision:
    if self.active is None: return StopDecision("allow")
    if background_running: return StopDecision("defer","background work is still running")
    evaluation = await self.evaluator.evaluate(self.active.condition, messages)
    self.active.iterations += 1; self.active.last_reason = evaluation.reason
    if evaluation.ok: self.active = None; return StopDecision("achieved", evaluation.reason)
    if evaluation.impossible: self.active = None; return StopDecision("failed", evaluation.reason)
    self.consecutive_blocks += 1
    if self.consecutive_blocks > self.block_cap:
        return StopDecision("limit", f"Stop hook blocked {self.block_cap} consecutive turns")
    return StopDecision("block", evaluation.reason)
# 接入点：原退出位置
decision = await self.goal.evaluate_after_turn(self.messages, background_running=self.background_running())
if decision.action == "block":
    self.messages.append({"role":"user","content":"[Goal still active]\n..."}); continue
return SessionResult(text=text, status=decision.action)
```

**和 Stage 3 的关系**：把"完成"定义为**可独立验证的判断**而非模型自述，用分离的判断器 + 明确完成条件保证 agent 真正干到目标达成。这是"自主持续执行"的收口机制，和 s16（管"一批工作怎样执行"）互补——s17 管"整件事是否完成"。

**课堂练习 17.1**：设 `/goal 所有测试通过`，让模型改代码后自认为完成，观察独立判断器是否因"还有测试失败"而 `block` 把它打回继续。

---

## 3. 把 17 课映射回 Stage 3 的 5 个学习动作（确认你掌握）

| Stage 3 动作               | 在 learn-claude-code 里对应哪课                                                       | 你怎么验证                                              |
| ------------------------ | ------------------------------------------------------------------------------- | -------------------------------------------------- |
| 动作1：读懂 harness 目录结构      | 看 s01–s17 的文件夹命名 = 能力堆叠顺序                                                       | 能说出"为什么 s03 在 s02 之后、s15 在最后"                      |
| 动作2：找出 5 大模块             | loop=s01, tool registry=s02, permission=s03, session/memory=s09, compaction=s08 | 能指出每模块在代码里的函数名                                     |
| 动作3：跑最小示例+加工具            | s01 跑通、s02 加 `@tool`                                                            | 你已跑过 `stage3_harness_demo`，再加一个 `reverse` 工具       |
| 动作4：观察一次完整 trace         | s01 的 loop 每一步 + s04 hooks 的 PreToolUse/PostToolUse                             | 你已有 `traces/*.trace.jsonl`，可对照本文件 s01/s03/s04 看每一步 |
| 动作5：裸 loop vs harness 对比 | 差异在 s03(权限)/s08(压缩)/s09(记忆)/s13(团队)                                             | 用本文件 §1 的对比思路写一份差异表                                |

**一句话确认**：读完 s01–s17 + 跑通 demo + 做完每课"课堂练习"，你就不是"知道 Stage 3 表面要求"，而是真正理解了"一辆生产级 agent 车"是怎么一层层造出来的——这正是 Stage 3 的灵魂。

---

## 4. 下一步（可选，非必须）

- 想要**中文逐字教程**：`learn-claude-code` 每节课都有 `README.zh.md`，直接 `git clone` 后顺着 s01 读到 s17。
- 想接**真实模型跑**：`pip install -r requirements.txt` + 配 `ANTHROPIC_API_KEY`，`python s01_agent_loop/code.py` 起步。
- 想补 **Stage 4（多 Agent）**：s13 Teams 已经是雏形，再去读仓库 Stage 4 说明。
- 想看 **claw0（网关/常驻视角）**：它是姐妹项目，讲 session/channel/gateway/heartbeat，和 learn-claude-code 互补。

> **总纲再念一遍**：Agency 来自模型，harness 给 agency 一个落地的载体。把车造好，模型会替你把剩下的事做完。
