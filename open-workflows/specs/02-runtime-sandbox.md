# Spec 02 — Runtime 与 Curated SDK

> 父文档：`../DESIGN.md`

## 1. 运行时

| 项 | 选择 |
|----|------|
| Node | 22+ |
| TypeScript | 5+ |
| 沙箱库 | `isolated-vm`（Anthropic、Replit 等同款选型） |
| Bundler | `esbuild`（脚本即时编译） |

## 2. isolated-vm 配置

### 2.1 创建 isolate

```typescript
import ivm from "isolated-vm";

const isolate = new ivm.Isolate({
  memoryLimit: config.sandbox.memoryLimitMb,    // 默 256
  inspector: false,                              // 生产关
  onCatastrophicError: (err) => {
    log.error("isolate 灾难性错误", err);
    process.exit(2);
  },
});

const context = await isolate.createContext();
```

### 2.2 注入全局

仅注入 Curated SDK 的 Reference（禁止 copy 真实对象到沙箱）：

```typescript
const jail = context.global;
await jail.set("agent",    new ivm.Reference(agentApi),    { copy: false });
await jail.set("tools",    new ivm.Reference(toolsApi),    { copy: false });
await jail.set("log",      new ivm.Reference(logApi),      { copy: false });
await jail.set("kv",       new ivm.Reference(kvApi),       { copy: false });
await jail.set("workflow", new ivm.Reference(workflowApi), { copy: false });
await jail.set("human",    new ivm.Reference(humanApi),    { copy: false });
```

### 2.3 沙箱内的 SDK 桥

由于 isolated-vm Reference 的限制，需要在沙箱内构造方便调用的代理。引擎在脚本前缀注入：

```javascript
// __setup__.js（透明前置代码，用户脚本看不见）
const _agent = agent;        // ivm.Reference
const _tools = tools;
const _log = log;
const _kv = kv;
const _workflow = workflow;
const _human = human;

globalThis.agent = {
  spawn(role, opts) { return _agent.applyIgnored("spawn", [role, opts], { result: { promise: true, copy: true }, arguments: { copy: true } }); }
};
globalThis.tools = { /* 同样模式 */ };
// ...
```

### 2.4 脚本执行

```typescript
const compiledJs = await esbuild.transform(orchestratorTsSource, {
  loader: "ts",
  target: "es2022",
  format: "esm",
  // 禁用 sourcemap 生成时引用文件
});

const script = await isolate.compileScript(SETUP_PREFIX + compiledJs.code);
await script.run(context, {
  timeout: config.maxRunMs,
  promise: true,
});
```

### 2.5 资源限制

| 资源 | 配置项 | 默认 | 实施 |
|------|-------|------|------|
| 内存 | `sandbox.memoryLimitMb` | 256 | isolate.memoryLimit |
| 墙钟 | `sandbox.maxRunMs` | 3600000 | script.run timeout |
| CPU | `sandbox.maxCpuMs` | 1800000 | isolate.cpuTime 监控 |

CPU 监控：用 `setInterval` 每秒检查 `isolate.cpuTime`，超限调用 `isolate.dispose()`。

## 3. Curated SDK 完整定义

### 3.1 `agent`

```typescript
declare const agent: {
  spawn(role: AgentRole, options: SpawnOptions): Promise<AgentResult>;
};

type AgentRole =
  | "auditor" | "fixer" | "reviewer" | "summarizer"
  | "planner" | "challenger" | "explorer" | "tester"
  | "documenter" | "translator" | "custom";   // 角色封闭集

interface SpawnOptions {
  prompt: string;
  cli?: CliName;                       // claude / codex / codebuddy
  systemPrompt?: string;
  tools?: ReadonlyArray<ToolName>;
  editPolicy?: "readonly" | "modify_only" | "full";
  allowedPaths?: ReadonlyArray<string>;
  outputSchema?: JsonSchema;
  maxTokens?: number;
  timeoutMs?: number;
}

interface AgentResult {
  taskId: string;
  cli: CliName;
  success: boolean;
  output: unknown;                     // 由 outputSchema 决定结构
  durationMs: number;
  tokensIn: number;
  tokensOut: number;
  cacheHitTokens: number;
  error: string | null;
  // 附 raw* 字段需通过 await result.raw() 异步获取，避免大对象传输
}
```

### 3.2 `tools`

```typescript
declare const tools: {
  // 只读
  findFiles(pattern: string, opts?: FindOptions): Promise<string[]>;
  readFile(path: string, opts?: ReadOptions): Promise<string>;
  grep(pattern: string, opts: GrepOptions): Promise<GrepMatch[]>;
  glob(pattern: string, opts?: GlobOptions): Promise<string[]>;
  gitDiff(opts?: GitOpts): Promise<string>;
  gitLog(opts?: GitOpts): Promise<GitLogEntry[]>;
  gitStatus(): Promise<GitStatus>;

  // 写（受 allowedWritePaths 严格限制）
  writeFile(path: string, content: string, opts?: WriteOptions): Promise<void>;
  editFile(path: string, edits: EditOp[]): Promise<void>;

  // 受限网络（默认禁用）
  httpGet?(url: string, opts?: HttpOptions): Promise<HttpResponse>;
};
```

每个工具实现层做二次校验（详 specs/03 §9）。

### 3.3 `log`

```typescript
declare const log: {
  info(message: string, context?: Record<string, unknown>): void;
  warn(message: string, context?: Record<string, unknown>): void;
  error(message: string, context?: Record<string, unknown>): void;
  debug(message: string, context?: Record<string, unknown>): void;
};
```

写入 `.ow/<run_id>/logs/orchestrator.jsonl`，同时 forwardto stdout（受 verbosity 控制）。

### 3.4 `kv`

```typescript
declare const kv: {
  get<T = unknown>(key: string): T | undefined;
  set<T = unknown>(key: string, value: T): void;
  delete(key: string): void;
  has(key: string): boolean;
  keys(): string[];
};
```

存储到 `.ow/<run_id>/kv.json`，每次 set 立即 persist（原子写）。

约束：
- key 长度 ≤ 256 字符
- value 必须 JSON 可序列化
- 单 value ≤ 1 MB
- 总大小 ≤ 100 MB

### 3.5 `workflow`

```typescript
declare const workflow: {
  // 检查点
  checkpoint(name: string, data: Record<string, unknown>): Promise<void>;

  // 错误终止（让脚本主动中止）
  fail(error: unknown): never;

  // 配置
  readonly config: {
    runId: string;
    maxParallel: number;
    maxSubagentsTotal: number;
    defaultCli: CliName;
    workdir: string;
    params: Record<string, unknown>;   // 用户启动时传入的参数
  };

  // 并发控制
  semaphore(limit: number): {
    run<T>(fn: () => Promise<T>): Promise<T>;
  };

  // 延迟
  sleep(ms: number): Promise<void>;
};
```

### 3.6 `human`

```typescript
declare const human: {
  approve(req: ApprovalRequest): Promise<ApprovalAnswer>;
  input(req: InputRequest): Promise<string>;
  confirm(message: string): Promise<boolean>;
};
```

通过 InteractionAdapter（stdio 或 skill 协议）转交。

## 4. 类型导出

引擎打包一份 `.d.ts` 文件作为生成 prompt 的一部分给 LLM：

```typescript
// ow-sdk.d.ts （注入到 LLM 生成 prompt 中）
declare global {
  const agent: AgentApi;
  const tools: ToolsApi;
  const log: LogApi;
  const kv: KvApi;
  const workflow: WorkflowApi;
  const human: HumanApi;
}

// 类型定义...
```

LLM 生成时 prompt 中明确："你必须只使用上述类型，不能 import 任何东西"。

## 5. SDK 实现层（host 侧）

```typescript
// awe/sdk/agent.ts
export const agentApi = {
  async spawn(role: string, options: unknown): Promise<unknown> {
    // 1. 参数校验（零信任：脚本传来的可能是任意值）
    validateRole(role);
    const opts = validateSpawnOptions(options);

    // 2. 安全二次校验（详 specs/03 §9）
    enforcePathPolicy(opts);
    enforceConcurrencyLimit();

    // 3. 调度器申请额度
    await scheduler.acquire(opts.cli ?? config.defaultCli);

    // 4. 调 CLI 适配层（与 agent-workflows 共享）
    const result = await cliAdapter.spawn({
      taskId: generateTaskId(role),
      role,
      prompt: opts.prompt,
      ...opts,
    });

    // 5. 持久化到 .ow/<run_id>/subagents/<task_id>.json
    await persistSubagentResult(result);

    // 6. 返回精简结果到沙箱
    return summarizeResult(result);
  },
};
```

### 5.1 零信任原则

- 沙箱传入的所有参数视为不可信
- 校验类型 / 范围 / 白名单
- 不直接转发到底层（防注入）
- 返回值 deep clone 后跨 isolate 边界传递

### 5.2 异步桥

isolated-vm 的 Reference 调用是同步阻塞的；要让 SDK 调用变 async，host 侧返回 Promise，沙箱内通过 `await` 等待。注意：

- `applyIgnored` + `{ result: { promise: true } }` 配置
- 错误必须包装成 SDK 标准错误类型
- 大对象（> 100KB）警告并建议改用 kv 或 checkpoint

## 6. 工具语义映射

继承 agent-workflows 的工具映射表（详 `../agent-workflows/specs/03-cli-adapter.md` §4）：

```
通用名         claude    codex     codebuddy
read_file      Read      read      read_file
write_file     Write     write     write_file
...
```

实现层共享。

## 7. 跨 isolate 数据传输

| 类型 | 传输方式 |
|------|---------|
| primitive (string/number/bool/null) | 直接 copy |
| 简单对象 | copy（deep clone） |
| 大对象 (> 100KB) | 用 kv 存，传 key |
| Promise | promise: true |
| 函数 | 不允许（Reference 才能） |
| symbol/BigInt | 不允许 |

## 8. 启动开销

实测：
- isolate 创建：~10ms
- context 创建：~5ms
- SDK 注入：~20ms
- 脚本编译（含 esbuild）：~30ms
- 总冷启动：~80ms

后续可优化：worker 池预热（v1.1+）。

## 9. 故障恢复

- isolate 内异常 → 捕获到 host → 写状态 → 报告错误码
- isolate 越界（内存/CPU 超限）→ host 监控发现 → dispose isolate → 写状态
- host 崩溃 → 下次 `ow resume` 加载 checkpoint
