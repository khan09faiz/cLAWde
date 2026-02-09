# 10_COST_OPTIMIZATION

**Last Updated:** 2026-02-09  
**Version:** 2.0.0  
**Author:** LawBotics Team

---

## Table of Contents

1. [Cost Model](#cost-model)
2. [Free Tier Limits](#free-tier-limits)
3. [Token Budget Strategy](#token-budget-strategy)
4. [Caching Strategy](#caching-strategy)
5. [Agent Selection Optimization](#agent-selection-optimization)
6. [Rate Limit Management](#rate-limit-management)
7. [Throughput Estimates](#throughput-estimates)

---

## Cost Model

LawBotics v2.0 operates at **$0 API cost** by using free tiers exclusively.

| Provider | Model | Cost per 1M Input Tokens | Cost per 1M Output Tokens |
|----------|-------|--------------------------|---------------------------|
| Groq | llama-3.3-70b-versatile | $0 (free tier) | $0 (free tier) |
| Groq | deepseek-r1-distill-llama-70b | $0 (free tier) | $0 (free tier) |
| Groq | llama-3.1-8b-instant | $0 (free tier) | $0 (free tier) |
| Groq | mixtral-8x7b-32768 | $0 (free tier) | $0 (free tier) |
| Google | gemini-2.0-flash | $0 (free tier) | $0 (free tier) |
| Google Colab | LLaMA 3.2-3B (fine-tuned) | $0 (free Colab GPU) | $0 (free Colab GPU) |

### Cost Comparison vs Paid Alternatives

| Configuration | Cost per Contract | Monthly (100 contracts) |
|--------------|-------------------|------------------------|
| **LawBotics v2.0 (Free tiers)** | **$0.00** | **$0.00** |
| GPT-4o | ~$0.15 | ~$15.00 |
| Claude 3.5 Sonnet | ~$0.12 | ~$12.00 |
| Claude 3 Opus (multi-agent) | ~$0.80 | ~$80.00 |

---

## Free Tier Limits

### Groq Free Tier

| Limit | Value | Impact |
|-------|-------|--------|
| Requests per minute | 30 RPM | 3 agents per batch |
| Requests per day | 14,400 RPD | ~1,440 full analyses/day |
| Tokens per minute | 15,000 TPM | ~3 concurrent analyses |
| Tokens per day | 500,000 TPD | ~50 full Tier 2 analyses/day |

### Google Gemini Free Tier

| Limit | Value | Impact |
|-------|-------|--------|
| Requests per minute | 15 RPM | Coordinator + Synthesis only |
| Requests per day | 1,500 RPD | ~750 analyses/day |
| Tokens per minute | 1,000,000 TPM | Effectively unlimited |
| Context window | 1,048,576 tokens | Full contracts fit easily |

### Tier 1 (Google Colab + Ollama)

| Limit | Value | Notes |
|-------|-------|-------|
| Throughput | ~10 contracts/min | On Colab T4 GPU |
| Cost | $0 | Free Google Colab GPU |
| Context | 4,096 tokens | Chunking required |

---

## Token Budget Strategy

### Per-Agent Token Budgets

| Agent | Input Budget | Output Budget | Peer Budget | Total/Analysis |
|-------|-------------|---------------|-------------|----------------|
| Clause Extractor | 4,000 | 2,000 | 500 | 6,500 |
| Risk Assessor | 6,000 | 3,000 | 1,000 | 10,000 |
| Financial Analyst | 5,000 | 2,500 | 500 | 8,000 |
| IP/Licensing | 5,000 | 2,500 | 500 | 8,000 |
| Domain Specialist | 5,000 | 2,000 | 500 | 7,500 |
| Compliance | 5,000 | 2,000 | 500 | 7,500 |
| Coordinator (Gemini) | 2,000 | 1,000 | — | 3,000 |
| Synthesis (Gemini) | 10,000 | 4,000 | — | 14,000 |

### Typical Tier 2 Analysis Budget

| Scenario | Agents Active | Total Tokens | Within Free Tier? |
|----------|--------------|-------------|-------------------|
| Simple (3 agents) | 3 | ~25,000 | Yes |
| Standard (5 agents) | 5 | ~45,000 | Yes |
| Complex (7 agents) | 7 | ~65,000 | Yes |
| Maximum (all agents) | 9 | ~80,000 | Yes |

Daily capacity at 500K Groq tokens/day: **~6-20 Tier 2 analyses/day** depending on complexity.

---

## Caching Strategy

### Response Caching

```typescript
// Cache identical contract analyses to avoid re-processing
interface AnalysisCache {
  contentHash: string;           // SHA-256 of document content
  tier1Results: any;
  tier2Results?: any;
  createdAt: number;
  expiresAt: number;             // 7-day TTL
}

async function getOrAnalyze(
  documentContent: string,
  options: AnalysisOptions
): Promise<AnalysisResult> {
  const hash = await sha256(documentContent);

  // Check cache
  const cached = await ctx.db
    .query("analysis_cache")
    .withIndex("by_contentHash", (q) => q.eq("contentHash", hash))
    .first();

  if (cached && cached.expiresAt > Date.now()) {
    return cached.results;
  }

  // Run fresh analysis
  const results = await runAnalysis(documentContent, options);

  // Cache results
  await ctx.db.insert("analysis_cache", {
    contentHash: hash,
    results,
    createdAt: Date.now(),
    expiresAt: Date.now() + 7 * 24 * 60 * 60 * 1000,  // 7 days
  });

  return results;
}
```

### Prompt Template Caching

Agent system prompts are static — build them once and reuse:

```typescript
const PROMPT_CACHE = new Map<string, string>();

function getAgentPrompt(agentId: string, context: ContractContext): string {
  const cacheKey = `${agentId}:${context.contractType}`;
  if (PROMPT_CACHE.has(cacheKey)) return PROMPT_CACHE.get(cacheKey)!;

  const prompt = buildAgentPrompt(agentId, context);
  PROMPT_CACHE.set(cacheKey, prompt);
  return prompt;
}
```

---

## Agent Selection Optimization

Not every contract needs every agent. Smart activation saves tokens:

```typescript
// Average agents activated per contract type
const ACTIVATION_STATS: Record<string, number> = {
  nda: 3,                // Clause + Risk + IP
  employment: 4,         // Clause + Risk + Employment + Compliance
  service: 4,            // Clause + Risk + Financial + Compliance
  license: 5,            // Clause + Risk + Financial + IP + Compliance
  "joint_venture": 6,    // Clause + Risk + Financial + IP + M&A + Compliance
  other: 5,              // Clause + Risk + Financial + IP + Compliance
};
```

### Token Savings from Smart Activation

| Strategy | Tokens Saved | Impact |
|----------|-------------|--------|
| Skip financial agent for NDAs | ~8,000/analysis | 10-15% savings |
| Skip M&A for service contracts | ~7,500/analysis | 10% savings |
| Skip employment for IP licenses | ~7,500/analysis | 10% savings |
| Reduce peer rounds when no conflicts | ~3,000/analysis | 5% savings |

---

## Rate Limit Management

### Staggered Execution

```typescript
class RateLimitManager {
  private queues: Map<string, Array<() => Promise<any>>> = new Map();
  private windows: Map<string, { count: number; resetAt: number }> = new Map();

  async execute<T>(provider: string, fn: () => Promise<T>): Promise<T> {
    const limits: Record<string, number> = {
      groq: 28,    // Stay under 30 RPM
      gemini: 13,  // Stay under 15 RPM
    };

    const maxRPM = limits[provider] ?? 30;
    const window = this.windows.get(provider);
    const now = Date.now();

    if (window && now < window.resetAt && window.count >= maxRPM) {
      const waitMs = window.resetAt - now;
      await new Promise((r) => setTimeout(r, waitMs));
      this.windows.set(provider, { count: 0, resetAt: now + 60000 });
    }

    if (!window || now >= window.resetAt) {
      this.windows.set(provider, { count: 1, resetAt: now + 60000 });
    } else {
      window.count++;
    }

    return fn();
  }
}
```

### Multi-User Fairness

```typescript
// Per-user daily quota to prevent single user exhausting free tier
const USER_DAILY_LIMITS = {
  tier2_analyses: 10,    // Max 10 deep analyses per user per day
  total_tokens: 100000,  // Max 100K tokens per user per day
};
```

---

## Throughput Estimates

| Metric | Tier 1 Only | Tier 2 (5 agents) | Tier 2 (all agents) |
|--------|-------------|-------------------|---------------------|
| Latency | 2-5s | 20-40s | 40-60s |
| Groq calls | 0 | 5-8 | 10-14 |
| Gemini calls | 0 | 1-2 | 2 |
| Tokens used | ~2K | ~45K | ~80K |
| Daily capacity | ~unlimited | ~10-20/day | ~6/day |
| Cost | $0 | $0 | $0 |

---

## Related Documentation

- [03_TIER2_MULTIAGENT_CORE.md](03_TIER2_MULTIAGENT_CORE.md) — Agent execution
- [05_COMPLEXITY_DETECTION.md](05_COMPLEXITY_DETECTION.md) — Tier routing
- [15_PERFORMANCE_BENCHMARKS.md](15_PERFORMANCE_BENCHMARKS.md) — Benchmark data

## Revision History

| Version | Date | Changes |
|---------|------|---------|
| 2.0.0 | 2026-02-09 | Free-tier cost model, Groq/Gemini budgets |
| 1.0.0 | 2025-01-01 | Single Gemini API cost tracking |
