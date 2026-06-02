# Open Workflows

> **跨平台开源版的 Claude Code Dynamic Workflows。**
>
> LLM 生成 TypeScript 编排脚本 → 多层校验 → isolated-vm 沙箱执行 → CLI 子进程派生 subagent。

**当前状态**：📐 设计冻结，待实施

## 与 `agent-workflows` 的关系

`agent-workflows` 与 `open-workflows` 是**同一愿景下的两个产品**，定位不同：

| | `agent-workflows`（DSL 路线） | `open-workflows`（代码路线） |
|---|---|---|
| 编排载体 | JSON DSL | TypeScript 脚本（V8 沙箱执行） |
| 表达力 | 受限 + code 节点补齐 | 图灵完备（受 SDK 边界约束） |
| LLM 生成稳定性 | 高（schema 强约束） | 中（多层校验补强） |
| 安全审计 | 直接读 JSON | 看 AST 衍生 DAG + SDK 调用摘要 |
| 可视化 | 原生 DAG | AST 提取后渲染 DAG |
| 用户群 | 企业级、合规、CI/CD | 个人开发者、探索性大任务、与 Claude Code 同构体验 |
| 与 Claude Code 关系 | 异构（JSON vs JS） | **同构**（都是 JS 脚本 + spawn 子代理） |
| 产品定位 | "跨平台 AI 编排平台" | "开源版 Claude Code Dynamic Workflows" |

两个项目共享：
- CLI Subagent 适配层（同一份代码，少量调整）
- 状态管理 / 断点续跑思路
- 可观测性框架
- SKILL.md 协议骨架

**用户可以二选一，也可以两者都装**——它们没有互斥关系。

## 文档地图

| 文件 | 内容 |
|------|------|
| [`DESIGN.md`](./DESIGN.md) | 总览、目标、原则、架构 |
| [`specs/01-script-spec.md`](./specs/01-script-spec.md) | LLM 生成的 orchestrator.ts 规范 |
| [`specs/02-runtime-sandbox.md`](./specs/02-runtime-sandbox.md) | isolated-vm 沙箱、Curated SDK、资源限制 |
| [`specs/03-validation-pipeline.md`](./specs/03-validation-pipeline.md) | **10 层防御**：怎么让 LLM 生成的代码安全 |
| [`specs/04-cli-subagent.md`](./specs/04-cli-subagent.md) | CLI subagent 派生（与 agent-workflows 共享） |
| [`specs/05-state.md`](./specs/05-state.md) | 状态、checkpoint、resume |
| [`specs/06-protocol.md`](./specs/06-protocol.md) | SKILL.md 协议 + ow CLI |
| [`ADR.md`](./ADR.md) | 关键决策 |
| [`IMPL-CHECKLIST.md`](./IMPL-CHECKLIST.md) | 实施清单 |

## 实施

- 一次性产品级 v1.0
- 实施由 AI 完成
- 严格遵循 ADR 决策

## 许可

MIT
