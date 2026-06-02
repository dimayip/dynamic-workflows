# AI Agent 动态工作流编排

> 把"用 LLM 一句话调度数十甚至上百个子代理协作完成大任务"做成跨平台、可审计、可复用、可生产的工程能力。

---

## 四个技术方案

| 方案 | 定位 | 约束强度 | 状态 |
|------|------|---------|------|
| [方案一：弱约束 SOP](codex-dynamic-workflows/SKILL.md) | 纯文档约束，依赖 LLM 自觉 | 弱 | ✅ 已可用 |
| [方案二：JSON DSL + Python 引擎](agent-workflows/README.md) | JSON DSL + 确定性引擎 + CLI subagent | 强 | 📐 设计冻结 |
| [方案三：TypeScript + isolated-vm 沙箱](open-workflows/README.md) | TS 脚本 + V8 沙箱，跨平台开源版 | 强 | 📐 设计冻结 |
| Claude Code DW（参考基线） | Anthropic 闭源原版 | 强 | 🔒 闭源 |

## 背景

2026 年 5 月，Anthropic 在 Claude Code 中发布 **Dynamic Workflows** 功能——LLM 现场写一段 JS 编排脚本，在 V8 沙箱内调度数百子代理。本质是：**把工作流的"编排"从 LLM 的概率推理域，迁移到代码的确定性执行域。**

本仓库探索**跨平台、开源**地实现等价能力的不同技术路线。

---

## 方案对比速览

| 维度 | 方案一 SOP | 方案二 DSL+Python | 方案三 TS 沙箱 | Claude Code DW |
|------|-----------|-------------------|---------------|----------------|
| 约束强度 | 弱 | 强 | 强 | 强 |
| 实现语言 | 纯文档 | Python 3.11+ | TypeScript / Node 22+ | JS (V8) |
| 可审计性 | 低 | 高 | 高 | 黑盒 |
| 跨平台 | 是 | 是 | 是 | 仅 Claude Code |
| 开源 | 是 | 是 | 是 | 否 |
