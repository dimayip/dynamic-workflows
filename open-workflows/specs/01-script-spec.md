# Spec 01 — orchestrator.ts 脚本规范

> 父文档：`../DESIGN.md`

## 1. 脚本结构

LLM 生成的 `orchestrator.ts` 必须遵守以下结构：

```typescript
/**
 * @workflow security-audit
 * @description 审计 src 目录认证缺陷
 * @generated_by claude-3.5-sonnet
 * @generated_at 2026-06-02T19:45:00Z
 */

// 顶层只允许：
// 1. 注释 + JSDoc 元信息
// 2. 顶层 const/let（基础类型/字面量）
// 3. type / interface 定义（v1.0 限简单结构）
// 4. async function main() 主函数
// 5. main().catch(workflow.fail)

const SCOPE = "src/**/*.ts";
const MAX_BATCH = 16;

interface Finding {
  file: string;
  line: number;
  severity: "critical" | "medium" | "low";
  desc: string;
}

async function main(): Promise<void> {
  // 主流程
  const files = await tools.findFiles(SCOPE, { maxResults: 1000 });
  log.info(`发现 ${files.length} 个候选文件`);

  // fan-out
  const sem = workflow.semaphore(MAX_BATCH);
  const findings = (await Promise.all(
    files.map(f => sem.run(() =>
      agent.spawn("auditor", {
        prompt: `审计 ${f} 中的认证缺陷`,
        tools: ["read_file", "grep"],
        editPolicy: "readonly",
        outputSchema: { /* ... */ },
      })
    ))
  )).flatMap(r => r.findings as Finding[]);

  // checkpoint
  await workflow.checkpoint("after-audit", { findings });

  // ... 后续阶段
}

main().catch((err) => workflow.fail(err));
```

## 2. 强制约束（每条违反 → ESLint/AST 拒绝）

| # | 约束 |
|---|------|
| 1 | 顶层不允许执行任何 SDK 调用（除 main 内） |
| 2 | 必须有且仅有一个 `main` 函数作为入口 |
| 3 | 必须以 `main().catch(workflow.fail)` 结尾 |
| 4 | 不允许 `import` / `require` |
| 5 | 不允许 `class`（v1.0 简化） |
| 6 | 不允许定义生成器、async generator |
| 7 | 不允许 `eval` / `new Function` / `setTimeout(string)` / `setInterval(string)` |
| 8 | 不允许直接访问 `globalThis` / `global` / `window` |
| 9 | 不允许 `as any` / `@ts-ignore` |
| 10 | 不允许嵌套深度 > 5 的 block |
| 11 | 不允许单函数超过 80 行 |
| 12 | 总行数 ≤ 800 |
| 13 | 所有 SDK 调用必须 `await` |
| 14 | 所有 Promise 必须 await 或 catch |
| 15 | 所有循环必须有可推断的终止条件 |
| 16 | 所有递归必须有 base case |
| 17 | spawn 嵌套深度 ≤ 3（脚本内显式调用层级） |

## 3. SDK 命名空间

唯一暴露给脚本的全局变量：

| 全局 | 类型 | 用途 |
|------|------|------|
| `agent` | AgentApi | 派生 subagent |
| `tools` | ToolsApi | 文件、搜索、git 等只读/受限工具 |
| `log` | LogApi | 日志（info/warn/error） |
| `kv` | KvApi | 跨 checkpoint 的状态存储 |
| `workflow` | WorkflowApi | checkpoint / fail / config / semaphore / sleep |
| `human` | HumanApi | 审批请求 |

详细类型定义见 `02-runtime-sandbox.md` §3。

## 4. 风格规范

参考 [Google TypeScript Style Guide](https://google.github.io/styleguide/tsguide.html)，子集化执行：

| 项 | 要求 |
|---|------|
| 命名 | camelCase 变量；PascalCase 类型；UPPER_SNAKE 常量 |
| 格式 | 2 空格缩进；100 列；单引号字符串（与 Prettier 默认）|
| 类型 | 强类型，禁 `any` 与 `as any` |
| 异步 | 一律 `async/await`，禁 `then`/`catch` 链 |
| 错误 | try/catch 必须 catch 具体类型或 unknown 处理 |
| 注释 | 关键逻辑写 JSDoc 说明意图 |

由 `prettier --check` + `eslint-config-google-typescript` + 自定义补丁规则强制。

## 5. 元信息（必填 JSDoc）

文件首行必须有：

```typescript
/**
 * @workflow <slug>                  // run_id 前缀
 * @description <一行描述>
 * @generated_by <模型名>             // 生成 LLM
 * @generated_at <ISO8601>
 * @sdk_version 1.0
 */
```

引擎启动时校验：
- `@sdk_version` 与当前引擎兼容
- `@workflow` 满足正则 `^[a-z][a-z0-9-]{0,63}$`

## 6. 推荐模式（few-shot）

LLM 生成 prompt 中提供以下模式作为参考：

### 6.1 简单 fan-out

```typescript
const items = await tools.findFiles(pattern);
const results = await Promise.all(
  items.map(item => agent.spawn("worker", { prompt: `处理 ${item}` }))
);
```

### 6.2 带并发限制的 fan-out

```typescript
const sem = workflow.semaphore(8);
const results = await Promise.all(
  items.map(item => sem.run(() => agent.spawn("worker", { ... })))
);
```

### 6.3 对抗式验证

```typescript
const claims = await Promise.all(items.map(i => agent.spawn("auditor", { ... })));
const verified = await Promise.all(
  claims.map(c => agent.spawn("challenger", {
    prompt: `反驳或确认：${JSON.stringify(c)}`,
  }))
);
const accepted = verified.filter(v => v.confirmed);
```

### 6.4 收敛迭代

```typescript
let pending = initial;
let iteration = 0;
while (pending.length > 0 && iteration < 5) {
  const results = await Promise.all(
    pending.map(p => agent.spawn("fixer", { ... }))
  );
  pending = results.filter(r => !r.success);
  await workflow.checkpoint(`iter-${iteration}`, { pending });
  iteration++;
}
```

### 6.5 LLM 决策驱动分支

```typescript
const decision = await agent.spawn("triager", {
  prompt: `决策...`,
  outputSchema: { strategy: { enum: ["a", "b", "c"] } },
});
switch (decision.strategy) {
  case "a": await ...; break;
  case "b": await ...; break;
  case "c": break;
}
```

### 6.6 动态扩展（运行时让 LLM 生成新计划）

```typescript
const plan = await agent.spawn("planner", {
  prompt: "根据 X 设计下一阶段任务清单",
  outputSchema: { tasks: { type: "array", items: { ... } } },
});
for (const task of plan.tasks.slice(0, 20)) {  // 静态上限
  await agent.spawn(task.role, task.options);
}
```

## 7. 反模式（拒绝生成）

```typescript
// ❌ 顶层 SDK 调用
const x = await agent.spawn(...);

// ❌ 无界循环
while (true) { await agent.spawn(...); }

// ❌ 动态 spawn 数量无界
const n = userInput.count;
for (let i = 0; i < n; i++) { await agent.spawn(...); }

// ❌ 字符串拼 shell
await tools.runShell(`rm ${userInput}`);   // 即使 SDK 不暴露 runShell 也禁

// ❌ 嵌套 spawn 链过深
agent.spawn("a", { ... }) → 内部又生成 spawn → 5 层以上

// ❌ 修改注入对象
agent.spawn = ...;     // 沙箱阻止
tools.read_file = ...;
```

## 8. 与 Claude Code DW 脚本的差异

| | Claude Code DW | open-workflows |
|---|---|---|
| 语言 | JS | TypeScript |
| spawn API | `agent.spawn` | `agent.spawn`（语义同） |
| 工具 | 内置 Read/Edit/... | `tools.readFile/...`（封装） |
| 沙箱 | Anthropic V8 | isolated-vm |
| 脚本可保存为 slash command | `.claude/workflows/` | `.ow/workflows/`（详 06-protocol.md） |
| 风格规范 | 无强制 | TypeScript strict + ESLint + 自定义规则 |
| 行数上限 | 无 | 800 行 |
| spawn 嵌套 | 无显式上限 | 3 层 |

## 9. 脚本编译

LLM 生成的是 TypeScript 源码。引擎流程：

```
1. 收 .ts 源码
2. tsc 编译（仅 type-check，不真生成 .js）
3. esbuild 转成 ES2022 JavaScript（target=es2022, format=esm）
4. 把 .js 喂给 isolated-vm
```

不打包外部依赖（脚本不允许 import）。

## 10. 复用脚本

通过 `ow.workflows/` 目录保存可复用脚本：

```
.ow/workflows/
├── security-audit.ts     # 项目级
└── ...

~/.ow/workflows/
└── ...                   # 用户级
```

调用：`ow run --script security-audit src/`，参数通过 `workflow.config.params` 暴露给脚本。
