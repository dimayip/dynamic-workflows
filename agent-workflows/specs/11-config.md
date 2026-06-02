# Spec 11 — 配置管理

> 父文档：`../DESIGN.md`

## 1. 配置层级（优先级从低到高）

```
1. 内置默认值（awe/config/defaults.toml）
2. 系统级（/etc/awe/config.toml，仅 Linux/macOS）
3. 用户级（~/.awe/config.toml）
4. 项目级（./.awe/config.toml）
5. 显式指定（--config <path>）
6. workflow.json 的 config 字段
7. 节点级配置
8. CLI 命令行参数
```

后者覆盖前者，深度合并（dict 字段合并，list 字段整体替换）。

## 2. 配置格式

TOML（人类可读 + 注释友好）。

## 3. 完整配置项

```toml
# ~/.awe/config.toml

[general]
default_cli = "claude"             # claude|codex|codebuddy
workflow_dir = ".workflow"          # 相对当前目录
locale = "zh-CN"                    # 错误信息 / 报告语言

[security]
allow_run_shell = false
allow_http_get = false
allow_modify_outside_cwd = false
mask_secrets_in_logs = true
forbidden_paths = [".env", "secrets/**", ".git/**", "~/.ssh/**", "~/.aws/**"]
subagent_env_passthrough = []       # 默认空，需显式声明

[concurrency]
max_parallel = 8
max_subagents_total = 200
default_timeout_seconds = 300
max_run_seconds = 7200

[concurrency.cli_limits.claude]
rpm = 60
tpm = 200_000

[concurrency.cli_limits.codex]
rpm = 60
tpm = 200_000

[retry]
max_attempts = 2
backoff = "exponential"
backoff_seconds = 2
max_backoff_seconds = 60
jitter_seconds = 1
on = ["rate_limit", "network", "model_error", "timeout", "subprocess_crash"]

[approval]
policy = "ask_on_destructive"       # auto|ask_on_destructive|always_ask
default_timeout_seconds = 1800
on_timeout = "reject"

[llm_plan]
max_recursion_depth = 3
max_generated_steps_per_call = 20

[loop]
max_iterations = 50

[observability]
log_level = "info"
log_format = "jsonl"                 # jsonl|text
trace_enabled = false
trace_endpoint = ""                  # OTLP endpoint
report_language = "zh-CN"

[cli_adapter.claude]
binary = "claude"                    # PATH 中的命令
extra_args = []                      # 始终附加
output_format = "json"

[cli_adapter.codex]
binary = "codex"
extra_args = []

[cli_adapter.codebuddy]
binary = "codebuddy"
extra_args = []

[generation]
default_gen_cli = "claude"           # 用哪个 CLI 生成 workflow.json
include_examples = true              # 在生成 prompt 中包含 few-shot
schema_validate_retries = 2          # schema 失败时让 LLM 重新生成的次数

[pricing]
enabled = true                       # 启用成本估算
table = "builtin"                    # builtin|<custom-path>

[ui]
color = true
progress_style = "rich"              # rich|plain|json
```

## 4. workflow.json 中的 config 字段

可以覆盖以下子集（不能比全局更宽松）：

```json
{
  "config": {
    "default_cli": "claude",
    "max_parallel": 8,
    "max_subagents_total": 200,
    "default_timeout_seconds": 300,
    "default_retry": { "max_attempts": 2, "backoff": "exponential" },
    "approval_policy": "ask_on_destructive",
    "subagent_env_passthrough": ["ANTHROPIC_API_KEY"]
  }
}
```

引擎在合并时校验：
- 不允许工作流降低安全级别（如全局禁 run_shell，工作流不能开启）
- 工作流可以收紧（全局允许 run_shell，工作流可禁用）

## 5. 配置加载实现

```python
# awe/config/settings.py

@dataclass(frozen=True)
class Config:
    general: GeneralConfig
    security: SecurityConfig
    concurrency: ConcurrencyConfig
    retry: RetryConfig
    approval: ApprovalConfig
    llm_plan: LlmPlanConfig
    loop: LoopConfig
    observability: ObservabilityConfig
    cli_adapter: dict[str, CliAdapterConfig]
    generation: GenerationConfig
    pricing: PricingConfig
    ui: UiConfig

    @classmethod
    def load(cls, paths: list[Path], overrides: dict | None = None) -> "Config":
        ...

    def merge_workflow_config(self, wf_config: dict) -> "Config":
        # 校验 + 合并 workflow.json 内的 config
        ...

    def merge_cli_overrides(self, cli_args: dict) -> "Config":
        ...

    def to_dict(self) -> dict: ...
    def to_redacted_dict(self) -> dict:
        # 用于日志输出，secrets 字段脱敏
        ...
```

## 6. 校验规则

启动时：
1. 解析所有配置文件（TOML）→ 报语法错误
2. 按优先级合并
3. 校验各字段类型与范围
4. 校验冲突（如 ci_mode 下 allow_run_shell=true 矛盾）
5. 检测路径黑名单是否包含 `.workflow` 自身（防误清理）

任何错误 → fail-fast，错误码 `AWE_CFG_xxx`。

## 7. 默认值文件

`awe/config/defaults.toml` 内置默认配置，与上文 §3 完全一致。

测试：每次发布前自动比对 docs/config-reference.md 与 defaults.toml 同步。

## 8. 环境变量覆盖

支持以下环境变量覆盖（用于容器/CI 场景）：

| 变量 | 对应配置项 |
|------|-----------|
| `AWE_DEFAULT_CLI` | general.default_cli |
| `AWE_MAX_PARALLEL` | concurrency.max_parallel |
| `AWE_AUTO_APPROVE` | approval.policy = auto |
| `AWE_LOG_LEVEL` | observability.log_level |
| `AWE_WORKFLOW_DIR` | general.workflow_dir |
| `AWE_CI_MODE` | 启用 ci-mode 全套配置 |

环境变量优先级介于"项目级配置"与"命令行参数"之间。

## 9. 敏感配置处理

- API key 等永远不写入配置文件（除非用户主动）
- 推荐通过环境变量传递
- `awe doctor` 会脱敏显示已配置的密钥（首尾 4 字符 + ***）

## 10. 配置 schema 的发布

`awe/config/schemas/config.schema.json` 提供完整 TOML schema（通过 `taplo` 等工具校验），方便 IDE 提示。

## 11. 配置迁移

发布新版本时，若配置 schema 变更：
- 旧字段标 deprecated（启动时警告）
- 提供 `awe config migrate` 命令自动升级
- 删除字段在下一个 major 版本

## 12. 多人协作

项目级 `.awe/config.toml` 应纳入 git；用户级 `~/.awe/config.toml` 不纳入。

`.awe/config.example.toml` 提供示例，含注释说明哪些应在团队共享。

## 13. CI/CD 推荐配置

```toml
# .awe/config.ci.toml
[general]
default_cli = "claude"

[security]
allow_run_shell = false
allow_http_get = false
mask_secrets_in_logs = true

[approval]
policy = "auto"

[concurrency]
max_parallel = 4              # CI 资源有限
max_run_seconds = 1800

[ui]
color = false
progress_style = "json"
```

CLI 启用：`awe --config .awe/config.ci.toml --ci-mode run "..."`
