# Open Workflows — 设计文档（总览）

> **项目代号**：`open-workflows`（占位名，可重命名为 `xworkflows` / `claude-flow-oss` 等）
> **版本**：v1.0 设计冻结稿
> **状态**：待实施
> **与 agent-workflows 的关系**：见 README.md

## 1. 一句话定位

> **开源、跨 AI Agent 平台的 Claude Code Dynamic Workflows 等价物：LLM 生成 TypeScript 编排脚本，在 isolated-vm 沙箱执行，通过 CLI 子进程派生 subagent，多层校验确保 LLM 生成代码安全无需用户逐字审计。**

## 2. 核心目标

| # | 目标 | 说明 |
|---|------|------|
| G1 | **同构体验** | 与 Claude Code DW 表达范式一致，便于用户跨工具迁移 |
| G2 | **跨平台** | 运行于任意支持 Node 22+ 的环境，subagent 用任意 CLI（claude/codex/codebuddy） |
| G3 | **代码强约束** | LLM 生成代码经过 10 层校验，用户审"DAG 视图 + SDK 调用摘要"即可 |
| G4 | **沙箱隔离** | isolated-vm 严格沙箱，不允许 fs / net / proc / require |
| G5 | **真并发** | 等价 V8 内 Promise.all，子进程派 CLI |
| G6 | **可恢复** | 检查点机制 + 重放 |
| G7 | **可观测** | 每个 SDK 调用结构化日志，可生成行为摘要 |
| G8 | **双入口** | Skill + CLI |

## 3. 设计原则

- **P1 限自由不限表达**：编排逻辑可任意写（图灵完备），但仅能调 Curated SDK
- **P2 安全四层防御**：编译时（TS/Lint）+ 运行时（沙箱）+ SDK 内（参数校验）+ 操作系统（资源限制）
- **P3 DAG 优先于代码**：从 AST 自动提取 DAG 视图，让用户审 DAG 而非代码
- **P4 失败显式**：脚本错误立即停止，状态可重入
- **P5 配置最小**：用户最少配置即可运行

## 4. 架构

```
                          用户
                           │
              ┌────────────┴────────────┐
              ▼                         ▼
        宿主 AI Agent               ow CLI
        (Claude Code 等)        (ow run / gen / ...)
              │                         │
              └────────────┬────────────┘
                           ▼
              ┌──────────────────────────────┐
              │ 生成阶段（任意 LLM）           │
              │ user_task → orchestrator.ts  │
              │ 受 SDK 类型 + Few-shot 约束   │
              └──────────────┬───────────────┘
                             ▼
              ┌──────────────────────────────┐
              │ 校验阶段（10 层防御）          │
              │ TS 编译 → ESLint → AST 校验   │
              │ → 复杂度 → DAG 提取 → 用户审   │
              └──────────────┬───────────────┘
                             ▼
              ┌──────────────────────────────┐
              │ 执行阶段（isolated-vm 沙箱）   │
              │  ┌─────────────────────────┐ │
              │  │ Curated SDK（注入沙箱）  │ │
              │  │ • agent.spawn(...)      │ │
              │  │ • tools.read_file(...)  │ │
              │  │ • log.info(...)         │ │
              │  │ • kv.get/set(...)       │ │
              │  │ • workflow.checkpoint() │ │
              │  └─────────────────────────┘ │
              │  无 fs/net/proc/require       │
              │  资源限制：内存/CPU/墙钟       │
              └──────────────┬───────────────┘
                             │ SDK 实现层
                             ▼
                    asyncio.gather 等价
                             │
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
          claude -p      codex exec     codebuddy ...
                    (CLI 子进程，与 agent-workflows 共享)
                             │
                             ▼
              ┌──────────────────────────────┐
              │ 持久化                        │
              │ .ow/<run_id>/                 │
              │   ├─ orchestrator.ts          │
              │   ├─ state.json               │
              │   ├─ dag.json (从 AST 提取)   │
              │   ├─ checkpoints/             │
              │   ├─ subagents/<task_id>.json │
              │   ├─ logs/                    │
              │   └─ final-report.md          │
              └──────────────────────────────┘
```

## 5. 与 Claude Code DW 的本质对比

| 维度 | Claude Code DW | open-workflows |
|------|----------------|----------------|
| 编排载体 | JS 脚本 | TypeScript 脚本 |
| 沙箱 | Anthropic 内部 V8 | isolated-vm（开源 V8 隔离） |
| Subagent 派生 | 内部 spawn API | CLI 子进程（同 agent-workflows）|
| 主对话隔离 | ✅ 完美 | ✅ 进程级隔离 |
| Prompt cache | 主对话深度复用 | 5 分钟窗口共享 |
| 跨平台 | ❌ | ✅ |
| 代码语言 | JS | TypeScript（更强类型约束） |
| LLM 生成校验 | Anthropic 内部（黑盒） | 10 层防御（开源透明） |
| 用户控制权 | 受限 | 完全 |

## 6. 与 `agent-workflows` 的本质对比

| 维度 | agent-workflows | open-workflows |
|------|-----------------|----------------|
| 编排载体 | JSON DSL | TypeScript 脚本 |
| 表达力 | 95%（+ code 节点补齐） | 100%（图灵完备）|
| 校验粒度 | DSL schema 节点级 | TS 类型 + Lint + AST 多层 |
| 用户审查方式 | 直接读 JSON | 审 DAG + SDK 调用摘要 |
| 适合用户 | 企业、合规、CI/CD | 个人、灵活探索 |
| 实施语言 | Python | Node.js + TypeScript |
| 生态 | jsonschema/jinja2/asteval | tsc/ESLint/isolated-vm |

## 7. 核心技术选型

| 组件 | 选择 | 替代 | 理由 |
|------|------|------|------|
| 运行时 | Node.js 22+ | Bun / Deno | 生态最成熟、isolated-vm 仅支持 Node |
| 语言 | TypeScript 5+ | JavaScript | 类型即文档、约束 LLM 生成、AST 工具丰富 |
| 沙箱 | `isolated-vm` | `vm2`（已废弃）/`workers_threads`/`process` | 唯一安全的 V8 隔离方案 |
| AST 工具 | `@babel/parser` + `@babel/traverse` | `acorn`/TypeScript Compiler API | 生态成熟、API 友好 |
| Lint | ESLint 9 + 自定义规则 | / | 业界标准、可写自定义 rule |
| CLI | `commander` 或 `yargs` | / | 成熟 |
| TUI | `ink`（React）或 `ora` | / | 进度可视化 |
| 测试 | Vitest | Jest | 速度、原生 ESM、TS 友好 |

## 8. 关键决策（详 ADR.md）

| # | 决策 | 选择 |
|---|------|------|
| ADR-001 | 运行时 | Node 22+ |
| ADR-002 | 语言 | TypeScript（不是裸 JS） |
| ADR-003 | 沙箱 | isolated-vm |
| ADR-004 | LLM 生成约束 | 10 层防御 |
| ADR-005 | Subagent | CLI 子进程（与 agent-workflows 共享） |
| ADR-006 | 状态 | 单文件 + checkpoint 机制 |
| ADR-007 | 双入口 | Skill + ow CLI |
| ADR-008 | 用户审查 | DAG 视图 + SDK 摘要，非代码逐行 |
| ADR-009 | SDK 边界 | Curated（封闭集，扩展需新版本） |
| ADR-010 | 不分阶段 | 一次性产品级 v1.0 |

## 9. 不在范围

- 不内置 LLM SDK 直调（必须经 CLI）
- 不实现自有 Agent CLI（复用现有）
- 不实现 GUI（仅 CLI / IDE Skill）
- 不实现工作流市场 / 远程托管
- 不支持 LLM 生成的代码做任意 require/import（受 SDK 边界约束）

## 10. 阅读引导

- 想看 LLM 生成的 TS 长什么样 → `specs/01-script-spec.md`
- 想看怎么沙箱执行 → `specs/02-runtime-sandbox.md`
- 想看怎么防止 LLM 生成的代码作恶 → **`specs/03-validation-pipeline.md`**
- 想看实施步骤 → `IMPL-CHECKLIST.md`
- 想知道为什么这样选 → `ADR.md`

## 11. 一句话总结

> **agent-workflows 是"安全 by design"，open-workflows 是"安全 by defense"。前者用 DSL 限制能力，后者用多层校验约束自由。两者都做到了"用户不需要逐行审 LLM 生成的内容"。**
