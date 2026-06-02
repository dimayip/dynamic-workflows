# Spec 05 — 状态管理、Checkpoint、Resume

> 父文档：`../DESIGN.md`

## 1. 与 agent-workflows 的差异

agent-workflows 是**节点级**状态管理（DSL 节点完成度天然就是 checkpoint）；
open-workflows 是**代码级**状态管理（脚本是图灵完备代码，需要显式 checkpoint）。

## 2. 运行时目录

```
.ow/<run_id>/
├── orchestrator.ts            # 不可变（脚本原文）
├── orchestrator.compiled.js   # esbuild 输出（运行时使用）
├── dag.json                   # 从 AST 提取的 DAG 视图
├── meta.json                  # 元信息
├── state.json                 # 主状态（运行时）
├── state.json.lock            # 进程锁
├── progress.json              # 实时进度
├── usage.json                 # token 与成本
├── kv.json                    # workflow.kv 持久化
├── checkpoints/
│   ├── after-audit.json       # 用户脚本中 checkpoint("after-audit", ...) 的快照
│   └── ...
├── subagents/
│   └── <task_id>.json         # 每 subagent 完整结果
├── logs/
│   ├── orchestrator.jsonl     # 主日志
│   └── subagents/<task_id>.log
└── final-report.md
```

## 3. state.json schema

```json
{
  "run_id": "run-...",
  "version": "1.0",
  "status": "running|completed|failed|aborted|awaiting_approval",
  "started_at": "...",
  "updated_at": "...",
  "finished_at": null,
  "script_hash": "sha256:...",
  "sdk_version": "1.0",
  "cli_versions": { "claude": "2.x", ... },
  "spawn_count": 23,
  "total_tokens": { "in": 12345, "out": 6789, "cache": 4567 },
  "checkpoints": [
    { "name": "after-audit", "at": "...", "data_path": "checkpoints/after-audit.json" }
  ],
  "approvals": [
    { "id": "...", "decision": "approve", "at": "..." }
  ],
  "errors": []
}
```

## 4. checkpoint 机制

### 4.1 用户视角（脚本内调用）

```typescript
const findings = await Promise.all(...);
await workflow.checkpoint("after-audit", { findings });

const fixes = await Promise.all(...);
await workflow.checkpoint("after-fix", { fixes });
```

### 4.2 引擎视角

```typescript
async function checkpoint(name: string, data: object) {
  const path = `.ow/${runId}/checkpoints/${name}.json`;
  await atomicWrite(path, JSON.stringify({
    name,
    at: new Date().toISOString(),
    spawn_count_at_checkpoint: state.spawn_count,
    data,
  }));
  state.checkpoints.push({ name, at: ..., data_path: path });
  await persistState();
}
```

## 5. resume 机制

由于脚本是 TypeScript 代码（图灵完备），**不能简单"从断点跳过"**。可行策略：

### 5.1 策略 A：Checkpoint 重放

适用：脚本设计为"checkpoint 之间幂等"

```typescript
async function main() {
  // 重启时跳过已完成阶段
  const findings = workflow.config.resumeFrom !== "after-audit"
    ? await runAuditPhase()
    : (await workflow.loadCheckpoint("after-audit")).findings;

  await workflow.checkpoint("after-audit", { findings });

  const fixes = workflow.config.resumeFrom !== "after-fix"
    ? await runFixPhase(findings)
    : (await workflow.loadCheckpoint("after-fix")).fixes;
  // ...
}
```

`workflow.config.resumeFrom` 是 `ow resume <run_id>` 时引擎注入的最近完成的 checkpoint 名。

### 5.2 策略 B：完整重跑（默认）

如果脚本不主动用 `loadCheckpoint`，重跑整个脚本，但：
- subagent 结果有缓存（同 prompt + 同 system → 直接返回缓存结果）
- 已 spawn 过的任务记入 spawn_count 计入限额
- subagent 缓存策略：

```
subagent 输入指纹 = sha256(role + system + prompt + tools + cli)
缓存命中 → 直接返回（不实际派 CLI）
缓存有效期：默认 1 小时（用户可配）
```

### 5.3 策略选择

| 场景 | 推荐策略 |
|------|---------|
| 短任务 < 10 min | 完整重跑（B） |
| 长任务 / 失败重启 | 显式 checkpoint（A） |
| LLM 生成的标准模板 | 模板内置 checkpoint（A） |

LLM 生成 orchestrator.ts 时，prompt 中提示："长任务请在阶段间使用 workflow.checkpoint()"。

## 6. 缓存系统

### 6.1 缓存键

```
sha256(JSON.stringify({
  role, cli, systemPrompt, prompt,
  tools: sorted(tools),
  editPolicy, allowedPaths: sorted(allowedPaths),
  outputSchema,
}))
```

### 6.2 缓存存储

```
.ow/<run_id>/subagents/<task_id>.json
.ow/cache/<sha256>.json     # 全局缓存（用户级 ~/.ow/cache）
```

### 6.3 缓存命中规则

- 命中条件：键完全相同
- 时效：默认 1 小时（重启后还在），用户配置可关
- resume 时：spawn 调用先查全局缓存

### 6.4 关闭缓存

- 工作流级：`workflow.config.cache = false`
- 全局级：`ow run --no-cache`
- subagent 级：`spawn(role, { cache: false })`

## 7. 原子写入

```typescript
async function atomicWrite(path: string, data: string) {
  const tmp = path + ".tmp";
  await fs.writeFile(tmp, data, "utf8");
  await fs.rename(tmp, path);
}
```

每次 state / checkpoint / kv 变更都用此函数。

## 8. 进程锁

```typescript
import lockfile from "proper-lockfile";

async function acquireRunLock(runDir: string) {
  return await lockfile.lock(runDir, {
    retries: 0,
    onCompromised: () => {
      log.error("锁丢失，可能是另一进程在操作此 run");
      process.exit(2);
    },
  });
}
```

防止：同一 run_id 被多个 ow 进程同时操作。

## 9. ow resume 命令

```bash
ow resume <run_id>
  [--from-checkpoint <name>]   # 强制从指定 checkpoint 开始
  [--no-cache]                 # 强制重跑所有 subagent
```

实施流程：

```
1. 加载 .ow/<run_id>/state.json
2. 校验 script_hash 与 orchestrator.ts 一致
3. 校验 sdk_version 兼容
4. 注入 workflow.config.resumeFrom = 最近 checkpoint 名
5. 启动沙箱执行
6. SDK 调用查缓存 → 命中跳过 → 不命中执行
```

## 10. 跨会话恢复约束

- orchestrator.ts 不可在运行后修改（修改 → script_hash 不匹配 → 拒绝 resume）
- script 修改后只能 `ow run` 启动新 run
- checkpoint 数据格式由用户脚本约定（引擎不强制结构）

## 11. 长任务最佳实践（写入 SKILL.md）

LLM 生成的 orchestrator.ts 应满足：
- 每个独立阶段 endwith `await workflow.checkpoint(...)`
- 检查 `workflow.config.resumeFrom` 决定是否重做该阶段
- 不在 checkpoint 之间持有大量内存状态（写入 kv 或 checkpoint）

模板片段：

```typescript
async function maybeRun<T>(name: string, fn: () => Promise<T>): Promise<T> {
  if (workflow.config.resumeFrom === name) {
    log.info(`从 checkpoint 恢复 ${name}`);
    return (await workflow.loadCheckpoint(name)).data as T;
  }
  const data = await fn();
  await workflow.checkpoint(name, { data });
  return data;
}

const findings = await maybeRun("audit", () => runAudit());
const fixes = await maybeRun("fix", () => runFix(findings));
```

## 12. 跨 run 共享状态

`.ow/cache/` 全局共享（用户级）；`.ow/<run_id>/` 仅当前 run。

未来可加 `~/.ow/runs/index.json` 维护 run 索引（v1.1+）。
