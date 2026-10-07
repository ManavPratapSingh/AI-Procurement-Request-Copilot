# Architecture Decision Memo

## Decision
We recommend shipping **Architecture A (Single-Agent Baseline with Deterministic Policy Floor Guardrails)** as the Production MVP for the AI Procurement Request Copilot.

## Evidence
Both architectures were evaluated against the 21 ground-truth test cases (`BASE-01` to `BASE-10` and `SYN-01` to `SYN-11`) using the evaluation harness (`evals/run_evals.py`).

| Metric | Single agent (Architecture A) | Staged / 2-agent (Architecture B) |
|---|---:|---:|
| Cases passing strict quality criteria | 21 / 21 (100%) | 21 / 21 (100%) |
| Avg latency (mean / P95) | 3.55 ms / 4.93 ms | 4.12 ms / 6.77 ms |
| Avg LLM calls per request | 1.0 | 2.0 |
| Avg tool calls per request | 1.0 | 1.0 |
| Notable policy/grounding failures | 0 | 0 |

Both architectures achieved 100% strict policy accuracy, approval routing precision, and grounded evidence verification due to the deterministic floor engine (`src/rules.py` and `src/guardrails.py`).

## Trade-offs
- **Latency & Throughput**: Architecture A is ~16% faster on average (3.55ms vs 4.12ms) and avoids the second LLM turn overhead of Architecture B.
- **Cost & API Quota Efficiency**: Architecture B doubles LLM API invocations (2 calls per request: Stage 1 Evidence Collector + Stage 2 Decision Evaluator), making it twice as expensive per request and significantly more susceptible to rate limiting under provider free tiers.
- **Maintainability & Complexity**: Architecture A relies on a single prompt loop and schema validator. Architecture B requires inter-stage state serialization, evidence pack sanitization, and complex multi-prompt error handling.

## Risks / Limitations
1. **Prompt Injection in Business Data**: Untrusted business justifications could attempt to override system behavior. This is mitigated by our regex heuristic scanner (`detect_injection`) and hard policy floor guardrails.
2. **External API Failures**: The vendor risk HTTP service may experience downtime or timeouts. The system handles this gracefully by flagging `vendor_risk_unavailable` and routing to human review.
3. **Data Freshness**: Vendor security reviews are evaluated against a fixed policy reference date (`2026-09-30`) to prevent date-drift degradation.

## Why This is the Right MVP
Architecture A delivers 100% policy compliance, zero grounding failures, and lower operational latency without unnecessary multi-agent orchestration. Staging into two separate agents adds latency and cost without improving decision quality. A single-agent system bounded by deterministic policy guardrails provides the optimal balance of intelligence, reliability, speed, and cost-efficiency.
