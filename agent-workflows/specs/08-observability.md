# Spec 08 — 可观测性

> 父文档：`../DESIGN.md`

## 1. 三层可观测性

```
┌─────────────────────────────────────────┐
│ 日志（logs/orchestrator.jsonl）           │  ← 详细事件流
├─────────────────────────────────────────┤
│ 进度（progress.json）                     │  ← 实时摘要（小文件）
├─────────────────────────────────────────┤
│ 指标（metrics in usage.json + state）     │  ← 数值统计
└─────────────────────────────────────────┘
```

## 2. 结构化日志

### 2.1 日志格式

每行一条 JSON：

```json
{
  "ts": "2026-06-02T19:48:23.123Z",
  "level": "info|warn|error|debug",
  "run_id": "run-...",
  "step_id": "audit",
  "task_id": "audit::3",
  "event": "subagent_started",
  "module": "cli_adapter.claude",
  "payload": { /* 事件载荷 */ }
}
```

### 2.2 标准事件类型

| event | level | 触发时机 |
|-------|-------|---------|
| `run_started` | info | orchestrator 启动 |
| `run_completed` | info | 工作流完成 |
| `run_failed` | error | 工作流失败 |
| `run_aborted` | warn | 用户中止 |
| `step_ready` | debug | 节点进入 ready |
| `step_started` | info | 节点开始 |
| `step_completed` | info | 节点完成 |
| `step_failed` | error | 节点失败 |
| `step_skipped` | info | 节点跳过 |
| `step_retry` | warn | 节点重试 |
| `subagent_started` | info | subagent 启动 |
| `subagent_completed` | info | subagent 完成 |
| `subagent_failed` | warn | subagent 失败 |
| `tool_called` | debug | 工具调用 |
| `approval_requested` | info | 发起审批 |
| `approval_received` | info | 收到审批 |
| `rate_limit_hit` | warn | 触发 429 |
| `concurrency_adjusted` | info | AIMD 调整并发 |
| `llm_plan_expanded` | info | 运行时插入新 steps |
| `state_persisted` | debug | 状态落盘 |

### 2.3 输出位置

- 文件：`logs/orchestrator.jsonl`（必）
- stdout：`--verbose` 时输出（CLI 入口默认 info 级别）
- 转发：通过 `on_event` 回调（用于宿主 Agent 转发用户）

### 2.4 Subagent 子日志

每个 subagent 的完整 stdout/stderr 单独存于 `logs/subagents/<task_id>.log`，方便排查。

### 2.5 日志轮转与体积

- `orchestrator.jsonl` 不轮转（一次运行一个文件）
- 单条目 > 64KB 截断 + 标记 `truncated: true`
- 超大输出（subagent 完整内容）只存 `results/<task_id>.json`，日志只记摘要

## 3. progress.json

实时进度（频繁覆写，小体积）：

```json
{
  "run_id": "run-...",
  "status": "running|awaiting_approval|completed|failed|aborted",
  "started_at": "...",
  "updated_at": "...",
  "elapsed_seconds": 45,
  "eta_seconds": 90,
  "steps": {
    "total": 12,
    "completed": 5,
    "running": ["audit"],
    "failed": [],
    "skipped": []
  },
  "current_step": {
    "id": "audit",
    "type": "subagent_fan_out",
    "progress": "12/16 subagents",
    "started_at": "..."
  },
  "tokens": {
    "in": 12345,
    "out": 6789,
    "cache_hit": 4567
  },
  "cost_estimate_usd": 0.34
}
```

更新时机：每个事件触发后、最少 1s 间隔。

## 4. 指标 metrics

### 4.1 实时指标对象

```python
class MetricsCollector:
    counters: dict[str, int]           # 累加型
    histograms: dict[str, list]        # 分布型
    gauges: dict[str, float]           # 瞬时值

    def inc(self, name: str, n: int = 1, **tags): ...
    def observe(self, name: str, value: float, **tags): ...
    def gauge(self, name: str, value: float, **tags): ...
```

### 4.2 标准指标

| 指标 | 类型 | 说明 |
|------|------|------|
| `awe.steps.total` | counter | 总节点数 |
| `awe.steps.completed` | counter | 完成数 |
| `awe.steps.failed` | counter | 失败数 |
| `awe.steps.duration_seconds` | histogram | 节点耗时分布（按 type 标签） |
| `awe.subagents.total` | counter | 总 subagent 数 |
| `awe.subagents.duration_seconds` | histogram | subagent 耗时（按 cli 标签） |
| `awe.subagents.tokens_in` | counter | 输入 token 累计（按 cli） |
| `awe.subagents.tokens_out` | counter | 输出 token 累计 |
| `awe.subagents.cache_hit_tokens` | counter | 缓存命中 token |
| `awe.rate_limit.hits` | counter | 429 次数（按 cli） |
| `awe.retry.count` | counter | 重试次数（按原因） |
| `awe.approval.requested` | counter | 审批请求次数 |
| `awe.approval.granted` | counter | 通过数 |
| `awe.approval.denied` | counter | 拒绝数 |
| `awe.concurrency.current` | gauge | 当前并发 |
| `awe.cost.usd` | counter | 累计成本估算 |

### 4.3 输出

- 实时：写入 `progress.json` 的子集
- 完整：完成时写入 `usage.json` + `metrics.json`
- 导出：`awe inspect <run_id> --metrics` 输出

## 5. Trace（OpenTelemetry 兼容，可选）

`config.trace.enabled = true` 时：
- 每个节点一个 span
- 每个 subagent 一个子 span
- 通过 OTLP 协议导出（默认 stdout / 可配 endpoint）
- span attributes 含 step_id / task_id / cli / status

实施推荐：用 `opentelemetry-api` + `opentelemetry-sdk`，Backend 可选。

## 6. 终端展示

CLI 入口默认输出：

```
[awe] run-20260602-194510-7f3a 启动
[01/12] discover_files          ✓ 0.3s    findings: 23 files
[02/12] audit (fan_out 16)      ███████░░ 12/16    [12.3s elapsed]
[03/12] triage                  ⏳ awaiting LLM...
[04/12] approval                ⊘ awaiting user...
```

实施推荐：`rich` 库提供进度条 + 状态行。

`--verbose` 切换为日志流模式（每条事件一行）。

`--quiet` 仅输出最终结果路径。

## 7. final-report.md

工作流完成后必生成：

```markdown
# 工作流报告：审计 src 的认证缺陷

**Run ID**: run-20260602-194510-7f3a
**状态**: 完成
**耗时**: 4 分 23 秒
**成本估算**: $0.34

## 执行概要

- 12 个节点全部完成
- 16 个 subagent 调用，平均 4.2s
- 1 次审批通过
- 0 次失败 / 2 次重试

## 关键产出

- 发现 23 处认证缺陷（critical 5，medium 18）
- 已修复 5 处 critical
- 修复后测试通过率 100%

## 时间线

| 节点 | 类型 | 状态 | 耗时 |
|------|------|------|------|
| discover_files | tool | ✓ | 0.3s |
| ... | ... | ... | ... |

## 审批记录

| 节点 | 决策 | 时间 | 来源 |
|------|------|------|------|
| approval | approve | ... | stdin |

## 安全摘要

- 写入路径：src/auth/middleware.ts, src/auth/jwt.ts
- 网络请求：0 次
- Shell 调用：0 次

## 详细数据

- state.json
- results/
- logs/

## 下一步建议

（由最后一个 subagent_call 生成的 summary）
```

## 8. 监控与告警（可选）

`config.monitoring`：
- webhook：完成/失败时 POST 到指定 URL
- email：通过 SMTP 发送报告（v1.1+）

## 9. 调试模式

`--debug`：
- 日志级别 debug
- subagent 命令行打印
- 状态写入更频繁（每次事件）
- 不删除中间临时文件
- 启用 trace

## 10. 性能剖析

`--profile` 启动 cProfile，结束时生成 `profile.prof`，方便定位引擎自身瓶颈。
