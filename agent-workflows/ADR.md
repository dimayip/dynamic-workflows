# ADR — 关键决策记录

> 父文档：`./DESIGN.md`

每条决策含：背景、选项、决策、依据、影响、回顾时机。

---

## ADR-001：引擎语言选 Python 3.11+

**背景**：需要选择 orchestrator 实现语言。

**选项**：
- Python：AI 生态强、跨平台、依赖少、`asyncio` 成熟
- Node.js：与 Claude Code 同构、`Promise` 友好、但跨进程派 CLI 优势不明显
- Go：单二进制分发、性能高、但 AI 生态弱、jinja-like 模板差

**决策**：Python 3.11+

**依据**：
- 用户群体（AI 领域开发者）大概率已装 Python
- jsonschema / jinja2 / asteval / typer / rich 生态完整
- asyncio 在 3.11 后稳定且性能足够
- 易于 AI 后续维护（生成 Python 代码质量高）

**影响**：
- 启动开销略高（200ms 级），但可接受
- Windows 安装稍麻烦，需文档引导
- 单二进制分发需借助 zipapp 或 PyInstaller

**回顾时机**：v2.0 前

---

## ADR-002：DSL 选 JSON 而非 YAML / JSON5 / TOML

**背景**：DSL 格式直接影响 LLM 生成稳定性与可校验性。

**选项**：
- JSON：schema 工具链最成熟、structured output 支持、无歧义、严格
- YAML：人类友好、但缩进敏感、LLM 易错、schema 工具弱
- JSON5：含注释、灵活、但生态小
- TOML：表达力对嵌套结构差

**决策**：JSON

**依据**：
- LLM 在 structured output（JSON Schema）模式下生成 JSON 的可靠性远高于 YAML
- jsonschema 校验工具链成熟
- 错误位置定位清晰

**影响**：
- 不能写注释（用 `comment` 字段补偿）
- 严格语法 → LLM 偶发逗号/引号错误时，需 retry 机制

**回顾时机**：用户反馈强烈要求 YAML 时

---

## ADR-003：Subagent 通过 CLI 子进程派生，不直调 LLM SDK

**背景**：subagent 派生方式决定整体架构。

**选项**：
- A. 直调 LLM SDK（OpenAI/Anthropic Python SDK）
- B. 派 CLI 子进程（`claude -p` / `codex exec`）
- C. 通过宿主 Agent 的 inbox/outbox 协议外包

**决策**：B（CLI 子进程）

**依据**：
- 复用用户已有的 CLI 认证 / 配额 / 工具链
- 跨平台一致：用户用什么 IDE 就用什么 CLI
- 真隔离：每个 subagent 独立进程
- 不需要管理 API key（用户在 CLI 已登录）

**影响**：
- token 成本比 in-session subagent 高 30%-80%
- 启动开销（每次 CLI 冷启动）
- 失去主对话的 prompt cache 复用，但能复用 5 分钟内的 cache

**回顾时机**：CLI 提供 daemon 模式时可优化

---

## ADR-004：异步模型选 asyncio 全栈，不混合多线程

**背景**：并发实现选择。

**选项**：
- asyncio 全栈：subprocess + I/O 全异步
- threading + sync：每个 subagent 一个线程
- multiprocessing：每个 subagent 一个 worker 进程

**决策**：asyncio 全栈

**依据**：
- subprocess 是 I/O bound，asyncio 最适合
- 单进程内更易做全局调度（信号量、token bucket、AIMD）
- 调试更简单（无锁竞争）

**影响**：
- 工具实现需要异步友好（用 aiofiles 等）
- 误用同步 I/O 会阻塞事件循环 → 用 `asyncio.to_thread` 兜底

**回顾时机**：性能瓶颈出现时

---

## ADR-005：状态用 state.json 单文件 + 原子写，不引入 DB

**背景**：状态持久化方案。

**选项**：
- 单 JSON 文件 + 原子写
- SQLite
- 多文件分片（每节点一个文件）

**决策**：单 JSON 文件

**依据**：
- 状态体积可控（几百 KB 量级）
- 单文件易备份、易诊断、易 git diff
- 原子写（tmp + rename）足够安全
- 不引入额外依赖

**影响**：
- 大工作流（>1000 节点）状态文件可能数 MB，写入有开销
- 周期快照 + 事件触发写入综合控制开销

**回顾时机**：超大规模工作流出现时引入 SQLite 后端

---

## ADR-006：不保留 inbox/outbox 协议作为 fallback

**背景**：CLI 不可用时是否退化为"宿主 Agent 派 subagent"模式。

**选项**：
- 保留：CLI 不可用 → 自动切换 inbox/outbox
- 不保留：CLI 不可用 → fail-fast

**决策**：不保留

**依据**：
- inbox/outbox 模式有"软约束"问题（前文已分析）
- 维护两套实现增加复杂度
- 用户应在 `awe doctor` 阶段就发现 CLI 缺失
- fail-fast 更明确

**影响**：
- 用户必须装至少一个 CLI 才能用
- 错误信息需明确引导用户装 CLI

**回顾时机**：v2.0 评估

---

## ADR-007：双入口共享 orchestrator API + 可插拔 InteractionAdapter

**背景**：Skill 与 CLI 入口如何共用代码。

**决策**：唯一 orchestrator API + 两个 InteractionAdapter（Stdio / SkillProtocol）。

**依据**：
- DRY：核心引擎代码 100% 复用
- 差异隔离在交互层：替换 Adapter 即可

**影响**：
- Adapter 接口需稳定（v1.0 后向兼容）
- 测试需双入口都覆盖

---

## ADR-008：默认 CLI 与宿主同款；CLI 入口 doctor 探测

**背景**：用户在 Claude Code 内调用时，subagent 应优先用 claude；纯 CLI 时由 doctor 探测可用。

**决策**：
- Skill 入口：从环境变量 `AWE_HOST_AGENT` 推断默认 CLI
- CLI 入口：doctor 探测，用户未指定时选第一个可用

**依据**：用户预期：在哪个 IDE 就用哪个 LLM。

---

## ADR-009：表达式语言用 Python 子集 + asteval（或自实现 AST 白名单）

**背景**：DSL 中需要表达式（branch.condition / transform.expression）。

**选项**：
- Python 子集（asteval）：用户熟悉、表达力够
- jq / JMESPath：更窄但不够通用
- 自定义 DSL：高成本

**决策**：Python 子集 + AST 白名单

**依据**：
- 用户/LLM 都熟悉 Python 语法
- AST 白名单确保安全
- 易于实现

**影响**：需仔细维护白名单防绕过。

> 与 ADR-021 协同：本决策的 AST 白名单基础设施可复用为 code 节点沙箱的内核，二者共享 `awe/sandbox/ast_validator.py` 中的核心校验逻辑（差异在白名单宽度：表达式更窄，code 节点稍宽）。

---

## ADR-010：直接产品级 v1.0，不分 MVP 阶段

**背景**：是否分阶段实施。

**决策**：一次性产品级实现。

**依据**：
- 实施由 AI 完成，工程量瓶颈不在人力
- 用户明确拒绝 MVP
- 设计已冻结，无需迭代发现需求

**影响**：实施周期更长，但避免重构成本。

---

## ADR-011：JSON Schema Draft 2020-12

**背景**：选择 schema 标准。

**决策**：Draft 2020-12（最新稳定）。

**依据**：
- Python `jsonschema` 库支持
- 含 `$dynamicRef` 等高级特性（用于 llm_plan 校验）
- 与现代 LLM structured output 兼容

---

## ADR-012：错误码用 AWE_<MODULE>_<NUM> 格式

**背景**：错误码命名。

**决策**：`AWE_DSL_001` / `AWE_CLI_002` 等。

**依据**：模块前缀便于定位、数字递增便于检索、字符串易读。

**影响**：维护错误码表需自动化（`scripts/update-error-codes.py`）。

---

## ADR-013：默认禁用 run_shell 与 http_get

**背景**：安全 vs 易用性权衡。

**决策**：默认禁用，用户显式开启。

**依据**：
- LLM 生成的 workflow 可能误调 shell 造成损害
- 大部分工作流（审计、修复、文档）不需要 shell
- 易用性损失可接受

**影响**：模板中需要 shell 时必须显式标注 + 用户确认。

---

## ADR-014：subagent 工作目录默认共享主目录

**背景**：subagent 隔离粒度。

**决策**：默认共享，节点可显式 `subagent_cwd` 隔离。

**依据**：
- 与 Claude Code 行为一致
- 大部分场景需要访问完整代码库
- 隔离需求由 `allowed_paths` 满足

---

## ADR-015：日志采用 JSONL，不用 plain text

**背景**：日志格式。

**决策**：默认 JSONL（machine-readable）。

**依据**：
- 便于工具消费（grep/jq）
- 字段化 → 易于过滤
- `--verbose` 也是 JSONL 而非 plain text，避免双重维护

**影响**：人类直接读 .jsonl 不友好 → 提供 `awe logs` 格式化展示。

---

## ADR-016：CLI 命令名 `awe`（短）+ `agent-workflows`（长别名）

**背景**：命令名取舍。

**决策**：短名 `awe` + 长别名 `agent-workflows`。

**依据**：
- 短名好打、不冲突（已查 npm/pypi）
- 长名描述性强、备用避免冲突

**影响**：包名 `agent-workflows` 上 PyPI；命令脚本同时安装两个名字。

---

## ADR-017：v1.0 不支持多机/多用户共享 .workflow

**背景**：分布式状态。

**决策**：v1.0 仅本地文件后端。

**依据**：
- 复杂度大幅提升
- 用户群体目前以单机为主
- 通过 StateBackend 抽象保留扩展点

**回顾时机**：用户社区出现明确需求

---

## ADR-018：默认 locale 中文，英文 fallback

**背景**：多语言策略。

**决策**：根据 `general.locale` 或系统 LANG 自适应；缺失字段用英文兜底。

**依据**：作者主要中文用户，但代码与文档需面向英文社区。

---

## ADR-019：使用 typer 而非 click 或 argparse

**背景**：CLI 框架。

**决策**：typer。

**依据**：
- 类型注解友好、与 mypy strict 兼容
- 自带 shell 自动补全
- 子命令组织清晰

---

## ADR-020：使用 rich 做终端展示

**背景**：终端 UI。

**决策**：rich。

**依据**：进度条、表格、彩色、跨平台（Windows 也支持）。

---

## ADR-021：引入 `code` 节点作为纯计算逃生舱

**背景**：DSL 内置的 transform / branch / loop 等节点无法表达某些复杂纯计算（图算法、自定义评分、多维聚合、复杂数据结构操作）。一种朴素方案是退回 JS 代码生成（V8 沙箱执行），但会失去 DSL 的全部安全/审计/可视化优势。

**选项**：
- A. 不加 code 节点，遇到表达力不足时让 LLM 改用更多 transform / branch 拼凑（语义晦涩、节点爆炸）
- B. 引入完全自由的 code 节点（可调 spawn / 工具 / I/O）→ 退化为 Claude Code 风格全代码方案
- C. **引入"纯计算"code 节点**（受限沙箱、零副作用、必须通过白名单 AST 校验）

**决策**：C。

**依据**：
- 工业先例：AWS Step Functions + Lambda、Airflow + PythonOperator、Temporal + Activity、LangGraph + Python 节点、n8n + Function 节点、Dify + Code 节点——全是"DSL 编排 + 代码计算"的成熟分工。
- 纯计算限制让 code 节点无法影响外部世界，所有 DSL 工程优势保留：DAG 可视化、审批门可静态发现、断点续跑、安全审计、跨语言可解析。
- 表达力补齐：原 5%-10% 的"非 LLM 复杂计算"差距被填平。
- LLM 生成纯函数代码的稳定性远高于生成完整 orchestrator 脚本（输入/输出明确、长度短、无副作用约束）。

**严格约束（详 specs/01 §4.11、specs/07 §13）**：
- 仅在白名单模块内 import（collections / math / re / json / itertools / functools 等）
- 禁止任何 I/O：文件、网络、subprocess、socket
- 禁止 spawn subagent / 调工具
- 禁止双下划线属性、eval / exec / compile
- 必须赋值 `result` 局部变量
- 受 timeout（默 5s，上限 30s）+ 内存限制（默 128 MB，上限 512 MB）
- 输入对象深拷贝注入，禁止修改
- 引擎二次静态校验 AST + 受限内置 + 子进程执行

**影响**：
- DSL 表达力达到"形式上图灵完备 + 实用上覆盖 100% 编排场景"
- 沙箱实现需要新增 `awe/sandbox/` 模块（AST 校验器 + 受限运行器 + 资源限制）
- LLM 生成约束需要在 SKILL.md 与 awe gen prompt 中明确引导
- 新增错误码 AWE_SEC_007/008/009/010
- 跨平台资源限制：Windows 需用 Job Objects 或子进程监控替代 resource 模块

**回顾时机**：
- 用户反馈 5s 超时不够（考虑可配置上调上限）
- 出现 Python 沙箱逃逸 CVE 时（重新评估方案）
- 决定支持 javascript code 节点时（新增 ADR）

**驳回 Option B 的理由**：
- 全代码 orchestrator = 失去 DSL 所有差异化优势
- 已在 ADR-002 / ADR-006 / ADR-010 系列决策中明确定位
- 全代码方案更接近"开源版 Claude Code"，是不同产品

---
