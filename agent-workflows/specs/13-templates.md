# Spec 13 — 模板与示例

> 父文档：`../DESIGN.md`

## 1. 内置模板清单

`templates/` 下：

| 模板 | 文件 | 场景 |
|------|------|------|
| 代码审计与修复 | `code-audit.json` | 安全/质量审计 + 选择性修复 |
| 大规模重构 | `refactor.json` | 跨多文件统一改造 |
| 跨语言/框架迁移 | `migration.json` | Vue 2→3、componentWill 替换 |
| 测试覆盖率提升 | `coverage-improvement.json` | 找未覆盖路径 + 补测试 |
| 文档批量生成 | `docs-generation.json` | API/模块自动写文档 |
| 多视角评审 | `multi-perspective-review.json` | 性能/安全/可维护性并行 |
| 数据迁移 | `data-migration.json` | DB schema 演进 + 数据搬运 |
| 依赖升级与适配 | `dependency-upgrade.json` | 主版本升级 |

## 2. 模板规约

每个模板必须含：
- `metadata.title` / `description` / `tags`
- 顶层注释说明使用方式
- 用 `${context.xxx}` 占位需要用户填充的部分
- 必含至少一个 `human_approval` 节点
- 必含至少一个对抗式验证（fan_out + 反驳 fan_out）
- 注释说明每个节点的目的

## 3. 模板调用方式

CLI：
```bash
awe run "审计 src" --template code-audit \
  --context-set scope="src/**/*.ts" \
  --context-set severity_threshold=medium
```

Skill：SKILL.md 提示 LLM 在生成 workflow.json 前先查看 `templates/` 是否有匹配模板，若有则基于模板修改而非从零生成。

## 4. 模板验证

每个模板必经：
- JSON Schema 静态校验
- 引擎动态校验（含 dry-run）
- 端到端测试（CI 用 mock CLI 跑通）

## 5. 示例工作流（examples/）

完整可运行的最小示例：

| 示例 | 演示重点 |
|------|---------|
| 01-hello-fanout | 最简扇出 + 聚合 |
| 02-audit-and-fix | 审计 → triage → approval → fix → verify 完整链 |
| 03-multi-perspective-review | parallel 块 + llm_decision 综合判断 |
| 04-with-llm-plan | llm_plan 动态扩展 |
| 05-cross-cli-mix | 不同节点用不同 CLI |
| 06-resume-demo | 中断 + 恢复 |
| 07-ci-mode | CI/CD 模式（auto-approve、禁交互） |

每个示例含：
- workflow.json
- README.md（说明、运行命令、预期结果）
- expected-final-report.md（mock 模式下的预期产出）

## 6. 用户自定义模板

用户可放置：
- `~/.awe/templates/` — 用户级
- `./.awe/templates/` — 项目级

`awe run --template <name>` 解析顺序：项目 → 用户 → 内置。

## 7. 模板参数化

模板顶层可声明 `parameters`：

```json
{
  "parameters": {
    "scope": { "type": "string", "required": true, "description": "扫描路径" },
    "severity_threshold": { "type": "string", "default": "medium", "enum": ["low","medium","high","critical"] }
  }
}
```

`awe run --template <name>` 时通过 `--param scope=...` 注入，实际写入 `context.*`。

## 8. 模板版本与兼容

- 模板顶层 `template_version: "1.0"`
- 与 awe DSL version 一致
- 升级时旧模板自动迁移或报弃用警告
