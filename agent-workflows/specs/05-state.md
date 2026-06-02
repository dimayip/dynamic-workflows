# Spec 05 — 状态管理与断点续跑

> 父文档：`../DESIGN.md`

## 1. .workflow/<run_id>/ 目录结构

```
.workflow/<run_id>/
├── workflow.json              # 不可变（运行期间不修改）
├── workflow.expanded.json     # 含 llm_plan 展开的所有 steps（可选）
├── state.json                 # 主状态文件（频繁更新）
├── state.json.lock            # 进程锁（fcntl/msvcrt）
├── results/
│   ├── <task_id>.json         # 每个 subagent 的完整输出
│   └── ...
├── logs/
│   ├── orchestrator.jsonl     # 引擎结构化日志
│   ├── subagents/
│   │   └── <task_id>.log      # subagent stdout+stderr
│   └── ...
├── usage.json                 # token 与成本累计
├── progress.json              # 实时进度（小文件，频繁覆写）
├── final-report.md            # 最终报告
└── meta.json                  # 元信息（创建时间、CLI 版本、awe 版本等）
```

## 2. state.json schema

```json
{
  "run_id": "run-20260602-194510-7f3a",
  "version": "1.0",
  "status": "running|completed|failed|aborted|awaiting_approval",
  "started_at": "2026-06-02T19:45:10Z",
  "updated_at": "2026-06-02T19:48:23Z",
  "finished_at": null,
  "current_steps": ["audit"],
  "queue_snapshot": {
    "pending": ["fix", "verify", "report"],
    "completed": ["discover"],
    "failed": [],
    "skipped": []
  },
  "steps": {
    "discover": {
      "status": "completed",
      "started_at": "...",
      "finished_at": "...",
      "attempts": 1,
      "last_error": null,
      "output_summary": { "files_count": 23 }
    },
    "audit": {
      "status": "running",
      "started_at": "...",
      "finished_at": null,
      "attempts": 1,
      "subagent_tasks": {
        "audit::0": { "status": "completed", "duration": 4.2 },
        "audit::1": { "status": "running" },
        "...": "..."
      }
    }
  },
  "variables": {
    "files": ["src/a.ts", "src/b.ts"],
    "all_findings": [ /* ... */ ]
  },
  "approvals": [
    { "step_id": "approval", "decision": "approve", "at": "...", "by": "stdin" }
  ],
  "expanded_steps": [
    { "parent": "dynamic_followup", "depth": 1, "steps": [...] }
  ],
  "recursion_depth": { "dynamic_followup": 1 }
}
```

## 3. 原子写入

```python
# awe/core/state.py
def atomic_write(path: Path, data: str):
    tmp = path.with_suffix(path.suffix + ".tmp")
    tmp.write_text(data, encoding="utf-8")
    if os.name == "posix":
        os.fsync(tmp.open("rb").fileno())
    tmp.replace(path)
```

写入时机：
- 节点状态变更（start / complete / fail / skip）
- 变量空间变更（output_var 写入）
- 审批决策记录
- llm_plan 插入新 steps
- 周期性快照（每 5s 一次，防进程崩溃丢失）

## 4. 进程锁

防止同一 run_id 被多个 orchestrator 同时操作：

```python
import fcntl  # Linux/macOS
class StateLock:
    def __init__(self, path: Path):
        self.path = path
        self.fd = None

    def __enter__(self):
        self.fd = open(self.path, "w")
        try:
            fcntl.flock(self.fd, fcntl.LOCK_EX | fcntl.LOCK_NB)
        except BlockingIOError:
            self.fd.close()
            raise StateLockedError(f"另一进程持有锁: {self.path}")
        return self

    def __exit__(self, *a):
        fcntl.flock(self.fd, fcntl.LOCK_UN)
        self.fd.close()
```

Windows 用 `msvcrt.locking`。跨平台封装在 `awe/core/state.py`。

## 5. 断点续跑

```python
# awe resume <run_id>
async def resume(run_id: str, config: Config) -> RunResult:
    run_dir = locate_run_dir(run_id)
    if not run_dir.exists():
        raise RunNotFound(run_id)

    workflow = json.loads((run_dir / "workflow.json").read_text())
    state = StateManager.load(run_dir)

    # 校验：workflow.json 与 state.json 的版本兼容
    if state.workflow_hash != hash_workflow(workflow):
        raise WorkflowMismatch("workflow.json 与状态不匹配")

    # 校验：之前在审批中？
    if state.status == "awaiting_approval":
        # 重新发起审批询问
        ...

    # 重建队列：跳过 completed / skipped，重做 failed（按 retry 配置）
    queue = ExecutionQueue.from_state(state, workflow)

    # 启动主循环（同 run）
    return await engine.run(workflow, run_id, config, ...)
```

## 6. Subagent 结果持久化

每个 subagent 完成后立即写入 `results/<task_id>.json`：

```json
{
  "task_id": "audit::0",
  "step_id": "audit",
  "cli": "claude",
  "status": "success|failed",
  "output": { /* 解析后的结构化输出 */ },
  "raw_stdout": "...",
  "raw_stderr": "...",
  "duration_seconds": 4.2,
  "tokens_in": 1234,
  "tokens_out": 567,
  "cache_hit_tokens": 890,
  "cost_estimate_usd": 0.012,
  "error": null,
  "started_at": "...",
  "finished_at": "..."
}
```

读取时机：
- 节点完成后聚合结果
- `awe inspect` 命令展示
- `usage.json` 汇总
- 跨会话 resume 时恢复 fan_out 中已完成的 task

## 7. usage.json

```json
{
  "run_id": "...",
  "by_cli": {
    "claude": { "tokens_in": 12345, "tokens_out": 6789, "cache_hit_tokens": 4567, "calls": 23, "cost_estimate_usd": 0.34 }
  },
  "by_step": {
    "audit": { "calls": 16, "tokens_in": 8000, "tokens_out": 3000 }
  },
  "totals": { "calls": 23, "cost_estimate_usd": 0.34 }
}
```

成本估算：基于公开费率表（实施时维护一份 `awe/config/pricing.json`，可关闭）。

## 8. progress.json

实时进度（每次状态变更覆写，体积小）：

```json
{
  "run_id": "...",
  "status": "running",
  "completed_steps": 5,
  "total_steps": 12,
  "current_step": "audit",
  "current_step_progress": "12/16 subagents",
  "elapsed_seconds": 45,
  "eta_seconds": 90
}
```

## 9. meta.json

```json
{
  "run_id": "...",
  "awe_version": "1.0.0",
  "python_version": "3.11.4",
  "platform": "darwin-arm64",
  "cli_versions": { "claude": "2.x", "codex": "..." },
  "cwd_at_start": "/Users/...",
  "config_resolved": { /* 完整解析后的配置 */ },
  "created_by": "skill|cli",
  "host_agent": "claude-code|codex|codebuddy|none"
}
```

## 10. 跨会话恢复约束

- workflow.json 在创建后**不可修改**（含 llm_plan 展开后的内容写入 expanded.json，原文件不变）
- state.json 含 workflow_hash，恢复时校验
- 已完成节点不重做
- 失败节点按 retry 配置重做
- 审批中节点恢复后重新询问
- 运行中节点（subagent_fan_out 中部分完成）：已完成 task 不重做，未完成 task 重新派发

## 11. 清理策略

- 默认保留所有 run（用户手动 `awe abort --delete <run_id>`）
- `awe gc --keep-last N --older-than 7d` 清理
- 清理时保留 final-report.md 与 usage.json，删除大体积日志

## 12. 多机/多用户

v1.0 不支持多机共享 .workflow/。若有需求：
- 状态后端抽象为 `StateBackend` 接口
- 默认 `FileStateBackend`，未来可扩展 `S3StateBackend` / `RedisStateBackend`
