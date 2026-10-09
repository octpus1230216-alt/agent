# learn-claude-code 研究笔记（s01–s17 全 17 课）

> 来源仓库：https://github.com/shareAI-lab/learn-claude-code
> 研究方法：通过 WebFetch 读取各节课 `README.zh.md`（base URL `https://raw.githubusercontent.com/shareAI-lab/learn-claude-code/main/`），并对 9 个核心章节额外读取 `code.py` 提取关键代码模式。
> 覆盖确认：17 节全部覆盖。其中 **s05 的 README.zh.md 返回 404**，已用英文 `README.md` 兜底，下文 s05 处已标注。其余 16 节均读取中文版成功。
> 说明：本报告用中文撰写，代码与术语保留英文原文。

---

## 全文主题脉络（一页速览）

整个课程是一条主线：**在一个不变的 `while True` Agent Loop 之上，逐层叠加机制**。

| 层 | 章节 | 解决的本质问题 |
|----|------|----------------|
| 内核 | s01 Agent Loop | 模型只输出文本，怎么让它"持续行动" |
| 工具 | s02 Tool Use, s03 Permission, s04 Hooks | 扩展模型能触达的边界 + 边界上的安全与扩展点 |
| 规划 | s05 TodoWrite, s06 Subagent | 复杂任务先想清楚、大任务隔离上下文 |
| 知识 | s07 Skill Loading, s08 Context Compact, s09 Memory | 上下文与长期知识的加载、压缩、持久化 |
| 协调 | s10 Task System, s11 Background, s12 Cron, s13 Teams | 多步骤/异步/定时/多 Agent 的编排 |
| 接入 | s14 MCP, s15 Integrated | 外挂工具与全机制合并 |
| 编排 | s16 Workflow | 固定多 Agent 脚本的可恢复编排 |
| 目标 | s17 Goal Loop | 模型说"停"不等于目标达成 |

---

## s01 · Agent Loop — 一个循环就够了

**Motto**：*"One loop & Bash is all you need"*（一个工具 + 一个循环 = 一个 Agent）

**核心问题**：模型输出一条 bash 命令后不会自己执行、也不会看结果继续推理；把"执行→粘贴回对话框→再执行"的人类中间层自动化，就是最小 agent。

**核心机制/抽象**：
- `while True` 循环：调用 LLM → 检查响应中是否有 `tool_use` block → 有则执行并把 `tool_result` 追加回 `messages` → 无则退出。
- 模型负责"决策"（调不调、调哪个工具），harness 负责"执行"（调用工具、把结果作为新消息追加）。
- 只有真实存在的 `tool_use` block 才进入执行阶段（绝不追加空的工具结果消息）。

**关键代码模式**（节自 `s01_agent_loop/code.py`）：

```python
def agent_loop(messages: list):
    while True:
        response = client.messages.create(
            model=MODEL, system=SYSTEM, messages=messages,
            tools=TOOLS, max_tokens=8000,
        )
        messages.append({"role": "assistant", "content": response.content})

        tool_calls = [
            block for block in response.content if block.type == "tool_use"
        ]
        if not tool_calls:
            return  # 模型没调工具 → 循环结束

        results = []
        for block in tool_calls:
            output = run_bash(block.input["command"])
            results.append({
                "type": "tool_result",
                "tool_use_id": block.id,
                "content": output,
            })
        messages.append({"role": "user", "content": results})  # 结果喂回 → 继续
```

**Build-on 关系**：无。这是地基，后续 16 章全部叠加在它之上，循环本身始终不变。

**对理解现代 agent harness 的贡献**：揭示了 agent 的本质不是模型，而是"把模型输出当动作、把动作结果当输入"的闭环。三十多行即可运行，是理解一切后续机制的锚点。

---

## s02 · Tool Use — 多加一个工具，只加一行

**Motto**：*"加一个工具, 只加一个 handler"*（循环不用动，新工具注册进 dispatch map 就行）

**核心问题**：只有一个 bash 工具时，读文件要 `cat`、写文件要 `echo >`、找文件要 `find`，既浪费 token 又易错；需要把"模型想做的事"映射为"专用工具"。

**核心机制/抽象**：
- `TOOLS`：告诉模型"我能做什么"的 JSON schema 数组。
- `TOOL_HANDLERS`：工具名 → 处理函数的 dispatch map（分发字典）。
- 加工具 = 在 `TOOLS` 加一条描述 + 在 `TOOL_HANDLERS` 加一行映射；`while True` 循环一行不改。
- `safe_path()`：file tools 的 workspace 越界校验。
- 多个 `tool_use` 按 `response.content` 原始顺序逐个执行。

**关键代码模式**（节自 `s02_tool_use/code.py`）：

```python
TOOL_HANDLERS = {
    "bash": run_bash, "read_file": run_read,
    "write_file": run_write, "edit_file": run_edit, "glob": run_glob,
}

# 循环里只改了一行——从硬编码 run_bash 变成查表：
for block in tool_calls:
    handler = TOOL_HANDLERS.get(block.name)    # 查表分发
    output = handler(**block.input) if handler else f"Unknown: {block.name}"
    results.append({"type": "tool_result", "tool_use_id": block.id, "content": output})

def safe_path(p: str) -> Path:
    path = (WORKDIR / p).resolve()
    if not path.is_relative_to(WORKDIR):
        raise ValueError(f"Path escapes workspace: {p}")
    return path
```

**Build-on 关系**：叠加于 s01 的循环；dispatch map 是后续 s03 权限、s04 hook、s14 MCP 工具池的共同骨架。

**对理解现代 agent harness 的贡献**：工具是"模型与真实世界的接口"，dispatch map 是把接口变成可扩展插件系统的关键抽象——决定了 agent 的能力边界可以被任意扩充而不动内核。

---

## s03 · Permission — 执行前做权限判断

**Motto**：*"工具执行前先做权限判断"*（权限管线决定哪些操作需要审批）

**核心问题**：bash 不受 `safe_path` 限制，"清理一下项目"可能执行 `rm -rf /`；安全边界必须由代码负责，且判断发生在工具执行之前。

**核心机制/抽象**（三道闸门，硬拒绝优先、软询问次之、都没命中则放行）：
- Gate 1 `DENY_LIST`：永远禁止的操作（`rm -rf /`、`sudo`、`shutdown`…），字符串匹配。
- Gate 2 `PERMISSION_RULES`：上下文相关规则（写工作区外、破坏性命令 `rm`/`chmod 777`…）。
- Gate 3 `ask_user`：规则命中后暂停等用户确认。
- `check_permission(block)`：三道闸门串联，插在工具执行前（循环只加一行）。

**关键代码模式**（节自 `s03_permission/code.py`）：

```python
DENY_LIST = ["rm -rf /", "sudo", "shutdown", "reboot", "mkfs", "dd if=", "> /dev/sda"]

PERMISSION_RULES = [
    {"tools": ["read_file", "write_file", "edit_file"],
     "check": lambda args: not (WORKDIR / args.get("path", "")).resolve().is_relative_to(WORKDIR),
     "message": "Writing outside workspace"},
    {"tools": ["bash"],
     "check": lambda args: contains_destructive_command(args.get("command", ""))
                           or any(kw in args.get("command", "") for kw in ["rm ", "> /etc/", "chmod 777"]),
     "message": "Potentially destructive command"},
]

def check_permission(block) -> bool:
    if block.name == "bash":
        reason = check_deny_list(block.input.get("command", ""))
        if reason:
            return False                       # Gate 1
    reason = check_rules(block.name, block.input)
    if reason:
        decision = ask_user(block.name, block.input, reason)
        if decision == "deny":
            return False                       # Gate 2 + 3
    return True

# 循环中仅插入：
for block in tool_calls:
    if not check_permission(block):
        results.append({... "content": "Permission denied."}); continue
    output = TOOL_HANDLERS[block.name](**block.input)
```

**Build-on 关系**：基于 s02 的多工具；权限逻辑在 s04 被重构成 PreToolUse hook。

**对理解现代 agent harness 的贡献**：agent 必须有权限边界，否则"行动能力"会变成"破坏能力"。三道闸门的层次（硬编码拒绝 → 规则 → 交互确认）是最小可用的安全模型。

---

## s04 · Hooks — 挂在循环上，不写进循环里

**Motto**：*"挂在循环上, 不写进循环里"*（hook 在工具执行前后注入扩展逻辑）

**核心问题**：s03 每加一个检查（日志、自动 git add）都要改 `agent_loop`，循环很快膨胀、认不出原貌；扩展应挂在外面，循环保持干净稳定。

**核心机制/抽象**（四个事件，覆盖一个完整 agent cycle）：
- `UserPromptSubmit`：用户输入后、进 LLM 前（输入验证/注入上下文）。
- `PreToolUse`：工具执行前（权限、日志）。返回非 `None` → 阻止本次执行。
- `PostToolUse`：工具执行后（副作用如自动 git add、大输出告警）。
- `Stop`：循环即将退出时（统计、清理、可强制继续）。
- `HOOKS` 注册表 + `register_hook()` / `trigger_hooks()`；循环只调 `trigger_hooks`，具体逻辑全在回调里。

**关键代码模式**（节自 `s04_hooks/code.py` 思路，原文以 README 为准）：

```python
HOOKS = {"UserPromptSubmit": [], "PreToolUse": [], "PostToolUse": [], "Stop": []}

def register_hook(event: str, callback):
    HOOKS[event].append(callback)

def trigger_hooks(event: str, *args):
    for callback in HOOKS[event]:
        result = callback(*args)
        if result is not None:   # 返回值 ≠ None → hook 说"停"
            return result
    return None

# 循环只改一处：s03 的 check_permission(block) 变为：
blocked = trigger_hooks("PreToolUse", block)
if blocked:
    results.append({"type": "tool_result", "tool_use_id": block.id, "content": str(blocked)})
    continue
```

> 注：s04 的 `code.py` 未在本研究中单独抓取（仅 README 已含完整机制说明）。核心模式已在上文呈现。

**Build-on 关系**：把 s03 硬编码的 `check_permission` 抽象为 PreToolUse hook；是 s10/s13/s14/s15 中"权限与扩展都挂在 hook 点"的源头。

**对理解现代 agent harness 的贡献**：扩展点（extension points）思想——让 harness 内核稳定，把日志、审计、权限、副作用都变成可插拔的回调，是工程可维护性的关键。

---

## s05 · TodoWrite — 没有计划的 Agent 会跑偏

> ⚠️ 读取状态：本文节 `README.zh.md` 在仓库返回 404，已用英文版 `README.md` 兜底（内容完整，以下据英文版整理）。

**Motto**：*"An agent without a plan goes wherever the wind blows"*（先列步骤，再执行）

**核心问题**：给 Agent 一个复杂任务，它做到一半会忘记最初目标（如"重命名成 snake_case"被测试失败吸走注意力），长对话里 system prompt 的影响力被工具结果稀释。

**核心机制/抽象**：
- `TodoManager`：进程内清单，校验更新、渲染状态（`[ ] pending` / `[>] in_progress` / `[x] completed`）。
- `todo_write` 工具：经同一 `TOOL_HANDLERS[block.name]` 分发；只更新计划状态，不增加执行能力（新增的是"规划能力"）。
- 提醒机制：连续 3 轮没有 `todo_write` 时，在结果中注入 `<reminder>Update your todos.</reminder>`。
- 约束：最多 20 项，每项非空 `content`，且同一时刻仅一个 `in_progress`。

**关键代码模式**（据 `s05_todo_write/README.md`）：

```python
class TodoManager:
    def __init__(self): self.items = []
    def update(self, todos):
        validated = [...]   # 解析+校验后再替换当前列表
        self.items = validated
        return self.render()
    def render(self): ...   # [ ]/[>]/[x]

TOOL_HANDLERS["todo_write"] = run_todo_write

# 提醒注入
rounds_since_todo = 0 if used_todo else rounds_since_todo + 1
if rounds_since_todo >= 3:
    results.append({"type": "text", "text": "<reminder>Update your todos.</reminder>"})
    rounds_since_todo = 0
```

**Build-on 关系**：复用 s04 的工具分发与 hooks；清单是 s10 持久化 Task System 的轻量前身。

**对理解现代 agent harness 的贡献**：显式规划（planning）能对抗长任务中的"目标漂移"，是 Agent 从"会执行"到"靠得住"的关键一步。

---

## s06 · Subagent — 给子任务一段独立上下文

**Motto**：*Subagent 从全新的 `messages[]` 开始。最终文本返回父循环，中间对话不进入父上下文。*

**核心问题**：Agent 为追调用链读了很多文件，中间细节在任务完成后仍占父循环 `messages[]`，浪费上下文；需要把子任务隔离到另一段对话。

**核心机制/抽象**：
- `run_subagent(prompt)`：同步运行一个全新 `messages[]` 的嵌套 Agent Loop，结束后只把**最终文本**作为父对话的 `tool_result` 返回。
- 隔离的是消息，不是进程/文件系统：父子共享 `WORKDIR`，写文件互相可见；子 Agent 用 `SUB_TOOLS`（无 `task` 工具，只允许一层委派）。
- `task` 工具经 handler map 分发，与调用其他工具一致。

**关键代码模式**（据 `s06_subagent/README.md`）：

```python
SUB_TOOLS = list(BASE_TOOLS)  # 无 task 工具

def run_subagent(prompt: str) -> str:
    messages = [{"role": "user", "content": prompt}]
    for _ in range(30):
        response = client.messages.create(model=MODEL, system=SUB_SYSTEM,
            messages=messages, tools=SUB_TOOLS, max_tokens=8000)
        messages.append({"role": "assistant", "content": response.content})
        tool_calls = [b for b in response.content if b.type == "tool_use"]
        if not tool_calls:
            return extract_text(response.content) or "(no summary)"
        results = []
        for block in tool_calls:
            output = execute_tool(block, SUB_HANDLERS)
            results.append({... "content": output})
        messages.append({"role": "user", "content": results})
    return "Subagent stopped after 30 turns without a final answer."

TOOLS = [*BASE_TOOLS, TASK_TOOL]
TOOL_HANDLERS = {**BASE_HANDLERS, "task": run_subagent}
```

**Build-on 关系**：基于 s04 内核 + 工具分发；是 s13 持久队友、s16 Workflow 子 agent 的"上下文隔离"原型。

**对理解现代 agent harness 的贡献**：上下文隔离是并行/委派的基础——一个 Agent 的中间过程不该污染另一个 Agent 的推理。

---

## s07 · Skill Loading — 用到时再加载

**Motto**：*system prompt 保存技能目录；`load_skill` 返回完整的 `SKILL.md`。*

**核心问题**：把 React 规范/SQL 指南/API 文档全塞进 system prompt，会与当前任务无关却仍占 token；知识应"按需注入"而非预载。

**核心机制/抽象**：
- `SkillLoader.scan()`：启动时扫描 `skills/*/SKILL.md`，解析 YAML frontmatter 的 `name`/`description` 建目录。
- system prompt 只放**技能名称和描述目录**（`catalog()`）；模型需要完整说明时调 `load_skill(name)`。
- `load_skill` 返回完整 `SKILL.md`，作为 `tool_result` 追加（和读文件一样自然）。

**关键代码模式**（据 `s07_skill_loading/README.md`）：

```python
class SkillLoader:
    def scan(self):
        for manifest in sorted(self.skills_dir.glob("*/SKILL.md")):
            content = manifest.read_text(encoding="utf-8")
            metadata, body = self.parse_frontmatter(content)
            self.skills[name] = {"name": name, "description": description, "content": content}

def build_system_prompt() -> str:
    return (f"You are a coding agent at {WORKDIR}...\n\n"
            f"Skills available:\n{SKILL_LOADER.catalog()}\n\n"
            "Use load_skill to read the full instructions when a skill applies.")

def load(self, name: str) -> str:
    skill = self.skills.get(name)
    return skill["content"] if skill else f"Error: Unknown skill '{name}'..."
```

**Build-on 关系**：基于 s04 内核；与 s08 压缩、s09 记忆共同构成"上下文预算管理"主题；s15 复用其"目录 + 按需加载"模式。

**对理解现代 agent harness 的贡献**：延迟加载（lazy loading）是控制上下文成本的通用范式——只把任务相关的知识送进上下文。

---

## s08 · Context Compact — 上下文总会满，先整理，再总结

**Motto**：*"上下文总会满，要有办法腾地方。"*（四步压缩，低成本操作优先）

**核心问题**：工具结果（长文件、测试日志、多文件搜索）会迅速撑爆上下文；直接让模型总结全历史既丢细节又费一次调用。需要在保留"目标/约束/进行中工作"的前提下腾空间。

**核心机制/抽象**（固定顺序的四步管线，只在必要时进入有损摘要）：
1. `tool_result_budget`：> 200_000 字符时，从最大结果开始持久化超长结果（> 30_000 字符）到 `.task_outputs/tool-results/<id>.txt`，上下文保留路径 + 前 2000 字符预览。
2. `snip_compact`：消息 > 50 条时，归档完整历史到 `.transcripts/`，保留前 3 + 后 46 条 + 一个归档标记（保护 `tool_use`/`tool_result` 配对不被截断）。
3. `micro_compact`：> `CONTEXT_CHAR_LIMIT`(50000) 时，保留最近 3 条已读结果，逐条缩短更早 > 120 字符的结果为可恢复路径，直到降到阈值 80%。
4. `compact_history`：仍超限 → 写 transcript + 让模型生成"只含事实"的状态摘要，用 `[Compacted]` 消息替换历史（且 `active_request` 与摘要明确分离）。

另含 `fit_tool_results`（未读新结果过大时先保预览）、`reactive_compact`（API 报 `prompt_too_long` 时的兜底，保留最近 5 条、`MAX_REACTIVE_RETRIES=1`）、`compact` 工具（模型主动请求压缩，必须先执行完本批次工具再摘要，避免孤立 tool_result）。

**关键代码模式**（节自 `s08_context_compact/code.py`）：

```python
class ContextCompactor:
    CONTEXT_CHAR_LIMIT = 50000
    TOOL_RESULT_BATCH_CHAR_LIMIT = 200000
    LARGE_RESULT_CHAR_LIMIT = 30000
    KEEP_RECENT_RESULTS = 3

    @staticmethod
    def estimate_chars(messages): return len(json.dumps(messages, default=str, ensure_ascii=False))

    def tool_result_budget(self, messages, max_chars=None):
        # 对最后一条 user 消息中的 tool_result 按大小排序，超 LARGE 的持久化
        ...
        block["content"] = self.persist_large_output(block.get("tool_use_id"), output)

    def snip_compact(self, messages, max_messages=50):
        # 归档中段消息，保留 head(3)+tail(46)+marker，保护 tool_use/tool_result 配对
        ...

    def micro_compact(self, messages, target_chars=None):
        # 旧 tool_result 持久化为 "[Earlier tool result saved at <path>]"
        ...

    def compact_history(self, messages, active_request):
        transcript = self.write_transcript(messages)
        summary = self.summarize_history(messages)
        return [self.summary_message("Compacted", active_request, summary, transcript)]

    def prepare(self, messages, active_request):
        messages = self.tool_result_budget(messages)      # Step 1
        messages = self.snip_compact(messages)            # Step 2
        if self.estimate_chars(messages) > self.CONTEXT_CHAR_LIMIT:
            target = int(self.CONTEXT_CHAR_LIMIT * 0.8)
            messages = self.micro_compact(messages, target)            # Step 3a
            if self.estimate_chars(messages) > self.CONTEXT_CHAR_LIMIT:
                messages = self.fit_tool_results(messages, target)     # Step 3b
            if self.estimate_chars(messages) > self.CONTEXT_CHAR_LIMIT:
                messages = self.compact_history(messages, active_request)  # Step 3c
        return messages
```

**Build-on 关系**：与 s07 同属上下文预算管理；是会话内压缩，与 s09（跨会话记忆）边界清晰；s15 直接复用此管线。

**对理解现代 agent harness 的贡献**：可恢复、确定性的压缩策略（先无损转存、最后才摘要）是长任务能在有限窗口持续工作的工程核心。

---

## s09 · Memory — 让重要信息跨会话保留下来

**Motto**：*"把以后还会用到的信息留下来。"*（文件存储 + 索引 + 相关性选择 + 按需召回）

**核心问题**：新会话 `messages` 没有上次对话；用户偏好/项目背景/排查线索下次还要用。全写进 system prompt 会让无关内容持续占 token；需要"哪些值得存、当前任务取回哪几条"。

**核心机制/抽象**（四件事：存储 / 召回 / 提取 / 整理）：
- 存储：`.memory/<slug>.md` 每记忆一文件，YAML frontmatter 记 `name`/`description`/`type`；`type` ∈ `user`/`feedback`/`project`/`reference`。`MEMORY.md` 是索引，`write_memory_file` 后 `rebuild_memory_index()` 重建。
- 召回：`select_relevant_memories()` 用轻量模型调用选最多 5 条相关索引（失败回退关键词），`load_memories()` 再读正文并限长。
- 提取：`extract_memories()` 回合结束后从对话提取，候选须带 `scope`；仅 `persistent` 才跨会话保存，`current_task` 只约束本次。
- 整理：`consolidate_memories()` 达阈值（10 条）时合并去重，带快照、失败回滚。

**关键代码模式**（节自 `s09_memory/code.py`）：

```python
def write_memory_file(name, mem_type, description, body) -> Path:
    path = memory_path(f"{memory_slug(name)}.md")
    path.write_text(memory_document(name, mem_type, description, body), encoding="utf-8")
    rebuild_memory_index()                      # 写后重建索引
    return path

def select_relevant_memories(messages, max_items=5) -> list:
    # 轻量 LLM 选 catalog 索引；except → keyword_memory_selection
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

**Build-on 关系**：与 s08 边界清晰（s08 管会话内上下文预算，s09 管会话外可复用知识）；s15 直接复用其运行时。

**对理解现代 agent harness 的贡献**：选择性持久记忆 + 相关性召回，是 agent "越用越懂用户/项目"的关键，也是上下文压缩之外另一条知识管理路径。

---

## s10 · Task System — 从执行清单到可协调的任务状态

**Motto**：*"大目标拆成小任务, 排好序, 持久化"*（文件持久化的任务图，多 agent 协作的基础）

**核心问题**：s05 的 TodoWrite 只有内容+状态，记不住依赖关系（"数据库表完成才能写 API"）和分工（"谁负责"）；Harness 无法判断某任务当前能否开始。

**核心机制/抽象**：
- `Task` dataclass：`id`/`subject`/`description`/`status`(pending|in_progress|completed)/`owner`/`blockedBy`。
- `.tasks/{id}.json` 文件持久化；`TaskStore` 校验 ID、读写、做环检测。
- 两阶段建图：先 `create_task` 拿运行时 ID，再 `update_task(addBlockedBy=...)` 加边（因同级工具调用在产生结果前已定型，无法互相引用刚生成的 ID）。
- 状态机：`pending --claim--> in_progress --complete--> completed`；`complete_task` 对比完成前后 `can_start` 集合，输出被解锁的下游任务。

**关键代码模式**（节自 `s10_task_system/code.py`）：

```python
@dataclass
class Task:
    id: str
    subject: str
    description: str
    status: str
    owner: str | None
    blockedBy: list[str]

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
    ready_before = {t.id for t in list_tasks()
                    if t.status == "pending" and t.blockedBy and can_start(t.id)}
    task.status = "completed"; TASKS.save(task)
    unblocked = [t.subject for t in list_tasks()
                 if t.status == "pending" and t.blockedBy
                 and t.id not in ready_before and can_start(t.id)]
    return f"Completed {task.id} ({task.subject})" + (f"\nUnblocked: {', '.join(unblocked)}" if unblocked else "")
```

**Build-on 关系**：基于 s04 内核 + 工具分发（保留 5 基础工具、Permission、Hooks、统一 `execute_tool`）；任务图是两阶段契约，被 s13 团队、s15 集成、s16 Workflow 复用。

**对理解现代 agent harness 的贡献**：可持久化、可依赖、可认领的任务图，是"多步骤工作可恢复进度"和"多 Agent 分工"的数据基础。

---

## s11 · Background Tasks — 慢操作放后台

**Motto**：*"慢操作放后台，Agent Loop 继续运行"*（后台线程执行命令，后续轮次收集完成结果）

**核心问题**：安装依赖、跑完整测试可能持续几分钟；同步执行会卡住整个 Agent Loop。不依赖该命令的后续工作（看文档、整理文件）本可并行。

**核心机制/抽象**：
- `should_run_background(name, input)`：仅当 `bash` 且 `run_in_background is True` 才进后台（不再按 `install`/`build`/`test` 关键词猜）。
- `BackgroundManager`：daemon 线程执行，`start()` 立即返回 `bg_id`；完成/失败入 `_ready` 队列。
- `collect_background_results()` / `inject_background_results()`：后续轮次开始收集已完成结果，以 `<task_notification>` 通知注入（不复用原 `tool_use_id`，保证一个 tool_use 只对应一个 tool_result）。
- 后台结果不会主动唤醒 Agent，下次进循环时才收集。

**关键代码模式**（据 `s11_background_tasks/README.md`）：

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

# 循环中
while True:
    inject_background_results(messages)         # 先收集已完成的后台结果
    response = client.messages.create(...)
def execute_tool(block):
    if should_run_background(block.name, block.input):
        task_id = start_background_task(block)
        output = f"[Background task {task_id} started]"
    else:
        output = call_tool(block)
```

**Build-on 关系**：基于 s04 内核 + `execute_tool` 同步入口；是 s12 cron、s15 集成的前置异步能力。

**对理解现代 agent harness 的贡献**：异步执行 + 通知注入，让 agent 不被慢操作阻塞，是"并发利用时间"的基础。

---

## s12 · Cron Scheduler — 按时间启动任务

**Motto**：（本课 README 未单列 motto）*"按时间启动一轮 Agent Loop"*（调度线程匹配 cron，空闲时交付）

**核心问题**：s11 只解决命令"开始后怎么跑"，不记录"何时开始"；"每天 9 点跑测试"若只靠当前 Loop，用户每次到点都得重发 prompt。

**核心机制/抽象**：
- `CronJob` dataclass：`id`/`cron`/`prompt`/`recurring`/`durable`/`pending_delivery`/`last_fired`。
- 五段式 cron：`分 时 日 月 周`，支持 `*`/`*/N`/`N`/`N-M`/`N,M`；`validate_cron()` 预校验。
- `poll_due_jobs()`：调度线程每秒匹配，到期先存 `pending_delivery`+`last_fired` 再入内存队列（防同分钟重复）。
- `queue_processor_loop()`：用 `agent_lock` 避免与用户会话同时改会话，空闲时才起一轮 Agent Loop，把 `[Scheduled] <prompt>` 追加为 user 消息。
- 持久化：`durable=True` → `.scheduled_tasks.json`（临时文件 + `os.replace` 原子更新）；至少一次交付语义。

**关键代码模式**（据 `s12_cron_scheduler/README.md`）：

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
        finally:
            agent_lock.release()

# 交付：把到期任务作为新 user 消息
fired = consume_cron_queue()
for job in fired:
    messages.append({"role": "user", "content": f"[Scheduled] {job.prompt}"})
```

**Build-on 关系**：基于 s04 内核（保留 5 工具 + Hooks）；不带入 s11 后台命令（传递的是待执行 prompt）；s15 集成 cron 与 background。

**对理解现代 agent harness 的贡献**：把"时间触发"作为一等公民接入 agent——让 agent 能自主在指定时刻启动工作，而非只能被动等待用户输入。

---

## s13 · Agent Teams — 团队运行时与协作协议

**Motto**：*"一个 Agent 装不下整项工作时，就让队友分头完成。"*（持久队友、共享任务认领、可选 worktree 与协作协议）

**核心问题**：重构整个后端这类工作适合并行，但用户只说目标、不设计团队。Harness 要回答：谁判断并行有用、队友如何跨任务保留身份、结果如何自动回 Lead、空闲队友能否接手、并行冲突用哪个工作目录、关机与审批如何可追踪。

**核心机制/抽象**：
- **Lead**：负责用户对话、提团队方案、等确认（`spawn_teammate` 前先确认）。
- **队友（TeammateRuntime）**：持久线程，`WORK → IDLE → WORK`，跨任务保留上下文；`MessageBus` 用 `.mailboxes/<name>.jsonl` 传递消息/结果/控制事件（不在模型上下文内共享 messages）。
- **task-bound worktree**：`Task.worktree` 可选字段；`create_worktree(name, task_id)`（仅 Lead）绑定独立 Git 分支+目录；`task_worktree_cwd()` 解析队友 cwd，认领后写入 `teammate_assignments`，所有文件工具从该 cwd 读（worktree 只分离工作目录，不是沙箱）。
- **原子认领**：`claim_task` 在 `task_store_lock()` 内改 owner 并绑 cwd，多个队友同时发现同一候选只有一个能推进到 in_progress。
- **结果与 IDLE 两事件**：`result` 回答产出，`idle_notification` 回答能否接新活。
- **类型化协议**：`ProtocolState` + `request_id` 做关机握手与计划审批闸门（`plan_gates`：未批准时锁住 bash/write/edit）。

**关键代码模式**（节自 `s13_agent_teams/code.py`）：

```python
class MessageBus:                       # 线程安全、破坏性读的文件邮箱
    def send(self, from_agent, to_agent, content, msg_type="message", metadata=None):
        msg = {"from": from_agent, "to": to_agent, "content": content, "type": msg_type, ...}
        with self._changed:
            with self._path(to_agent).open("a", encoding="utf-8") as h:
                h.write(json.dumps(msg, ensure_ascii=True) + "\n")
            self._changed.notify_all()
    def wait_for_messages(self, agent, timeout=None):
        with self._changed:
            while not self.peek(agent):
                self._changed.wait(remaining)   # 阻塞等待唤醒
            return self._read_unlocked(agent)

def claim_task(task_id, owner="agent") -> str:        # 原子认领 + 绑定 cwd
    with task_store_lock():
        task = load_task(task_id)
        if task.status != "pending" or task.owner: return "unavailable"
        if not can_start(task_id): return "Blocked by: ..."
        cwd, error = task_worktree_cwd(task)
        if error: return f"Cannot claim {task_id}: {error}"
        task.owner = owner; task.status = "in_progress"; save_task(task)
        teammate_assignments[owner] = {"task_id": task.id, "cwd": cwd}
        advance_assignment_version(owner)

def task_worktree_cwd(task):            # 任务级工作目录解析
    if not task.worktree: return WORKDIR, None
    return _registered_worktree(task.worktree)

def spawn_teammate_thread(name, role, prompt, task_id=None, require_plan=False):
    runtime = TeammateRuntime(name, role, prompt, task_id, require_plan)
    threading.Thread(target=runtime.run, daemon=True).start()   # 持久线程
```

**Build-on 关系**：复用 s10 基础工具 + Hooks + Permission + Task System（两阶段建图，仅 Lead 可 `update_task`）；不含 s11/s12；s15 集成团队与协议。

**对理解现代 agent harness 的贡献**：多 Agent 协作的完整范式——身份持久化、消息总线隔离、任务原子认领、worktree 隔离、类型化控制协议，是"一个 agent 装不下整项工作"的工程答案。

---

## s14 · MCP Tools — 发现并调用外部工具

**Motto**：（本课 README 未单列 motto，属 Harness 层 "MCP Tools"）*"通过统一发现与调用协议，在运行时连接外部服务"*

**核心问题**：基础工具都写死在 `code.py`；每接一个外部系统（文档、部署平台）都要手写工具定义+调用逻辑，外部系统改动也要改课程代码。

**核心机制/抽象**：
- `MCPClient`：`register(tool_defs, handlers)` 存发现结果，`call_tool(name, args)` 执行（错误返回给模型，不结束 Loop）。
- `connect_mcp(name)`：连接 server 并发现工具；模型先只看到 `connect_mcp`，连接后下一轮出现 `mcp__docs__search` 等。
- `assemble_tool_pool()`：每轮把基础工具 + 已连接 MCP 工具合并（工具名统一 `mcp__{server}__{tool}`，`normalize_mcp_name` 清洗并查冲突与 64 字符上限）。
- `MCP_HOST_POLICY`：权限来自**宿主**配置（非 server 自述的 `readOnlyHint`），未配置默认 `confirm`。

**关键代码模式**（节自 `s14_mcp_plugin/code.py`）：

```python
MCP_HOST_POLICY = {
    ("docs", "search"): "allow", ("docs", "get_version"): "allow",
    ("deploy", "status"): "allow", ("deploy", "trigger"): "confirm",
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
            tools.append({"name": prefixed, "description": ..., "input_schema": ...})
            handlers[prefixed] = lambda *, client=server, tool=tool_def["name"], **kw: client.call_tool(tool, kw)
    return tools, handlers
```

**Build-on 关系**：基于 s04 内核 + 工具分发；把 s02 的 dispatch map 泛化为"运行时发现的动态工具池"；s15 与 Task/Background/Cron/Teams 合并。

**对理解现代 agent harness 的贡献**：用统一协议（tools/list、tools/call + 命名前缀）把外部能力变成可插拔工具，是 agent 接入真实世界系统的标准方式。

---

## s15 · Integrated Harness — 多种机制，一个循环

**Motto**：*"多种机制，一个循环"*（工具、权限、记忆、任务、团队、插件都挂在同一个 `while True` 上）

**核心问题**：前面机制各自独立示例，需要把它们接到同一个运行时，让各机制从同一处进入模型循环、产生的事件回到同一段对话。

**核心机制/抽象**（不引入新机制，只做集成；循环仍是同一结构）：
- 单次 flow：用户输入 → `UserPromptSubmit` hooks → cron/background 通知注入 → context compact → memory+skills+MCP 状态组装 system prompt → LLM → 有 `tool_use`? → `PreToolUse` hooks+permission → 分发(内置/MCP/background) → `PostToolUse` hooks → `tool_result`/`task_notification` 回 messages → 下一轮；无 tool_use → `Stop` hooks。
- 内置工具池 **26 个**：bash/read/write/edit/glob + todo_write/task/load_skill/compact + 6 个 task 工具 + 3 个 cron + spawn/list/send/request_shutdown/request_plan/review_plan + create_worktree + connect_mcp。
- 两层计划并存：`todo_write`（会话内轻量）+ task graph（跨会话、可依赖、可认领）。
- 两种 delegation：`task`（一次性 subagent，隔离上下文只返摘要）vs `spawn_teammate`（持久队友线程，WORK→result→IDLE）。
- 复用 s09 Memory、s08 压缩管线，并包一层错误恢复（429/529 退避、`max_tokens` 升级、prompt too long → reactive compact）。
- 权限不写死在工具行，而是 `PreToolUse` hook；MCP 权限用宿主白名单，文件越界直接拒，异步轮次拒交互确认。

**关键代码模式**（据 `s15_integrated_harness/README.md`）：

```python
# 权限作为 PreToolUse hook（不是硬编码在工具执行行）
blocked = trigger_hooks("PreToolUse", block)
if blocked:
    results.append(tool_result(block.id, blocked)); continue

# 压缩管线（每轮 LLM 前）
tool_result_budget → snip_compact → micro_compact → compact_history

# 后台 bash：显式标记才进后台
should_run_background → start_background_task → placeholder tool_result
后台完成 → task_notification → 下一轮注入 messages

# MCP：connect_mcp 后下一轮工具池出现 mcp__docs__search
assemble_tool_pool()  # BUILTIN_TOOLS + connected MCP tools
```

**Build-on 关系**：集成 s04(s04 内核/hooks/权限) + s05(s05 todo) + s06(subagent) + s07(skill) + s08(compact) + s09(memory) + s10(task) + s11(background) + s12(cron) + s13(teams/worktree) + s14(mcp)。

**对理解现代 agent harness 的贡献**：展示一个生产级 harness 的全貌——所有机制围绕一个循环协作，事件统一回同一段对话，是前面 14 章的"总装图"。

---

## s16 · Workflow Runtime — 模型决定单步，脚本决定编排

**Motto**：*"一次 tool_use，跑完一整套编排"*（`Workflow` 工具启动可恢复的脚本运行时，协调多次 agent 调用）

**核心问题**：s01–s15 每轮都由模型决定下一步，适合"下一步取决于上一步发现"的任务；但重复固定流程（如代码审查多维度并行+逐条验证+合并去重）需要并行、稳定结果结构、可恢复。

**核心机制/抽象**：
- `Workflow` 工具加入 s15 工具池：模型只提供 name/args/可选 `resume_from_run_id`，不提交代码。
- 编排原语（脚本收 `ExecutionState`，不直接读写文件/跑 shell）：`agent(prompt,{schema,label,phase})`、`parallel(thunks)`（等齐屏障）、`pipeline(items,*stages)`（每个 item 独立分阶段、不等齐）、`phase(title)`、`log(msg)`、`workflow(name,args)`（嵌套一层）。
- 结构化输出：`agent({schema})` 要求子 agent 只返匹配 schema 的 JSON，校验失败重试一次。
- 存储：`.runtime/` 下快照 + journal（`<runId>.journal.jsonl`）+ lock；`resume_from_run_id` 按**稳定调用键**（基于 kind/label/prompt/schema 的哈希，不依赖并发顺序）命中缓存，未改的 `agent()` 直接用 journal 结果。
- `validate_meta`：执行前校验 workflow 元数据（name 须 1–64 安全 slug），思路同 s12 校验 cron。
- 生命周期事件流：`task_started` → `task_progress`* → `task_notification`。

**关键代码模式**（据 `s16_workflow_runtime/README.md`）：

```python
# 编排原语：pipeline 每个 item 独立跑完所有 stage，不等齐
async def pipeline(self, items, *stages):
    async def run_item(item, idx):
        value = item
        for stage in stages:
            value = await stage(value, item, idx)
        return value
    return await asyncio.gather(*[run_item(it, i) for i, it in enumerate(items)])

# 稳定调用键：不依赖并发完成顺序，用内容哈希
def key(self, kind, label, prompt, schema):
    basis = f"{kind}|{label}|{prompt}|{json.dumps(schema, sort_keys=True)}"
    return f"{kind}-{_stable_hash(basis) % 10**10:010d}"
# agent() 内部：journal 命中则直接用缓存，不重跑
cached = self.journal.cached(key)
if cached is not MISS:
    self.task.progress_event("workflow_agent", label=label, status="cached"); return cached

# journal 一条条记每个 agent() 结果
class WorkflowJournal:
    def record(self, key, value):
        self._f.write(json.dumps({"key": key, "value": value}) + "\n"); self._f.flush()
```

**Build-on 关系**：基于 s15 集成宿主（复用其工具池与 API client）；与 s06 子 agent 区别：s06 是模型临场派一次，s16 把编排写成可恢复宿主代码。

**对理解现代 agent harness 的贡献**：把"固定多 Agent 脚本"从对话历史里抽出来写成可恢复代码，并用 journal + 稳定键实现断点续跑，是复杂可重复编排的工程范式。

---

## s17 · Goal Loop — 模型提出停止，独立判断器决定是否继续

**Motto**：*"模型不再调用工具，只代表这一轮想停；目标是否完成，再交给一个独立判断器。"*

**核心问题**：s01 以来退出条件=模型不再调工具即返回；但"修到测试全过""完成所有验收项"这类任务，模型可能只完成一部分就认为做完了。无 tool_use 只证明本轮结束，不证明目标达成。

**核心机制/抽象**：
- `/goal <完成条件>`：会话级 Stop hook，保存条件并立即作为本轮任务交主模型（无需再输入"开始"）。
- 独立判断器 `PromptGoalEvaluator`：**另一次无工具模型调用**，只读对话、判断完成条件（看到 condition + 对话记录 + 工具结果）。返回 `{ok, reason, impossible}`。
- `GoalController.evaluate_after_turn`：无活跃目标→`allow`（同 s01）；有后台运行→`defer`（保留目标、不判断）；`ok`→`achieved`；`impossible`→`failed`；否则 `block`（把 reason 追加回 `messages` 并 `continue`）。`block_cap` 限制连续阻止次数。
- 后台未完成时 `defer`，结果回对话后再判断；退出出口为全局 `max_turns` 与 `block_cap`，达到则还权用户但**不伪装完成、不清目标**。
- 单会话仅一个活跃 Goal；`/goal` 查看、`/goal 新条件` 替换、`/goal clear` 清除；`GoalController.restore()` 可恢复活跃 Goal。

**关键代码模式**（节自 `s17_goal_loop/code.py`）：

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
            messages=[{"role": "user", "content": prompt}], max_tokens=self.max_tokens)
        return GoalEvaluation(**_parse_json_object(_extract_text(response.content)))

async def evaluate_after_turn(self, messages, background_running=False) -> StopDecision:
    if self.active is None: return StopDecision("allow")
    if background_running: return StopDecision("defer", "background work is still running")
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
    self.messages.append({"role": "user", "content": "[Goal still active]\n..."}); continue
return SessionResult(text=text, status=decision.action)
```

**Build-on 关系**：基于 s04 Kernel（5 工具 + 四类 hook），新增 Goal 四部件（GoalState / PromptGoalEvaluator / GoalController / AgentSession）；与 s16 互补——s16 管"一批工作怎样执行"，s17 管"整件事是否完成"。

**对理解现代 agent harness 的贡献**：把"完成"定义为可独立验证的判断而非模型自述，用分离的判断器 + 明确完成条件（结束状态/验证方式/限制条件）保证 agent 真正干到目标达成，是"自主持续执行"的收口机制。

---

## 覆盖与读取状态确认

- **17 节全部覆盖**：s01, s02, s03, s04, s05, s06, s07, s08, s09, s10, s11, s12, s13, s14, s15, s16, s17 ✅
- **读取失败/兜底文件**：
  - `s05_todo_write/README.zh.md` → 返回 **404**，已用 `s05_todo_write/README.md`（英文）兜底，笔记已在 s05 节标注 ⚠️。
  - 其余 16 节中文 README 均读取成功。
- **code.py 读取**：s01, s02, s03, s08, s09, s10, s13, s14, s17 共 9 个核心章节已抓取（按要求）；s04/s05/s06/s07/s11/s12/s15/s16 的 code.py 未单独抓取，其关键代码模式已通过对应 README 充分呈现（README 内含完整片段/表格）。
- 总体：无文件完全缺失；仅 s05 中文版缺失，已按任务要求用英文版兜底并标注。
