# AGENTS.md
每次回复时都叫我【dima，你好：】

PURPOSE: Compliance manual for any general-purpose AI Agent (any domain, any task: chat, research, customer support, data analysis, content generation, automation, coding, tool use) at every LLM call site.

SCOPE: Project- and domain-agnostic. Loaded automatically per agents.md protocol. Overrides local style when in conflict.

KEYWORDS: MUST / SHOULD / MAY per RFC 2119.

FORMAT: Each rule = `IF <trigger> THEN <action>`. Cite rule IDs (A1..A9, B1..B12, C1..C25, U1..U7) in implementation notes and change-promotion records.

CLASSES:
- [A] engineering-curable; apply mandatory fix; target error rate <1%
- [B] statistically mitigable; apply mitigation + monitor; residual 1–10%
- [C] currently unsolvable at inference; redesign required

---

## §0 Protocol (run BEFORE every LLM call site)

```
1. RESTATE task: "agent does X for user, in domain Z, with output type Y"
2. MATCH rules from §A/§B/§C (multi-hit allowed)
3. PER hit:
   A → apply engineering fix; add regression test
   B → apply mitigation; add monitor + alert; update eval
   C → redesign with tool / KG / DB / rules / human-in-loop;
       if redesign infeasible → STOP and report rule ID
4. APPLY §U universal hardening
5. VERIFY against §R checklist before shipping
```

---

## §A Engineering-curable (apply fix)

| ID | Trigger | Action |
|----|---------|--------|
| A1 | Task includes arithmetic ≥4-digit, dates / duration, percentages, statistics, currency, or unit conversion | Route to deterministic tool (calculator / code interpreter / SQL); re-execute and verify before returning |
| A2 | Query references "latest / today / current / since {cutoff} / news / price / rate / weather / status of {entity}" | Attach search or RAG result with `retrieved_at` timestamp; refuse if no fresh source |
| A3 | Query targets internal documents, private records, or non-public knowledge | RAG over private corpus; require chunk_id citation; refuse on no-cite |
| A4 | Output is structured (JSON / XML / SQL / function-call args / tool input) | Use native structured-output mode (OpenAI `response_format=json_schema`, Anthropic `tools`, Outlines / XGrammar / GBNF); validate parse on receive; on parse fail → 1 retry with explicit error feedback then escalate |
| A5 | Generation length >500 tokens OR open-ended writing | Set `top_p≤0.95` AND (`repetition_penalty≥1.05` OR `no_repeat_ngram_size≥3`); set `max_tokens` cap |
| A6 | Output is an executable artifact (code / shell command / SQL / automation script / API call payload) | Execute in sandbox with feedback loop (generate → execute → repair); cap 1–3 cycles; escalate on cap-hit |
| A7 | Latency-sensitive OR cost-sensitive call site | Set `max_tokens`; set per-field `maxLength` in JSON schema; monitor p99 length |
| A8 | Task has >3 sequential dependent steps | Decompose into explicit nodes; orchestrate via state machine (LangGraph / explicit graph); define termination condition |
| A9 | Generating a query for a structured store (SQL / Cypher / GraphQL / aggregation pipeline / DSL filter) | Inject schema in prompt; dry-run / EXPLAIN; retry-on-error 1x; sandbox with read-only role for read paths |

---

## §B Statistically mitigable (apply mitigation + monitor)

| ID | Trigger | Action | Monitor |
|----|---------|--------|---------|
| B1 | Context >8k tokens AND critical info present | Place critical info at start OR end of context; rerank-then-truncate | Position-of-truth recall |
| B2 | User message contains stance / leading phrasing ("isn't this wrong?", "我觉得 X 对吗") | Strip user stance from LLM-facing prompt; add explicit `IGNORE_USER_OPINION`; require new evidence before any "rethink" prompt | Sycophancy delta on agree/disagree pairs |
| B3 | Answer is a factual claim outside a guaranteed RAG hit | Require citation OR Chain-of-Verification OR Self-Consistency n≥5; block critical-domain answers lacking citation | Citation-coverage rate; SelfCheck score |
| B4 | Using LLM-as-judge | Use a judge model from a different family than the candidate; use length-controlled metric | Inter-judge agreement; family-bias delta |
| B5 | RAG context contradicts model parametric belief | Use Context-Aware Decoding; add explicit `TRUST_CONTEXT_OVER_MEMORY`; require model to enumerate which context lines were used | Context-attribution rate |
| B6 | LLM ranks or selects from N candidates | Shuffle position across calls; permutation-average (PriDe pattern) | First-position selection bias |
| B7 | Prompt includes user-supplied OR web-retrieved text | Spotlight (delimit + tag `<untrusted>...</untrusted>` + base64 or marker variant); restrict tool whitelist to read-only on untrusted scope; add output-side jailbreak classifier | Injection-success rate; tool-use-on-untrusted alarms |
| B8 | Conversation >8 turns OR session crosses context limit | Re-inject critical constraints every N turns; summary-then-continue beyond limit; plant canary constraint at turn 1 and check at turn N | Constraint-retention rate |
| B9 | Reasoning chain is required | Convert to PoT (executable code) for verifiable steps; treat textual chain-of-thought as draft only | PoT-vs-text answer divergence |
| B10 | Eval uses preference or win-rate | Use length-controlled metric (LC-AlpacaEval style) | Length–win correlation |
| B11 | Safety / refusal-sensitive endpoint | Measure under-refusal (jailbreak success) AND over-refusal (StrongREJECT-style); both gate release | Dual refusal-calibration metrics |
| B12 | Stateful agent with growing memory | Page memory (MemGPT pattern); separate working / episodic / long-term tiers; cap working-set tokens | Working-set hit-rate; eviction churn |

---

## §C Currently unsolvable (redesign required)

This section lists §C failure modes that have a concrete agent-runtime trigger and a concrete redesign action. Other §C modes that exist only as awareness items (training-side / evaluation-side / product-design concerns without a runtime trigger) are out of scope here. For ALL listed §C: depend on the redesign, not on the LLM solving it end-to-end. When the user requests an end-to-end LLM solution for a §C trigger, surface the rule ID and propose the redesign; if redesign is infeasible, STOP and report.

ID gaps (C5, C11, C13, C15, C16, C18, C19, C24, C25) are intentional: those modes are documented for humans in the companion README, with no Agent trigger.

| ID | Trigger | Required redesign |
|----|---------|-------------------|
| C1 | Task asks LLM to recall a fact in the reverse direction it was trained ("A's father is B" → "B's son is ?") | KG / structured DB lookup |
| C2 | Task requires composing N independent skills end-to-end out-of-distribution | External decomposition; one skill per call |
| C3 | Logic depends on negation ("not X", "without Y", "exclude Z") | Structured filter on attributes; also test the affirmative form |
| C4 | Query targets a long-tail entity (rare person / niche term / private record) | RAG mandatory; treat parametric memory as unreliable for rare entities |
| C6 | Task is tokenization-sensitive (count letters, char-edit, reverse string, base-N conversion) | Deterministic tool |
| C7 | A reasoning trace will be used as audit log or proof | PoT or external verifier produces the audit artifact; chain-of-thought text is a draft only |
| C8 | Task is multi-step symbolic theorem proving | Proof assistant (Lean / Coq) is the prover; LLM acts as tactic suggester |
| C9 | Endpoint may receive unanswerable questions | Upstream refusal classifier classifies before LLM generation |
| C10 | Pipeline uses model-emitted confidence to gate a downstream action | Replace with ensemble disagreement signal |
| C12 | Agent loop with tool use and goals | Set hard tool / step / cost budgets and a kill-switch on every loop |
| C14 | Pipeline asks the model to reconsider an answer it already produced | Force a second pass with explicit "produce a different answer than the first" instruction; compare both |
| C17 | Query contains relative time expressions ("3 weeks before X", "next quarter") | Pre-resolve all dates upstream; pass absolute ISO-8601 timestamps to LLM |
| C20 | Task involves numerical reasoning over large arrays / matrices / tabular data | Tool (NumPy / SQL / pandas / spreadsheet engine); pass results back to LLM only for narration |
| C21 | Task reasoning spans multiple loosely-coupled artifacts (files / documents / records / sub-systems) | Scope LLM to one artifact per call; external orchestrator (index / planner / static analyzer) handles cross-artifact integration |
| C22 | Producing artifacts in a low-resource modality (rare programming language, low-resource natural language, niche domain DSL) | Add reference example + verification harness; lower model autonomy |
| C23 | Single context exceeds ~100k tokens AND target info is at the tail | Map-Reduce: chunk → per-chunk extract → reduce; one LLM call per chunk |

---

## §U Universal hardening (apply unconditionally)

| ID | Rule |
|----|------|
| U1 | Set temperature explicitly per call: factual=0; generative ≤0.7 |
| U2 | Set `seed` when supported; record it in logs |
| U3 | Log per call: `prompt_template_id`, `model_id`, `model_version`, `temperature`, `top_p`, `seed`, `input_tokens`, `output_tokens`, `latency_ms`, `structured_output_valid`, `retry_count`, `tool_calls` |
| U4 | Every LLM call site has ≥1 golden-output regression test; CI runs them on every change |
| U5 | LLM call sits behind circuit breaker; non-LLM fallback path exists (cached / rule-based / human queue) |
| U6 | Log token cost per call; alert on cost-per-task drift >20% |
| U7 | Redact PII before send; rotate prompt logs ≤30 days |

---

## §R Release checklist (gate every change to a production LLM call site)

```
[ ] Release notes list hit rule IDs (A?/B?/C?) for each LLM call site
[ ] §A hits: engineering fix applied; regression test added; CI green
[ ] §B hits: mitigation applied; monitor + alert threshold defined; eval golden updated
[ ] §C hits: redesign applied; user informed; no end-to-end LLM dependency
[ ] §U1–U7 applied
[ ] Cost & latency budgets defined per call site
[ ] Non-LLM fallback exists and is tested
[ ] Logs scrubbed of PII; prompt template versioned
```

Release is BLOCKED if any unchecked.

---

## §D Default branch (task matches no rule above)

IF no rule matches AND no deterministic alternative exists:
- State the uncertainty in implementation notes and the typed return contract
- Set conservative defaults: `temperature=0`, `max_tokens=<small>`, validation=strict, fallback=human-review
- Add a TODO with the unresolved failure-mode hypothesis
- Ship telemetry from the first call

END
