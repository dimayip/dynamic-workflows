# Agent Workflows

> 跨 AI Agent 平台的、产品级动态工作流编排系统。
>
> Skill 与 CLI 双入口共享同一 Python 引擎，通过 CLI 子进程派生 subagent 实现真并发。

**当前状态**：📐 设计冻结，待实施

## 是什么

- **跨平台**：可在 Claude Code、Codex、CodeBuddy 等任意 AI Agent 中作为 Skill 使用，也可作为独立 CLI 工具调用。
- **JSON DSL 编排**：用 `workflow.json` 表达完整工作流（fan_out、对抗验证、循环、审批门、动态扩展）。
- **代码强约束**：Python 引擎确定性执行，不依赖宿主 LLM 的自觉性。
- **真并发**：通过 `claude / codex / codebuddy` CLI 子进程派生 subagent，asyncio 调度。
- **可恢复**：跨会话、跨重启、跨日的断点续跑。
- **安全 by default**：工具白名单、路径白名单、表达式沙箱、密钥隔离。

## 设计灵感

- [Claude Code Dynamic Workflows](https://www.anthropic.com/news/claude-opus-4-8) 的"编排代码化"理念
- [`codex-dynamic-workflows` Skill](../codex-dynamic-workflows/) 的 SOP 与决策规则
- LangGraph / Temporal / n8n 的 DSL 编排经验

## 文档地图

| 文件 | 内容 |
|------|------|
| [`DESIGN.md`](./DESIGN.md) | 总览、目标、原则、架构 |
| [`specs/01-dsl.md`](./specs/01-dsl.md) | workflow.json DSL 规范 |
| [`specs/02-engine.md`](./specs/02-engine.md) | Python orchestrator |
| [`specs/03-cli-adapter.md`](./specs/03-cli-adapter.md) | CLI subagent 适配层 |
| [`specs/04-concurrency.md`](./specs/04-concurrency.md) | 并发、限流、429 处理 |
| [`specs/05-state.md`](./specs/05-state.md) | 状态管理与断点续跑 |
| [`specs/06-error-approval.md`](./specs/06-error-approval.md) | 错误处理、重试、审批门 |
| [`specs/07-security.md`](./specs/07-security.md) | 安全模型 |
| [`specs/08-observability.md`](./specs/08-observability.md) | 日志、指标、进度 |
| [`specs/09-skill-protocol.md`](./specs/09-skill-protocol.md) | SKILL.md 协议 |
| [`specs/10-cli-entry.md`](./specs/10-cli-entry.md) | awe CLI 命令规范 |
| [`specs/11-config.md`](./specs/11-config.md) | 配置层级与项 |
| [`specs/12-layout.md`](./specs/12-layout.md) | 项目结构与运行时目录 |
| [`specs/13-templates.md`](./specs/13-templates.md) | 内置模板与示例 |
| [`ADR.md`](./ADR.md) | 关键决策记录 |
| [`IMPL-CHECKLIST.md`](./IMPL-CHECKLIST.md) | 实施检查清单 |

## 阅读建议

- 想快速了解整体 → 读 `DESIGN.md`
- 想知道决策为什么这样 → 读 `ADR.md`
- 准备实施 → 按 `IMPL-CHECKLIST.md` 顺序

## 实施计划

> 实施由 AI 完成，**一次性交付产品级 v1.0**，不分 MVP 阶段。

实施前必须：
1. 通读所有设计文档
2. 确认 ADR 中所有决策（如有争议先讨论）
3. 按 `IMPL-CHECKLIST.md` 执行
4. 完成后跑通 `IMPL-CHECKLIST.md §18` 全部验收

## 与本仓库其他 Skill 的关系

- `codex-dynamic-workflows`：早期版本，使用文档约束 + LLM 自觉。本项目是其下一代演进版。
- 未来可能将 `codex-dynamic-workflows` 标注为 deprecated，并将其文档约束逻辑作为本项目 Skill 入口的一部分。

## 状态

🚧 设计完成，未实施。

## 许可

MIT
