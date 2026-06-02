# Spec 10 — awe CLI 入口规范

> 父文档：`../DESIGN.md`

## 1. 命令树

```
awe
├── run "<task>" [options]            # 一站式：生成 + 执行
├── gen "<task>" [options]            # 仅生成 workflow.json
├── execute <workflow.json> [opts]    # 执行已有 workflow
├── resume <run_id> [options]         # 恢复运行
├── inspect <run_id> [options]        # 查看状态
├── list [options]                    # 列出所有 run
├── doctor                            # 检查环境
├── logs <run_id> [options]           # 查看日志
├── abort <run_id>                    # 中止
├── gc [options]                      # 清理旧 run
├── version                           # 版本信息
└── help [command]
```

实施：`typer` 框架。

## 2. 全局选项

| 参数 | 说明 |
|------|------|
| `--config <path>` | 指定配置文件 |
| `--workflow-dir <path>` | 指定 .workflow 根目录（默认 ./.workflow） |
| `--cli-runner <name>` | 强制使用某 CLI（claude/codex/codebuddy） |
| `--max-parallel <N>` | 全局并发上限 |
| `--auto-approve` | 自动批准所有审批 |
| `--ci-mode` | CI/CD 安全模式（禁用交互、shell、http） |
| `--dry-run` | 仅校验，不执行 |
| `--verbose` / `-v` | 详细日志 |
| `--quiet` / `-q` | 仅输出结果路径 |
| `--debug` | 调试模式（日志 debug 级 + 完整 trace） |
| `--no-color` | 禁用彩色输出 |
| `--json-output` | 关键事件输出 JSON 行（机器可读） |

## 3. `awe run "<task>"`

完整流程：检测 CLI → 生成 → 审查 → 执行 → 报告。

```bash
awe run "审计 src 目录的认证缺陷" \
  [--gen-cli claude] \           # 用哪个 CLI 来生成 workflow.json
  [--exec-cli claude] \          # 用哪个 CLI 派 subagent
  [--no-confirm] \               # 跳过 workflow.json 审查
  [--template <name>] \          # 基于模板生成（见 13-templates.md）
  [--from-prompt <file>]         # 从文件读 task
```

退出码：见 §13。

输出（默认）：

```
[awe] 检测可用 CLI: claude ✓ (2.1.154), codex ✓ (1.0.x)
[awe] 生成 workflow.json (使用 claude)...
[awe] ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
[awe] 已生成 12 个节点的工作流
[awe] 类型分布: tool=2, subagent_fan_out=3, llm_decision=1, ...
[awe] 预估 subagent 调用: 16 个（fan_out audit 阶段）
[awe] 预估成本: $0.30 - $1.20（取决于代码量）
[awe] 是否查看完整 JSON? [y/N/edit] n
[awe] 是否执行? [y/N] y
[awe] ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
[awe] Run ID: run-20260602-194510-7f3a
[awe] 工作目录: ./.workflow/run-20260602-194510-7f3a
...
```

## 4. `awe gen "<task>"`

```bash
awe gen "审计 src 的认证缺陷" \
  --cli claude \
  --output workflow.json \
  --schema-only          # 仅输出 schema 不调 LLM
```

输出：写入 workflow.json，stdout 打印路径。

## 5. `awe execute <workflow.json>`

```bash
awe execute workflow.json \
  --interactive stdio|skill \
  --run-id <id>          # 自定义 ID（默认时间戳+随机）
```

`--interactive skill` 时输出格式遵循 09-skill-protocol.md。

## 6. `awe resume <run_id>`

```bash
awe resume run-20260602-194510-7f3a \
  [--from-step <step_id>]   # 强制从指定步骤开始（v1.1+）
```

行为：
- 校验 workflow.json 与 state.json 一致性
- 已完成节点跳过
- 失败节点按 retry 配置重做
- 审批中节点重新询问

## 7. `awe inspect <run_id>`

```bash
awe inspect run-20260602-194510-7f3a \
  [--step <step_id>] \      # 仅看某个节点
  [--metrics] \             # 输出 metrics
  [--results] \             # 输出 subagent 结果
  [--json]                  # JSON 格式
```

输出（默认）：

```
Run: run-20260602-194510-7f3a
状态: completed
开始: 2026-06-02 19:45:10
结束: 2026-06-02 19:48:33
耗时: 3 分 23 秒
成本: $0.34

节点概览:
  ✓ discover_files       0.3s
  ✓ audit (fan_out 16)   45.2s
  ✓ triage              2.1s
  ✓ approval (manual)    -
  ...

变量空间（摘要）:
  files: 23 项
  all_findings: 47 项
  triage_plan: { strategy: "fix_top_n", top_n: 10 }
  ...
```

## 8. `awe list`

```bash
awe list \
  [--status running|completed|failed|all] \
  [--last N] \
  [--since 7d] \
  [--workflow-dir <path>]
```

输出表：run_id / 创建时间 / 状态 / 节点数 / 耗时 / 成本。

## 9. `awe doctor`

```bash
awe doctor [--verbose]
```

检查项：
- Python 版本 ≥ 3.11
- 必须依赖已安装
- 各 CLI 是否安装、登录、版本兼容
- 写入权限（.workflow 目录）
- 配置文件是否合法
- 网络连通性（可选 ping API endpoint）

输出（示例）：

```
awe doctor v1.0.0

环境
  Python ........... 3.11.7    ✓
  Platform ......... darwin-arm64
  awe version ...... 1.0.0     ✓

CLI
  claude ........... 2.1.154   ✓ logged in
  codex ............ 1.0.5     ✓ logged in
  codebuddy ........ -         ✗ not installed

权限
  .workflow/ ....... rw        ✓
  ~/.awe/ .......... rw        ✓

配置
  ~/.awe/config.toml ✓ valid
  ./.awe/config.toml not found

网络
  api.anthropic.com  ✓ reachable
  api.openai.com ... ✓ reachable

总评: 1 项警告（codebuddy 未安装，工作流引用此 CLI 时会失败）
```

## 10. `awe logs <run_id>`

```bash
awe logs run-... \
  [--tail N] \
  [--follow] \
  [--level info|debug|error] \
  [--step <step_id>] \
  [--task <task_id>] \
  [--subagent]      # 切换到 subagent 子日志
```

## 11. `awe abort <run_id>`

```bash
awe abort run-... \
  [--delete]              # 同时删除 .workflow 目录
```

通过文件锁通知运行中的 orchestrator 停止；如果无运行进程，仅修改 state.json 状态。

## 12. `awe gc`

```bash
awe gc \
  --keep-last 20 \
  --older-than 7d \
  --keep-failed false
```

清理：
- 删除非 final-report 的中间文件
- 保留 state.json + final-report.md + usage.json
- `--full` 删除整个 run 目录

## 13. 退出码

| Code | 含义 |
|------|------|
| 0 | 成功 |
| 1 | 一般失败 |
| 2 | 配置错误（schema 无效、引用错误） |
| 3 | 环境问题（CLI 不可用、权限不足） |
| 4 | 用户中止（abort 或 SIGINT） |
| 5 | 资源耗尽 |
| 6 | 网络错误 |
| 7 | 认证错误 |
| 43 | 等待审批超时 |
| 130 | SIGINT |

详细错误码（带 AWE_xxx 前缀）由 logs 与 stderr 给出。

## 14. 配置发现顺序

```
1. --config <path> 指定的文件
2. ./.awe/config.toml （项目级）
3. ~/.awe/config.toml （用户级）
4. /etc/awe/config.toml （系统级，可选）
5. 内置默认值
```

后者覆盖前者；`--xxx` 命令行参数最高优先级。

## 15. 输出模式

### 15.1 默认（人类可读）

彩色 + 表格 + 进度条（`rich` 库）。

### 15.2 `--json-output`

每个事件输出一行 JSON：
```json
{"event":"step_completed","step_id":"audit","duration_seconds":45.2}
```

适合 CI/CD pipe 给监控。

### 15.3 `--quiet`

仅输出最终 final-report.md 路径。

## 16. shell 自动补全

```bash
awe completion bash > /etc/bash_completion.d/awe
awe completion zsh  > ~/.zfunc/_awe
awe completion fish > ~/.config/fish/completions/awe.fish
```

实施：typer 自带支持。

## 17. 安装与分发

- PyPI：`pip install agent-workflows`
- 命令名：`awe`（短）+ `agent-workflows`（长别名）
- 单文件 zipapp：`awe.pyz`（备选分发方式）
- Homebrew formula：`brew install agent-workflows`（社区维护）

## 18. CLI 测试要点

- 每个命令必须有：`awe <cmd> --help` 测试
- run/execute：用 mock CLI 适配器跑端到端
- resume：模拟中断后恢复
- inspect/list/logs：基于固定 fixture 输出对比
- doctor：模拟各 CLI 状态
- 跨平台测试：Linux / macOS / Windows
