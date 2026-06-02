# Spec 04 — 并发与限流子系统

> 父文档：`../DESIGN.md`

## 1. 三层并发控制

实际并发 = `min(全局, 节点, CLI)`

```
全局：config.max_parallel = 8        ← 工作流总并发
节点：fan_out.max_parallel = 16     ← 节点配置覆盖
CLI ：claude RPM=50, TPM=200k       ← 按 CLI 限流
```

## 2. 全局信号量

```python
# awe/schedule/concurrency.py
class GlobalConcurrency:
    def __init__(self, limit: int):
        self.sem = asyncio.Semaphore(limit)

    async def acquire(self): await self.sem.acquire()
    def release(self): self.sem.release()

    async def __aenter__(self): await self.acquire(); return self
    async def __aexit__(self, *a): self.release()
```

## 3. Token Bucket（按 CLI）

```python
# awe/schedule/ratelimit.py
class CliTokenBucket:
    """RPM + TPM 双桶限流"""
    def __init__(self, rpm: int, tpm: int):
        self.rpm_capacity = rpm
        self.tpm_capacity = tpm
        self.rpm_tokens = float(rpm)
        self.tpm_tokens = float(tpm)
        self.last_refill = time.monotonic()
        self.lock = asyncio.Lock()

    async def acquire(self, estimated_tokens: int = 2000):
        while True:
            async with self.lock:
                self._refill()
                if self.rpm_tokens >= 1 and self.tpm_tokens >= estimated_tokens:
                    self.rpm_tokens -= 1
                    self.tpm_tokens -= estimated_tokens
                    return
            await asyncio.sleep(0.1)

    def _refill(self):
        now = time.monotonic()
        elapsed = now - self.last_refill
        self.rpm_tokens = min(self.rpm_capacity,
                              self.rpm_tokens + elapsed * (self.rpm_capacity / 60))
        self.tpm_tokens = min(self.tpm_capacity,
                              self.tpm_tokens + elapsed * (self.tpm_capacity / 60))
        self.last_refill = now

    def report_actual(self, estimated: int, actual: int):
        # 校正预估值
        async def _correct():
            async with self.lock:
                self.tpm_tokens = max(0, self.tpm_tokens - max(0, actual - estimated))
        asyncio.create_task(_correct())
```

默认值（实施时需根据用户 plan 调整）：

| CLI | RPM | TPM |
|-----|-----|-----|
| claude (Pro) | 30 | 80k |
| claude (Max 5x) | 60 | 200k |
| claude (Max 20x) | 120 | 400k |
| codex | 60 | 200k |
| codebuddy | 60 | 200k |

## 4. 429 处理协议

```python
# awe/schedule/ratelimit.py
class RateLimitHandler:
    def __init__(self):
        self.cooldowns: dict[str, datetime] = {}      # CLI -> until
        self.consecutive_429: dict[str, int] = defaultdict(int)
        self.lock = asyncio.Lock()

    async def on_429(self, cli: str, retry_after: int | None):
        async with self.lock:
            self.consecutive_429[cli] += 1
            wait_seconds = retry_after or self._compute_backoff(self.consecutive_429[cli])
            self.cooldowns[cli] = datetime.utcnow() + timedelta(seconds=wait_seconds)
            logger.warning(f"{cli} 429: cooldown {wait_seconds}s (consecutive {self.consecutive_429[cli]})")

    async def wait_if_cooling(self, cli: str):
        until = self.cooldowns.get(cli)
        if until and datetime.utcnow() < until:
            await asyncio.sleep((until - datetime.utcnow()).total_seconds())

    def on_success(self, cli: str):
        self.consecutive_429[cli] = 0

    def _compute_backoff(self, n: int) -> int:
        # 2, 4, 8, 16, 30, 60, 60, 60...
        return min(60, 2 ** n)
```

行为规则：
- 429 → **冻结整个 CLI 池**（不是单个 task）`wait_seconds`
- 优先 `Retry-After` 头
- 连续 429 上升 → 指数退避到 60s 上限
- 任意成功 → 重置计数

## 5. 自适应并发（AIMD）

```python
# awe/schedule/adaptive.py
class AdaptiveConcurrency:
    """Additive Increase Multiplicative Decrease"""
    def __init__(self, initial: int, max_limit: int, min_limit: int = 1):
        self.current = initial
        self.max = max_limit
        self.min = min_limit
        self.success_streak = 0

    def on_success(self):
        self.success_streak += 1
        # 每连续 N 次成功 +1
        if self.success_streak >= 10 and self.current < self.max:
            self.current += 1
            self.success_streak = 0

    def on_rate_limit(self):
        self.current = max(self.min, self.current // 2)
        self.success_streak = 0

    def on_error(self):
        self.success_streak = 0

    @property
    def limit(self) -> int:
        return self.current
```

整合：fan_out 节点开始时取 `min(配置, AdaptiveConcurrency.limit)`。

## 6. Scheduler 总装

```python
# awe/schedule/__init__.py
class Scheduler:
    def __init__(self, config: Config):
        self.global_sem = GlobalConcurrency(config.max_parallel)
        self.cli_buckets: dict[str, CliTokenBucket] = {}
        self.rate_handler = RateLimitHandler()
        self.adaptive: dict[str, AdaptiveConcurrency] = {}

    async def acquire(self, cli: str, estimated_tokens: int = 2000):
        await self.rate_handler.wait_if_cooling(cli)
        bucket = self._bucket(cli)
        await bucket.acquire(estimated_tokens)
        await self.global_sem.acquire()

    def release(self, cli: str):
        self.global_sem.release()

    async def report_result(self, cli: str, result: SubagentResult):
        if result.error_category == "rate_limit":
            await self.rate_handler.on_429(cli, retry_after=result.retry_after)
            self._adaptive(cli).on_rate_limit()
        elif result.success:
            self.rate_handler.on_success(cli)
            self._adaptive(cli).on_success()
            if result.tokens_in is not None:
                self._bucket(cli).report_actual(2000, result.tokens_in + result.tokens_out)
        else:
            self._adaptive(cli).on_error()
```

## 7. fan_out 调度伪代码

```python
async def execute_fan_out(self, node, ctx):
    items = ctx.resolve_ref(node["items_from"])
    cli = node["agent"].get("cli", ctx.config.default_cli)
    max_par = min(node.get("max_parallel", 4),
                  ctx.scheduler._adaptive(cli).limit)
    sem = asyncio.Semaphore(max_par)

    async def run_one(item):
        async with sem:
            await ctx.scheduler.acquire(cli)
            try:
                req = build_request(node["agent"], item, ctx)
                result = await ctx.cli_registry[cli].spawn(req, ...)
                await ctx.scheduler.report_result(cli, result)
                return result
            finally:
                ctx.scheduler.release(cli)

    results = await asyncio.gather(*[run_one(it) for it in items],
                                    return_exceptions=True)
    return aggregate(results, node)
```

## 8. Prompt Cache 利用策略

- 同 fan_out 内 system_prompt 完全相同（保证 cache prefix）
- 启动 fan_out 前打印一条事件 `cache_warmup`，提示用户首条会比较慢
- 5 分钟内连续派 → cache 命中
- metrics 记录 `cache_hit_tokens` 用于事后分析

## 9. 限流配置层级

```
默认 → 全局配置（~/.awe/config.toml）→ 项目配置（.awe/config.toml）
   → workflow.json/config → 节点 → CLI 参数（--max-parallel）
```

## 10. 测试要点

- 模拟 429：Mock CLI 返回 rate_limit，断言全局降速
- AIMD 收敛：连续成功后并发应回升
- TPM 校正：实际 token 大于估计时，下次申请会被限制
- 并发上限：实际同时运行的 task 数 ≤ min(各层限制)
