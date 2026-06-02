# 实施检查清单（IMPL-CHECKLIST）

> 实施前先通读 `DESIGN.md` 与所有 `specs/` 文件。
> 本清单按"产品级一次性交付"组织，不分 MVP 阶段。

## 0. 准备

- [ ] Python 3.11+ 可用
- [ ] 至少一个 CLI（claude / codex）已安装并登录
- [ ] 阅读 DESIGN.md / specs/01..13 / ADR.md
- [ ] 确认 ADR 决策是否有争议，有则讨论后更新文档再开始

## 1. 项目初始化

- [ ] `pyproject.toml`（按 12-layout.md §2）
- [ ] `awe/version.py` 含 `__version__ = "1.0.0"`
- [ ] 仓库目录骨架（按 12-layout.md §1）
- [ ] `.gitignore`、`LICENSE`、`README.md`、`CHANGELOG.md` 占位
- [ ] CI 配置（lint / mypy strict / test / coverage）

## 2. 配置子系统（specs/11）

- [ ] `awe/config/defaults.toml` 完整版
- [ ] `awe/config/schemas/config.schema.json`
- [ ] `awe/config/settings.py`：Config 数据类 + 加载器 + 合并 + 校验
- [ ] 单元测试：层级合并、安全级别校验、环境变量覆盖

## 3. DSL 校验子系统（specs/01）

- [ ] `awe/config/schemas/workflow.schema.json` 完整版（draft 2020-12）
- [ ] `awe/core/schema.py`：静态校验 + 动态校验（工具/CLI/引用/循环依赖/code 节点 AST 预检）
- [ ] `awe/render/template.py`：Jinja2 SandboxedEnvironment + filter 白名单
- [ ] `awe/render/expression.py`：AST 白名单或 asteval 包装
- [ ] `awe/render/refs.py`：`${var.path}` 解析
- [ ] code 节点专项二次校验（complement to runtime sandbox）：在 schema 校验阶段就做 AST 预检，提前拒绝违规代码
- [ ] 单元测试：所有节点类型 schema 正反例

## 4. 状态管理（specs/05）

- [ ] `awe/core/state.py`：StateManager + atomic_write + StateLock（跨平台）
- [ ] `awe/core/types.py`：所有 TypedDict / dataclass
- [ ] state.json schema 完整覆盖
- [ ] 断点续跑：load_or_create / hash 校验 / queue 重建
- [ ] 单元测试：原子写、锁、resume、并发安全

## 5. 工具注册中心（specs/07）

- [ ] `awe/tools/registry.py`：装饰器 + Schema + 调用
- [ ] `awe/tools/fs.py`：find_files / read_file / write_file / edit_file（含路径校验）
- [ ] `awe/tools/search.py`：grep / glob（基于 ripgrep 或 Python 实现）
- [ ] `awe/tools/git.py`：git_diff / git_log / git_status
- [ ] `awe/tools/shell.py`：run_shell（默认禁用 + 命令白名单）
- [ ] `awe/tools/http.py`：http_get（默认禁用 + 域名白名单）
- [ ] PathPolicy 实现 + 二次校验
- [ ] 单元测试：路径越界、危险命令拦截

## 6. CLI 适配层（specs/03）

- [ ] `awe/cli_adapter/base.py`：CliAdapter ABC + DetectResult + SubagentRequest/Result
- [ ] `awe/cli_adapter/claude.py`：完整实现（含输出解析、错误分类、超时、kill）
- [ ] `awe/cli_adapter/codex.py`：完整实现
- [ ] `awe/cli_adapter/codebuddy.py`：占位（待官方 CLI 文档）
- [ ] `awe/cli_adapter/registry.py`：注册 + detect 聚合
- [ ] `awe/cli_adapter/tool_mapping.py`：完整工具映射表
- [ ] 输出协议：JSON 提取 + 重试 + 兜底
- [ ] 环境隔离：黑白名单
- [ ] 单元测试：mock subprocess 测各类输出
- [ ] e2e 测试：真实 CLI（CI 选标可手动触发）

## 7. 调度子系统（specs/04）

- [ ] `awe/schedule/concurrency.py`：GlobalConcurrency
- [ ] `awe/schedule/ratelimit.py`：CliTokenBucket + RateLimitHandler
- [ ] `awe/schedule/adaptive.py`：AdaptiveConcurrency (AIMD)
- [ ] `awe/schedule/backoff.py`：退避计算
- [ ] `awe/schedule/__init__.py`：Scheduler 总装
- [ ] 单元测试：429 全局降速、AIMD 收敛、TPM 校正、跨 CLI 隔离

## 8. 节点处理器（specs/02）

- [ ] `awe/nodes/base.py`：NodeHandler ABC + NodeOutput
- [ ] `awe/nodes/tool.py`
- [ ] `awe/nodes/code.py`：调用 sandbox.runner 执行受限代码
- [ ] `awe/nodes/subagent_call.py`
- [ ] `awe/nodes/subagent_fan_out.py`：含 batch / aggregate / fail_strategy
- [ ] `awe/nodes/llm_decision.py`：含 schema 强制 + retry 一次
- [ ] `awe/nodes/llm_plan.py`：含 max_recursion_depth / max_generated_steps + 子 schema 校验
- [ ] `awe/nodes/branch.py`：switch + if/else
- [ ] `awe/nodes/loop.py`：while + for_each
- [ ] `awe/nodes/sequence.py`
- [ ] `awe/nodes/parallel.py`
- [ ] `awe/nodes/set_var.py`
- [ ] `awe/nodes/transform.py`：所有 operation
- [ ] `awe/nodes/human_approval.py`
- [ ] `awe/nodes/sleep.py`
- [ ] `awe/nodes/noop.py`
- [ ] 单元测试：每个 handler 独立可测

## 8.1 code 节点沙箱（specs/01 §4.11、specs/07 §13）

- [ ] `awe/sandbox/__init__.py`
- [ ] `awe/sandbox/allowed_modules.py`：白名单模块清单
- [ ] `awe/sandbox/builtins.py`：受限 __builtins__ 构造
- [ ] `awe/sandbox/ast_validator.py`：AST 白名单校验（节点类型、import、属性、内置函数）
- [ ] `awe/sandbox/runner.py`：子进程执行 + 资源限制 + 超时 + 跨平台兼容
- [ ] 单元测试矩阵（specs/07 §13.8）：所有违规模式必拒绝、所有合法模式必通过
- [ ] Windows 兼容（Job Objects 或子进程监控降级）
- [ ] 输入对象深拷贝注入

## 9. 引擎与执行循环（specs/02）

- [ ] `awe/core/engine.py`：run() + ExecutionQueue + 主循环
- [ ] `awe/core/context.py`：ExecutionContext
- [ ] `awe/core/exceptions.py`：完整异常层级
- [ ] `awe/core/interaction.py`：InteractionAdapter ABC + StdioAdapter + SkillProtocolAdapter
- [ ] `awe/core/approval.py`：审批流 + auto_approve_when 求值
- [ ] 错误分类与重试策略
- [ ] 集成测试：mock CLI 跑端到端

## 10. 可观测性（specs/08）

- [ ] `awe/observability/logger.py`：JSONL 日志器
- [ ] `awe/observability/progress.py`：progress.json 写入器
- [ ] `awe/observability/metrics.py`：counters / histograms / gauges
- [ ] `awe/observability/trace.py`：OTel 兼容（opt-in）
- [ ] 终端 UI：rich 进度条
- [ ] final-report.md 生成器
- [ ] 单元测试：日志脱敏、metrics 聚合、报告内容

## 11. CLI 入口（specs/10）

- [ ] `awe/cli/main.py`：typer 应用
- [ ] 子命令：run / gen / execute / resume / inspect / list / doctor / logs / abort / gc / version / completion
- [ ] 全局选项处理
- [ ] 错误码到退出码映射
- [ ] shell 自动补全
- [ ] 单元测试：每命令 --help、参数解析、退出码

## 12. SKILL.md（specs/09）

- [ ] `SKILL.md` 完整版（含 schema、few-shot 示例、协议说明、加固语句）
- [ ] frontmatter 正确（name / description / version）
- [ ] 反 prompt 注入加固段
- [ ] 各平台启动建议章节

## 13. 模板与示例（specs/13）

- [ ] 8 个内置模板完整可运行
- [ ] 7 个 examples 完整 + README + expected-final-report
- [ ] 模板单元测试（schema + dry-run + e2e mock）

## 14. 文档（docs/）

- [ ] mkdocs 站点骨架
- [ ] index / getting-started / 概念 / 参考 / 食谱
- [ ] 错误码参考（`specs/_error_codes.md` 自动生成 + 文档站）
- [ ] 变更日志规范

## 15. 测试与质量门

- [ ] 单元测试覆盖 ≥ 80%
- [ ] mypy --strict 通过
- [ ] ruff 全绿
- [ ] e2e 测试可手动跑通
- [ ] 性能基线测试通过（DESIGN.md §5.2）
- [ ] 跨平台测试（Linux / macOS / Windows）

## 16. 发布

- [ ] PyPI 包发布脚本
- [ ] GitHub Actions release workflow
- [ ] CHANGELOG.md 1.0.0 章节
- [ ] README.md 完整使用说明
- [ ] LICENSE（MIT）

## 17. dogfooding

- [ ] 用 awe 自己审计 awe 的代码
- [ ] 用 awe 跑出本仓库的安全审计报告
- [ ] 修复发现的所有 critical 问题

## 18. 验收标准

实施完成必须同时满足：

- [ ] 所有 specs/ 描述的功能 100% 实现
- [ ] 所有 ADR 决策被尊重
- [ ] 所有验收测试（IMPL-CHECKLIST 各项 + 性能基线）通过
- [ ] 至少在 Claude Code 与一个其他 Agent（Codex 或 CodeBuddy）上跑通端到端工作流
- [ ] final-report.md 完整且符合 specs/08 §7 规范
- [ ] 无 mypy strict / ruff 错误
- [ ] CI 全绿
- [ ] 单元测试覆盖 ≥ 80%
- [ ] e2e 测试至少 5 个场景通过

## 19. 实施风险与对策

| 风险 | 对策 |
|------|------|
| CLI 行为变化 | 适配器版本检测 + 兼容矩阵文档 |
| Prompt cache 不稳定 | 不依赖具体命中率，仅作优化 |
| LLM 生成不稳定 | schema 校验 retry + 模板 few-shot |
| 跨平台路径问题 | pathlib + 完整 Windows 测试 |
| 长任务断电中断 | state.json 周期快照 + resume |
| 依赖版本冲突 | pyproject.toml 严格固定主版本，可选项隔离 |

## 20. 完成后步骤

- [ ] 在 README 和 PyPI 发布版本说明
- [ ] 在原 codex-dynamic-workflows skill 仓库的 README 加链接指向 agent-workflows
- [ ] 写发布博客 / 推文（可选）
- [ ] 收集首批用户反馈
