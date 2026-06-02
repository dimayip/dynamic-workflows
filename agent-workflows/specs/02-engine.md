# Spec 02 — Python Orchestrator 设计

> 父文档：`../DESIGN.md`

## 1. 包结构

```
awe/
├── __init__.py
├── __main__.py                 # python -m awe
├── version.py
├── cli/
│   ├── main.py                 # CLI 入口（typer）
│   ├── commands/               # run/gen/execute/resume/inspect/list/doctor/logs/abort
│   └── prompts/                # awe gen 用的 system prompts
├── core/
│   ├── engine.py               # 主执行循环
│   ├── state.py                # 状态机+持久化
│   ├── context.py              # ExecutionContext
│   ├── schema.py               # workflow.json 校验
│   ├── approval.py             # 审批门
│   ├── interaction.py          # InteractionAdapter（stdio/skill）
│   ├── exceptions.py
│   └── types.py
├── nodes/
│   ├── base.py                 # NodeHandler ABC
│   ├── tool.py
│   ├── code.py                 # 纯计算逃生舱（受限沙箱）
│   ├── subagent_call.py
│   ├── subagent_fan_out.py
│   ├── llm_decision.py
│   ├── llm_plan.py
│   ├── branch.py
│   ├── loop.py
│   ├── sequence.py
│   ├── parallel.py
│   ├── set_var.py
│   ├── transform.py
│   ├── human_approval.py
│   ├── sleep.py
│   └── noop.py
├── schedule/
│   ├── concurrency.py          # 信号量
│   ├── ratelimit.py            # token bucket
│   ├── adaptive.py             # AIMD
│   └── backoff.py              # 退避
├── cli_adapter/
│   ├── base.py                 # CliAdapter ABC
│   ├── claude.py
│   ├── codex.py
│   ├── codebuddy.py
│   ├── registry.py
│   └── tool_mapping.py
├── tools/
│   ├── registry.py
│   ├── fs.py                   # find_files/read_file/write_file
│   ├── shell.py                # run_shell（受限）
│   ├── search.py               # grep/ripgrep
│   ├── git.py                  # git_diff/git_log（只读）
│   └── http.py                 # http_get（只读，可禁用）
├── render/
│   ├── template.py             # Jinja2 sandbox
│   ├── expression.py           # asteval 或自实现 AST 白名单
│   └── refs.py                 # ${var.path} 解析
├── sandbox/
│   ├── __init__.py
│   ├── ast_validator.py        # code 节点 AST 白名单校验
│   ├── runner.py               # 子进程执行 + 资源限制 + 超时
│   ├── builtins.py             # 受限 __builtins__
│   └── allowed_modules.py      # 白名单模块清单
├── observability/
│   ├── logger.py
│   ├── progress.py
│   ├── metrics.py
│   └── trace.py                # OTel 兼容（可选）
└── config/
    ├── settings.py
    └── schemas/
        └── workflow.schema.json
```

## 2. 核心数据类型

```python
# awe/core/types.py
class StepStatus(str, Enum):
    PENDING = "pending"
    READY = "ready"
    RUNNING = "running"
    COMPLETED = "completed"
    FAILED = "failed"
    SKIPPED = "skipped"
    AWAITING_APPROVAL = "awaiting_approval"

@dataclass
class RunMetadata:
    run_id: str
    workflow_path: Path
    started_at: datetime
    config: dict
    cli_versions: dict[str, str]

@dataclass
class StepRecord:
    id: str
    type: str
    status: StepStatus
    started_at: datetime | None
    finished_at: datetime | None
    attempts: int
    last_error: str | None
    output_summary: dict | None     # 完整输出在 variables，此处仅摘要

@dataclass
class SubagentResult:
    task_id: str
    cli: str
    success: bool
    output: dict | None
    raw_stdout: str
    raw_stderr: str
    exit_code: int
    duration_seconds: float
    tokens_in: int | None
    tokens_out: int | None
    cache_hit_tokens: int | None
    error: str | None

@dataclass
class RunResult:
    run_id: str
    success: bool
    final_report_path: Path
    state_path: Path
    duration_seconds: float
    metrics: dict
```

## 3. NodeHandler 接口

```python
# awe/nodes/base.py
class NodeHandler(ABC):
    type_name: str

    @abstractmethod
    async def execute(self, node: dict, ctx: "ExecutionContext") -> "NodeOutput": ...

class NodeOutput(NamedTuple):
    status: StepStatus
    output: Any                       # 写入 output_var
    new_steps: list[dict] | None      # llm_plan 用
    next_action: Literal["abort","await_approval", None] | None
```

每种节点类型对应一个 `*Handler` 类，注册到 `node_dispatcher`。

## 4. ExecutionContext

```python
class ExecutionContext:
    run_id: str
    workflow: dict
    state: StateManager
    variables: VariableSpace
    scheduler: Scheduler
    cli_registry: CliAdapterRegistry
    tool_registry: ToolRegistry
    interaction: InteractionAdapter
    logger: Logger
    metrics: MetricsCollector
    config: Config

    # 工具方法
    def render_template(self, template: str, extra: dict | None = None) -> str: ...
    def evaluate(self, expression: str) -> Any: ...
    def resolve_ref(self, ref: str) -> Any: ...
    def emit_event(self, event: dict) -> None: ...
```

## 5. 主执行循环

```python
async def run(workflow: dict, run_id: str, config: Config,
              interaction: InteractionAdapter,
              on_event: Callable | None = None) -> RunResult:

    # 1. Schema 静态校验
    schema.validate(workflow)

    # 2. 加载/创建状态
    state = StateManager.load_or_create(run_id, workflow, config)

    # 3. 引擎层动态校验：工具/CLI/变量引用/循环依赖
    schema.validate_runtime(workflow, ctx)

    # 4. 构建上下文
    ctx = ExecutionContext(...)

    # 5. 主循环
    queue = ExecutionQueue.from_workflow(workflow["steps"])
    while not queue.is_empty():
        ready = queue.pop_ready(state)
        if not ready:
            await asyncio.sleep(0.05)
            continue

        # 同时刻多个 ready 节点 → 并发启动
        await asyncio.gather(*[execute_step(s, ctx, queue) for s in ready])

    # 6. 生成报告
    return finalize(ctx)
```

## 6. 单步执行细节

```python
async def execute_step(step: dict, ctx: ExecutionContext, queue: ExecutionQueue):
    if not should_execute(step, ctx):
        ctx.state.mark_skipped(step["id"])
        return

    handler = ctx.node_dispatcher.get(step["type"])
    attempt = 0
    while True:
        attempt += 1
        ctx.state.start_step(step["id"], attempt)
        try:
            timeout = step.get("timeout_seconds", ctx.config.default_timeout)
            result = await asyncio.wait_for(handler.execute(step, ctx), timeout=timeout)
            break
        except Exception as e:
            error = classify_error(e)
            policy = step.get("retry", ctx.config.default_retry)
            if should_retry(error, attempt, policy):
                await backoff_sleep(attempt, policy)
                continue
            await handle_terminal_error(step, e, ctx, queue)
            return

    if result.next_action == "await_approval":
        ctx.state.set_status(step["id"], AWAITING_APPROVAL)
        ctx.state.persist()
        answer = await ctx.interaction.request_approval(step, ctx)
        result = process_approval_answer(step, answer)

    if result.new_steps:
        schema.validate_subschema(result.new_steps)
        queue.insert_after(step["id"], result.new_steps,
                           depth=ctx.state.recursion_depth(step["id"]) + 1)

    if step.get("output_var") and result.output is not None:
        ctx.variables.set(step["output_var"], result.output)

    ctx.state.complete_step(step["id"], result)
    ctx.state.persist()
    ctx.emit_event({"type": "step_completed", "id": step["id"]})
```

## 7. 执行队列

`ExecutionQueue`：
- DAG + 拓扑调度
- 节点扁平化为全局表
- 依赖：显式 `depends_on` + 嵌套块隐式依赖（块内顺序、块对外依赖父）
- 并发节点同时返回 ready
- `insert_after(node_id, new_nodes, depth)` 支持 llm_plan 运行时插入

接口：
```python
class ExecutionQueue:
    @classmethod
    def from_workflow(cls, steps: list[dict]) -> "ExecutionQueue": ...
    def pop_ready(self, state: StateManager) -> list[dict]: ...
    def insert_after(self, node_id: str, new_nodes: list[dict], depth: int = 0) -> None: ...
    def is_empty(self) -> bool: ...
```

## 8. 异步模型规则

- 全栈 `asyncio`
- subprocess：`asyncio.create_subprocess_exec`
- 阻塞 I/O：`asyncio.to_thread`（如大文件读写）
- 并发原语：`asyncio.Semaphore` / `asyncio.Lock` / `asyncio.Event`
- **不使用 threading / multiprocessing**（除非工具明确需要）

## 9. 错误分类（详见 06-error-approval.md）

| 类别 | 来源 | 重试 |
|------|------|------|
| `config_error` | schema/引用/工具未注册 | ❌ fail-fast |
| `tool_error` | 工具异常 | 默认重试 1 |
| `network_error` | CLI 网络失败 | 重试 + 指数退避 |
| `rate_limit_error` | 429 | 全局降速 + 重试 |
| `auth_error` | CLI 未登录 / token 过期 | ❌ 升级到人 |
| `model_error` | LLM 输出非 schema | 重试 1 |
| `subprocess_crash` | 子进程崩溃 | 重试 1 |
| `timeout_error` | 节点超时 | 按 retry 配置 |
| `business_error` | subagent 报告失败 | 按 on_error 处理 |

## 10. 入口契约

```python
# awe/__init__.py
from .engine import run

# 公开 API
def run(
    workflow: dict | Path,
    run_id: str | None = None,
    config: Config | None = None,
    interaction: InteractionAdapter | None = None,
    on_event: Callable[[dict], None] | None = None,
) -> RunResult: ...
```

CLI 入口：`awe.cli.main.app`（typer 应用）。

## 11. 性能基线

- 启动 → 退出（空工作流）：< 500ms
- 100 set_var 顺序：< 2s
- 16 并发 subagent 调度开销（不含 LLM 时间）：< 100ms

## 12. 类型与质量

- 全量类型注解，`mypy --strict` 通过
- 公开 API 必须 docstring
- 所有 raise 的异常有错误码
- 单元覆盖 ≥ 80%

## 13. 扩展点

- 新增节点类型：实现 `NodeHandler` + 注册
- 新增工具：`@tool_registry.register("name")` + Schema
- 新增 CLI：实现 `CliAdapter` + 加入 registry
- 新增 InteractionAdapter：实现 ABC（如 HTTP / WebSocket）
- 新增 code 节点支持的 language（v1.0 仅 python，未来可加 javascript via QuickJS / V8）
- 新增白名单模块 / 内置函数：在 `sandbox/allowed_modules.py` / `sandbox/builtins.py` 增加，必须经过安全 review
