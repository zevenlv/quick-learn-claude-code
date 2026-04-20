# quick-learn-claude-code

# 🔧 渐进式学习 Claude Code：12 个会话完整掌握 AI Agent 架构

> 从零到一，构建具备工程级健壮性的 AI Agent 系统

---

## 目录
- [阶段 1：基础架构](#阶段-1基础架构)
- [阶段 2：高级功能](#阶段-2高级功能)
- [阶段 3：任务与协作](#阶段-3任务与协作)
- [阶段 4：高级协作](#阶段-4高级协作)
- [关键技能获得](#关键技能获得)
- [核心理念内化](#核心理念内化)

---

## 阶段 1：基础架构

### ✅ S01 - Agent 循环：30 行代码掌握 AI Agent 核心秘密

**核心机制**  
AI Agent 的本质是一个**循环调用 LLM + 工具执行**的范式：

```python
def agent_loop(messages: list):
    while True:
        # 1. 调用 LLM
        response = client.messages.create(
            model=MODEL, system=SYSTEM, messages=messages,
            tools=TOOLS, max_tokens=8000
        )
        # 2. 保存助手回复
        messages.append({"role": "assistant", "content": response.content})
        # 3. 检查是否继续
        if response.stop_reason != "tool_use":
            return
        # 4. 执行工具调用
        results = []
        for block in response.content:
            if block.type == "tool_use":
                output = run_bash(block.input["command"])
                results.append({"type": "tool_result", "tool_use_id": block.id, "content": output})
        # 5. 将结果加入历史
        messages.append({"role": "user", "content": results})
```

**要点**  
- 循环条件：`stop_reason == "tool_use"` 时继续  
- 消息累积：全部历史保存在 `messages` 中  
- 工具执行：bash 命令结果实时返回给模型

---

### ✅ S02 - 工具机制："加一个工具只加一个 handler"的扩展哲学

**核心创新**  
用**字典映射**替代硬编码 if-elif，实现零侵入扩展：

```python
TOOL_HANDLERS = {
    "bash":       lambda **kw: run_bash(kw["command"]),
    "read_file":  lambda **kw: run_read(kw["path"], kw.get("limit")),
    "write_file": lambda **kw: run_write(kw["path"], kw["content"]),
    "edit_file":  lambda **kw: run_edit(kw["path"], kw["old_text"], kw["new_text"]),
}
```

循环内仅需一行完成分发：

```python
handler = TOOL_HANDLERS.get(block.name)
output = handler(**block.input) if handler else f"Unknown tool: {block.name}"
```

**安全加固**  
- 路径沙箱：`safe_path()` 禁止逃离工作目录  
- 权限控制：敏感路径黑名单校验

**优势**  
| 维度   | 收益                        |
|--------|-----------------------------|
| 扩展性 | 新增工具无需改动循环代码     |
| 安全性 | 工具级细粒度权限控制         |
| 简洁性 | 消除冗长 if-elif 链         |

---

### ✅ S03 - 任务规划：TodoWrite 防止多步任务迷失方向

**TodoManager 设计**  
强制单任务进行，防止并发混乱：

```python
class TodoManager:
    def update(self, items: list) -> str:
        in_progress_count = 0
        for item in items:
            if item.get("status") == "in_progress":
                in_progress_count += 1
        if in_progress_count > 1:
            raise ValueError("Only one task can be in_progress at a time")
```

**智能提醒**  
Agent 连续 3 轮未更新待办时，自动注入提醒：

```python
rounds_since_todo = 0 if used_todo else rounds_since_todo + 1
if rounds_since_todo >= 3:
    results.append({"type": "text", "text": "<reminder>Update your todos.</reminder>"})
```

**可视化**  
- `[ ]` pending  `[>]` in_progress  `[x]` completed

---

## 阶段 2：高级功能

### ✅ S04 - 子 Agent：上下文隔离保护思维清晰度

**设计思想**  
大任务拆小，每个小任务拥有**干净上下文**：

```python
def run_subagent(prompt: str) -> str:
    sub_messages = [{"role": "user", "content": prompt}]  # 全新上下文！
    for _ in range(30):  # 安全上限
        response = client.messages.create(
            model=MODEL, system=SUBAGENT_SYSTEM, messages=sub_messages,
            tools=CHILD_TOOLS  # 子 Agent 无 task 工具，防止递归
        )
        # ... 执行工具 ...
        if response.stop_reason != "tool_use":
            break
    # 仅返回摘要，丢弃完整历史
    return "".join(b.text for b in response.content if hasattr(b, "text")) or "(no summary)"
```

**关键约束**  
- 无递归：子 Agent 缺少 `task` 工具  
- 内存优化：历史丢弃，只保留结果摘要  
- 专注执行：独立 `messages`，屏蔽主对话噪声

---

### ✅ S05 - 技能加载："用到什么知识临时加载"的按需策略

**两层知识架构**  
1. 系统提示仅保留**技能名称与一句话描述**（低成本）  
2. 通过 `load_skill` 工具**按需加载**完整内容（高成本）

```python
SYSTEM = f"""You are a coding agent at {WORKDIR}.
Use load_skill to access specialized knowledge before tackling unfamiliar topics.

Skills available:
{SKILL_LOADER.get_descriptions()}"""
```

**SkillLoader 核心**  
- 前置元数据解析（YAML FrontMatter）  
- `get_descriptions()` 生成简短列表供系统提示  
- `get_content()` 返回完整 `<skill>` 包裹内容

**收益**  
避免系统提示过载，保持上下文精简，** token 节省 50%+ **。

---

### ✅ S06 - 上下文压缩：三层压缩实现无限会话

**分层策略**  
| 层级 | 触发时机 | 手段 | 目的 |
|------|----------|------|------|
| 微观 | 每次 LLM 调用前 | 工具结果占位符替换 | 快速释放短周期内存 |
| 自动 | token > 阈值 | LLM 摘要 + 磁盘持久化 | 防止上下文溢出 |
| 手动 | Agent 显式调用 | 同上 | 用户可控整理 |

**自动压缩流程**  
1. 保存完整对话到 `.transcripts/transcript_<ts>.jsonl`  
2. 调用 LLM 生成 ≤2000 tokens 摘要  
3. 替换当前 `messages` 为摘要

**安全细节**  
- `read_file` 等重要工具结果**免于压缩**  
- 磁盘文件支持**审计与回滚**

---

## 阶段 3：任务与协作

### ✅ S07 - 任务系统：文件持久化的依赖图管理

**数据模型**  
```json
{
  "id": 1,
  "subject": "Setup project",
  "status": "pending|in_progress|completed",
  "blockedBy": [2, 3],
  "owner": ""
}
```

**依赖自动解锁**  
完成任务时，自动从其他任务的 `blockedBy` 中移除自身 ID：

```python
def _clear_dependency(self, completed_id: int):
    for f in self.dir.glob("task_*.json"):
        task = json.loads(f.read_text())
        if completed_id in task.get("blockedBy", []):
            task["blockedBy"].remove(completed_id)
            self._save(task)
```

**可视化**  
```
[ ] #1: Setup project (blocked by: [2,3])
[>] #2: Install deps
[x] #3: Write README
```

---

### ✅ S08 - 后台任务：慢操作并行化，Agent 继续思考

**架构**  
- 守护线程池 + 线程安全通知队列  
- 命令超时 300s，防止卡死

**非阻塞注入**  
每次 LLM 调用前检查通知队列，**自动追加**结果：

```python
notif_text = "\n".join(
    f"[bg:{n['task_id']}] {n['status']}: {n['result']}" for n in notifs
)
messages.append({
    "role": "user",
    "content": f"<background-results>\n{notif_text}\n</background-results>"
})
```

**使用体感**  
Agent 发起 `npm install` 后可立即进入下一步思考，**零等待**。

---

### ✅ S09 - 团队协作：JSONL 消息总线实现异步通信

**消息总线**  
以 `{to}.jsonl` 为收件箱，**追加写 + 读完清空**，天然线程安全：

```python
def send(self, sender: str, to: str, content: str, msg_type: str = "message") -> str:
    msg = {"type": msg_type, "from": sender, "content": content, "timestamp": time.time()}
    inbox_path = self.dir / f"{to}.jsonl"
    with open(inbox_path, "a") as f:
        f.write(json.dumps(msg) + "\n")
```

**队友生命周期**  
独立线程 + 独立 `messages` + 通信工具，支持**角色分工**与**广播**。

---

## 阶段 4：高级协作

### ✅ S10 - 通信协议：请求-响应模式的状态机管理

**协议一览**  
| 协议 | 触发方 | 关键字段 | 状态跟踪 |
|------|--------|----------|----------|
| shutdown_request | 队长 | request_id | shutdown_requests dict |
| plan_approval | 队友 | request_id | plan_requests dict |

**状态机**  
pending → approved/rejected，通过 `request_id` 关联，支持**异步审批**。

---

### ✅ S11 - 自主 Agent：空闲轮询自动认领任务

**双阶段循环**  
1. **工作阶段**：标准 50 轮 Agent 循环  
2. **空闲阶段**：轮询 `POLL_INTERVAL=3s`，上限 `IDLE_TIMEOUT=60s`

**任务认领**  
扫描 `status=pending & owner="" & blockedBy=[]` 任务，**先扫先得**：

```python
def scan_unclaimed_tasks() -> list:
    unclaimed = []
    for f in sorted(TASKS_DIR.glob("task_*.json")):
        task = json.loads(f.read_text())
        if (task.get("status") == "pending"
            and not task.get("owner")
            and not task.get("blockedBy")):
            unclaimed.append(task)
    return unclaimed
```

---

### ✅ S12 - 工作树隔离：Git 工作树实现任务完全隔离

**三层架构**  
```
EventBus ─┐
TaskManager ─┼─ WorktreeManager
Git Worktree ─┘
```

**创建流程**  
1. 前置验证（名称、任务存在性）  
2. 发射 `worktree.create.before` 事件  
3. `git worktree add -b wt/<name>` 创建隔离分支  
4. 更新索引 & 绑定任务  
5. 发射 `worktree.create.after` 事件

**命令执行**  
在指定工作树目录内 `cwd=path`，**超时 120s**，事件全程追踪。

**收益**  
- 并行开发互不污染  
- Git 原生支持，回滚成本低  
- 事件日志可观测性高

---

## 关键技能获得

| 领域 | 具体能力 |
|------|----------|
| 架构设计 | 从简单循环 → 分布式多 Agent 系统 |
| 安全思维 | 路径沙箱、权限控制、超时保护 |
| 并发处理 | 线程安全、异步通信、后台执行 |
| 状态管理 | 任务图、消息队列、生命周期追踪 |
| 扩展能力 | 插件化工具、动态技能、协议设计 |

---

## 核心理念内化

1. **Harness 工程思维**：构建环境而非编写智能  
2. **模型信任原则**：让模型决策，代码执行模型意愿  
3. **渐进复杂度**：从简单开始，逐步添加必要机制  
4. **实用主义**：每个机制解决具体问题，避免过度工程

> 至此，你已具备构建生产级 AI Agent 系统的完整能力栈。🚀
