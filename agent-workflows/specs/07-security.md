# Spec 07 — 安全模型

> 父文档：`../DESIGN.md`

## 1. 威胁模型

| 威胁 | 描述 | 缓解 |
|------|------|------|
| Prompt 注入 | LLM 生成的 workflow.json 含恶意意图 | schema 强约束、用户审批、工具白名单 |
| Subagent 输出注入 | subagent 返回内容诱导越权操作 | 输出走 schema 校验、路径二次校验 |
| 表达式注入 | DSL 表达式执行任意代码 | sandbox + AST 白名单 |
| 模板注入 | Jinja 模板被构造执行任意代码 | SandboxedEnvironment + filter 白名单 |
| **`code` 节点逃逸** | LLM 生成的 code 节点尝试 I/O / 系统访问 | AST 白名单 + 受限 import + 资源限制 + 二次静态校验 |
| 路径越界 | subagent 修改 allowed_paths 外文件 | 主进程二次校验路径 |
| 命令注入 | tool/run_shell 拼接用户输入 | 默认禁用 / 受限 / 参数化 |
| 密钥泄露 | secrets 出现在日志 / state | 默认黑名单 + 脱敏 |
| 资源耗尽 | 无限循环 / 大并发 | max_iterations / max_subagents_total / max_recursion_depth |

## 2. 工具白名单

仅以下工具默认注册：

| 工具 | 类别 | 写入能力 | 默认状态 |
|------|------|---------|---------|
| `find_files` | 文件 | 否 | 启用 |
| `read_file` | 文件 | 否 | 启用 |
| `write_file` | 文件 | 是 | 启用（受 allowed_paths） |
| `edit_file` | 文件 | 是 | 启用（受 allowed_paths） |
| `grep` | 搜索 | 否 | 启用 |
| `glob` | 搜索 | 否 | 启用 |
| `git_diff` | git | 否 | 启用 |
| `git_log` | git | 否 | 启用 |
| `git_status` | git | 否 | 启用 |
| `http_get` | 网络 | 否 | **默认禁用**（需配置开启） |
| `run_shell` | 系统 | 是 | **默认禁用**（需配置开启 + 命令白名单） |

每工具签名：

```python
@tool_registry.register("find_files")
class FindFilesTool(Tool):
    schema = {  # 参数 JSON Schema
        "type": "object",
        "required": ["pattern"],
        "properties": {
            "pattern": {"type": "string"},
            "exclude": {"type": "array", "items": {"type": "string"}},
            "max_results": {"type": "integer", "default": 1000}
        }
    }
    async def execute(self, params: dict, ctx: ToolContext) -> Any: ...
```

## 3. 路径访问控制

```python
class PathPolicy:
    def __init__(self, allowed: list[str], forbidden: list[str] | None = None):
        # 转为绝对路径 + glob 匹配规则
        ...

    def check_read(self, path: Path) -> None:
        if not self._matches_allowed(path):
            raise PathAccessDenied(path)

    def check_write(self, path: Path) -> None:
        self.check_read(path)
        if self._matches_forbidden(path):
            raise PathAccessDenied(path)
```

应用点：
- 所有内置工具调用前校验
- subagent 返回的"修改文件清单"二次校验

默认禁止：
- 工作目录之外（除非 allowed_paths 明确）
- `.git/` 内部（除非显式开启）
- `.env` / `secrets/` / `credentials/`（路径黑名单）
- 用户家目录内的隐藏文件（`.aws/`、`.ssh/`、`.kube/`）

## 4. 表达式与模板沙箱

### 4.1 表达式

实施推荐 `asteval` 或自实现 AST 白名单：

允许节点：
- `Constant`、`Name`、`Load`
- `BinOp`、`UnaryOp`、`BoolOp`、`Compare`
- `Subscript`、`Attribute`（受限）
- `Call`（仅白名单函数）
- `List`、`Tuple`、`Dict`、`Set`
- `IfExp`、`ListComp`（受限循环深度）

禁止节点：
- `Import`、`ImportFrom`
- `Lambda`、`FunctionDef`
- `Assign`（写操作）
- `Yield`、`Await`
- 双下划线属性

### 4.2 模板

Jinja2 `SandboxedEnvironment`：
- 不传 loader（无法 include/import）
- filter 白名单：`length` / `default` / `upper` / `lower` / `trim` / `join` / `first` / `last` / `count` / `json` / `int` / `float` / `sort` / `unique`
- 禁用：`{% set %}` 全局、`{% include %}`、`{% import %}`、自定义扩展

## 5. 密钥与环境变量

### 5.1 密钥黑名单（默认从环境隔离）

```python
SECRET_KEYS_BLACKLIST = [
    r"(?i).*token.*",
    r"(?i).*key.*",
    r"(?i).*secret.*",
    r"(?i).*password.*",
    r"(?i).*credential.*",
    r"AWS_.*",
    r"GOOGLE_.*",
    r"AZURE_.*",
]
```

### 5.2 白名单（必须显式声明）

workflow.json/config 可声明：
```json
{ "config": { "subagent_env_passthrough": ["ANTHROPIC_API_KEY", "OPENAI_API_KEY"] } }
```

只有白名单内的环境变量会传给 subagent。

### 5.3 日志脱敏

写入日志前过滤：
- 命中黑名单的环境变量值替换为 `***`
- workflow.json 中的字符串字段，命中常见 secret 模式（高熵 + 长度阈值）→ 提示警告

### 5.4 不在 workflow.json 中存放密钥

- schema 层警告：字符串字段值匹配密钥模式 → 警告
- 提供 `${env.SECRET}` 引用语法（v1.1+）

## 6. Subagent 沙箱

每个 subagent 子进程隔离：
- `--cwd` 限定工作目录
- `--allowed-tools` 工具白名单
- `--disallowed-tools` 工具黑名单
- 网络访问：依赖各 CLI 的能力（claude `--no-network` 等，若支持）
- 环境变量：仅白名单 passthrough

## 7. 资源耗尽防护

| 限制 | 默认 | 配置项 |
|------|------|--------|
| 单工作流总 subagent 数 | 200 | `config.max_subagents_total` |
| 单 fan_out 节点 max_parallel | 16 | `节点.max_parallel` |
| llm_plan 嵌套深度 | 3 | `节点.max_recursion_depth` |
| llm_plan 单次生成 steps | 20 | `节点.max_generated_steps` |
| loop 最大迭代 | 50 | `节点.max_iterations` |
| 单 subagent 超时 | 300s | `节点.timeout_seconds` |
| 总运行时长 | 7200s | `config.max_run_seconds` |

超限 → 立即终止 + 写状态。

## 8. 审计日志

`logs/orchestrator.jsonl` 必须包含：
- 启动事件（含 awe 版本、CLI 版本、用户配置摘要）
- 每个节点开始/完成
- 每次 subagent spawn 与结果
- 每次审批请求与决策
- 每次错误（脱敏后）
- 每次工具调用（含参数摘要、不含完整内容）
- 退出事件（含总耗时、总 token、状态）

日志不可篡改要求：
- 仅追加（append-only）
- 启动时记录初始 hash
- 退出时计算最终 hash 写入 meta.json（v1.1+）

## 9. 默认安全配置

```toml
# ~/.awe/config.toml （默认）
[security]
allow_run_shell = false
allow_http_get = false
allow_modify_outside_cwd = false
mask_secrets_in_logs = true
forbidden_paths = [".env", "secrets/**", ".git/**", "~/.ssh/**", "~/.aws/**"]
subagent_env_passthrough = []   # 默认空，需用户显式声明
```

工作流可在 config 字段覆盖（但不能比默认更宽松，除非用户在全局配置中显式允许）。

## 10. CI/CD 安全模式

`--ci-mode` 标志：
- 禁用 `human_approval`（节点遇到 → fail）
- 强制 `auto_approve_when` 必须为常量表达式
- 禁用 `run_shell`、`http_get`
- 强制启用日志脱敏

## 11. 审计与合规

- final-report.md 含"安全摘要"章节：列出本次运行涉及的写入路径、shell 调用、网络请求
- 可选 `--audit-mode`：所有 subagent 强制 readonly + 不实际执行写入工具

## 12. 安全相关错误码

| Code | 含义 |
|------|------|
| `AWE_SEC_001` | 路径越界访问尝试 |
| `AWE_SEC_002` | 模板/表达式尝试调用禁用函数 |
| `AWE_SEC_003` | 密钥模式出现在 workflow.json |
| `AWE_SEC_004` | 资源耗尽超限 |
| `AWE_SEC_005` | 工具未注册或被禁用 |
| `AWE_SEC_006` | 子进程环境变量违规 |
| `AWE_SEC_007` | code 节点 AST 校验失败（违禁 import / 调用） |
| `AWE_SEC_008` | code 节点超时或内存超限 |
| `AWE_SEC_009` | code 节点未赋值 result 变量 |
| `AWE_SEC_010` | code 节点尝试修改 input_vars 注入对象 |

## 13. `code` 节点沙箱模型

### 13.1 核心原则

> code 节点是**纯计算逃生舱**，不允许任何副作用。所有副作用必须通过对应 DSL 节点表达。

### 13.2 沙箱实现

推荐方案：**自实现 AST 白名单 + RestrictedPython 加固**。

执行流程：

```
1. 读取 code 字段
2. ast.parse 解析为 AST
3. 遍历 AST 节点，检查：
   - 禁止节点类型（Import 中的非白名单模块、Lambda 副作用等）
   - 禁止内置函数调用（open、eval、exec 等）
   - 禁止双下划线属性访问
4. AST 检查通过 → 编译为 code object（不允许 import 自定义模块）
5. 准备受限 globals/locals 命名空间：
   - __builtins__ = 仅保留白名单内置
   - 注入 input_vars（深拷贝）
   - 注入白名单模块（如 collections, math）
6. 在子进程中执行（用 multiprocessing 或 subprocess）以做：
   - 内存限制（resource.RLIMIT_AS）
   - CPU 限制（resource.RLIMIT_CPU 或 timeout）
   - signal.alarm 兜底超时
7. 子进程结束后从 result 变量取值
8. 异常 / 超时 / 违规 → 标记节点失败
```

### 13.3 白名单严格定义

#### 13.3.1 允许的 AST 节点类型

```
Module, Expression, Expr, Constant, Name, Load, Store, Del,
Assign, AugAssign, AnnAssign,
BinOp, UnaryOp, BoolOp, Compare,
List, Tuple, Dict, Set, ListComp, SetComp, DictComp, GeneratorExp,
Subscript, Slice, Index, Attribute,
Call, keyword, Starred,
If, IfExp, For, While, Break, Continue, Pass,
FunctionDef, Return, arguments, arg,
Lambda, comprehension,
Try, ExceptHandler, Raise,
With, withitem,
ClassDef（仅含义为 dataclass / namedtuple 的简单定义）
```

#### 13.3.2 禁止的 AST 节点类型

```
Import 中的非白名单模块
ImportFrom 中的非白名单模块
Global, Nonlocal（影响命名空间）
Yield, YieldFrom, Await, AsyncFunctionDef, AsyncFor, AsyncWith
Exec（Python 2 残留）
```

#### 13.3.3 白名单 import 模块（与 01-dsl.md §4.11 一致）

```python
ALLOWED_MODULES = {
    "collections", "heapq", "bisect", "itertools", "functools", "operator",
    "math", "statistics", "fractions", "decimal", "random",
    "re", "string", "textwrap",
    "datetime",
    "json",
    "dataclasses", "enum",
}
```

#### 13.3.4 禁止的属性访问模式

通过 AST 在 `Attribute` 节点上检查：
- 任何以 `__` 开头的属性名
- `subprocess`、`os`、`sys`、`socket`、`urllib`、`http`、`pathlib` 等危险模块的属性

#### 13.3.5 禁止内置函数

通过受限 `__builtins__` 实现，仅暴露：
```python
SAFE_BUILTINS = {
    "len", "sum", "min", "max", "abs", "round", "pow", "divmod",
    "int", "float", "str", "bool", "list", "tuple", "dict", "set", "frozenset", "bytes",
    "range", "enumerate", "zip", "map", "filter", "sorted", "reversed",
    "any", "all", "isinstance", "type",
    "print",   # 仅 redirect 到日志
    "True", "False", "None",
    "ValueError", "KeyError", "IndexError", "TypeError",  # 异常类
}
```

### 13.4 资源限制

| 资源 | 默认 | 上限 | 实施 |
|------|------|------|------|
| 执行时间 | 5s | 30s | signal.alarm + multiprocessing.Process.join(timeout) |
| 内存 | 128 MB | 512 MB | resource.RLIMIT_AS（Linux/macOS）+ memory_profiler 监测 |
| CPU 时间 | 5s | 30s | resource.RLIMIT_CPU |
| 子进程创建 | 0 | 0 | 禁用（不在白名单） |
| 文件描述符 | 0 个新增 | 0 | resource.RLIMIT_NOFILE = 当前 |
| 网络 | 禁用 | 禁用 | socket 模块不在白名单 |

Windows：用 Job Objects 限制（实施时引入 `pywin32`）。

### 13.5 输入对象保护

`input_vars` 注入的值必须是**深拷贝**：
- 防止 code 节点修改原对象影响其他节点
- 用 `copy.deepcopy` 实现
- 大对象（> 10 MB）警告并询问是否继续

### 13.6 输出约定

执行结束后：
- 沙箱必须存在名为 `result` 的局部变量 → 写入 output_var
- 不存在 → AWE_SEC_009 节点失败
- result 必须是 JSON 可序列化（写 state.json 用）→ 否则 AWE_SEC_009

### 13.7 LLM 生成 code 节点的约束

SKILL.md 与 awe gen 的 system prompt 必须告诉 LLM：

- 仅用于纯计算
- 不要尝试 import 白名单外的模块
- 必须赋值 result
- 优先用 transform 节点；只在 transform 不够时才用 code
- 代码 ≤ 80 行
- 写注释说明意图

### 13.8 沙箱测试矩阵

每条违规模式都必须有单元测试：

```python
# 必须拒绝执行
- import os
- import subprocess
- open('/etc/passwd')
- eval('...')
- exec('...')
- compile('...', '', 'exec')
- __import__('os')
- getattr(obj, '__class__')
- 写全局变量
- 修改 input_vars 注入对象
- 死循环 / 长循环
- 大内存分配
- 调用 globals() / locals()

# 必须允许执行
- 复杂数据变换
- 自定义函数 + 递归（适度）
- 列表/字典理解
- 数学计算
- 字符串处理
- json.dumps / json.loads
- 白名单模块的所有合理用法
```

### 13.9 跨平台考量

| 平台 | 资源限制方案 |
|------|------------|
| Linux | resource 模块 + setrlimit |
| macOS | resource 模块 + setrlimit（部分限制软上限） |
| Windows | Job Objects（pywin32）/ 进程超时 + 内存监控线程兜底 |

Windows 不能使用 `resource` 模块时，**降级为子进程 + 监控**：父进程定期检查子进程内存，超限则 kill。
