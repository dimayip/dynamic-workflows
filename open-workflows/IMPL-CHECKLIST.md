# 实施检查清单（IMPL-CHECKLIST）

## 0. 准备

- [ ] Node 22+ 可用
- [ ] 至少一个 CLI（claude / codex）已安装并登录
- [ ] 阅读 DESIGN.md / specs/01..06 / ADR.md
- [ ] 确认 ADR 决策无争议

## 1. 项目初始化

- [ ] `package.json`（含 commander / ink / isolated-vm / esbuild / typescript / vitest 等依赖）
- [ ] `tsconfig.json`（strict / ES2022 / NodeNext）
- [ ] `eslint.config.js`（v9 flat + 自定义插件）
- [ ] `.gitignore` / `LICENSE` / `README.md` / `CHANGELOG.md`
- [ ] CI 配置（lint / typecheck / test / build / e2e）
- [ ] 仓库目录结构（详 specs/12 等价物，参考 agent-workflows/specs/12-layout.md 调整为 Node 风格）

## 2. 配置子系统

- [ ] `~/.ow/config.toml` 默认值
- [ ] `ow/config/schema.json`（含 jsonschema 校验）
- [ ] 配置加载器（默认 → 系统 → 用户 → 项目 → 命令行）
- [ ] 单元测试

## 3. 类型与 SDK 定义

- [ ] `ow-sdk.d.ts` 完整版（所有 SDK 类型）
- [ ] `ow/sdk/types.ts` 内部使用
- [ ] AgentRole / ToolName / CliName 等封闭枚举
- [ ] JSON Schema 定义同步
- [ ] 类型导出测试（确认 LLM 生成 prompt 拿到的就是这套）

## 4. 校验流水线（specs/03，核心模块）

### 4.1 L1 TypeScript

- [ ] `ow/validate/typescript.ts`：调 tsc 或 TS Compiler API
- [ ] strict 模式 + 自定义 lib（仅暴露 ow-sdk.d.ts）
- [ ] 错误格式化输出
- [ ] 单元测试：所有 spec/01 强制约束的反例必须报错

### 4.2 L2 ESLint

- [ ] `@open-workflows/eslint-plugin/`：30+ 自定义规则
- [ ] 每条规则单元测试（valid + invalid 各 3+ 例）
- [ ] eslint.config.js 集成

### 4.3 L3 AST 校验

- [ ] `ow/validate/ast.ts`：基于 @babel/parser + @babel/traverse
- [ ] 白名单节点类型
- [ ] 危险模式检测（原型污染、双下划线访问、shell 字符串等）
- [ ] 单元测试：spec/03 §4 矩阵

### 4.4 L4 复杂度门

- [ ] `ow/validate/complexity.ts`：行数 / 圈复杂度 / 嵌套深度 / spawn 静态数量 / spawn 嵌套深度
- [ ] 单元测试

### 4.5 L5 DAG 提取

- [ ] `ow/dag/extract.ts`：从 AST 提取节点 + 边
- [ ] `ow/dag/render.ts`：ASCII / Mermaid / JSON 输出
- [ ] 静态可推断性分析（spawn 数量预估、tool 调用预估）
- [ ] 单元测试：典型脚本能否正确生成 DAG

### 4.6 L6 用户审批

- [ ] CLI 入口：ASCII DAG + 摘要 + 询问
- [ ] Skill 入口：通过协议消息转交
- [ ] dry-run 模式

### 4.7 校验编排

- [ ] `ow/validate/pipeline.ts`：串联 L1-L4，任一失败即停
- [ ] 错误回灌让 LLM 重新生成（最多 2 次）

## 5. 沙箱与运行时（specs/02）

- [ ] `ow/runtime/sandbox.ts`：isolated-vm wrapper
- [ ] SDK 桥（setup prefix 注入）
- [ ] 资源限制（memory / cpu / wall）
- [ ] 错误处理（沙箱越界 / 内存爆 / 超时）

## 6. SDK 实现（host 侧）

- [ ] `ow/sdk/agent.ts`：spawn + 二次校验 + 调度器集成
- [ ] `ow/sdk/tools.ts`：findFiles/readFile/writeFile/grep/glob/git*/httpGet
- [ ] `ow/sdk/log.ts`：JSONL 日志
- [ ] `ow/sdk/kv.ts`：持久化 KV
- [ ] `ow/sdk/workflow.ts`：checkpoint / fail / config / semaphore / sleep
- [ ] `ow/sdk/human.ts`：approve / input / confirm

每个 SDK 函数：参数零信任校验 + 单元测试

## 7. CLI 适配层（specs/04）

- [ ] `ow/cli-adapter/base.ts`：CliAdapter 接口
- [ ] `ow/cli-adapter/claude.ts`
- [ ] `ow/cli-adapter/codex.ts`
- [ ] `ow/cli-adapter/codebuddy.ts`（占位）
- [ ] `ow/cli-adapter/registry.ts`
- [ ] `ow/cli-adapter/tool-mapping.ts`
- [ ] 输出解析 + 错误分类
- [ ] 单元测试 + e2e 测试

## 8. 调度子系统

复用 agent-workflows 调度策略（Node 实现版）：

- [ ] `ow/schedule/concurrency.ts`：全局信号量
- [ ] `ow/schedule/ratelimit.ts`：token bucket（RPM + TPM）
- [ ] `ow/schedule/adaptive.ts`：AIMD
- [ ] `ow/schedule/backoff.ts`
- [ ] `ow/schedule/scheduler.ts`：总装
- [ ] 429 处理（全局降速）
- [ ] 单元测试

## 9. 状态管理（specs/05）

- [ ] `ow/state/manager.ts`：StateManager + atomicWrite + 进程锁（proper-lockfile）
- [ ] state.json schema
- [ ] checkpoint 机制
- [ ] resume 流程
- [ ] subagent 缓存（按 deterministic 哈希）
- [ ] 单元测试

## 10. 可观测性

- [ ] `ow/observability/logger.ts`：JSONL 结构化日志
- [ ] `ow/observability/progress.ts`：progress.json
- [ ] `ow/observability/metrics.ts`：counters / histograms / gauges
- [ ] final-report.md 生成器
- [ ] 单元测试

## 11. CLI 入口（specs/06）

- [ ] `ow/cli/main.ts`：commander 应用
- [ ] 子命令：run / gen / compile / execute / resume / inspect / list / doctor / logs / abort / gc / cache / render-dag / version / completion
- [ ] 全局选项处理
- [ ] 退出码映射
- [ ] shell 自动补全
- [ ] ink 进度 UI
- [ ] 单元测试 + e2e

## 12. SKILL.md（specs/06）

- [ ] `SKILL.md` 完整版
- [ ] 内嵌完整 SDK 类型定义
- [ ] few-shot 示例（5+ 个）
- [ ] 协议说明
- [ ] 反 prompt 注入加固

## 13. 模板与示例

- [ ] `templates/`：8+ 内置模板（参考 agent-workflows 模板列表）
- [ ] `examples/`：完整可运行示例
- [ ] 模板验证测试

## 14. 文档

- [ ] mkdocs / docusaurus 站点
- [ ] 入门指南 / 概念 / 参考 / 食谱
- [ ] 错误码表
- [ ] CHANGELOG

## 15. 测试与质量门

- [ ] 单元测试 ≥ 80%
- [ ] tsc --strict 通过
- [ ] eslint 全绿
- [ ] e2e 至少 5 场景通过
- [ ] 跨平台（Linux / macOS / Windows）
- [ ] 性能基线
  - 启动 → 退出（noop 脚本） < 200ms
  - 100 步顺序 spawn < 2s（不含 LLM 时间）

## 16. 安全审计

- [ ] 沙箱逃逸测试矩阵（spec/03 §13.8 等价）
- [ ] 已知 V8 / isolated-vm CVE 检查
- [ ] 依赖审计（npm audit）
- [ ] 密钥泄露扫描

## 17. 发布

- [ ] npm 发布脚本
- [ ] GitHub Actions release workflow
- [ ] CHANGELOG 1.0.0
- [ ] README 完整使用说明

## 18. dogfooding

- [ ] 用 ow 自己审计 ow 的代码
- [ ] 修复发现的 critical 问题

## 19. 验收标准

- [ ] 所有 specs/ 描述的功能 100% 实现
- [ ] 所有 ADR 决策被尊重
- [ ] 所有验收测试通过
- [ ] 至少在 Claude Code + 一个其他 Agent 跑通端到端
- [ ] final-report.md 完整
- [ ] CI 全绿
- [ ] 单元测试 ≥ 80%
- [ ] e2e 至少 5 场景通过
- [ ] 校验流水线对 spec/03 完整测试矩阵全部正确（违规拒绝、合法通过）
- [ ] 沙箱逃逸测试全过

## 20. 完成后

- [ ] 在 README 与 PyPI/npm 写版本说明
- [ ] 在 agent-workflows README 加 open-workflows 链接，说明定位差异
- [ ] 跟进首批用户反馈
