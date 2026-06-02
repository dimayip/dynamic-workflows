# Spec 03 — CLI Subagent 适配层

> 父文档：`../DESIGN.md`

## 1. 抽象接口

```python
# awe/cli_adapter/base.py
class CliAdapter(ABC):
    name: str                              # "claude"|"codex"|"codebuddy"
    rate_limits: RateLimitsConfig          # 默认值

    @abstractmethod
    async def detect(self) -> DetectResult: ...
    # 返回：installed/version/authenticated/notes

    @abstractmethod
    async def spawn(self, request: SubagentRequest, cwd: Path, env: dict) -> SubagentResult: ...

    @abstractmethod
    def translate_tools(self, generic_tools: list[str]) -> list[str]: ...

    @abstractmethod
    def parse_output(self, stdout: str, stderr: str, exit_code: int) -> dict: ...

    def normalize_error(self, exit_code: int, stderr: str) -> ErrorCategory: ...
    # 返回：rate_limit | auth_error | network | invalid_input | model_error | timeout | unknown
```

## 2. SubagentRequest / Result

```python
@dataclass
class SubagentRequest:
    task_id: str
    role: str
    prompt: str
    system_prompt: str | None
    tools: list[str]                       # 通用名
    edit_policy: Literal["readonly","modify_only","full"]
    allowed_paths: list[str]
    output_schema: dict | None
    max_tokens: int | None
    timeout_seconds: int

@dataclass
class DetectResult:
    name: str
    installed: bool
    version: str | None
    authenticated: bool | None
    notes: str
    capabilities: dict                     # 该 CLI 支持哪些特性
```

## 3. 各 CLI 适配实现要点

### 3.1 Claude (`claude` CLI)

启动命令：
```bash
claude -p "<prompt>" \
  --output-format json \
  --permission-mode bypassPermissions \
  --allowed-tools "Read,Grep,Edit" \
  --append-system-prompt "<system>" \
  --max-turns 10 \
  --cwd <path>
```

输出解析：
- `--output-format json` 返回单个 JSON 对象
- 关键字段：`result`（最终回复）、`usage`（token 信息）、`is_error`
- 当 `output_schema` 存在 → 在 prompt 末尾追加："请以下列 JSON schema 输出：..."，从 `result` 中提取 JSON 块

错误识别：
- exit_code != 0 + stderr 含 `rate limit` → `rate_limit`
- stderr 含 `unauthorized` / `not logged in` → `auth_error`
- stderr 含 `network` / `timeout` → `network`

### 3.2 Codex (`codex` CLI)

启动命令：
```bash
codex exec "<prompt>" \
  --full-auto \
  --json \
  --cwd <path>
```

输出解析：
- JSONL 流，逐行解析
- 最后一条 `type: "response"` 的对象为最终结果

工具传递：通过 prompt 引导（codex CLI 的工具控制能力较弱时）。

### 3.3 CodeBuddy (`codebuddy` CLI)

启动命令（占位，待官方文档确认）：
```bash
codebuddy run "<prompt>" \
  --headless \
  --output json \
  --cwd <path>
```

实施前必做：查阅 CodeBuddy 官方 CLI 文档，补全实际参数。

## 4. 工具语义映射表

```python
# awe/cli_adapter/tool_mapping.py
TOOL_MAP = {
    "claude": {
        "read_file":  "Read",
        "write_file": "Write",
        "edit_file":  "Edit",
        "grep":       "Grep",
        "glob":       "Glob",
        "run_shell":  "Bash",
        "web_fetch":  "WebFetch",
    },
    "codex": {
        "read_file":  "read",
        "write_file": "write",
        "edit_file":  "edit",
        "grep":       "grep",
        "glob":       "glob",
        "run_shell":  "shell",
        "web_fetch":  "fetch",
    },
    "codebuddy": { /* 待补 */ },
}
```

未映射工具 → 适配器抛 `UnsupportedToolError` → 引擎层 fail-fast。

## 5. 输出协议（强制结构化）

策略优先级：
1. 优先使用 CLI 的 `--output-format json` / `--json`
2. prompt 末尾追加：`请以下列 JSON Schema 严格输出最终结果，且 JSON 必须包裹在 \`\`\`json ... \`\`\` 内：<schema>`
3. 解析失败 → 重试一次（带"格式错误"的纠正提示）
4. 仍失败 → 任务标 `failed`，按 fail_strategy 处理

提取规则（按顺序尝试）：
- a. CLI 原生 JSON 模式的字段直接取
- b. `result` 字段中提取 ` ```json ... ``` ` 代码块
- c. `result` 字段中提取首个 `{...}` 平衡括号块
- d. 整体 stdout 解析（最后兜底）

## 6. 路径权限隔离

- `--cwd <path>`：限定工作目录
- `--allowed-tools` / `--disallowed-tools`：工具白名单
- **二次校验**：subagent 返回的文件路径，主进程必须再次校验是否在 `allowed_paths` 内（防 prompt 注入诱导越界）

## 7. 进程管理

```python
async def spawn(self, req, cwd, env):
    cmd = self._build_cmd(req)
    proc = await asyncio.create_subprocess_exec(
        *cmd,
        stdin=asyncio.subprocess.DEVNULL,
        stdout=asyncio.subprocess.PIPE,
        stderr=asyncio.subprocess.PIPE,
        cwd=str(cwd),
        env=self._sanitize_env(env),
    )
    try:
        stdout, stderr = await asyncio.wait_for(
            proc.communicate(), timeout=req.timeout_seconds
        )
    except asyncio.TimeoutError:
        proc.terminate()
        try:
            await asyncio.wait_for(proc.wait(), timeout=5)
        except asyncio.TimeoutError:
            proc.kill()
            await proc.wait()
        raise SubagentTimeout(req.task_id)
    return self._build_result(proc.returncode, stdout, stderr, ...)
```

## 8. 环境变量隔离

`_sanitize_env`：
- 继承 PATH / HOME / SHELL / TERM 等系统变量
- 显式传递必要的 CLI 配置（`ANTHROPIC_API_KEY` 等，仅在用户已配置时）
- **不传 awe 自身的内部状态变量**
- **不传未声明的 secrets**（默认黑名单 + 用户白名单）

## 9. CLI 检测与降级

启动时由 `cli_adapter.registry`：
1. 对每个适配器调 `detect()`
2. 收集可用 CLI 列表
3. 如果 workflow 指定 CLI 不可用 → fail-fast，输出明确错误码 + 修复建议
4. 如果用户未指定 → 选择 config.default_cli；若仍不可用 → 选择第一个可用

`awe doctor` 命令使用同一 detect 逻辑产生健康报告。

## 10. Prompt Cache 利用策略

- 同一 fan_out 内所有 subagent 使用**完全相同**的 system_prompt（影响 cache prefix）
- system_prompt 渲染后做 hash，记入 metrics 用于事后分析 cache 命中率
- 实施层：CLI 提供原生 cache control 的（如 Claude）→ 添加参数；不提供的 → 依靠 Anthropic 服务端自动缓存（5 分钟窗口）

## 11. CLI 适配测试矩阵

每个适配器必须有：
- detect() 返回正确状态的测试（已安装 + 已登录、未安装、未登录）
- spawn() 用 mock 子进程的输出解析测试
- normalize_error() 对各类 stderr 的分类测试
- translate_tools() 完整映射测试
- 端到端测试（真实 CLI，可选 skip）

## 12. 添加新 CLI 的步骤

1. 实现 `CliAdapter` 子类
2. 添加到 `cli_adapter/registry.py`
3. 补全 `tool_mapping.py`
4. 写适配测试
5. 更新 `awe doctor` 输出
6. 更新 `01-dsl.md` 的 `cli_name` 枚举（若需要硬枚举）
