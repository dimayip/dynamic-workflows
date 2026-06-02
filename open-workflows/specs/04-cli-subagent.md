# Spec 04 — CLI Subagent 适配（与 agent-workflows 共享）

> 父文档：`../DESIGN.md`

## 1. 共享原则

CLI subagent 适配层是 `agent-workflows` 与 `open-workflows` 的**共享组件**。两者依赖同一份 CLI 适配实现：

```
agent-workflows/                open-workflows/
└── (Python: awe/cli_adapter)    └── (Node: ow-cli-adapter)
              │                              │
              └──────── 同一份契约 ──────────┘
```

由于实现语言不同，不能直接共享代码，但**契约层完全一致**：
- 命令行参数构造规则
- 输出解析逻辑
- 错误分类映射
- 工具语义映射

## 2. Node 侧实现

### 2.1 抽象接口

```typescript
// ow/cli-adapter/base.ts
export interface CliAdapter {
  readonly name: string;
  readonly rateLimits: RateLimitsConfig;

  detect(): Promise<DetectResult>;
  spawn(req: SubagentRequest, opts: SpawnRunOptions): Promise<SubagentResult>;
  translateTools(generic: ReadonlyArray<string>): string[];
  parseOutput(stdout: string, stderr: string, exitCode: number): ParsedOutput;
  normalizeError(exitCode: number, stderr: string): ErrorCategory;
}

export interface SubagentRequest {
  taskId: string;
  role: string;
  prompt: string;
  systemPrompt?: string;
  tools: ReadonlyArray<string>;
  editPolicy: "readonly" | "modify_only" | "full";
  allowedPaths: ReadonlyArray<string>;
  outputSchema?: object;
  maxTokens?: number;
  timeoutMs: number;
}

export interface SubagentResult {
  taskId: string;
  cli: string;
  success: boolean;
  output: unknown;
  rawStdout: string;
  rawStderr: string;
  exitCode: number;
  durationMs: number;
  tokensIn: number | null;
  tokensOut: number | null;
  cacheHitTokens: number | null;
  error: string | null;
  errorCategory: ErrorCategory | null;
}

export type ErrorCategory =
  | "rate_limit" | "auth" | "network" | "invalid_input"
  | "model_error" | "timeout" | "subprocess_crash" | "unknown";
```

### 2.2 Claude 适配器

```typescript
// ow/cli-adapter/claude.ts
export class ClaudeAdapter implements CliAdapter {
  name = "claude" as const;

  async detect() {
    const { exitCode, stdout } = await execa("claude", ["--version"], { reject: false });
    return {
      installed: exitCode === 0,
      version: parseVersion(stdout),
      authenticated: await this.checkAuth(),
    };
  }

  async spawn(req, opts) {
    const args = [
      "-p", req.prompt,
      "--output-format", "json",
      "--permission-mode", "bypassPermissions",
      "--allowed-tools", this.translateTools(req.tools).join(","),
      "--cwd", opts.cwd,
    ];
    if (req.systemPrompt) args.push("--append-system-prompt", req.systemPrompt);
    if (req.maxTokens) args.push("--max-tokens", String(req.maxTokens));

    const proc = execa("claude", args, {
      timeout: req.timeoutMs,
      reject: false,
      env: this.sanitizeEnv(opts.env),
    });

    const result = await proc;
    return this.buildResult(req, result);
  }

  translateTools(generic) {
    return generic.map(t => TOOL_MAP[t] ?? throwUnsupported(t));
  }

  parseOutput(stdout) {
    // claude --output-format json 返回单 JSON 对象
    const obj = JSON.parse(stdout);
    return { result: obj.result, usage: obj.usage };
  }

  normalizeError(exitCode, stderr) {
    if (/rate.?limit/i.test(stderr)) return "rate_limit";
    if (/unauthor|not logged/i.test(stderr)) return "auth";
    // ...
  }
}
```

### 2.3 Codex / CodeBuddy 适配器

按相同接口实现。Codex 的输出是 JSONL，需逐行解析取最后一条 `type: "response"`。CodeBuddy 待官方文档确认。

### 2.4 工具映射表

```typescript
const TOOL_MAP: Record<string, Record<string, string>> = {
  claude: {
    read_file: "Read",
    write_file: "Write",
    edit_file: "Edit",
    grep: "Grep",
    glob: "Glob",
    git_diff: "Bash",   // 通过 Bash 限定的 git 调用
    // ...
  },
  codex: { /* ... */ },
  codebuddy: { /* 待补 */ },
};
```

## 3. 进程管理

```typescript
async function runCli(cmd, args, opts) {
  const proc = execa(cmd, args, {
    stdin: "ignore",
    stdout: "pipe",
    stderr: "pipe",
    cwd: opts.cwd,
    env: opts.env,
    timeout: opts.timeoutMs,
    killSignal: "SIGTERM",
  });

  // 5s 后强制 kill
  proc.catch(() => {});
  setTimeout(() => {
    if (!proc.killed) proc.kill("SIGKILL");
  }, opts.timeoutMs + 5000);

  return await proc;
}
```

## 4. 输出协议

与 agent-workflows specs/03-cli-adapter.md §5 完全一致：
1. 优先 CLI 原生 JSON 模式
2. 失败 → 提取 ` ```json...``` ` 代码块
3. 仍失败 → 平衡括号匹配
4. 仍失败 → 节点失败 + retry 一次（带"格式错误"提示）

## 5. 环境变量隔离

```typescript
const SECRET_KEYS_BLACKLIST = [
  /token/i, /key/i, /secret/i, /password/i, /credential/i,
  /^AWS_/, /^GOOGLE_/, /^AZURE_/,
];

function sanitizeEnv(passthroughList: string[]): Record<string, string> {
  const result: Record<string, string> = {};
  for (const [k, v] of Object.entries(process.env)) {
    if (SAFE_BUILTINS.includes(k)) result[k] = v ?? "";
    else if (passthroughList.includes(k)) result[k] = v ?? "";
    // 黑名单的 secrets 必须 explicit passthrough
  }
  return result;
}
```

## 6. CLI 检测与降级

`ow doctor` 执行 detect 流程；启动时如果脚本指定的 cli 不可用 → fail-fast。

## 7. Prompt Cache 策略

同 agent-workflows specs/04-concurrency.md §8。重点：
- 同一 fan-out 内 systemPrompt 完全相同（cache prefix 相同）
- 5 分钟窗口内连续派 → 命中
- metrics 记录 cache_hit_tokens

## 8. 与 agent-workflows 的实现共享建议

未来可考虑：
- 抽取 CLI 适配为独立 npm 包 + pip 包，跨语言契约
- 用 protobuf / JSON schema 定义 SubagentRequest / SubagentResult，两边自动生成
- 共享测试 fixture（mock CLI 输出样本）

v1.0 不强求；先各自实现，保持契约一致即可。
