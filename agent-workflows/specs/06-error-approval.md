# Spec 06 — 错误处理、重试与审批门

> 父文档：`../DESIGN.md`

## 1. 错误分类矩阵

| 类别 | 来源 | 默认行为 | 是否可重试 |
|------|------|---------|-----------|
| `config_error` | schema 校验失败、引用未定义、循环依赖 | fail-fast | ❌ |
| `tool_error` | Python 工具执行异常 | retry 1 → escalate | ✅ |
| `network_error` | CLI 网络失败、连接超时 | exp backoff retry 3 | ✅ |
| `rate_limit_error` | 429 | 全局降速 + retry | ✅ |
| `auth_error` | CLI 未登录、token 过期 | fail-fast，提示用户 | ❌ |
| `model_error` | LLM 输出非 schema、内容截断 | retry 1 + 纠正提示 | ✅ |
| `subprocess_crash` | 子进程崩溃、信号杀死 | retry 1 | ✅ |
| `timeout_error` | 节点超时 | 按 retry.on 配置 | 视配置 |
| `business_error` | subagent 报告业务失败 | 按 on_error 处理 | 视配置 |
| `approval_denied` | 用户拒绝审批 | 按 on_error 处理 | ❌ |
| `internal_error` | 引擎自身 bug | 写日志 → fail-fast | ❌ |

实现：`awe/core/exceptions.py` 定义异常层级，`awe/cli_adapter/base.py` 的 `normalize_error` 把 stderr 映射到类别。

## 2. 异常层级

```python
class AweError(Exception):
    code: str            # AWE_xxx 错误码
    category: str
    suggestion: str | None

class ConfigError(AweError): ...
class ToolError(AweError): ...
class NetworkError(AweError): ...
class RateLimitError(AweError):
    retry_after: int | None
class AuthError(AweError): ...
class ModelOutputError(AweError):
    raw_output: str
    schema: dict
class SubprocessCrash(AweError): ...
class TimeoutError(AweError): ...
class BusinessError(AweError):
    details: dict
class ApprovalDenied(AweError): ...
class InternalError(AweError): ...
```

## 3. 错误码命名规约

`AWE_<MODULE>_<NUMBER>`：
- DSL：`AWE_DSL_001..099`
- ENGINE：`AWE_ENG_001..099`
- CLI：`AWE_CLI_001..099`
- TOOL：`AWE_TOOL_001..099`
- STATE：`AWE_STATE_001..099`
- AUTH：`AWE_AUTH_001..099`
- NET：`AWE_NET_001..099`

实施时维护 `specs/_error_codes.md`，每个码含：含义、典型原因、修复建议。

## 4. 重试策略

```json
{
  "retry": {
    "max_attempts": 3,
    "backoff": "constant|linear|exponential",
    "backoff_seconds": 2,
    "max_backoff_seconds": 60,
    "jitter_seconds": 1,
    "on": ["rate_limit", "network", "model_error", "timeout", "subprocess_crash"]
  }
}
```

退避计算：
- constant：`backoff_seconds`
- linear：`backoff_seconds * attempt`
- exponential：`min(max, backoff_seconds * 2^(attempt-1))`
- 实际等待 = 计算值 + uniform(0, jitter_seconds)

## 5. 错误升级路径

```
节点失败
  ↓
按 retry.on 判断是否可重试
  ↓ 不可重试 / 重试耗尽
按 on_error 处理：
  - abort：终止整个工作流
  - skip：标记跳过，继续下游（依赖此节点的下游也跳过）
  - escalate：转为 human_approval 节点（特殊态），让用户决定
  - fallback：执行 fallback_step 指定的节点
```

`escalate` 实施细节：
- 引擎暂停，构造一个临时 human_approval 节点
- 选项：`retry`、`skip`、`abort`、`edit_workflow`
- 用户选 `edit_workflow` → 写状态后退出，让用户手动改 workflow.json + resume

## 6. fan_out 失败聚合

```
fail_strategy:
  - all_or_nothing：任一 task 失败 → 节点失败
  - best_effort：失败的标记 null，结果数组带 _failed_count
  - min_success_rate：成功率 < min_success_rate → 节点失败
```

聚合后的输出形态：

```json
{
  "results": [...],            // 成功结果
  "_meta": {
    "total": 16,
    "succeeded": 14,
    "failed": 2,
    "failed_tasks": [
      { "task_id": "audit::3", "error": "...", "category": "rate_limit" }
    ]
  }
}
```

下游节点可访问 `audit._meta.succeeded` 等字段做判断。

## 7. 审批门机制

### 7.1 触发场景

| 场景 | 处理 |
|------|------|
| 显式 `human_approval` 节点 | 直接发起审批 |
| `on_error: escalate` | 错误升级时构造临时审批 |
| llm_plan 生成的新 steps 含敏感操作 | 强制插入审批节点 |
| `approval_policy: always_ask` | 每个 subagent_fan_out 前自动询问 |
| `approval_policy: ask_on_destructive` | edit_policy != readonly 时询问 |
| `approval_policy: auto` | 不询问 |

### 7.2 审批协议（InteractionAdapter）

```python
class InteractionAdapter(ABC):
    @abstractmethod
    async def request_approval(self, req: ApprovalRequest) -> ApprovalAnswer: ...

    @abstractmethod
    async def emit_progress(self, event: dict) -> None: ...

    @abstractmethod
    async def emit_log(self, level: str, msg: str, context: dict) -> None: ...

@dataclass
class ApprovalRequest:
    step_id: str
    title: str
    summary: str
    details: str
    options: list[str]
    timeout_seconds: int | None

@dataclass
class ApprovalAnswer:
    decision: str
    timestamp: datetime
    by: str            # "stdin"|"skill"|"auto"|"timeout"
    note: str | None
```

### 7.3 StdioAdapter 实现

```python
class StdioAdapter(InteractionAdapter):
    async def request_approval(self, req):
        print(f"\n[审批请求] {req.title}")
        print(f"摘要: {req.summary}")
        print(f"详情:\n{req.details}")
        print(f"选项: {' / '.join(req.options)}")
        # 阻塞读 stdin
        loop = asyncio.get_event_loop()
        ans = await loop.run_in_executor(None, input, "答复> ")
        return ApprovalAnswer(decision=ans.strip().lower(), ...)
```

### 7.4 SkillProtocolAdapter 实现

```python
class SkillProtocolAdapter(InteractionAdapter):
    async def request_approval(self, req):
        # 通过 stdout 输出特殊标记的 JSON 行
        marker = {
            "type": "awe.approval_request",
            "version": 1,
            "payload": asdict(req),
        }
        print(f"<<AWE:JSON>>{json.dumps(marker)}<<END>>", flush=True)
        # 等 stdin 接收答复（宿主写回）
        line = await self._read_marked_line("awe.approval_answer")
        return ApprovalAnswer(**line["payload"])
```

宿主侧 SKILL.md 必须说明：看到 `<<AWE:JSON>>{type:"awe.approval_request", ...}<<END>>` 后向用户提问，得到答复后写回 `<<AWE:JSON>>{type:"awe.approval_answer", payload:{decision:"..."}}<<END>>`。

### 7.5 自动批准与超时

- `auto_approve_when` 表达式为真 → 立即返回 approve
- `timeout_seconds` 到 → 按 `on_timeout` 处理（默认 reject）
- `--auto-approve` 全局开关 → 等同 `approval_policy: auto`

### 7.6 审批审计

每次审批记录到 `state.approvals`：
```json
{
  "step_id": "approval",
  "decision": "approve",
  "at": "...",
  "by": "stdin|skill|auto|timeout",
  "note": "用户备注（可选）",
  "options_shown": ["approve","reject"],
  "request_summary": "..."
}
```

完成后写入 `final-report.md` 的"审批记录"章节。

## 8. 失败可视化

- 终端：错误码 + 类别 + 简短建议
- progress.json：当前失败步骤 ID 与原因
- final-report.md：失败章节列出所有失败步骤

## 9. 调试协助

- `awe inspect <run_id> --step <step_id>`：展示该步骤完整 state + result + log
- `awe logs <run_id> --tail 100`：实时跟踪日志
- `awe replay <run_id> --from-step <step_id>`：从指定步骤重放（v1.1+）
