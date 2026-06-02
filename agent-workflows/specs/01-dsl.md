# Spec 01 — workflow.json DSL 规范

> 父文档：`../DESIGN.md`

## 1. 顶层结构

```json
{
  "version": "1.0",
  "metadata": {
    "title": "string",
    "description": "string?",
    "tags": ["string"],
    "created_by": "string?",
    "created_at": "ISO8601?"
  },
  "context": { /* 任意键值，用于变量引用 */ },
  "config": {
    "default_cli": "claude|codex|codebuddy",
    "max_parallel": 8,
    "max_subagents_total": 200,
    "default_timeout_seconds": 300,
    "default_retry": { "max_attempts": 2, "backoff": "exponential" },
    "approval_policy": "auto|ask_on_destructive|always_ask"
  },
  "vars": { /* 工作流级初始变量 */ },
  "steps": [ /* 节点数组 */ ]
}
```

## 2. 节点通用字段

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `id` | string | ✓ | `^[a-z][a-z0-9_-]{0,63}$`，全局唯一 |
| `type` | enum | ✓ | 见 §3 |
| `name` | string | ✗ | 人类可读名 |
| `comment` | string | ✗ | 注释 |
| `depends_on` | string \| string[] | ✗ | 显式依赖；省略时默认依赖前一节点 |
| `condition` | expression | ✗ | false 则跳过 |
| `timeout_seconds` | int | ✗ | 覆盖 config.default_timeout |
| `retry` | RetryPolicy | ✗ | 覆盖 config.default_retry |
| `on_error` | ErrorPolicy | ✗ | abort\|skip\|escalate\|fallback |
| `output_var` | string | ✗ | 写入 state.variables 的变量名 |

`RetryPolicy`：
```json
{ "max_attempts": 3, "backoff": "constant|linear|exponential", "backoff_seconds": 2, "on": ["rate_limit", "network", "model_error"] }
```

`ErrorPolicy`：
```json
{ "action": "abort|skip|escalate|fallback", "fallback_step": "step_id?", "escalate_message": "string?" }
```

## 3. 节点类型清单（封闭集）

| Type | 执行方 | 用途 |
|------|--------|------|
| `tool` | Python | 调用注册工具（白名单） |
| `code` | Python 沙箱 | **纯计算**（用户提供代码片段，无副作用） |
| `subagent_call` | CLI 子进程 | 单次 LLM 任务 |
| `subagent_fan_out` | CLI 子进程批 | 并发扇出 |
| `llm_decision` | CLI 子进程 | LLM 在 schema 内做选择 |
| `llm_plan` | CLI 子进程 | 运行时生成新 steps（逃生舱） |
| `branch` | Python | 数据驱动分支 |
| `loop` | Python | while/for-each |
| `sequence` | Python | 嵌套顺序块 |
| `parallel` | Python | 嵌套并行块 |
| `set_var` | Python | 变量赋值 |
| `transform` | Python | 数据变换（受限 operation 集） |
| `human_approval` | Python+Interaction | 人工审批 |
| `sleep` | Python | 延迟 |
| `noop` | Python | 占位 |

**节点类型分类**（按"是否影响外部世界"）：

| 分类 | 节点 | 特点 |
|------|------|------|
| **副作用类** | tool / subagent_* / llm_* / human_approval | 与外部世界交互（文件、LLM、用户） |
| **纯计算类** | code / branch / loop / sequence / parallel / set_var / transform | 仅在变量空间内做计算与流控 |
| **占位类** | sleep / noop | 控制时序 |

`code` 节点专门用于 transform/branch 等内置纯计算节点表达力不足的场景（复杂图算法、自定义评分、多维数据结构操作等），让 DSL 在保留所有安全/审计优势的前提下补齐表达力。

## 4. 各节点详细规范

### 4.1 `tool`

```json
{
  "id": "discover",
  "type": "tool",
  "tool": "find_files",
  "params": { "pattern": "${context.scope}", "exclude": ["**/node_modules/**"] },
  "output_var": "files"
}
```
- `tool` 必须是已注册工具
- `params` 经模板渲染后传入

### 4.2 `subagent_call`

```json
{
  "id": "summarize",
  "type": "subagent_call",
  "agent": {
    "role": "summarizer",
    "cli": "claude",
    "prompt_template": "总结：{{audit_results}}",
    "system_prompt": "你是简洁的技术汇报员",
    "tools": ["read_file"],
    "edit_policy": "readonly",
    "allowed_paths": ["${context.scope}"],
    "output_schema": { "type": "object", "properties": {} },
    "max_tokens": 4000
  },
  "output_var": "summary"
}
```

`agent` 字段对照：

| 字段 | 必填 | 说明 |
|------|------|------|
| `role` | ✓ | 角色名（影响 system prompt 模板） |
| `cli` | ✗ | 默认 config.default_cli |
| `prompt_template` | ✓ | Jinja2 模板 |
| `system_prompt` | ✗ | 覆盖默认 |
| `tools` | ✗ | 通用工具名数组 |
| `edit_policy` | ✗ | readonly\|modify_only\|full（默 readonly） |
| `allowed_paths` | ✗ | 路径白名单（modify 必填） |
| `output_schema` | ✗ | JSON Schema，强制结构化输出 |
| `max_tokens` | ✗ | 单次最大输出 |

### 4.3 `subagent_fan_out`

```json
{
  "id": "audit",
  "type": "subagent_fan_out",
  "items_from": "files",
  "item_var": "file",
  "max_parallel": 8,
  "batch_size": 16,
  "agent": { /* 同 4.2 */ },
  "aggregate": "array|flatten_findings|merge_objects|custom",
  "fail_strategy": "all_or_nothing|best_effort|min_success_rate",
  "min_success_rate": 0.8,
  "output_var": "all_findings"
}
```

| 字段 | 说明 |
|------|------|
| `items_from` | 迭代源（变量引用或字面数组） |
| `item_var` | 当前项变量名（默 `item`） |
| `max_parallel` | 阶段内并发上限 |
| `batch_size` | 单批 OS 进程上限 |
| `aggregate` | array=数组拼接；flatten_findings=取每项 findings 数组并合并；merge_objects=对象合并；custom 需配 transform |
| `fail_strategy` | all_or_nothing/best_effort/min_success_rate |

### 4.4 `llm_decision`

```json
{
  "id": "triage",
  "type": "llm_decision",
  "agent": {
    "role": "triager",
    "prompt_template": "Critical {{critical|length}} 个，决定策略",
    "output_schema": {
      "type": "object", "required": ["strategy"],
      "properties": {
        "strategy": { "enum": ["fix_all", "fix_top_n", "skip"] },
        "top_n": { "type": "integer", "minimum": 1 }
      }
    }
  },
  "output_var": "triage_plan"
}
```

输出强制符合 schema，不符合 → 重试一次 → 仍失败 → 节点失败。

### 4.5 `llm_plan`（逃生舱）

```json
{
  "id": "dynamic_followup",
  "type": "llm_plan",
  "agent": {
    "role": "planner",
    "prompt_template": "根据 {{verified_findings}}，生成下一阶段处理 steps",
    "output_schema": "$ref:#/$defs/steps_array"
  },
  "max_generated_steps": 20,
  "max_recursion_depth": 3,
  "insert_position": "after|at_end"
}
```

- 生成的 steps 必经同一份 schema 校验
- 嵌套深度不得 > `max_recursion_depth`
- `insert_position`：after = 插当前节点之后；at_end = 末尾追加

### 4.6 `branch`

`switch` 形式：
```json
{
  "id": "select_targets",
  "type": "branch",
  "switch": "triage_plan.strategy",
  "cases": {
    "fix_all":   [ /* steps */ ],
    "fix_top_n": [ /* steps */ ],
    "skip":      [ { "id": "x", "type": "noop" } ]
  },
  "default": [ /* steps */ ]
}
```

`if/else` 形式：
```json
{
  "id": "check",
  "type": "branch",
  "if": "high_findings | length > 0",
  "then": [ /* steps */ ],
  "else": [ /* steps */ ]
}
```

### 4.7 `loop`

```json
{
  "id": "fix_loop",
  "type": "loop",
  "mode": "while",
  "condition": "unfixed | length > 0 and iteration < 3",
  "max_iterations": 5,
  "iteration_var": "iteration",
  "body": [ /* steps */ ]
}
```

或 `for_each`：
```json
{ "type": "loop", "mode": "for_each", "items_from": "files", "item_var": "file", "body": [ /* steps */ ] }
```

### 4.8 `sequence` / `parallel`

```json
{
  "id": "review",
  "type": "parallel",
  "max_parallel": 3,
  "branches": [
    { "id": "perf",  "steps": [ /* */ ] },
    { "id": "sec",   "steps": [ /* */ ] },
    { "id": "maint", "steps": [ /* */ ] }
  ]
}
```

`sequence` 同结构但顺序执行。

### 4.9 `set_var`

```json
{ "id": "init", "type": "set_var", "vars": { "round": 0, "errors": [] } }
```

### 4.10 `transform`

```json
{
  "id": "filter_high",
  "type": "transform",
  "input": "all_findings",
  "operation": "filter",
  "expression": "item.severity in ['critical','high']",
  "output_var": "high_findings"
}
```

支持 operation：`filter` / `map` / `sort` / `reduce` / `group_by` / `unique` / `flatten` / `take` / `skip` / `merge`。

> 当 transform 表达不出某种纯计算（复杂评分、图算法、多维聚合等），改用 §4.11 `code` 节点。

### 4.11 `code`（纯计算逃生舱）

> 用 LLM 生成的纯计算代码补齐 DSL 表达力，**不允许任何副作用**。

```json
{
  "id": "compute_priority",
  "type": "code",
  "language": "python",
  "input_vars": ["findings", "dependency_graph"],
  "output_var": "prioritized",
  "timeout_seconds": 5,
  "memory_limit_mb": 128,
  "code": "from collections import defaultdict\n\nscores = defaultdict(float)\nfor f in findings:\n    deps = dependency_graph.get(f['file'], [])\n    scores[f['id']] = len(deps) * 1.5 + (10 if f['severity']=='critical' else 1)\n\nresult = sorted(findings, key=lambda f: scores[f['id']], reverse=True)"
}
```

#### 字段

| 字段 | 必填 | 说明 |
|------|------|------|
| `language` | ✓ | `python`（v1.0 仅支持 python；后续可加 javascript） |
| `input_vars` | ✓ | 注入到沙箱命名空间的变量名列表（必须是已存在的工作流变量） |
| `output_var` | ✓ | 沙箱执行后取回的变量名（沙箱内必须赋值给名为 `result` 的局部变量） |
| `code` | ✓ | 代码字符串 |
| `timeout_seconds` | ✗ | 默认 5；上限 30 |
| `memory_limit_mb` | ✗ | 默认 128；上限 512 |
| `code_id` | ✗ | 代码片段的稳定标识（用于日志/缓存/复用，可选） |

#### 严格约束（违反任一 → 校验失败 / 沙箱拒绝执行）

**禁止**：
- `import` 任何不在白名单内的模块
- 任何形式的 I/O：文件、网络、subprocess、`open`、`socket`
- 调用 `agent.spawn` / 工具 / 数据库（这些只能用 DSL 节点表达）
- 写入除 `result` 以外的全局变量
- 双下划线属性（`__class__`、`__import__` 等）
- `eval` / `exec` / `compile`
- `globals()` / `locals()` / `vars()` 修改命名空间
- 修改 `input_vars` 注入的对象（应当只读）
- 长循环 / 死循环（受 timeout 控制）

**白名单可 import 模块**：
- 数据结构：`collections`、`heapq`、`bisect`、`itertools`、`functools`、`operator`
- 数学：`math`、`statistics`、`fractions`、`decimal`、`random`（仅伪随机）
- 字符串：`re`、`string`、`textwrap`
- 时间：`datetime`、`time.monotonic`（不允许 `time.sleep`）
- JSON：`json`
- 数据处理：`dataclasses`、`enum`

**白名单内置函数**：
- `len/sum/min/max/abs/round`
- `int/float/str/bool/list/tuple/dict/set/frozenset`
- `range/enumerate/zip/map/filter/sorted/reversed`
- `any/all/print`（print 仅写日志，不影响 stdout 协议）
- `isinstance/type`

**禁止内置函数**：
- `open`、`input`、`__import__`、`getattr/setattr/delattr`（受限）、`hasattr`、`callable`、`compile`、`eval`、`exec`、`memoryview`

#### 执行语义

```
1. 引擎解析 input_vars → 从变量空间取值（深拷贝，防止 code 篡改原对象）
2. 注入到沙箱局部命名空间
3. 在沙箱执行 code（受 timeout + memory + AST 白名单约束）
4. 沙箱执行后必须存在名为 `result` 的局部变量
5. 取 result 写入 output_var
6. 任何沙箱违例 / 超时 / 异常 → 节点失败（按 retry/on_error 处理）
```

#### LLM 生成约束

LLM 生成 `code` 节点时必须：
- 代码独立可读（注释说明意图）
- 代码长度 ≤ 80 行（超出 → 应拆为多个 code 节点或考虑用 transform）
- 必须在末尾或显式赋值 `result = ...`
- 不假设 `input_vars` 之外的任何变量存在

引擎在 schema 校验后做**二次静态校验**：
- AST 解析代码
- 检查 import / 调用是否在白名单
- 检查是否赋值了 `result`
- 失败 → 拒绝执行，让 LLM 重新生成

#### 与其他节点的能力边界

| 想做的事 | 应当用的节点（不是 code） |
|---------|------------------------|
| 读文件 | `tool: read_file` |
| 写文件 | `tool: write_file` |
| 派 subagent | `subagent_call` / `subagent_fan_out` |
| LLM 决策 | `llm_decision` |
| 等待用户 | `human_approval` |
| 简单过滤/排序 | `transform`（不要用 code） |
| 调用外部 API | 必须先封装为注册的 `tool` |

> **判定原则**：如果一段逻辑可以表达为"输入数据 → 计算 → 输出数据"且不依赖任何外部资源，才用 code 节点；否则用对应的副作用节点。

#### 代码沙箱实现

详见 `specs/07-security.md` §13。

### 4.12 `human_approval`

```json
{
  "id": "confirm_fix",
  "type": "human_approval",
  "title": "确认修复",
  "summary": "将修改 {{fix_targets|length}} 处代码",
  "details_template": "{% for t in fix_targets %}- {{t.file}}:{{t.line}}\n{% endfor %}",
  "options": ["approve", "reject", "edit"],
  "timeout_seconds": 1800,
  "on_timeout": "reject",
  "auto_approve_when": "config.approval_policy == 'auto'",
  "output_var": "approval_result"
}
```

### 4.13 `sleep` / `noop`

```json
{ "id": "cooldown", "type": "sleep", "seconds": 30 }
{ "id": "ph", "type": "noop", "reason": "..." }
```

## 5. 变量与表达式

### 5.1 变量空间

| 命名空间 | 来源 |
|---------|------|
| `context.*` | workflow.json/context |
| `config.*` | workflow.json/config |
| `vars.*` | workflow.json/vars + set_var |
| `<step_id>.*` 或 `<output_var>.*` | 上游步骤输出 |
| `runtime.*` | run_id / iteration / now / cwd / env |

### 5.2 引用语法

- **JSON 字段中**：`${var.path}` 或 `${var[0]}` —— 渲染时整体替换
- **模板字符串**：`{{ expression }}` —— Jinja2

### 5.3 表达式语言

Python 表达式子集（实施用 `asteval` 或自实现 AST 白名单）：

| 允许 | 禁止 |
|------|------|
| 字面量、变量、算术、比较、布尔 | `import`、`__xxx__`、I/O、subprocess |
| 索引、属性、切片 | `eval`/`exec`、动态属性 |
| 内置：len/sum/min/max/abs/round/int/str/bool/list/dict | lambda 副作用 |
| 字符串只读方法：startswith/endswith/lower/upper/strip/split | 文件/网络函数 |
| 受限列表理解 | 任何写操作 |

### 5.4 模板渲染

Jinja2 `SandboxedEnvironment`：
- 关闭 loader（不能 include/import）
- filter 白名单：length / default / upper / lower / trim / join / first / last / count / json / int / float / sort / unique
- 禁用 `{% set %}` 写工作流变量

## 6. JSON Schema 约束

完整 schema 由 `awe/config/schemas/workflow.schema.json` 定义，遵循 JSON Schema Draft 2020-12，提供 `$defs`：

- `retry_policy` / `error_policy` / `agent_spec` / `cli_name` / `expression` / `step` / `steps_array`

校验流程：
1. JSON Schema 静态校验
2. 引擎层动态校验：tool 名是否注册、CLI 是否可用、变量引用是否定义、循环依赖检测、表达式语法预编译

任一阶段失败 → 拒绝执行。

## 7. 命名规约

| 类别 | 正则 |
|------|------|
| 节点 ID | `^[a-z][a-z0-9_-]{0,63}$` |
| 变量名 | `^[a-z_][a-z0-9_]{0,63}$` |
| 工具名 | `^[a-z_][a-z0-9_]{0,63}$` |
| 角色名 | `^[a-z][a-z0-9_-]{0,63}$` |

## 8. 完整示例

见 `../templates/code-audit.json`、`../templates/migration.json` 等。

## 9. 版本与演化

- DSL 版本由顶层 `version` 字段声明
- 引擎对未知 `version` → 拒绝执行
- 兼容策略：`x.y` 同主版本兼容，跨主版本需迁移工具

## 10. 校验错误规范

引擎拒绝执行时返回：
```json
{
  "error_code": "AWE_DSL_001",
  "stage": "schema|reference|tool|cli|expression|cycle",
  "step_id": "...",
  "message": "...",
  "suggestion": "...?"
}
```

错误码表见 `../specs/_error_codes.md`（实施时建立）。
