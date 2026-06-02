# Agent Workflows — 产品级设计文档（总览）

> **项目代号**：`agent-workflows`（占位名，可重命名）
> **版本**：v1.0 设计冻结稿
> **状态**：待实施
> **更新**：2026-06-02

## 文档地图

| 文件 | 内容 |
|------|------|
| `DESIGN.md`（本文档） | 总览、目标、原则、架构 |
| `specs/01-dsl.md` | workflow.json DSL 完整规范 |
| `specs/02-engine.md` | Python orchestrator 架构与接口 |
| `specs/03-cli-adapter.md` | CLI subagent 适配层 |
| `specs/04-concurrency.md` | 并发、限流、429 处理 |
| `specs/05-state.md` | 状态管理、断点续跑、恢复 |
| `specs/06-error-approval.md` | 错误处理、重试、审批门 |
| `specs/07-security.md` | 安全模型 |
| `specs/08-observability.md` | 日志、指标、进度 |
| `specs/09-skill-protocol.md` | SKILL.md 协议规范 |
| `specs/10-cli-entry.md` | awe CLI 命令规范 |
| `specs/11-config.md` | 配置层级与项 |
| `specs/12-layout.md` | 项目结构与运行时目录 |
| `specs/13-templates.md` | 内置模板与示例 |
| `ADR.md` | 关键决策记录 |
| `IMPL-CHECKLIST.md` | 实施检查清单 |

> 实施顺序：阅读 `DESIGN.md` → `specs/01..13` → `ADR.md` → 按 `IMPL-CHECKLIST.md` 执行。

---

## 1. 设计目标

### 1.1 一句话定位

**跨 AI Agent 平台的、用 JSON DSL 表达、Python 引擎确定性执行、CLI subagent 实现真并发的动态工作流编排系统。Skill 与 CLI 双入口共享同一引擎。**

### 1.2 核心目标 G1–G8

| # | 目标 | 含义 |
|---|------|------|
| G1 | 编排逻辑代码化 | 控制流由 Python 确定性执行，不依赖宿主 LLM 自觉 |
| G2 | 软约束最小化 | LLM 仅在两处出现：生成 workflow.json、单 subagent 任务 |
| G3 | 跨平台 | 支持 claude / codex / codebuddy CLI，可扩展 |
| G4 | 真并发 | asyncio + 子进程，等价 V8 内并发 |
| G5 | 可恢复 | 跨会话/跨重启/跨日断点续跑 |
| G6 | 可审计 | 每步可观测、可重放、可静态分析 |
| G7 | 安全默认 | 工具白名单、路径白名单、表达式沙箱、密钥隔离 |
| G8 | 双入口共引擎 | Skill 与 CLI 共用 orchestrator |

### 1.3 设计原则 P1–P5

- **P1 显式优于隐式**：所有控制流、错误处理、审批门必须 DSL 显式声明
- **P2 失败优于带病**：schema 校验失败、CLI 不可用 → fail-fast
- **P3 小核心 + 可扩展**：节点类型封闭，扩展靠 `llm_plan`（运行时插入 steps）+ `tool` 注册（工具）+ `code` 节点（纯计算逃生舱）
- **P4 无状态可重入**：任何时刻 kill，state.json 可精确恢复
- **P5 文档即契约**：SKILL.md / workflow.schema.json / CLI 协议三者构成对外契约

### 1.4 与现有方案对比

| 项 | Claude Code DW | codex-dynamic-workflows | **本方案** |
|---|---|---|---|
| 编排载体 | JS（图灵完备） | 文档（LLM 自觉） | JSON DSL（受限 + llm_plan + code 节点） |
| 控制流 | V8 沙箱 | LLM 推理 | Python 引擎 |
| Subagent | 内置 spawn | 模拟 | CLI 子进程 |
| 跨平台 | ❌ | ⚠️ | ✅ |
| 真并发 | ✅ 进程内 | ❌ | ✅ asyncio+subproc |
| 中间隔离 | ✅ 脚本变量 | ❌ 污染主对话 | ✅ 进程隔离 |
| 跨会话恢复 | ❌ | 弱 | ✅ |
| 上限 | ~1000 | ~10 | ~256 |
| 复杂纯计算表达力 | ✅ JS 任意 | ❌ | ✅ code 节点（受限沙箱内任意 Python 纯函数） |
| 整体可视化 | ❌（看代码） | ⚠️ | ✅ DAG + code 节点局部 |
| 安全边界 | V8 | 无 | OS + 工具白名单 + code 沙箱 |

### 1.5 不在范围

- 不内置 LLM SDK 直调（必须经 CLI）
- 不实现自有 Agent CLI（复用现有）
- 不实现 GUI
- 不实现工作流市场 / 远程托管

---

## 2. 系统架构

### 2.1 架构图

```
              用户
               │
        ┌──────┴──────┐
        ▼             ▼
   宿主 AI Agent   awe CLI
        │             │
   SKILL.md       cli/main.py
        │             │
        └──────┬──────┘
               ▼
       LLM 生成 workflow.json
       Schema 校验 + 用户审批
               ▼
   ┌───────────────────────────┐
   │    Python Orchestrator    │
   │  Engine | Scheduler |     │
   │  CLIAdapter | State |     │
   │  Tool | Render | Approval │
   │  Observability            │
   └────────────┬──────────────┘
                │ asyncio.gather
       ┌────────┼────────┐
       ▼        ▼        ▼
    claude   codex   codebuddy
    (隔离子进程)
       │        │        │
       └────────┼────────┘
                ▼
        .workflow/<run_id>/
        ├ workflow.json
        ├ state.json
        ├ results/<task_id>.json
        ├ logs/
        ├ usage.json
        └ final-report.md
```

### 2.2 核心组件

| 组件 | 职责 | 位置 |
|------|------|------|
| 入口层 | 触发并启动 | `SKILL.md` + `awe/cli/main.py` |
| 生成层 | LLM 生成 JSON | 宿主 / `awe gen` |
| Schema 校验 | 拒绝非法配置 | `awe/core/schema.py` |
| Engine | 节点调度循环 | `awe/core/engine.py` |
| Scheduler | 并发与限流 | `awe/schedule/` |
| CLI Adapter | CLI 统一接口 | `awe/cli_adapter/` |
| Tool Registry | 工具白名单 | `awe/tools/` |
| State Manager | 持久化与恢复 | `awe/core/state.py` |
| Render | 模板与表达式 | `awe/render/` |
| Approval Handler | 审批门交互 | `awe/core/approval.py` |
| InteractionAdapter | stdio / skill 协议 | `awe/core/interaction.py` |
| Observability | 日志、指标、进度 | `awe/observability/` |

### 2.3 数据流（核心）

```
1. 用户描述任务
2. LLM + SKILL.md/awe gen → workflow.json
3. Schema 校验（失败回灌让 LLM 修正）
4. 创建 .workflow/<run_id>/，保存 workflow.json
5. 用户审批（必要时）
6. orchestrator 启动 → 加载 workflow.json + state.json
7. Engine 主循环：取节点 → 校验依赖 → 调 NodeHandler → 写 state → 输出进度
8. fan_out / subagent_call：Scheduler 申请额度 → CLI Adapter 派子进程 → 收集 → 解析
9. human_approval：通过 InteractionAdapter 阻塞等答复
10. llm_plan：派 subagent 生成新 steps → 校验 → 插入队列
11. 完成 → 生成 final-report.md → 退出码 0
```

### 2.4 进程模型

| 项 | 说明 |
|----|------|
| 主进程 | orchestrator.py 长驻 |
| 子进程 | 每 subagent 一个 CLI 进程，`asyncio.create_subprocess_exec` |
| 通信 | stdin（prompt）+ stdout（结构化结果）+ stderr（日志）+ exit code |
| 工作目录 | 默认共享主目录；节点可配 `subagent_cwd` 隔离 |
| 信号 | SIGTERM/SIGINT → 优雅停止：等当前批 → 写状态 → 退出 |
| 重启 | `awe resume <run_id>` 从 state.json 恢复 |

---

## 3. 双入口设计

### 3.1 入口对比

| 维度 | Skill 入口 | CLI 入口 |
|------|-----------|---------|
| 触发 | 宿主加载 SKILL.md | shell `awe run` |
| 生成 | 宿主 LLM | awe 调用 LLM CLI |
| 交互 | 宿主转交 | stdin/stdout 直接 |
| 审批 | 宿主转交 | stdin 阻塞 |
| 进度 | 宿主转发日志 | 直接 print（可选 TUI） |
| 场景 | 对话式、IDE | 程序、CI/CD、定时 |

### 3.2 共享 orchestrator API

两入口最终都调用：
```
awe.engine.run(
    workflow: dict | Path,
    run_id: str | None,
    config: Config,
    interaction: InteractionAdapter,
    on_event: Callable | None,
) -> RunResult
```

差异完全集中在 `InteractionAdapter`：
- `StdioAdapter`：直接 print + input
- `SkillProtocolAdapter`：输出特殊标记的 JSON 行（`<<AWE:APPROVAL>>{...}<<END>>`），由宿主解析转交

### 3.3 Skill 入口流程

```
1. 用户："帮我审计 src 安全问题"
2. 宿主加载 SKILL.md，识别为工作流任务
3. 宿主按 SKILL.md 引导，让 LLM 生成 workflow.json
4. 宿主调用 awe execute <workflow.json>
5. orchestrator 输出结构化日志，宿主解析转发用户
6. 遇 <<AWE:APPROVAL>>，宿主向用户提问
7. 用户回答 → 宿主写入 stdin
8. 完成 → 宿主读 final-report.md → 总结给用户
```

### 3.4 CLI 入口流程

```
$ awe run "审计 src 的安全问题"
[awe] 检测可用 CLI: claude ✓, codex ✓
[awe] 生成 workflow.json...
[awe] 已生成（12 步），查看? [y/N/edit] y
       <显示 JSON>
[awe] 是否执行? [y/N] y
[awe] 启动 run-20260602-194510-7f3a
[01/12] discover_files     ✓ (0.3s)
[02/12] audit_fan_out      → 派 16 个 subagent ...
       ...
[awe] 审批：发现 23 处问题，是否修复? [y/N] y
       ...
[12/12] generate_report    ✓ (1.2s)
[awe] 完成: .workflow/run-.../final-report.md
```

---

## 4. 核心设计决策摘要

详见 `ADR.md`，此处仅列结论：

| # | 决策 | 选择 |
|---|------|------|
| ADR-001 | 引擎语言 | Python 3.11+（asyncio 强、AI 生态、跨平台） |
| ADR-002 | DSL 格式 | JSON（schema 工具链成熟，LLM 易生成） |
| ADR-003 | Subagent 派生 | CLI 子进程（不直调 SDK） |
| ADR-004 | 异步模型 | asyncio 全栈（不混合多线程） |
| ADR-005 | 状态存储 | state.json 单文件 + 原子写（不引入 DB） |
| ADR-006 | inbox/outbox 兜底 | 不保留（CLI 不可用 fail-fast） |
| ADR-007 | 双入口 | 共享 orchestrator API + 可插拔 InteractionAdapter |
| ADR-008 | 默认 CLI | 与宿主同款；CLI 入口探测可用 CLI |
| ADR-009 | 表达式语言 | Python 子集 + Jinja2 sandbox（不自创 DSL） |
| ADR-010 | 模式 | 完整产品级 v1.0（不分 MVP 阶段） |

---

## 5. 实施约束与质量门

### 5.1 必须满足

- 类型完全（mypy strict 通过）
- 单元测试覆盖 ≥ 80%
- 端到端测试覆盖所有节点类型
- 所有公开 API 有 docstring
- 所有错误码有文档对照
- 所有配置项有默认值与说明
- 所有 CLI 命令有 `--help`

### 5.2 性能基线

- 空工作流（仅 noop）启动 → 退出 < 500ms
- 100 节点工作流（全 set_var）顺序执行 < 2s
- 16 并发 subagent 调度开销 < 100ms

### 5.3 兼容性

- Python 3.11+
- Linux / macOS / Windows（Windows 经测试通过）
- claude CLI ≥ 2.x、codex CLI 任一稳定版

---

## 6. 阅读引导

- 想看完整 DSL → `specs/01-dsl.md`
- 想看引擎接口 → `specs/02-engine.md`
- 想看怎么解决 429 → `specs/04-concurrency.md`
- 想看怎么写 SKILL.md → `specs/09-skill-protocol.md`
- 想看实施步骤 → `IMPL-CHECKLIST.md`
- 想知道为什么这样设计 → `ADR.md`

---

## 7. 术语表

| 术语 | 含义 |
|------|------|
| **Workflow** | 一份 workflow.json + 一次执行实例 |
| **Run** | 一次工作流执行，对应一个 run_id |
| **Step / Node** | workflow 中的一个节点（一个执行单元） |
| **Subagent** | 由 orchestrator 派出的 CLI 子进程（执行单个 LLM 任务） |
| **Host Agent** | 用户使用的 IDE Agent（Claude Code / Codex / CodeBuddy） |
| **CLI Runner** | 用作 subagent 的 CLI 工具（claude / codex / codebuddy） |
| **InteractionAdapter** | 区分 stdio 与 skill 协议的交互层 |
| **Tool** | DSL 中可调用的 Python 内置功能（fs/shell/...） |
| **DSL** | workflow.json 表达的领域特定语言 |

---

**下一步**：开始撰写 `specs/01-dsl.md`，定义 DSL 完整规范。
