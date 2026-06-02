# Workflows

> **AI Agent 动态工作流编排：四个方案，一个目标。**
>
> 把"用 LLM 一句话调度数十甚至上百个子代理协作完成大任务"这件事，做成跨平台、可审计、可复用、可生产的工程能力。

## 背景

2026 年 5 月，Anthropic 在 Claude Code 中发布 **Dynamic Workflows** 功能——LLM 现场写一段 JS 编排脚本，在 V8 沙箱内调度数百子代理，完成跨服务的代码迁移、安全审计、对抗式验证等大任务。这个能力的本质是：

> **把工作流的"编排"从 LLM 的概率推理域，迁移到代码的确定性执行域。**

但 Claude Code DW 有两个关键限制：
1. **平台锁定**：仅在 Claude Code 自家产品内可用
2. **闭源**：实现细节、调度策略、安全模型都是黑盒

本仓库探索**跨平台、开源**地实现等价能力的不同技术路线，并把它们的设计差异、能力边界、适用场景透明地呈现出来。

---

## 本仓库包含的四个技术方案

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                  │
│   弱约束 ──────────────────────────────────────────── 强约束     │
│                                                                  │
│   ┌──────────────────────┐    ┌──────────────────────┐          │
│   │ codex-dynamic-       │    │ agent-workflows      │          │
│   │ workflows            │    │                      │          │
│   │                      │    │  JSON DSL + Python   │          │
│   │  纯 SOP 文档约束      │    │  确定性引擎 + CLI    │          │
│   │  依赖 LLM 自觉        │    │  subagent           │          │
│   │  最轻量              │    │  最安全可审计         │          │
│   └──────────────────────┘    └──────────────────────┘          │
│                                                                  │
│   ┌──────────────────────┐    ┌──────────────────────┐          │
│   │ Claude Code DW       │    │ open-workflows       │          │
│   │  (参考标杆)           │    │                      │          │
│   │                      │    │  TypeScript 脚本 +   │          │
│   │  JS 脚本 + 内置 V8   │    │  isolated-vm 沙箱 +  │          │
│   │  闭源、平台锁定       │    │  10 层防御          │          │
│   │                      │    │  跨平台开源等价物    │          │
│   └──────────────────────┘    └──────────────────────┘          │
│                                                                  │
│   表达力强 ←──────────────────────────────────────→ 表达力弱     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

| 项目 | 一句话定位 | 实施状态 |
|------|-----------|---------|
| [`codex-dynamic-workflows`](./codex-dynamic-workflows/) | 用 SOP 文档引导宿主 Agent 自觉做 dynamic workflow | ✅ 可用 |
| [`agent-workflows`](./agent-workflows/) | JSON DSL + Python 引擎 + CLI subagent 的跨平台 AI 编排平台 | 📐 设计冻结，待实施 |
| [`open-workflows`](./open-workflows/) | TypeScript + isolated-vm 沙箱实现的开源版 Claude Code DW | 📐 设计冻结，待实施 |
| Claude Code DW（参考） | Anthropic 闭源原版，作为对照基线 | ✅ Anthropic 官方提供 |

---

## 四方案横向对比（一图全览）

### 总览表

| 维度 | Claude Code DW | codex-dynamic-workflows | agent-workflows | open-workflows |
|------|----------------|------------------------|-----------------|----------------|
| **编排载体** | JS 脚本 | 纯 SOP 文档 | JSON DSL + code 节点 | TypeScript 脚本 |
| **控制流执行** | V8 沙箱（闭源） | LLM 推理 | Python 引擎 | isolated-vm 沙箱 |
| **Subagent 派生** | 内置 spawn API | LLM 自觉调度 | CLI 子进程 | CLI 子进程 |
| **跨平台** | ❌ 仅 Claude Code | ⚠️ 任意支持 skill 的 Agent | ✅ 任意支持 subagent | ✅ Node 22+ 任意环境 |
| **真并发** | ✅ 进程内 | ❌ 串行/弱并发 | ✅ asyncio + subprocess | ✅ Promise.all + subprocess |
| **中间状态隔离** | ✅ 脚本变量 | ❌ 污染主对话 | ✅ CLI 进程隔离 | ✅ isolate + 进程隔离 |
| **跨会话恢复** | ❌ | ⚠️ 弱 | ✅ state.json | ✅ checkpoint + cache |
| **上限规模** | ~1000 subagent | ~10 | ~256（API 配额） | ~256（API 配额） |
| **表达力** | 100% 图灵完备 | 100%（依赖 LLM）| 95% + code 节点 ≈ 100% | 100% 图灵完备 |
| **LLM 生成稳定性** | 高（专门优化） | 不需要生成 | 极高（schema 强约束） | 高（10 层校验） |
| **安全审计** | ⚠️ 看代码 | ⚠️ 看 SOP 输出 | ✅ 直接读 JSON | ✅ AST 衍生 DAG + SDK 摘要 |
| **可视化** | ❌ 看代码 | ❌ 看输出 | ✅ 原生 DAG | ✅ AST 提取 DAG |
| **风险等级** | 低（沙箱 + 平台保障） | 中（依赖自觉） | 极低（代码强约束） | 低（10 层防御） |
| **学习曲线** | 低（写 JS） | 极低（自然语言） | 中（学 DSL） | 中（写受限 TS） |
| **token 成本基线** | 1.0× | 1.5× | 1.3-1.8× | 1.3-1.8× |
| **是否需要装宿主 CLI** | 仅 Claude Code | 任一 Agent IDE | 任一 CLI（claude/codex/codebuddy） | 任一 CLI |
| **实施语言** | TypeScript（闭源） | / | Python 3.11+ | Node 22+ + TypeScript 5+ |

### 安全模型对比

| 安全维度 | Claude Code DW | codex-dynamic-workflows | agent-workflows | open-workflows |
|---------|----------------|------------------------|-----------------|----------------|
| 编排流程合规 | V8 + 平台调度 | LLM 软约束 | Python 引擎硬约束 | sandbox + 10 层校验 |
| 工具白名单 | 内置 | 文档建议 | Python 注册中心 | Curated SDK |
| 路径访问控制 | 平台内置 | 文档建议 | PathPolicy + 二次校验 | SDK 实现层二次校验 |
| 表达式沙箱 | V8 | / | Jinja2 + AST 白名单 | TS 类型 + ESLint + AST |
| 资源限制 | 平台内置 | / | 完全自建 | isolated-vm + 自建 |
| 429 协调 | 内部调度器 | / | token bucket + AIMD | token bucket + AIMD |
| 审批门 | / | 文档建议 | 显式节点 + InteractionAdapter | human SDK + InteractionAdapter |
| 密钥隔离 | 平台保障 | / | 黑名单 + 脱敏 | 黑名单 + 脱敏 |

### 用户体验对比

| 场景 | Claude Code DW | codex-dynamic-workflows | agent-workflows | open-workflows |
|------|----------------|------------------------|-----------------|----------------|
| 触发方式 | "create a workflow" 关键词 / ultracode | 在 Skill 中调用 | Skill / `awe run` CLI | Skill / `ow run` CLI |
| 用户审查内容 | JS 脚本（可选） | SOP 文档输出 | JSON 文件 | DAG 视图 + SDK 摘要 |
| 审查心智负担 | 中-高（读代码） | 低（读 SOP） | 低（读 JSON） | 低（看 DAG） |
| 进度展示 | 平台内置 | 文档建议 | 终端进度 / Skill 转发 | ink TUI / Skill 转发 |
| 中断恢复 | 同会话内 | 弱 | 跨会话精确恢复 | 跨会话 + 缓存 |
| 复用工作流 | `.claude/workflows/*.js` | 复制 SOP 文档 | `templates/*.json` | `.ow/workflows/*.ts` |

---

## 如何选择适合自己的方案

```
你的需求是什么？
│
├─ 只想用 Claude Code 内的内置功能
│  └─→ 用 Claude Code DW（不在本仓库范围）
│
├─ 想要轻量启动，对质量稳定性要求一般
│  ├─ 已经在用 IDE Agent → codex-dynamic-workflows
│  └─ 任务复杂度低 / 一次性使用
│
├─ 企业级 / 合规 / 审计要求高 / CI/CD 集成
│  └─→ agent-workflows
│      • 安全 by design
│      • JSON 可静态分析、可 git diff
│      • 适合团队协作、模板复用
│
├─ 想要 Claude Code 同款体验，但要跨平台
│  └─→ open-workflows
│      • 与 Claude Code DW 编程范式一致
│      • TypeScript 类型约束 + 沙箱执行
│      • 个人开发者 / 探索性大任务
│
└─ 不确定 / 都试试
   └─→ 三个并存，不冲突
       • 小任务 → codex-dynamic-workflows
       • 关键流程 → agent-workflows
       • 实验性大任务 → open-workflows
```

---

## 核心设计哲学

四个方案在"安全 vs 表达力"的光谱上各占一个位置：

```
表达力强 ──────────────────────────────────────→ 表达力受控
          │                              │
          ▼                              ▼
    Claude Code DW                  codex-dynamic-
    open-workflows                  workflows
          ▲                              ▲
          │                              │
安全 by sandbox                    安全 by SOP
（沙箱限制能力）                  （依赖 LLM 自觉）

                      agent-workflows
                            │
                            ▼
                    安全 by design
                  （DSL 受限能力 + code 沙箱）
```

- **agent-workflows**：用结构化的 DSL **限制能力本身** → 不该做的事根本表达不出来
- **open-workflows**：保持完整表达力，用沙箱 + 多层校验 **限制能力的执行边界** → 表达任意但只能调 SDK
- **codex-dynamic-workflows**：靠文档约束让 LLM **自觉不做**坏事 → 最轻量但风险最高
- **Claude Code DW**：闭源黑盒，由平台方保证 → 用户无需自己管

**没有银弹**——选择哪个取决于你的风险偏好、表达力需求、跨平台诉求与团队工程基础。

---

## 各项目快速链接

### [codex-dynamic-workflows](./codex-dynamic-workflows/)

**纯 Skill 方案**——通过 SOP 文档引导 LLM 自觉地做编排、审批、验证。

```text
Install the AI agent skill at https://github.com/DannyMac180/skills/tree/main/codex-dynamic-workflows
```

或本地复制：
```bash
git clone https://github.com/DannyMac180/skills.git
mkdir -p "${CODEX_HOME:-$HOME/.codex}/skills"
cp -R skills/codex-dynamic-workflows "${CODEX_HOME:-$HOME/.codex}/skills/"
```

调用示例：
```
Use $codex-dynamic-workflows to plan and run a supervised multi-agent workflow for ...
```

→ 详见 [codex-dynamic-workflows/SKILL.md](./codex-dynamic-workflows/SKILL.md)

### [agent-workflows](./agent-workflows/)

**JSON DSL + Python 引擎方案**——产品级跨平台 AI 编排平台，强调安全可审计。

技术栈：
- Python 3.11+ + asyncio
- JSON DSL（含 code 节点纯计算逃生舱）
- CLI subagent 派生
- 14 个文档完整覆盖架构、安全、可观测性、CLI 等

入口：
```bash
# Skill 入口（IDE Agent 触发）
"Use agent-workflows to ..."

# CLI 入口（程序触发）
awe run "审计 src 目录的认证缺陷"
```

→ 详见 [agent-workflows/README.md](./agent-workflows/README.md) + [DESIGN.md](./agent-workflows/DESIGN.md)

### [open-workflows](./open-workflows/)

**TypeScript + isolated-vm 方案**——开源版 Claude Code DW，强调表达力与同构体验。

技术栈：
- Node.js 22+ + TypeScript 5+
- isolated-vm 严格沙箱
- 10 层防御让用户无需逐行审代码
- 与 Claude Code DW 同构编程范式

入口：
```bash
# Skill 入口
"Use open-workflows to ..."

# CLI 入口
ow run "审计 src 目录的认证缺陷"
```

→ 详见 [open-workflows/README.md](./open-workflows/README.md) + [DESIGN.md](./open-workflows/DESIGN.md)

特别推荐阅读：[open-workflows/specs/03-validation-pipeline.md](./open-workflows/specs/03-validation-pipeline.md) — 详细说明如何让 LLM 生成的 TS 代码安全可信。

---

## 仓库结构

```
.
├── README.md                       # 本文件：四方案总览与对比
├── codex-dynamic-workflows/        # Skill 方案（已实施）
│   ├── SKILL.md
│   ├── agents/
│   ├── references/
│   └── scripts/
├── agent-workflows/                # JSON DSL 方案（设计完成）
│   ├── README.md
│   ├── DESIGN.md
│   ├── ADR.md                      # 21 条决策记录
│   ├── IMPL-CHECKLIST.md
│   └── specs/                      # 13 份详细规范
└── open-workflows/                 # TypeScript 沙箱方案（设计完成）
    ├── README.md
    ├── DESIGN.md
    ├── ADR.md                      # 22 条决策记录
    ├── IMPL-CHECKLIST.md
    └── specs/                      # 6 份详细规范
```

---

## 设计原则

四个方案共享以下核心原则：

1. **代码确定性优先于 LLM 概率性** —— 编排逻辑能用代码就不用 LLM
2. **跨平台优先于单一生态** —— 用户应能选择宿主 Agent 与 LLM CLI
3. **可审计优先于黑盒便利** —— 用户应能看懂工作流要做什么
4. **资源边界显式** —— 上限、超时、配额都必须显式声明
5. **失败应当 fail-fast** —— 不允许带病运行
6. **状态可恢复** —— 长任务中断后能续跑

---

## 进一步阅读

- 想了解为什么 Claude Code DW 是范式革新 → [agent-workflows/DESIGN.md §1](./agent-workflows/DESIGN.md)
- 想了解 LLM 生成代码如何安全 → [open-workflows/specs/03-validation-pipeline.md](./open-workflows/specs/03-validation-pipeline.md)
- 想了解 JSON DSL 的能力边界 → [agent-workflows/specs/01-dsl.md](./agent-workflows/specs/01-dsl.md)
- 想了解 CLI subagent 派生机制 → [agent-workflows/specs/03-cli-adapter.md](./agent-workflows/specs/03-cli-adapter.md)
- 想了解每个决策的依据 → [agent-workflows/ADR.md](./agent-workflows/ADR.md) / [open-workflows/ADR.md](./open-workflows/ADR.md)

---

## 致谢

- [DannyMac180](https://github.com/DannyMac180) — `codex-dynamic-workflows` 原作者，本仓库 fork 起点
- Anthropic — Claude Code Dynamic Workflows 范式启发
- 业界先例：AWS Step Functions、Temporal、Apache Airflow、LangGraph、n8n、Dify — 编排架构参考

## License

MIT
