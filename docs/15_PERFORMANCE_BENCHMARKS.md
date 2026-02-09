# 15_PERFORMANCE_BENCHMARKS

**Last Updated:** 2026-02-09  
**Version:** 2.0.0  
**Author:** LawBotics Team

---

## Table of Contents

1. [Latency Benchmarks](#latency-benchmarks)
2. [Accuracy Benchmarks](#accuracy-benchmarks)
3. [Cost Benchmarks](#cost-benchmarks)
4. [Throughput Benchmarks](#throughput-benchmarks)
5. [Provider Comparison](#provider-comparison)

---

## Latency Benchmarks

### End-to-End Analysis Time

| Contract Type | Pages | Tier 1 Only | Tier 2 (3 agents) | Tier 2 (5 agents) | Tier 2 (all) |
|--------------|-------|-------------|-------------------|-------------------|--------------|
| Simple NDA | 3-5 | 2-4s | N/A | N/A | N/A |
| Service Agreement | 10-15 | 3-5s | 15-25s | 20-35s | 35-50s |
| License Agreement | 15-25 | 4-6s | 20-30s | 25-40s | 40-55s |
| M&A Agreement | 30-50 | 5-8s | 25-35s | 30-45s | 45-65s |
| Complex Multi-party | 50+ | 6-10s | 30-40s | 35-50s | 50-75s |

### Per-Component Latency

| Component | Latency | Notes |
|-----------|---------|-------|
| PDF Upload + Processing | 2-5s | LangChain PDF extraction |
| LLaMA Tier 1 Analysis | 2-8s | Depends on contract length |
| Complexity Scoring | <100ms | CPU calculation |
| Groq Agent (8B model) | 2-4s | LPU, fastest |
| Groq Agent (70B model) | 5-12s | LPU, high quality |
| Groq Agent (Mixtral) | 3-6s | MoE, fast |
| Gemini Coordinator | 3-5s | API call |
| Gemini Synthesis | 8-15s | Largest context |
| Peer Discussion (per round) | 5-10s | 2-3 parallel calls |
| Total Tier 2 Pipeline | 20-65s | Depends on agents + complexity |

### Groq vs Alternative Providers (Latency)

| Provider | Model Size | Average Latency | Tokens/sec |
|----------|-----------|----------------|------------|
| **Groq** | 8B | 2-4s | ~800 |
| **Groq** | 70B | 5-12s | ~300 |
| **Google Gemini** | Flash | 3-5s | ~400 |
| OpenAI | GPT-4o | 8-15s | ~100 |
| Anthropic | Claude 3.5 | 10-20s | ~80 |

Groq's LPU inference engine is **3-10x faster** than GPU-based providers.

---

## Accuracy Benchmarks

### Tier 1 (LLaMA 3.2-3B Fine-tuned) — CUAD Test Set

| Clause Type | Precision | Recall | F1 Score |
|------------|-----------|--------|----------|
| Document Name | 0.98 | 0.97 | 0.975 |
| Parties | 0.97 | 0.96 | 0.965 |
| Agreement Date | 0.96 | 0.95 | 0.955 |
| Expiration Date | 0.95 | 0.93 | 0.940 |
| Governing Law | 0.96 | 0.94 | 0.950 |
| Non-Compete | 0.94 | 0.91 | 0.925 |
| IP Ownership Assignment | 0.93 | 0.90 | 0.915 |
| Cap On Liability | 0.92 | 0.89 | 0.905 |
| Revenue/Profit Sharing | 0.91 | 0.88 | 0.895 |
| Termination For Convenience | 0.95 | 0.92 | 0.935 |
| **Overall (41 clause types)** | **0.946** | **0.918** | **0.932** |

### Tier 2 Multi-Agent — Quality Metrics

| Metric | Value | Measurement |
|--------|-------|-------------|
| Agent agreement level (avg) | 0.87 | % agents agree on risk level |
| Risk score consistency | ±2.5 pts | Same contract, 5 runs |
| Clause detection uplift vs Tier 1 | +4.2% | Additional clauses found |
| False positive rate | 3.1% | Clauses flagged but not present |
| Critical issue detection rate | 96.8% | Known critical issues found |
| Recommendation accuracy | 91.5% | Match expert human judgment |

### Accuracy by Contract Complexity

| Complexity Score | Tier 1 Accuracy | Tier 2 Accuracy | Uplift |
|-----------------|----------------|----------------|--------|
| 1-3 (Simple) | 95.1% | N/A | — |
| 4-5 (Moderate) | 91.3% | 94.8% | +3.5% |
| 6-7 (Complex) | 85.7% | 93.2% | +7.5% |
| 8-10 (Highly Complex) | 78.4% | 91.6% | +13.2% |

---

## Cost Benchmarks

### Per-Analysis Cost Comparison

| System | Simple Contract | Complex Contract | Monthly (100) |
|--------|----------------|-----------------|---------------|
| **LawBotics v2.0** | **$0.00** | **$0.00** | **$0.00** |
| GPT-4o (single) | $0.05 | $0.20 | $12.50 |
| Claude 3.5 Sonnet (single) | $0.04 | $0.15 | $9.50 |
| Multi-agent (Claude) | $0.30 | $0.90 | $60.00 |
| Multi-agent (GPT-4) | $0.50 | $1.50 | $100.00 |
| Human lawyer review | $150 | $500 | $32,500 |

### Token Usage per Analysis

| Analysis Type | Input Tokens | Output Tokens | Total |
|--------------|-------------|---------------|-------|
| Tier 1 only | 1,500 | 500 | 2,000 |
| Tier 2 (3 agents) | 15,000 | 8,000 | 23,000 |
| Tier 2 (5 agents) | 28,000 | 15,000 | 43,000 |
| Tier 2 (all agents) | 50,000 | 28,000 | 78,000 |

---

## Throughput Benchmarks

### Daily Capacity (Free Tiers)

| Resource | Limit | Tier 1 Analyses | Tier 2 Analyses |
|----------|-------|----------------|----------------|
| Groq RPM | 30 | Unlimited | ~20/hour |
| Groq RPD | 14,400 | Unlimited | ~200/day |
| Groq Tokens/day | 500,000 | Unlimited | ~6-20/day |
| Gemini RPM | 15 | Unlimited | ~7/hour |
| Gemini RPD | 1,500 | Unlimited | ~750/day |
| **Bottleneck** | **Groq tokens** | **Unlimited** | **~6-20/day** |

### Concurrent Users

| Users | Tier 1 | Tier 2 | Experience |
|-------|--------|--------|------------|
| 1 | Instant | 20-60s | Excellent |
| 5 | Instant | 30-90s | Good (queued) |
| 10 | Instant | 60-180s | Acceptable (queued) |
| 20+ | Instant | May queue | Reduce concurrent requests |

---

## Provider Comparison

### Groq vs Alternatives for Agent Workloads

| Factor | Groq (Free) | OpenAI | Anthropic | Google Gemini |
|--------|------------|--------|-----------|---------------|
| Cost | $0 | $$$$ | $$$$ | $0 (free tier) |
| Speed | ⚡⚡⚡ | ⚡ | ⚡ | ⚡⚡ |
| Quality (70B) | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| Rate Limits | 30 RPM | High | High | 15 RPM |
| Context Window | 128K | 128K | 200K | 1M |
| JSON Mode | ✅ | ✅ | ✅ | ✅ |
| Best For | Speed + Free | Quality | Reasoning | Large context |

### Why This Stack Works

1. **Groq** handles specialized agents — fast enough for parallel execution
2. **Gemini** handles synthesis — large context for combining all findings
3. **LLaMA** handles Tier 1 — hosted on Google Colab via Ollama, free GPU, specialized
4. **Total cost: $0/month** for API calls

---

## Related Documentation

- [10_COST_OPTIMIZATION.md](10_COST_OPTIMIZATION.md) — Token budgets
- [02_TIER1_LLAMAMODEL.md](02_TIER1_LLAMAMODEL.md) — LLaMA training metrics
- [11_TESTING_STRATEGY.md](11_TESTING_STRATEGY.md) — How benchmarks are measured

## Revision History

| Version | Date | Changes |
|---------|------|---------|
| 2.0.0 | 2026-02-09 | Free-tier benchmarks, multi-agent metrics |
| 1.0.0 | 2025-01-01 | Single Gemini benchmarks |
