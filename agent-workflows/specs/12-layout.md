# Spec 12 — 项目目录与运行时布局

> 父文档：`../DESIGN.md`

## 1. 仓库目录结构

```
agent-workflows/
├── pyproject.toml                 # 项目元信息 + 依赖（PEP 621）
├── README.md                      # 用户文档
├── LICENSE                        # MIT
├── CHANGELOG.md
├── DESIGN.md                      # 架构总览（本系列文档）
├── ADR.md                         # 决策记录
├── IMPL-CHECKLIST.md              # 实施清单
├── SKILL.md                       # Skill 入口（包含完整生成约束）
├── specs/
│   ├── 01-dsl.md
│   ├── 02-engine.md
│   ├── 03-cli-adapter.md
│   ├── 04-concurrency.md
│   ├── 05-state.md
│   ├── 06-error-approval.md
│   ├── 07-security.md
│   ├── 08-observability.md
│   ├── 09-skill-protocol.md
│   ├── 10-cli-entry.md
│   ├── 11-config.md
│   ├── 12-layout.md
│   ├── 13-templates.md
│   └── _error_codes.md
├── awe/                           # Python 包
│   ├── __init__.py
│   ├── __main__.py
│   ├── version.py
│   ├── cli/
│   ├── core/
│   ├── nodes/
│   ├── schedule/
│   ├── cli_adapter/
│   ├── tools/
│   ├── render/
│   ├── sandbox/                   # code 节点的受限沙箱
│   ├── observability/
│   └── config/
│       ├── settings.py
│       ├── defaults.toml
│       ├── pricing.json
│       └── schemas/
│           ├── workflow.schema.json
│           └── config.schema.json
├── templates/                     # 内置模板（DSL 文件）
│   ├── code-audit.json
│   ├── refactor.json
│   ├── migration.json
│   ├── coverage-improvement.json
│   ├── docs-generation.json
│   └── README.md
├── examples/                      # 用户参考示例
│   ├── 01-hello-fanout/
│   ├── 02-audit-and-fix/
│   ├── 03-multi-perspective-review/
│   ├── 04-with-llm-plan/
│   ├── 05-cross-cli-mix/
│   └── README.md
├── tests/
│   ├── unit/
│   │   ├── test_dsl_schema.py
│   │   ├── test_engine.py
│   │   ├── test_state.py
│   │   ├── test_concurrency.py
│   │   ├── test_render.py
│   │   ├── test_expression.py
│   │   ├── test_cli_adapter_*.py
│   │   ├── test_tools_*.py
│   │   └── ...
│   ├── integration/
│   │   ├── test_engine_with_mock_cli.py
│   │   ├── test_resume.py
│   │   ├── test_approval.py
│   │   └── ...
│   ├── e2e/
│   │   ├── test_real_claude.py    # 需 CLI + API key
│   │   └── ...
│   ├── fixtures/
│   │   ├── workflows/
│   │   └── states/
│   └── conftest.py
├── docs/                          # 用户文档站点（mkdocs）
│   ├── index.md
│   ├── getting-started.md
│   ├── concepts/
│   ├── reference/
│   ├── recipes/
│   └── mkdocs.yml
├── scripts/                       # 开发脚本
│   ├── lint.sh
│   ├── test.sh
│   ├── release.sh
│   └── update-error-codes.py
├── .github/
│   └── workflows/
│       ├── ci.yml                 # 测试 + 类型 + lint
│       ├── e2e.yml                # 真实 CLI 集成（手动触发）
│       └── release.yml            # PyPI 发布
└── .awe/                          # 本仓库自身的 awe 配置（用于 dogfooding）
    └── config.toml
```

## 2. pyproject.toml 关键内容

```toml
[project]
name = "agent-workflows"
version = "1.0.0"
requires-python = ">=3.11"
dependencies = [
    "typer>=0.12",
    "rich>=13",
    "jsonschema>=4",
    "jinja2>=3.1",
    "asteval>=1",                  # 或自实现表达式
    "tomli>=2; python_version<'3.11'",
    "pydantic>=2",                 # 配置数据类
    "aiofiles>=23",
]

[project.optional-dependencies]
trace = ["opentelemetry-api", "opentelemetry-sdk"]
dev = ["pytest", "pytest-asyncio", "mypy", "ruff", "coverage"]

[project.scripts]
awe = "awe.cli.main:app"
agent-workflows = "awe.cli.main:app"

[tool.ruff]
line-length = 100
target-version = "py311"

[tool.mypy]
strict = true
python_version = "3.11"
```

## 3. 运行时目录（每次 run）

```
.workflow/run-<timestamp>-<rand>/
├── workflow.json              # 不可变
├── workflow.expanded.json     # llm_plan 展开后的全量 steps（如有）
├── state.json                 # 主状态
├── state.json.lock            # 进程锁
├── progress.json              # 实时进度
├── usage.json                 # token 与成本
├── meta.json                  # 元信息（awe 版本、CLI 版本等）
├── final-report.md            # 最终报告
├── results/
│   └── <task_id>.json         # 每 subagent 完整输出
└── logs/
    ├── orchestrator.jsonl     # 引擎结构化日志
    └── subagents/
        └── <task_id>.log      # 每 subagent 原始 stdout+stderr
```

## 4. 用户级目录

```
~/.awe/
├── config.toml                # 用户配置
├── runs/                      # 全局 run 索引（v1.1+，可选）
└── cache/
    └── pricing.json           # 缓存的费率表
```

## 5. 系统级目录（可选）

```
/etc/awe/
└── config.toml                # 系统级默认（Linux/macOS）
```

## 6. 命名约定

- run_id：`run-YYYYMMDD-HHMMSS-<6 位 hex>`
- task_id：`<step_id>::<index>`（fan_out）或 `<step_id>::call`（单次）
- workflow.json hash：SHA-256 前 16 位

## 7. .gitignore 推荐

仓库使用方应在自己的项目添加：

```
# AWE
.workflow/
.awe/runs/
*.tmp
```

仓库自身：

```
.workflow/
.awe/runs/
__pycache__/
*.pyc
.pytest_cache/
.mypy_cache/
.coverage
htmlcov/
dist/
build/
*.egg-info/
.tox/
.venv/
```

## 8. 跨平台路径

- 所有路径处理使用 `pathlib.Path`
- Windows 路径分隔符自动处理
- 配置文件路径使用 `~`（自动展开 HOME）
- 锁文件 Windows 用 `msvcrt.locking`，POSIX 用 `fcntl.flock`

## 9. 文件大小预算

| 文件 | 典型大小 | 上限 |
|------|---------|------|
| workflow.json | 5-50 KB | 1 MB |
| state.json | 10-200 KB | 10 MB |
| progress.json | < 5 KB | 50 KB |
| logs/orchestrator.jsonl | 100 KB - 5 MB | 100 MB |
| results/<task_id>.json | 1-100 KB | 10 MB |
| logs/subagents/<task_id>.log | 1-500 KB | 50 MB |

超限处理：截断 + 标记。

## 10. 国际化

错误信息与报告：
- 内置中文 / 英文
- 由 `general.locale` 控制
- 资源放在 `awe/i18n/<locale>.json`
- 实施推荐：`gettext` 或自实现简单查表
