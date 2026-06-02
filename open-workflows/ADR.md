# ADR — open-workflows 关键决策记录

> 父文档：`./DESIGN.md`

## ADR-001：Node.js 22+ 作为运行时

**背景**：脚本要在 V8 沙箱执行，宿主必须能跑 V8。

**选项**：
- Node 22+：成熟、isolated-vm 支持
- Bun：性能更好但 isolated-vm 不支持
- Deno：内置沙箱但与 isolated-vm 生态不兼容

**决策**：Node 22+

**依据**：
- `isolated-vm` 是开源界唯一安全 V8 隔离方案，仅 Node 支持
- Node LTS 生态稳定
- Claude Code 用户已装 Node

**回顾时机**：isolated-vm 出现 Bun 移植版

---

## ADR-002：脚本语言选 TypeScript 而非 JS

**背景**：让 LLM 生成可执行脚本。

**选项**：
- 裸 JS：自由度高，但类型约束弱
- TypeScript：强类型，可静态校验

**决策**：TypeScript（编译为 ES2022 后入沙箱）

**依据**：
- LLM 在有完整类型定义时生成稳定性大幅提升
- 类型即文档：SDK 边界用类型表达
- `tsc --strict` 是 LLM 生成 JS 安全性的第一道关卡
- 业界先例：Vercel AI SDK、Modal、Replit Agent 都用 TS 约束 LLM 输出

**影响**：
- 脚本编译开销（~30ms）
- 需要 esbuild / tsc 链路
- 沙箱内执行的是编译后 JS，原始 TS 仅校验

---

## ADR-003：沙箱选 isolated-vm

**背景**：Node 内执行不可信代码。

**选项**：
- `vm`（Node 内置）：不安全，可逃逸
- `vm2`：已知漏洞 + 项目废弃
- `isolated-vm`：真正的 V8 isolate，工业级安全
- 子进程 + worker_threads：隔离性强但 SDK 集成复杂

**决策**：`isolated-vm`

**依据**：
- 工业级（Anthropic、Replit 等使用）
- 真正的 V8 isolate（独立堆 + 独立 event loop）
- 资源限制原生支持（memoryLimit、cpuTime）
- API 简洁

**影响**：
- 必须用 Reference 跨边界传值（性能开销）
- 不支持原生 promise 跨边界，需要 `applyIgnored + promise: true`
- 不支持 inspector（生产环境无所谓）

**回顾时机**：isolated-vm 出现安全问题或 Node 内置 isolate API 时

---

## ADR-004：10 层防御 + 用户审 DAG 不审代码

**背景**：LLM 生成的代码如何安全又方便地审查。

**选项**：
- A. 让用户逐行审 TS（不可行）
- B. 仅靠运行时沙箱（攻击面大、出错时损失大）
- C. 多层校验 + AST 衍生 DAG 让用户审 DAG（本方案）

**决策**：C，10 层叠加。

**依据**：
- 任何单层都不够（深度防御）
- 用户审 DAG + 调用摘要 ≈ 审 JSON DSL 的认知负担
- 业界先例（Vercel、Modal 等都做类似多层校验）

**影响**：
- 校验链路长（~5s/次）
- 错误反馈链路：让 LLM 看到具体校验失败原因 → 重生成
- 必须维护一系列工具（tsc / ESLint / 自定义规则 / AST 校验器）

---

## ADR-005：Subagent 派生用 CLI 子进程（与 agent-workflows 共享）

**背景**：subagent 派生方式。

**决策**：CLI 子进程

**依据**：
- 与 agent-workflows 完全共享契约（CLI 适配）
- 真隔离
- 不绑定 LLM SDK
- 用户用自己已登录的 CLI 配额

**影响**：
- token 成本比 Claude Code 内部 spawn 高 30-80%
- 但跨平台 / 用户可控

---

## ADR-006：Curated SDK 而非自由全局

**背景**：脚本能调用什么。

**决策**：仅暴露 6 个全局对象（agent/tools/log/kv/workflow/human），各为封闭 API 集。

**依据**：
- 限制能力 = 限制风险
- 类型完整 = LLM 生成稳定
- 扩展通过版本化（v1.1 加新 API，旧脚本兼容）

**影响**：
- 用户无法用脚本访问任意 npm 包（这是优点）
- 新需求需要发版增加 SDK 函数

---

## ADR-007：双入口共享 orchestrator + 可插拔 InteractionAdapter

**背景**：与 agent-workflows ADR-007 同。

**决策**：唯一 ow.engine.run() + StdioAdapter / SkillProtocolAdapter

---

## ADR-008：用户审"DAG 视图 + SDK 摘要"，不审代码

**背景**：可审计性。

**决策**：
- 默认审查模式：DAG + 摘要 + 资源预估 + 风险清单
- 可选模式：完整代码（用户主动 `[y/edit]`）

**依据**：
- 完整代码审查心智负担过高
- DAG 表达 80% 行为信息
- SDK 调用摘要表达另 15% 副作用信息
- 剩 5% 真正的"代码逻辑"用户可选择是否查看

**影响**：
- 必须实现可靠的 AST → DAG 提取器
- DAG 提取失败时 fallback 到"必须读代码"模式

---

## ADR-009：SDK 边界封闭（agent role / tools / cli 都是枚举）

**背景**：SDK 类型定义的灵活度。

**决策**：所有关键参数用联合类型枚举，不允许字符串自由值。

**依据**：
- TS 联合类型可被静态校验
- 防 LLM 自创不存在的角色/工具
- 错误信息明确（"role 'xxx' 不在白名单"）

**影响**：
- 新增角色/工具需发版
- 通过 `role: "custom"` 提供逃生口（受额外审查）

---

## ADR-010：一次性产品级 v1.0

**背景**：与 agent-workflows ADR-010 同。

**决策**：不分 MVP，直接产品级。

---

## ADR-011：与 agent-workflows 并行存在，不互斥

**背景**：两个产品定位。

**决策**：两个项目独立维护，共享 CLI 适配契约。

**依据**：
- 服务不同用户群（详 README.md 对比表）
- 相互参考 / 相互启发
- 长期可融合（如把 agent-workflows 的 DSL 作为 ow 的高层抽象）

**回顾时机**：用户社区出现强烈合并需求

---

## ADR-012：v1.0 不支持 LLM 生成的脚本 import 任何包

**背景**：脚本里能不能 require 第三方包。

**选项**：
- 完全禁（本方案）
- 受限白名单（如允许 lodash）
- 自由 import

**决策**：完全禁。

**依据**：
- 脚本是 LLM 生成 + 沙箱执行，引入外部代码会扩大攻击面
- 需要的功能通过 SDK 提供（Curated）
- 简化沙箱实现

**回顾时机**：用户反馈 SDK 内置功能不足时（v1.1+ 考虑白名单包）

---

## ADR-013：使用 esbuild 编译 TS

**背景**：TS → JS 链路。

**决策**：esbuild

**依据**：
- 速度（< 30ms 全量）
- API 简洁
- 与 tsc 配合：tsc 仅 type-check，esbuild 真编译

**影响**：esbuild 的 ESM 输出与 isolated-vm 兼容性需测试

---

## ADR-014：使用 commander 而非 yargs

**背景**：CLI 框架。

**决策**：commander。

**依据**：API 直观、类型友好、依赖小

---

## ADR-015：使用 ink 做交互式 UI

**背景**：终端 UI。

**决策**：ink（React for CLI）。

**依据**：
- 进度条、表格、富交互友好
- 与 ow 的"审 DAG"场景天然适配（可渲染 DAG 树）
- 可降级到 ora（简单进度）

---

## ADR-016：使用 Vitest 而非 Jest

**背景**：测试框架。

**决策**：Vitest。

**依据**：速度、原生 ESM、TS 友好、配置简单。

---

## ADR-017：与 agent-workflows 共享 CLI 适配契约（不共享代码）

**背景**：避免双重维护。

**决策**：
- v1.0：契约一致，各自实现（Python / Node）
- v1.1+：抽取共享 npm + pip 包

**依据**：
- v1.0 优先快速交付
- 实施由 AI 完成，工作量可接受
- 共享层抽取需要稳定的契约（先各自实现稳定）

---

## ADR-018：subagent 缓存默认开启

**背景**：缓存策略。

**决策**：默认开启，TTL 1 小时。

**依据**：
- 长任务 resume 时大幅省 token
- LLM 生成的脚本通常多次重跑
- 风险：缓存误命中。通过 deterministic 哈希降到极低

**影响**：用户可关闭（`--no-cache` / `cache: false`）

---

## ADR-019：默认禁止 tools.runShell 与 tools.httpGet

**背景**：高风险工具默认状态。

**决策**：默认完全禁用，配置可显式开启。

---

## ADR-020：日志 JSONL 格式

**背景**：与 agent-workflows ADR-015 同步。

**决策**：JSONL。

---

## ADR-021：脚本不可在运行后修改

**背景**：脚本不可变性。

**决策**：orchestrator.ts 一旦 `ow run` 则不可修改；script_hash 入 state.json 校验。

**依据**：
- 防止"运行中改代码"造成的不一致
- 修改后只能 `ow run` 启动新 run

---

## ADR-022：v1.0 不实现 worker 池

**背景**：isolate 启动 ~80ms 是否需要预热。

**决策**：v1.0 不预热，按需创建。

**依据**：
- 80ms 对长任务可忽略
- worker 池增加复杂度
- v1.1+ 评估
