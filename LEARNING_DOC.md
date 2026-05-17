# MiniCode Python 项目架构分析与学习指南

> 本文档旨在帮助开发者深入理解 MiniCode Python 的核心架构设计，掌握每部分实现细节，便于将相关技术写入简历。

---

## 目录

1. [项目概述与设计哲学](#1-项目概述与设计哲学)
2. [核心架构设计](#2-核心架构设计)
3. [Agent Loop（智能体循环）](#3-agent-loop智能体循环)
4. [工具系统](#4-工具系统)
5. [状态管理](#5-状态管理)
6. [会话管理](#6-会话管理)
7. [记忆系统](#7-记忆系统)
8. [上下文管理](#8-上下文管理)
9. [权限系统](#9-权限系统)
10. [TUI 实现](#10-tui-实现)
11. [MCP 集成](#11-mcp-集成)
12. [简历技术要点](#12-简历技术要点)

---

## 1. 项目概述与设计哲学

### 1.1 项目定位

MiniCode Python 是一个**终端 AI 编程助手**，实现了一个轻量级的 `model -> tool -> model` 智能体循环架构，核心参考了 Claude Code 的设计理念。

**设计哲学**：构建一个**更可控、更轻量**的终端编程助手，而非追求功能大而全。

```
Claude Code (参考实现)
       │
       ▼
┌─────────────────┐
│  功能繁多复杂   │  → 企业级应用
└────────┬────────┘
         │
         ▼ 简化/重构
┌─────────────────┐
│ MiniCode Python │  → 教学/轻量/可控
└─────────────────┘
```

### 1.2 核心能力矩阵

| 能力模块 | 实现文件 | 功能说明 |
|---------|---------|---------|
| Agent Loop | `agent_loop.py` | 智能体核心循环，支持并发/串行工具执行 |
| 工具系统 | `tooling.py` + `tools/` | 统一工具注册表，支持 20+ 内置工具 |
| 状态管理 | `state.py` | Zustand 风格状态容器 |
| 会话持久化 | `session.py` | 增量保存，崩溃恢复 |
| 记忆系统 | `memory.py` | 三层 BM25 语义搜索 |
| 上下文管理 | `context_manager.py` | 多级压缩，Token 估算 |
| 权限系统 | `permissions.py` | 路径/命令/编辑多级审批 |
| TUI | `tty_app.py` + `tui/` | 事件驱动全屏终端 |
| MCP | `mcp.py` | 外部工具服务器集成 |
| Skills | `skills.py` | 本地技能加载 |

### 1.3 项目结构详解

```
MiniCode-Python/
├── minicode/                      # 核心包 (50+ Python 文件)
│   ├── main.py                   # CLI 入口，参数解析，环境初始化
│   │
│   ├── agent_loop.py             # 【核心】智能体循环
│   │   ├── run_agent_turn()     # 主循环函数
│   │   ├── _execute_single_tool() # 工具执行（带异常安全网）
│   │   ├── _model_next()        # 模型调用封装
│   │   └── 并发调度逻辑         # ThreadPoolExecutor
│   │
│   ├── tooling.py               # 【核心】工具基础设施
│   │   ├── ToolDefinition       # 工具定义数据结构
│   │   ├── ToolMetadata         # 工具元数据（能力标记）
│   │   ├── ToolRegistry        # 工具注册表（O(1)查找）
│   │   ├── ToolContext         # 工具执行上下文
│   │   └── ToolResult          # 工具执行结果
│   │
│   ├── tools/                   # 【核心】内置工具集
│   │   ├── read_file.py         # 文件读取
│   │   ├── edit_file.py         # 文件编辑
│   │   ├── grep_files.py       # 内容搜索
│   │   ├── run_command.py      # 命令执行
│   │   ├── list_files.py       # 目录列表
│   │   ├── file_tree.py        # 文件树
│   │   ├── code_review.py     # 代码审查
│   │   ├── diff_viewer.py      # Diff 查看
│   │   └── ask_user.py        # 用户交互
│   │
│   ├── state.py                 # 【状态】Zustand 风格状态管理
│   │   ├── Store<T>            # 泛型状态容器
│   │   ├── AppState            # 应用状态数据类
│   │   └── Updater 函数         # 不可变更新模式
│   │
│   ├── session.py               # 【持久化】会话管理
│   │   ├── SessionData         # 会话数据结构
│   │   ├── AutosaveManager     # 自动保存管理器
│   │   ├── save_session()     # 增量/全量保存
│   │   └── load_session()     # 恢复会话
│   │
│   ├── memory.py                # 【记忆】三层记忆系统
│   │   ├── MemoryEntry         # 记忆条目
│   │   ├── MemoryFile          # 记忆文件
│   │   ├── MemoryManager       # 记忆管理器
│   │   ├── _bm25_score()      # BM25 算法
│   │   └── _auto_classify_*()  # 自动分类
│   │
│   ├── context_manager.py        # 【上下文】窗口管理
│   │   ├── ContextManager      # 上下文管理器
│   │   ├── estimate_tokens()   # Token 估算
│   │   ├── compact_messages()  # 多级压缩
│   │   └── _summarize_*()     # 分层摘要
│   │
│   ├── permissions.py          # 【安全】权限管理
│   │   ├── PermissionManager   # 权限管理器
│   │   ├── PermissionGate     # 权限门卫
│   │   ├── _classify_*()      # 危险命令分类
│   │   └── _normalize_*()     # 路径规范化
│   │
│   ├── anthropic_adapter.py    # 【适配器】Anthropic API
│   ├── openai_adapter.py       # 【适配器】OpenAI API
│   ├── mock_model.py           # 【适配器】Mock 模型
│   │
│   ├── mcp.py                  # 【扩展】MCP 协议实现
│   ├── skills.py               # 【扩展】Skills 加载
│   │
│   ├── tty_app.py              # 【UI】TTY 应用入口
│   ├── tui/                    # 【UI】TUI 组件
│   │   ├── renderer.py         # 屏幕渲染
│   │   ├── chrome.py          # 界面框架
│   │   ├── input_handler.py   # 输入处理
│   │   ├── input_parser.py    # 输入解析
│   │   ├── session_flow.py    # 会话流程
│   │   ├── event_flow.py     # 事件流
│   │   └── types.py          # 类型定义
│   │
│   ├── prompt.py               # 系统提示词构建
│   ├── prompt_pipeline.py     # 提示词管道
│   │
│   └── config.py               # 配置管理
│
├── tests/                       # 测试套件
└── pyproject.toml              # 项目配置
```

---

## 2. 核心架构设计

### 2.1 架构分层视图

```
┌─────────────────────────────────────────────────────────────────┐
│                        Presentation Layer                       │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────┐  │
│  │    main.py       │  │   run_tty_app()  │  │   /commands  │  │
│  │  (CLI Entry)     │  │   (TUI Loop)     │  │  (Slash Cmds)│  │
│  └──────────────────┘  └──────────────────┘  └──────────────┘  │
├─────────────────────────────────────────────────────────────────┤
│                         Agent Layer                              │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                      run_agent_turn()                     │  │
│  │  ┌─────────┐    ┌──────────┐    ┌─────────────┐           │  │
│  │  │  Model  │───▶│  Tools   │───▶│   Model     │           │  │
│  │  │  Next   │    │ Executor │    │  Continue   │           │  │
│  │  └─────────┘    └──────────┘    └─────────────┘           │  │
│  │       ▲              │                                  │  │
│  │       └──────────────┴                                  │  │
│  └──────────────────────────────────────────────────────────┘  │
├─────────────────────────────────────────────────────────────────┤
│                       Adapter Layer                             │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌─────────┐  │
│  │ Anthropic  │  │  OpenAI    │  │   Mock     │  │ Model   │  │
│  │ Adapter   │  │  Adapter   │  │   Model    │  │Registry │  │
│  └────────────┘  └────────────┘  └────────────┘  └─────────┘  │
├─────────────────────────────────────────────────────────────────┤
│                       Infrastructure Layer                      │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐  │
│  │  State   │  │  Session │  │  Memory  │  │   Context    │  │
│  │ (Store)  │  │(Persist) │  │ (BM25)   │  │ (Compaction) │  │
│  └──────────┘  └──────────┘  └──────────┘  └──────────────┘  │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐  │
│  │Permisson │  │   MCP    │  │  Skills  │  │   Config     │  │
│  │ (Gate)   │  │(Protocol)│  │ (Loader) │  │  (Runtime)   │  │
│  └──────────┘  └──────────┘  └──────────┘  └──────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 设计模式详解

#### 2.2.1 适配器模式（Adapter Pattern）

**问题**：需要支持多种模型提供商（Anthropic、OpenAI），但不希望业务代码耦合具体实现。

**解决方案**：
```python
# minicode/types.py
class AgentStep(NamedTuple):
    type: str                           # "assistant" | "tool_calls"
    content: str = ""
    calls: list[dict] = field(default_factory=list)
    kind: str | None = None            # "final" | "progress"
    diagnostics: StepDiagnostics | None = None

# 定义协议（接口）
class ModelAdapter(Protocol):
    def next(
        self,
        messages: list[ChatMessage],
        on_stream_chunk: Callable[[str], None] | None = None,
    ) -> AgentStep: ...

# Anthropic 实现
class AnthropicModelAdapter:
    def next(self, messages, on_stream_chunk=None) -> AgentStep:
        # 构建 Anthropic 格式请求
        # 发送 HTTP 请求
        # 解析响应
        return AgentStep(type="tool_calls", calls=tool_calls, ...)

# OpenAI 实现
class OpenAIModelAdapter:
    def next(self, messages, on_stream_chunk=None) -> AgentStep:
        # 构建 OpenAI 格式请求
        # 发送 HTTP 请求
        # 解析响应
        return AgentStep(type="tool_calls", calls=tool_calls, ...)

# 使用方透明
def run_agent_turn(model: ModelAdapter, ...):
    next_step = model.next(messages)  # 无需关心具体实现
```

**优势**：
- 业务代码解耦
- 新增模型只需实现协议
- 便于测试（MockModelAdapter）

#### 2.2.2 注册表模式（Registry Pattern）

**问题**：工具数量多，需要高效查找工具。

**解决方案**：
```python
# minicode/tooling.py
class ToolRegistry:
    def __init__(self, tools: list[ToolDefinition], ...):
        # O(n) 遍历建立索引
        self._tool_index: dict[str, ToolDefinition] = {t.name: t for t in tools}

    def find(self, name: str) -> ToolDefinition | None:
        # O(1) 查找
        return self._tool_index.get(name)

    def execute(self, tool_name: str, input_data: Any, context: ToolContext) -> ToolResult:
        tool = self.find(tool_name)  # 快速定位
        if tool is None:
            return ToolResult(ok=False, output=f"Unknown tool: {tool_name}")
        # ... 执行逻辑
```

**时间复杂度对比**：
| 操作 | 无索引 | 有索引 |
|------|--------|--------|
| 查找工具 | O(n) | O(1) |
| n=20 工具 | 20 次比较 | 1 次哈希 |

#### 2.2.3 状态模式（State Pattern / Updater Pattern）

**问题**：状态更新需要可追踪、可预测，避免直接修改。

**解决方案**：
```python
# minicode/state.py
class Store(Generic[T]):
    def set_state(self, updater: Callable[[T], T]) -> None:
        prev = self._state
        next_state = updater(prev)
        if next_state is prev:
            return  # 跳过无变化更新
        self._state = next_state
        for listener in self._listeners:
            listener()  # 通知所有订阅者

# 定义 Updater 函数（不可变更新）
def increment_tool_calls() -> Callable[[AppState], AppState]:
    def updater(state: AppState) -> AppState:
        state.tool_call_count += 1
        return state
    return updater

def add_cost(cost: float) -> Callable[[AppState], AppState]:
    def updater(state: AppState) -> AppState:
        state.total_cost_usd += cost
        state.api_calls += 1
        return state
    return updater

# 使用
store.set_state(increment_tool_calls())
store.set_state(add_cost(0.0025))
```

**优势**：
- 所有状态变化经过 Store
- 便于调试和记录
- 支持撤销/重做

---

## 3. Agent Loop（智能体循环）

### 3.1 执行流程详解

文件：`minicode/agent_loop.py`

```python
def run_agent_turn(
    model: ModelAdapter,
    tools: ToolRegistry,
    messages: list[ChatMessage],
    cwd: str,
    permissions: PermissionManager,
    store: Store[AppState] | None = None,
    max_steps: int = 50,
    context_manager: ContextManager | None = None,
    metrics_collector: AgentMetricsCollector | None = None,
) -> list[ChatMessage]:
```

**详细时序图**：
```
┌────────┐     ┌────────┐     ┌────────┐     ┌────────┐     ┌────────┐
│ User   │     │ Model  │     │ Tools  │     │Permis- │     │ Store  │
│ Input  │     │ (LLM)  │     │        │     │ sions  │     │        │
└───┬────┘     └────┬───┘     └───┬────┘     └───┬────┘     └───┬────┘
    │              │              │              │              │
    │───1.消息────▶│              │              │              │
    │              │              │              │              │
    │◀──2.响应─────│              │              │              │
    │              │              │              │              │
    │◀──3.tool_───│──4.检查──────▶│              │              │
    │   calls      │              │              │              │
    │              │              │◀──5.权限────│              │
    │              │              │   检查       │              │
    │              │              │              │              │
    │              │              │──6.执行工具─▶│              │
    │              │              │◀──7.结果────│              │
    │              │              │              │              │
    │◀──8.tool_────│◀─9.添加─────│              │              │
    │   results    │   结果消息   │              │              │
    │              │              │              │              │
    │              │◀─10.循环────│              │              │
    │              │              │              │              │
    │◀─11.最终────│              │              │              │
    │   响应       │              │              │              │
```

### 3.2 消息处理状态机

```
                    ┌─────────────────┐
                    │  START         │
                    └────────┬────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │  CALL_MODEL    │◀────────────────┐
                    │  model.next()  │                 │
                    └────────┬────────┘                 │
                             │                          │
              ┌──────────────┼──────────────┐          │
              ▼              ▼              ▼          │
     ┌────────────┐  ┌────────────┐  ┌────────────┐   │
     │ assistant │  │tool_calls  │  │  progress  │   │
     │  (最终)   │  │  (工具)    │  │  (进度)    │   │
     └─────┬──────┘  └─────┬──────┘  └─────┬──────┘   │
           │               │               │          │
           │               │    添加 Nudge 消息       │
           │               │    继续循环 ─────────────┘
           │               │
           │               ▼
           │      ┌─────────────────┐
           │      │  EXECUTE_TOOLS  │
           │      │  - 并发执行     │
           │      │  - 串行执行     │
           │      │  - 错误处理     │
           │      └────────┬────────┘
           │               │
           │               ▼
           │      ┌─────────────────┐
           │      │  PROCESS_RESULT │
           │      │  - 分类错误     │
           │      │  - 添加 Nudge   │
           │      │  - 继续循环     │
           │      └────────┬────────┘
           │               │
           └───────────────┴──────────▶ 返回消息列表
```

### 3.3 并发工具执行详解

**核心代码**：
```python
# Phase 1: 识别可并发工具 vs 需串行工具
concurrent_calls, serial_calls = tool_scheduler.schedule_calls(calls, tools)

# Phase 2: 并发执行只读工具
with concurrent.futures.ThreadPoolExecutor(
    max_workers=max_workers,
    thread_name_prefix="mc-tool",
) as pool:
    future_to_call = {
        pool.submit(
            _execute_single_tool,
            call, tools, cwd, permissions, runtime, None, step,
            None, None,  # 无 UI 回调
        ): call
        for call in concurrent_calls
    }
    for future in concurrent.futures.as_completed(future_to_call):
        result = future.result()
        _results.append((call, result))

# Phase 3: 串行执行写操作/危险命令
for call in serial_calls:
    result = _execute_single_tool(call, ...)
    _results.append((call, result))
```

**工具分类决策**：
```python
class ToolScheduler:
    def schedule_calls(self, calls, tools):
        concurrent = []
        serial = []
        for call in calls:
            tool_def = tools.find(call["toolName"])
            if tool_def and tool_def.is_concurrency_safe:
                concurrent.append(call)
            else:
                serial.append(call)
        return concurrent, serial
```

**is_concurrency_safe 判断**：
```python
@property
def is_concurrency_safe(self) -> bool:
    if self.metadata:
        return self.metadata.is_concurrency_safe or self.metadata.is_read_only
    return self.is_read_only
```

### 3.4 全局异常安全网

```python
def _execute_single_tool(...) -> ToolResult:
    tool_name = call["toolName"]
    tool_input = call["input"]

    try:
        # 正常执行路径
        if on_tool_start:
            on_tool_start(tool_name, tool_input)

        result = tools.execute(tool_name, tool_input, context)

        if on_tool_result:
            on_tool_result(tool_name, result.output, not result.ok)

        return result

    except (KeyboardInterrupt, SystemExit):
        raise  # 允许中断传播

    except Exception as exc:
        # 全局安全网：捕获任何意外异常
        import traceback
        tb_excerpt = "".join(traceback.format_exception(...)[-3:])

        return ToolResult(
            ok=False,
            output=f"[{type(exc).__name__}] Tool execution pipeline crashed: {exc}\n"
                   f"Traceback:\n{tb_excerpt}"
        )
```

**设计目的**：
- 单工具崩溃不应导致整个会话失败
- 工具执行管道（钩子、状态更新等）的意外错误被捕获
- 用户仍可继续对话

### 3.5 Nudge 机制（智能提示）

```python
# 定义 Nudge 策略
NUDGE_CONTINUE = (
    "Continue immediately from your <progress> update with concrete tool calls, "
    "code changes, or an explicit <final> answer only if the task is complete."
)

NUDGE_AFTER_TOOL_RESULT = (
    "Continue from your progress update. You have already used tools in this turn, "
    "so treat plain status text as progress, not a final answer."
)

NUDGE_AFTER_EMPTY_RESPONSE = (
    "Your last response was empty after recent tool results. Continue immediately "
    "by trying the next concrete step."
)

# 错误分类和 Nudge 生成
classified = ErrorClassifier.classify(result.output, tool_name=call["toolName"])
nudge = NudgeGenerator.generate(classified, retry_count=tool_error_count)
result_output = result.output + "\n\n[System note: " + nudge + "]"
```

---

## 4. 工具系统

### 4.1 工具定义结构

```python
# minicode/tooling.py

@dataclass(slots=True)
class ToolDefinition:
    name: str                           # 工具名称
    description: str                    # 工具描述（供模型理解）
    input_schema: dict[str, Any]       # JSON Schema 输入验证
    validator: Validator               # 验证函数
    run: Runner                        # 执行函数
    metadata: ToolMetadata | None = None

@dataclass
class ToolMetadata:
    """工具元数据，定义工具能力"""
    name: str
    description: str
    capabilities: set[ToolCapability] = field(default_factory=set)
    input_schema: dict[str, Any] = field(default_factory=dict)
    is_enabled: bool = True
    max_result_size_chars: int = 10_000
    tags: list[str] = field(default_factory=list)

class ToolCapability(str, Enum):
    READ_ONLY = "read_only"           # 只读工具
    DESTRUCTIVE = "destructive"       # 破坏性操作
    CONCURRENCY_SAFE = "concurrency_safe"  # 可并发执行
    REQUIRES_PERMISSION = "requires_permission"  # 需要权限
```

### 4.2 工具注册流程

```python
# minicode/tools/__init__.py
def create_default_tool_registry(cwd: str, runtime: dict | None = None) -> ToolRegistry:
    tools = [
        # 文件操作
        ToolDefinition(
            name="read_file",
            description="Read the contents of a file...",
            input_schema={"type": "object", "properties": {...}},
            validator=read_file_input,
            run=read_file_run,
            metadata=ToolMetadata(
                name="read_file",
                capabilities={ToolCapability.READ_ONLY, ToolCapability.CONCURRENCY_SAFE},
            ),
        ),

        # 编辑操作
        ToolDefinition(
            name="edit_file",
            description="Make targeted edits to a file...",
            input_schema={...},
            validator=edit_file_input,
            run=edit_file_run,
            metadata=ToolMetadata(
                name="edit_file",
                capabilities={ToolCapability.REQUIRES_PERMISSION},
            ),
        ),

        # ... 更多工具
    ]

    return ToolRegistry(tools=tools)
```

### 4.3 工具执行上下文

```python
@dataclass(slots=True)
class ToolContext:
    cwd: str                           # 当前工作目录
    permissions: PermissionManager | None = None
    _runtime: dict | None = None      # 运行时配置

@dataclass(slots=True)
class ToolResult:
    ok: bool
    output: str
    backgroundTask: BackgroundTaskResult | None = None
    awaitUser: bool = False           # 是否等待用户输入
```

### 4.4 输入验证机制

```python
# 示例：read_file 输入验证
def read_file_input(input_data: Any) -> dict[str, Any]:
    """验证 read_file 工具的输入"""
    if not isinstance(input_data, dict):
        raise ValueError("Input must be a dictionary")

    path = input_data.get("path")
    if not path or not isinstance(path, str):
        raise ValueError("path is required and must be a string")

    offset = input_data.get("offset", 0)
    if not isinstance(offset, int) or offset < 0:
        raise ValueError("offset must be a non-negative integer")

    limit = input_data.get("limit", 0)
    if not isinstance(limit, int) or limit < 0:
        raise ValueError("limit must be a non-negative integer")

    return {
        "path": path,
        "offset": offset,
        "limit": limit,
        "show_line_numbers": input_data.get("show_line_numbers", False),
    }
```

### 4.5 智能输出截断

```python
# minicode/tooling.py

_TOOL_OUTPUT_LIMITS: dict[str, int] = {
    "read_file": 40_000,
    "grep_files": 20_000,
    "run_command": 30_000,
    "web_fetch": 20_000,
    # ...
}

_LARGE_OUTPUT_THRESHOLD = 50_000  # 触发截断的阈值

def _smart_truncate_output(output: str, tool_name: str) -> str:
    """智能截断大输出，保留关键信息"""

    if tool_name == "read_file":
        # 文件读取：保留头部 60% + 尾部 40%
        head_lines = max(1, int(max_lines * 0.6))
        tail_lines = max(1, max_lines - head_lines)
        return f"{head}\n\n... [{omitted} lines omitted] ...\n\n{tail}"

    if tool_name in ("run_command", "run_with_debug"):
        # 命令输出：保留头部 40% + 尾部 40% + 错误行
        error_lines = [
            (i, line) for i, line in enumerate(lines)
            if re.search(r'(?i)(error|fail|exception|traceback)', line)
        ]
        return f"{head}\n... [{omitted} lines omitted] ...\n{error_text}\n\n{tail}"

    if tool_name in ("grep_files", "web_search"):
        # 搜索结果：保留前 N 匹配 + 摘要
        return f"{head}\n... [{omitted} more matches omitted] ..."
```

---

## 5. 状态管理

### 5.1 Zustand 风格 Store 完整实现

```python
# minicode/state.py

T = TypeVar("T")

class Store(Generic[T]):
    """Zustand 风格状态管理"""

    def __init__(
        self,
        initial_state: T,
        on_change: Callable[[T, T], None] | None = None,
    ):
        self._state = initial_state
        self._listeners: list[Callable[[], None]] = []
        self._on_change = on_change
        self._update_count = 0

    def get_state(self) -> T:
        """获取当前状态（引用）"""
        return self._state

    def set_state(self, updater: Callable[[T], T]) -> None:
        """通过 updater 函数更新状态"""
        prev = self._state
        next_state = updater(prev)

        # 跳过无变化更新
        if next_state is prev:
            return

        # 调用变化回调
        if self._on_change:
            self._on_change(next_state, prev)

        self._state = next_state
        self._update_count += 1

        # 通知所有订阅者
        for listener in self._listeners:
            try:
                listener()
            except Exception:
                pass  # 不让订阅者错误中断状态更新

    def subscribe(self, listener: Callable[[], None]) -> Callable[[], None]:
        """订阅状态变化"""
        self._listeners.append(listener)

        def unsubscribe():
            if listener in self._listeners:
                self._listeners.remove(listener)

        return unsubscribe
```

### 5.2 AppState 数据类

```python
@dataclass
class AppState:
    """全局应用状态"""

    # Session 信息
    session_id: str = ""
    workspace: str = ""
    model: str = "unknown"

    # 上下文跟踪
    message_count: int = 0
    tool_call_count: int = 0
    token_usage: int = 0
    context_window_size: int = 128_000
    context_usage_percentage: float = 0.0

    # 成本跟踪
    total_cost_usd: float = 0.0
    api_calls: int = 0
    api_errors: int = 0

    # 任务跟踪
    active_tasks: int = 0
    completed_tasks: int = 0

    # UI 状态
    is_busy: bool = False
    active_tool: str | None = None
    status_message: str = ""

    # 功能开关
    verbose: bool = False
    skills_enabled: bool = True
    mcp_enabled: bool = True

    # 时间戳
    created_at: float = field(default_factory=time.time)
    last_updated: float = field(default_factory=time.time)

    def update_timestamp(self) -> None:
        self.last_updated = time.time()
```

### 5.3 Updater 函数集合

```python
# 状态更新函数工厂

def update_message_count(count: int) -> Callable[[AppState], AppState]:
    def updater(state: AppState) -> AppState:
        state.message_count = count
        state.update_timestamp()
        return state
    return updater

def increment_tool_calls() -> Callable[[AppState], AppState]:
    def updater(state: AppState) -> AppState:
        state.tool_call_count += 1
        state.update_timestamp()
        return state
    return updater

def update_context_usage(
    tokens: int,
    window_size: int | None = None,
) -> Callable[[AppState], AppState]:
    def updater(state: AppState) -> AppState:
        state.token_usage = tokens
        if window_size is not None:
            state.context_window_size = window_size
        if state.context_window_size > 0:
            state.context_usage_percentage = (
                tokens / state.context_window_size * 100
            )
        state.update_timestamp()
        return state
    return updater

def add_cost(cost_usd: float) -> Callable[[AppState], AppState]:
    def updater(state: AppState) -> AppState:
        state.total_cost_usd += cost_usd
        state.api_calls += 1
        state.update_timestamp()
        return state
    return updater

def record_api_error() -> Callable[[AppState], AppState]:
    def updater(state: AppState) -> AppState:
        state.api_errors += 1
        state.api_calls += 1
        state.update_timestamp()
        return state
    return updater

def set_busy(tool_name: str | None = None) -> Callable[[AppState], AppState]:
    def updater(state: AppState) -> AppState:
        state.is_busy = True
        state.active_tool = tool_name
        state.status_message = f"Running {tool_name}..." if tool_name else "Working..."
        state.update_timestamp()
        return state
    return updater

def set_idle() -> Callable[[AppState], AppState]:
    def updater(state: AppState) -> AppState:
        state.is_busy = False
        state.active_tool = None
        state.status_message = "Ready"
        state.update_timestamp()
        return state
    return updater
```

### 5.4 全局 Store 单例

```python
# 全局 Store 管理
_global_store: Store[AppState] | None = None

def get_global_store() -> Store[AppState]:
    global _global_store
    if _global_store is None:
        _global_store = create_app_store()
    return _global_store

def set_global_store(store: Store[AppState]) -> None:
    global _global_store
    _global_store = store
```

---

## 6. 会话管理

### 6.1 会话数据结构

```python
# minicode/session.py

@dataclass
class SessionData:
    """完整会话状态"""
    session_id: str
    created_at: float
    updated_at: float
    workspace: str
    messages: list[dict[str, Any]] = field(default_factory=list)
    transcript_entries: list[dict[str, Any]] = field(default_factory=list)
    history: list[str] = field(default_factory=list)
    permissions_summary: dict[str, Any] = field(default_factory=dict)
    skills: list[dict[str, Any]] = field(default_factory=list)
    mcp_servers: list[dict[str, Any]] = field(default_factory=list)
    metadata: SessionMetadata = field(default=None)

    # 增量保存跟踪
    _last_saved_msg_count: int = field(default=0, repr=False)
    _last_saved_transcript_count: int = field(default=0, repr=False)
    _delta_save_count: int = field(default=0, repr=False)
    _last_full_save_hash: str = field(default="", repr=False)

    def has_delta(self) -> bool:
        """检查是否有未保存的更改"""
        return (
            len(self.messages) != self._last_saved_msg_count
            or len(self.transcript_entries) != self._last_saved_transcript_count
        )

    def update_metadata(self) -> None:
        """更新元数据"""
        self.updated_at = time.time()
        self.metadata.updated_at = self.updated_at
        self.metadata.message_count = len(self.messages)
```

### 6.2 增量保存策略

```
时间线：
───────────────────────────────────────────────────────────────▶

1. 首次保存 (force_full=True)
   ┌─────────────────────────────────┐
   │ sessions/{session_id}.json      │  ← 全量保存
   │ (包含所有消息)                   │
   └─────────────────────────────────┘

2. 第2次保存 (_delta_save_count=1)
   ┌─────────────────────────────────┐
   │ sessions/deltas/{session_id}/  │
   │   delta_0001.json               │  ← 仅新增消息
   └─────────────────────────────────┘

3. 第3-10次保存 (增量)
   ┌─────────────────────────────────┐
   │ sessions/deltas/{session_id}/  │
   │   delta_0002.json               │
   │   delta_0003.json               │
   │   ...                           │
   │   delta_0009.json               │
   └─────────────────────────────────┘

4. 第11次保存 (_delta_save_count=10, 达到阈值)
   ┌─────────────────────────────────┐
   │ sessions/{session_id}.json      │  ← 全量合并保存
   │ (合并所有 delta)                 │
   └─────────────────────────────────┘
   删除 delta_*.json 文件
```

### 6.3 Delta 文件格式

```json
// sessions/deltas/{session_id}/delta_0001.json
{
  "ts": 1715932800.123,           // 时间戳
  "msg_offset": 5,                // 这批消息的起始偏移
  "transcript_offset": 10,        // 这批转录的起始偏移
  "messages": [
    {"role": "user", "content": "..."},
    {"role": "assistant", "content": "..."}
  ],
  "transcripts": [
    {"kind": "user", "body": "..."}
  ]
}
```

### 6.4 AutosaveManager

```python
class AutosaveManager:
    """自动保存管理器"""

    def __init__(self, session: SessionData, interval: int = 30):
        self.session = session
        self.interval = interval          # 保存间隔（秒）
        self._last_save_time = time.time()
        self._dirty = False
        self._full_save_counter = 0

    def mark_dirty(self) -> None:
        """标记需要保存"""
        self._dirty = True

    def should_save(self) -> bool:
        """检查是否应该保存"""
        if not self._dirty:
            return False
        elapsed = time.time() - self._last_save_time
        return elapsed >= self.interval

    def save_if_needed(self) -> bool:
        """如果需要则保存"""
        if self.should_save():
            save_session(self.session, force_full=False)  # 增量保存
            self._last_save_time = time.time()
            self._dirty = False
            self._full_save_counter += 1
            return True
        return False

    def force_save(self) -> None:
        """强制立即全量保存"""
        save_session(self.session, force_full=True)
        self._last_save_time = time.time()
        self._dirty = False
        self._full_save_counter = 0
```

---

## 7. 记忆系统

### 7.1 三层记忆架构

```
┌─────────────────────────────────────────────────────────────────┐
│                     记忆系统层次图                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  Layer 1: User Memory (~/.mini-code/memory/)              │   │
│   │  ├── 作用域：跨项目共享                                    │   │
│   │  ├── 持久化：~/.mini-code/memory/memory.json             │   │
│   │  └── 用途：用户偏好、跨项目约定                           │   │
│   └─────────────────────────────────────────────────────────┘   │
│                              ▲                                  │
│                              │                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  Layer 2: Project Memory (.mini-code-memory/)            │   │
│   │  ├── 作用域：项目内共享，可提交到 Git                     │   │
│   │  ├── 持久化：{project}/.mini-code-memory/memory.json     │   │
│   │  └── 用途：项目架构决策、技术债记录                        │   │
│   └─────────────────────────────────────────────────────────┘   │
│                              ▲                                  │
│                              │                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  Layer 3: Local Memory (.mini-code-memory-local/)        │   │
│   │  ├── 作用域：本地私有，不提交到 Git                       │   │
│   │  ├── 持久化：{project}/.mini-code-memory-local/memory.json│  │
│   │  └── 用途：本地调试信息、个人笔记                        │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 7.2 记忆条目结构

```python
class MemoryScope(str, Enum):
    USER = "user"           # 跨项目用户记忆
    PROJECT = "project"      # 项目共享记忆
    LOCAL = "local"         # 本地私有记忆

@dataclass
class MemoryEntry:
    """记忆条目"""
    id: str                           # 唯一标识
    scope: MemoryScope                # 作用域
    category: str                    # 分类：architecture, code-pattern, testing, ...
    content: str                     # 内容
    created_at: float = field(default_factory=time.time)
    updated_at: float = field(default_factory=time.time)
    tags: list[str] = field(default_factory=list)
    usage_count: int = 0             # 被引用次数
```

### 7.3 BM25 搜索算法

**BM25（Best Matching 25）**是一种经典的信息检索算法，用于评估文档与查询的相关性。

```python
# BM25 参数
_BM25_K1 = 1.5  # Term frequency scaling factor
_BM25_B = 0.75  # Document length normalization

def _bm25_score(
    query_tokens: list[str],
    doc_tokens: list[str],
    idf: dict[str, float],
    avgdl: float,
) -> float:
    """
    BM25 算法公式：

    score(q,d) = Σ IDF(qi) * (tf(qi,d) * (k1 + 1)) /
                          (tf(qi,d) + k1 * (1 - b + b * |d|/avgdl))

    其中：
    - tf(qi,d): 词项 qi 在文档 d 中的词频
    - IDF(qi): 词项 qi 的逆文档频率
    - |d|: 文档长度
    - avgdl: 平均文档长度
    - k1, b: 调参
    """
    if not query_tokens or not doc_tokens or avgdl == 0:
        return 0.0

    doc_len = len(doc_tokens)
    tf_doc = _compute_tf(doc_tokens)

    score = 0.0
    for term in set(query_tokens):
        if term not in idf:
            continue
        tf = tf_doc.get(term, 0.0)
        if tf == 0:
            continue

        # BM25 核心公式
        numerator = tf * (k1 + 1)
        denominator = tf + k1 * (1 - b + b * (doc_len / avgdl))
        score += idf[term] * (numerator / denominator)

    return score

def _compute_tf(tokens: list[str]) -> dict[str, float]:
    """计算词频（归一化）"""
    counts = Counter(tokens)
    total = len(tokens)
    return {term: count / total for term, count in counts.items()}

def _compute_idf(documents: list[list[str]]) -> dict[str, float]:
    """
    计算逆文档频率（IDF）

    公式：log((N + 1) / (df + 1)) + 1

    其中：
    - N: 文档总数
    - df: 包含词项的文档数
    """
    n = len(documents)
    doc_freq: dict[str, int] = {}

    for doc_tokens in documents:
        seen = set(doc_tokens)
        for term in seen:
            doc_freq[term] = doc_freq.get(term, 0) + 1

    return {
        term: math.log((n + 1) / (df + 1)) + 1
        for term, df in doc_freq.items()
    }
```

### 7.4 搜索评分公式

```python
# 最终搜索评分 = 多重因素加权

total_score = (
    bm25_score              # BM25 语义相关性
    + substring_score       # 子串匹配 (精确匹配=2.0, 部分匹配=1.0)
    + tag_score             # 标签匹配 (精确=5.0, 部分=1.5)
    + usage_bonus           # 使用频率加成: log1p(usage_count) * 0.3
    + recency_bonus          # 新近度加成: 1.0 / (1.0 + age_hours / 24.0) * 0.5
)
```

### 7.5 自动分类规则

```python
_CLASSIFICATION_RULES = [
    ("architecture", [
        "architecture", "design", "pattern", "api", "rest",
        "backend", "service", "架构", "设计", "模式"
    ]),
    ("code-pattern", [
        "function", "method", "def", "class", "函数", "方法", "类"
    ]),
    ("testing", [
        "test", "assert", "pytest", "unit", "测试", "断言"
    ]),
    ("configuration", [
        "config", "settings", "env", "配置", "设置", "环境"
    ]),
    ("workflow", [
        "git", "commit", "branch", "merge", "工作流", "分支", "合并"
    ]),
    ("security", [
        "security", "auth", "permission", "安全", "认证", "权限"
    ]),
    ("performance", [
        "performance", "optimization", "benchmark", "性能", "优化", "基准"
    ]),
    ("convention", [
        "convention", "style", "naming", "规范", "风格", "命名"
    ]),
]
```

---

## 8. 上下文管理

### 8.1 Token 估算算法

```python
# minicode/context_manager.py

# 预编译正则：快速检测 CJK 字符
_CJK_PATTERN = re.compile(r'[一-鿿぀-ゟ゠-ヿ가-힯]')

# LRU 缓存：避免重复计算
_token_cache: dict[str, int] = {}
_TOKEN_CACHE_MAX = 1024

def estimate_tokens(text: str) -> int:
    """
    Token 估算策略：

    - 中文/日文/韩文：约 1.5 字符/token
    - 英文/代码：约 4 字符/token

    使用正则表达式快速统计而非逐字符检查
    """
    if not text:
        return 0

    # 缓存查找（短文本用原文作 key，长文本用 hash）
    cache_key = text if len(text) < 256 else hash(text)
    if cache_key in _token_cache:
        return _token_cache[cache_key]

    # 使用正则快速统计 CJK 字符数量
    cjk_count = len(_CJK_PATTERN.findall(text))

    # 计算 ASCII 字符
    ascii_chars = len(text) - cjk_count

    # 估算
    result = max(1, int(cjk_count / 1.5 + ascii_chars / 4.0))

    # 缓存结果
    if len(_token_cache) < _TOKEN_CACHE_MAX:
        _token_cache[cache_key] = result

    return result
```

### 8.2 消息 Token 估算

```python
def estimate_message_tokens(message: dict[str, Any]) -> int:
    """估算单条消息的 token 数"""
    tokens = 0

    # 角色开销
    role = message.get("role", "")
    if role == "system":
        tokens += 3
    elif role == "user":
        tokens += 4
    elif role == "assistant":
        tokens += 3
    elif role == "assistant_tool_call":
        tokens += 7
    elif role == "tool_result":
        tokens += 6
    elif role == "assistant_progress":
        tokens += 3

    # 内容 token
    content = message.get("content", "")
    if isinstance(content, str):
        tokens += estimate_tokens(content)

    # 工具调用输入
    if "input" in message:
        input_str = json.dumps(message["input"]) if isinstance(message["input"], dict) else str(message["input"])
        tokens += estimate_tokens(input_str)

    return tokens
```

### 8.3 多级压缩策略

```python
# 压缩目标层级
_COMPACTION_LEVELS = [0.70, 0.50, 0.30]  # 轻度70%/中度50%/深度30%

def should_auto_compact(self) -> bool:
    """检查是否需要触发自动压缩"""
    stats = self.get_stats()
    # 根据压缩级别调整阈值
    threshold = 0.95 - (self._compaction_level * 0.10)
    threshold = max(0.60, threshold)  # 最低 60%
    return stats.usage_percentage >= (threshold * 100)

def compact_messages(self) -> list[dict[str, Any]]:
    """
    多级渐进压缩流程：

    Phase 1: 移除 progress 消息（最低优先级）
    Phase 2: 自适应截断大工具结果
    Phase 3: 压缩 tool_call + result 对为摘要
    Phase 4: 基于优先级的消息删除
    """
    stats = self.get_stats()
    target_pct = self._COMPACTION_LEVELS[min(self._compaction_level, 2)]
    target_tokens = int(self.context_window * target_pct)

    # Phase 1: 移除 progress 消息
    filtered = [m for m in other_messages if m.get("role") != "assistant_progress"]

    if estimate_messages_tokens(filtered) <= target_tokens:
        return self._finalize_compaction(...)

    # Phase 2: 截断大工具结果
    for m in filtered:
        if m.get("role") != "tool_result":
            continue
        content = m.get("content", "")
        if len(content) <= threshold:
            continue

        # 根据工具类型选择截断策略
        if tool_name in _EDIT_TOOLS:
            threshold = 3000  # 编辑工具保留更多
        elif tool_name in _READ_TOOLS:
            threshold = 1500  # 读取工具可更激进
        elif is_error:
            threshold = 4000  # 错误保留最多

        # 智能截断：头部 + 省略标记 + 尾部
        truncated = smart_truncate(content, threshold)
        m["content"] = truncated

    # Phase 3: 压缩工具对
    compressed = []
    i = 0
    while i < len(filtered):
        msg = filtered[i]
        if (msg.get("role") == "assistant_tool_call" and
            i + 1 < len(filtered) and
            filtered[i + 1].get("role") == "tool_result"):

            call_msg = msg
            result_msg = filtered[i + 1]
            summary = self._compress_tool_pair(call_msg, result_msg)
            compressed.append({"role": "assistant", "content": summary})
            i += 2
        else:
            compressed.append(msg)
            i += 1

    # Phase 4: 优先级删除
    while estimate_messages_tokens(compressed) > target_tokens:
        # 移除最低优先级消息（从最旧的开始）
        # 保护最近 6 条消息
        removable_end = max(MIN_MESSAGES_TO_KEEP, len(compressed) - PROTECTED_RECENT)
        # 找到最低优先级消息并删除
        ...
```

### 8.4 分层摘要算法

```python
def _build_layered_summary(info: _ExtractedInfo, max_summary_tokens: int = 2000) -> str:
    """
    分层摘要构建：

    Layer 1: 用户意图 (35% 预算)      - 最高优先级
    Layer 2: 决策与文件路径 (20%)     - 关键选择
    Layer 3: 关键工具结果 (15%)       - 错误和重要结果
    Layer 4: 助手结论 (15%)           - 达成结论
    Layer 5: 代码片段 (10%)           - 代码模式
    Layer 6: 工具使用摘要 (5%)        - 活动日志
    """
    lines = []
    layer_budgets = [0.35, 0.20, 0.15, 0.15, 0.10, 0.05]

    def _remaining_budget() -> int:
        return max(0, max_summary_tokens - estimate_tokens("\n".join(lines)))

    # Layer 1: 用户意图
    if info.user_intents:
        budget = int(max_summary_tokens * layer_budgets[0])
        lines.append("## User requests:")
        for intent in info.user_intents[:12]:
            if estimate_tokens("\n".join(lines)) > budget:
                lines.append(f"  ... and {len(info.user_intents) - info.user_intents.index(intent)} more")
                break
            lines.append(f"- {intent}")

    # ... 其他 Layer
```

---

## 9. 权限系统

### 9.1 PermissionManager 完整结构

```python
class PermissionManager:
    def __init__(self, workspace_root: str, prompt: PromptHandler | None = None):
        self.workspace_root = _normalize_path(workspace_root)
        self.prompt = prompt
        self.auto_checker = AutoModeChecker(mode=PermissionMode.DEFAULT)

        # 路径权限
        self.allowed_directory_prefixes: set[str] = set()
        self.denied_directory_prefixes: set[str] = set()
        self.session_allowed_paths: set[str] = set()
        self.session_denied_paths: set[str] = set()

        # 命令权限
        self.allowed_command_patterns: set[str] = set()
        self.denied_command_patterns: set[str] = set()
        self.session_allowed_commands: set[str] = set()
        self.session_denied_commands: set[str] = set()

        # 编辑权限
        self.allowed_edit_patterns: set[str] = set()
        self.denied_edit_patterns: set[str] = set()
        self.session_allowed_edits: set[str] = set()
        self.session_denied_edits: set[str] = set()
        self.turn_allowed_edits: set[str] = set()
        self.turn_allow_all_edits = False

        self._initialize()
```

### 9.2 路径权限检查流程

```
ensure_path_access(path, intent)
         │
         ▼
┌─────────────────────────┐
│ 1. 快速路径检查        │  ← 检查 path 是否在 workspace_root 内
│    workspace_root       │     （最常见情况，优化路径）
└───────────┬─────────────┘
            │
     ┌──────┴──────┐
     ▼             ▼
  通过          继续检查
                │
                ▼
         ┌─────────────────────────┐
         │ 2. 拒绝列表检查         │  ← O(1) 查找
         │    session_denied_paths│
         │    denied_prefixes     │
         └───────────┬─────────────┘
                     │
              ┌──────┴──────┐
              ▼             ▼
            拒绝          继续检查
                         │
                         ▼
                  ┌─────────────────────────┐
                  │ 3. 批准列表检查         │
                  │    session_allowed_paths│
                  │    allowed_prefixes     │
                  └───────────┬─────────────┘
                              │
                       ┌──────┴──────┐
                       ▼             ▼
                     通过          继续检查
                                  │
                                  ▼
                           ┌─────────────────────────┐
                           │ 4. 自动模式评估         │
                           │    auto_checker.        │
                           │    assess_risk()        │
                           └───────────┬─────────────┘
                                       │
                                ┌──────┴──────┐
                                ▼             ▼
                            批准            继续
                                          │
                                          ▼
                                   ┌─────────────────────────┐
                                   │ 5. 用户交互提示         │
                                   │    prompt()             │
                                   └─────────────────────────┘
```

### 9.3 危险命令分类

```python
def _classify_dangerous_command(command: str, args: list[str]) -> str | None:
    """分类命令危险等级"""
    normalized_args = [arg.strip() for arg in args if arg.strip()]
    signature = f"{command} {' '.join(normalized_args)}"

    # Git 危险操作
    if command == "git":
        if "reset" in normalized_args and "--hard" in normalized_args:
            return f"git reset --hard can discard local changes ({signature})"
        if "clean" in normalized_args:
            return f"git clean can delete untracked files ({signature})"
        if "push" in normalized_args and "--force" in normalized_args:
            return f"git push --force rewrites remote history ({signature})"

    # 灾难性删除命令
    if command == "rm":
        combined_flags = "".join(
            arg for arg in normalized_args if arg.startswith("-")
        ).lower()
        if "r" in combined_flags and "f" in combined_flags:
            if any(arg in {"/", "/*"} for arg in normalized_args):
                return f"rm -rf targeting root is catastrophic ({signature})"
            return f"rm -rf can cause catastrophic data loss ({signature})"

    # 磁盘操作命令
    if command in {"dd", "mkfs", "fdisk", "format"}:
        return f"{command} can modify or destroy disk partitions ({signature})"

    # 权限操作命令
    if command == "chmod" and "777" in normalized_args:
        return f"chmod 777 opens permissions to all users ({signature})"

    # 代码执行命令
    if command in {"node", "python", "bash", "sh", "powershell"}:
        return f"{command} can execute arbitrary local code ({signature})"

    return None  # 非危险命令
```

### 9.4 路径规范化与缓存

```python
# LRU 缓存：Path.resolve() 是昂贵操作（stat syscall）
_normalize_path_cached = lru_cache(maxsize=512)(
    lambda p: str(Path(p).resolve())
)

def _is_within_directory(root: str, target: str) -> bool:
    """检查 target 是否在 root 目录内"""
    if _is_win:
        # Windows: 大小写不敏感
        target_str = target.lower()
        root_str = root.lower().rstrip("\\/")
        return target_str.startswith(root_str + "\\") or target_str.startswith(root_str + "/")
    else:
        # Unix: 直接字符串比较
        root_str = root.rstrip(os.sep)
        return target == root_str or target.startswith(root_str + os.sep)
```

### 9.5 原子写入

```python
def _write_permission_store(store: dict[str, Any]) -> None:
    """原子写入：先写临时文件，再 rename"""
    import tempfile

    MINI_CODE_PERMISSIONS_PATH.parent.mkdir(parents=True, exist_ok=True)

    # 1. 创建临时文件
    fd, tmp_path = tempfile.mkstemp(
        dir=MINI_CODE_PERMISSIONS_PATH.parent,
        suffix=".tmp"
    )
    try:
        with os.fdopen(fd, 'w', encoding='utf-8') as f:
            json.dump(store, f, indent=2)
            f.write('\n')

        # 2. 原子替换（rename 是原子操作）
        os.replace(tmp_path, MINI_CODE_PERMISSIONS_PATH)
    except Exception:
        # 3. 清理临时文件
        try:
            os.unlink(tmp_path)
        except OSError:
            pass
        raise
```

---

## 10. TUI 实现

### 10.1 TTY 应用架构

```
┌─────────────────────────────────────────────────────────────────┐
│                        run_tty_app()                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                   Event Loop                              │   │
│  │                                                          │   │
│  │    ┌─────────┐    ┌─────────┐    ┌─────────┐            │   │
│  │    │  Read   │───▶│  Parse  │───▶│ Handle  │            │   │
│  │    │  Input  │    │  Event  │    │  Event  │            │   │
│  │    └─────────┘    └─────────┘    └────┬────┘            │   │
│  │                                        │                 │   │
│  │    ┌───────────────────────────────────┴──────────────┐ │   │
│  │    │                                              │   │ │   │
│  │    ▼                                              ▼   │ │   │
│  │  ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐      │ │   │
│  │  │  Text  │  │ Key    │  │Resize  │  │ Agent  │      │ │   │
│  │  │  Event │  │ Event  │  │ Event  │  │Result  │      │ │   │
│  │  └────────┘  └────────┘  └────────┘  └────────┘      │ │   │
│  │                                                      │ │   │
│  └──────────────────────────────────────────────────────┘ │   │
│                            │                               │   │
│                            ▼                               │   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    Rerender                             │   │
│  │  ThrottledRenderer (防止闪烁，60fps 限制)               │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 10.2 关键组件交互

```python
# main.py 中启动 TUI
run_tty_app(
    runtime=runtime,
    tools=tools,
    model=model,
    messages=messages,
    cwd=cwd,
    permissions=permissions,
    resume_session=args.resume,
    list_sessions_only=args.list_sessions,
    memory_manager=memory_mgr,
    context_manager=context_mgr,
)

# tty_app.py 中
def run_tty_app(...) -> list[ChatMessage]:
    # 1. 构建运行时状态
    args, state = build_tty_runtime_state(...)

    # 2. 节流渲染器
    throttled = _ThrottledRenderer(
        lambda: _render_screen(args, state),
        min_interval=0.016  # ~60fps
    )

    def rerender() -> None:
        throttled.request()

    # 3. 进入 TTY 模式
    enter_tty_runtime()

    # 4. 事件循环
    with _RawModeContext():
        while not should_exit:
            # 读取输入
            chunk = os.read(sys.stdin.fileno(), 4096)

            # 解析
            parsed = parse_input_chunk(chunk)

            # 处理事件
            for event in parsed.events:
                _handle_tty_event(args, state, event, ...)

            # 渲染
            throttled.flush()
```

### 10.3 节流渲染器

```python
class _ThrottledRenderer:
    """节流渲染器：合并快速连续的渲染请求"""

    def __init__(self, render_fn: Callable[[], None], min_interval: float = 0.016):
        self.render_fn = render_fn
        self.min_interval = min_interval
        self._pending = False
        self._last_render = 0.0

    def request(self) -> None:
        """请求渲染（可能被合并）"""
        if self._pending:
            return  # 已有待处理请求

        now = time.time()
        elapsed = now - self._last_render

        if elapsed >= self.min_interval:
            # 可以立即渲染
            self._do_render()
        else:
            # 延迟渲染，标记待处理
            self._pending = True
            # 设置定时器在 min_interval 后触发渲染

    def flush(self) -> None:
        """强制刷新待处理的渲染"""
        if self._pending:
            self._do_render()
```

---

## 11. MCP 集成

### 11.1 MCP 协议概述

MCP（Model Context Protocol）是一种标准化协议，用于 AI 模型与外部工具服务器的通信。

```
┌──────────────┐                          ┌──────────────┐
│   MiniCode   │         stdio           │  MCP Server  │
│   (Client)   │◀────────────────────────▶│  (Tool Host) │
└──────────────┘                          └──────────────┘
       │                                        │
       │  ──── initialize ────▶                 │
       │  ◀─── capabilities ────                │
       │                                        │
       │  ──── tools/list ────▶                 │
       │  ◀─── tool definitions ────           │
       │                                        │
       │  ──── tools/call ────▶                 │
       │  ◀─── tool result ────                 │
       │                                        │
```

### 11.2 MCP 服务器管理

```python
class MCPServer:
    def __init__(self, name: str, command: list[str], env: dict):
        self.name = name
        self.command = command

        # 启动进程
        self.process = subprocess.Popen(
            command,
            stdin=subprocess.PIPE,
            stdout=subprocess.PIPE,
            stderr=subprocess.PIPE,
            env={**os.environ, **env},
        )

        # 发送初始化
        self._send({"jsonrpc": "2.0", "id": 0, "method": "initialize", ...})
        response = self._recv()  # 获取服务器能力

    def _send(self, message: dict) -> None:
        """发送 JSON-RPC 消息"""
        data = json.dumps(message) + "\n"
        self.process.stdin.write(data.encode("utf-8"))
        self.process.stdin.flush()

    def _recv(self) -> dict:
        """接收 JSON-RPC 消息"""
        line = self.process.stdout.readline()
        return json.loads(line)
```

### 11.3 MCP 工具封装

```python
class MCPClient:
    def __init__(self, server: MCPServer):
        self.server = server
        self._tools = self._discover_tools()

    def _discover_tools(self) -> list[ToolDefinition]:
        """发现 MCP 服务器提供的工具"""
        self.server._send({
            "jsonrpc": "2.0",
            "id": 1,
            "method": "tools/list"
        })
        response = self.server._recv()

        tools = []
        for tool in response.get("result", {}).get("tools", []):
            # 封装为本地 ToolDefinition
            tools.append(ToolDefinition(
                name=f"mcp__{self.server.name}__{tool['name']}",
                description=tool['description'],
                input_schema=tool['inputSchema'],
                validator=self._wrap_validator(tool['name']),
                run=self._wrap_runner(tool['name']),
            ))
        return tools

    def _wrap_runner(self, tool_name: str):
        """包装 MCP 工具为本地 runner"""
        def run(input_data, context):
            self.server._send({
                "jsonrpc": "2.0",
                "id": uuid.uuid4().hex[:8],
                "method": "tools/call",
                "params": {
                    "name": tool_name,
                    "arguments": input_data
                }
            })
            response = self.server._recv()
            return ToolResult(
                ok=response.get("error") is None,
                output=json.dumps(response.get("result", {}))
            )
        return run
```

---

## 12. 简历技术要点

### 12.1 核心技术亮点详细版

#### A. Agent 架构设计
```
【项目描述】
MiniCode Python 是一个终端 AI 编程助手，实现了轻量级的 model->tool->model 智能体循环。

【技术实现】
- 核心循环：run_agent_turn() 实现模型调用 → 工具执行 → 结果处理的循环
- 并发执行：使用 ThreadPoolExecutor 实现只读工具并发、串行工具顺序执行
- 异常安全网：全局 try-catch 确保单工具崩溃不影响整体会话
- 智能增强：错误分类（ErrorClassifier）、提示生成（NudgeGenerator）、工具调度（ToolScheduler）

【关键技术点】
model-tool-model loop, concurrent.futures.ThreadPoolExecutor, 适配器模式, 全局异常处理
```

#### B. 上下文窗口管理
```
【项目描述】
实现自适应上下文窗口管理，支持多级渐进压缩策略。

【技术实现】
- Token 估算：正则 + 缓存，支持中英文混合文本
- 压缩层级：70% → 50% → 30% 三级目标
- 压缩策略：移除 progress 消息 → 截断大工具结果 → 压缩工具对 → 优先级删除
- 分层摘要：BM25 + 标签匹配 + 使用频率 + 新近度多因素评分

【关键技术点】
LRU cache, token estimation, multi-level compaction, layered summarization, BM25
```

#### C. 三层记忆系统
```
【项目描述】
设计并实现跨会话的三层记忆系统，支持 BM25 语义搜索。

【技术实现】
- 记忆分层：User Memory / Project Memory / Local Memory
- 搜索算法：Okapi BM25 + IDF + 多因素评分
- 自动分类：关键词规则匹配自动分类
- 数据持久化：JSON + Markdown 双格式，原子写入

【关键技术点】
BM25, TF-IDF, 语义搜索, 自动分类, 原子写入, 三层架构
```

#### D. 状态管理模式
```
【项目描述】
Zustand 风格状态管理，支持不可变更新和订阅者通知。

【技术实现】
- Store 类：泛型容器，set_state(updater) 模式
- Updater 函数：increment_tool_calls(), add_cost() 等
- 订阅机制：subscribe(listener) 返回取消订阅函数
- 全局单例：get_global_store()

【关键技术点】
Zustand pattern, immutable update, observer pattern, 泛型编程
```

#### E. 会话持久化
```
【项目描述】
增量会话持久化，支持崩溃恢复和自动保存。

【技术实现】
- 增量策略：首次全量，后续增量 delta 文件
- 合并阈值：每 N 次增量执行一次全量合并
- 自动保存：AutosaveManager 30s 间隔 + dirty 标记
- 原子写入：临时文件 + rename 确保一致性

【关键技术点】
incremental save, delta file, atomic write, crash recovery
```

#### F. 权限安全体系
```
【项目描述】
多级权限审批体系，路径/命令/编辑全面管控。

【技术实现】
- 路径权限：workspace 内快速路径 + 前缀匹配
- 命令权限：危险命令分类（git reset --hard, rm -rf 等）
- 编辑权限：diff preview + 多级决策（once/turn/all/always）
- 缓存优化：LRU 缓存路径规范化结果

【关键技术点】
permission gate, path normalization, lru_cache, dangerous command detection
```

#### G. TUI 交互设计
```
【项目描述】
事件驱动全屏终端应用，支持输入解析和渲染节流。

【技术实现】
- 事件循环：select/poll 非阻塞输入 + 事件分发
- 输入解析：parse_input_chunk 处理文本/按键事件
- 渲染节流：ThrottledRenderer 60fps 限制
- 信号处理：SIGWINCH 终端 resize 监听

【关键技术点】
event loop, non-blocking I/O, throttling, signal handler, TTY
```

### 12.2 简历项目描述模板

```markdown
## 项目名称：MiniCode Python - 终端 AI 编程助手

### 项目概述
MiniCode Python 是一个参考 Claude Code 设计的终端 AI 编程助手，实现了轻量级的 model->tool->model 智能体循环架构。

### 技术亮点
1. **智能体架构**：实现并发/串行混合工具执行，支持全局异常安全网
2. **上下文管理**：多级渐进压缩（70%→50%→30%）+ BM25 语义记忆搜索
3. **状态管理**：Zustand 风格不可变状态更新，支持订阅者通知
4. **会话持久化**：增量 delta 保存 + 原子写入，支持崩溃恢复
5. **权限安全**：路径/命令/编辑多级审批 + LRU 缓存优化
6. **TUI 实现**：事件驱动架构 + 渲染节流防止闪烁

### 关键技能
Python, 智能体架构设计, 并发编程, 状态管理, BM25 算法, TUI 开发
```

### 12.3 面试可能问题

| 问题 | 考察点 | 回答要点 |
|------|--------|---------|
| Agent Loop 如何处理工具并发？ | 并发编程 | ThreadPoolExecutor，工具分类（只读/写），结果合并 |
| 为什么选择 BM25 而不是 TF-IDF？ | 信息检索 | BM25 对短文本更好，避免词频饱和问题 |
| 状态管理为什么用 Updater 模式？ | 设计模式 | 不可变性，可追踪，可预测，便于调试 |
| 增量保存如何保证一致性？ | 系统设计 | 原子写入（临时文件+rename），崩溃恢复机制 |
| TUI 如何避免闪烁？ | 前端优化 | 节流渲染，脏标记，双缓冲（可选） |

---

## 附录：关键代码索引

### A. Agent Loop 相关

| 功能 | 文件:函数 | 行号 |
|------|----------|------|
| Agent 循环入口 | `agent_loop.py:run_agent_turn()` | 194 |
| 工具执行 | `agent_loop.py:_execute_single_tool()` | 63 |
| 并发执行 | `agent_loop.py:concurrent.futures` | 421-441 |
| 模型调用 | `agent_loop.py:_model_next()` | 169 |

### B. 工具系统相关

| 功能 | 文件:类 | 行号 |
|------|---------|------|
| 工具注册表 | `tooling.py:ToolRegistry` | 265 |
| 工具执行 | `tooling.py:ToolRegistry.execute()` | 293 |
| 工具定义 | `tooling.py:ToolDefinition` | 230 |
| 智能截断 | `tooling.py:_smart_truncate_output()` | 37 |

### C. 状态管理相关

| 功能 | 文件:类/函数 | 行号 |
|------|-------------|------|
| Store 类 | `state.py:Store` | 22 |
| AppState | `state.py:AppState` | 109 |
| Updater 示例 | `state.py:increment_tool_calls()` | 239 |

### D. 记忆系统相关

| 功能 | 文件:函数 | 行号 |
|------|----------|------|
| BM25 算法 | `memory.py:_bm25_score()` | 359 |
| 记忆搜索 | `memory.py:MemoryFile.search()` | 580 |
| TF-IDF | `memory.py:_compute_idf()` | 333 |

### E. 上下文管理相关

| 功能 | 文件:函数 | 行号 |
|------|----------|------|
| Token 估算 | `context_manager.py:estimate_tokens()` | 78 |
| 多级压缩 | `context_manager.py:compact_messages()` | 502 |
| 分层摘要 | `context_manager.py:_build_layered_summary()` | 272 |

---

*文档版本：2.0（深度扩展版）*
*最后更新：2026/05/17*