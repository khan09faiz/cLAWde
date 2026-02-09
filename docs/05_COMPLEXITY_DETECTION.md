# 05_COMPLEXITY_DETECTION

**Last Updated:** 2026-02-09  
**Version:** 2.0.0  
**Author:** LawBotics Team

---

## Table of Contents

1. [Complexity Scoring Algorithm](#complexity-scoring-algorithm)
2. [Scoring Factors](#scoring-factors)
3. [Tier Routing Logic](#tier-routing-logic)
4. [User Override](#user-override)
5. [Calibration & Tuning](#calibration--tuning)
6. [Edge Cases](#edge-cases)

---

## Complexity Scoring Algorithm

The complexity score (0–10) determines whether a contract requires Tier 2 multi-agent analysis. By default, Tier 1 (LLaMA) handles all contracts. Tier 2 activates only when complexity ≥ 6.0.

```typescript
interface ComplexityFactors {
  clauseCount: number;
  ambiguousClauseRatio: number;     // Clauses with confidence < 0.7
  uniqueClauseTypes: number;
  contractLength: number;            // Word count
  financialComplexity: number;       // 0-10
  crossJurisdiction: boolean;
  multiParty: boolean;               // > 2 parties
  hasUnusualProvisions: boolean;
  tier1Confidence: number;           // LLaMA's overall confidence
  contractValue: number | null;
  regulatoryDomains: number;         // Number of applicable regulations
}

function calculateComplexityScore(factors: ComplexityFactors): number {
  let score = 0;

  // 1. Clause density (max 2.0 points)
  const clauseDensity = factors.clauseCount / Math.max(factors.contractLength / 1000, 1);
  score += Math.min(clauseDensity * 0.5, 2.0);

  // 2. Ambiguity ratio (max 2.0 points)
  score += factors.ambiguousClauseRatio * 2.0;

  // 3. Clause diversity (max 1.5 points)
  if (factors.uniqueClauseTypes > 15) score += 1.5;
  else if (factors.uniqueClauseTypes > 10) score += 1.0;
  else if (factors.uniqueClauseTypes > 5) score += 0.5;

  // 4. Financial complexity (max 1.5 points)
  score += (factors.financialComplexity / 10) * 1.5;

  // 5. Structural complexity (max 1.5 points)
  if (factors.crossJurisdiction) score += 0.5;
  if (factors.multiParty) score += 0.5;
  if (factors.hasUnusualProvisions) score += 0.5;

  // 6. Tier 1 uncertainty (max 1.0 point)
  score += (1 - factors.tier1Confidence) * 1.0;

  // 7. Regulatory complexity (max 0.5 points)
  score += Math.min(factors.regulatoryDomains * 0.25, 0.5);

  return Math.min(Math.round(score * 10) / 10, 10);
}
```

---

## Scoring Factors

| Factor | Weight | Range | Measurement |
|--------|--------|-------|-------------|
| Clause density | 2.0 | 0–2.0 | Clauses per 1K words |
| Ambiguity ratio | 2.0 | 0–2.0 | % clauses with LLaMA conf < 0.7 |
| Clause diversity | 1.5 | 0–1.5 | Count of unique CUAD clause types |
| Financial complexity | 1.5 | 0–1.5 | Multi-currency, variable pricing, penalties |
| Structural complexity | 1.5 | 0–1.5 | Cross-jurisdiction + multi-party + unusual provisions |
| Tier 1 uncertainty | 1.0 | 0–1.0 | 1 − LLaMA overall confidence |
| Regulatory complexity | 0.5 | 0–0.5 | Number of applicable regulations |

**Total maximum: 10.0 points**

### Financial Complexity Sub-Score

```typescript
function calculateFinancialComplexity(clauses: any[]): number {
  let score = 0;
  
  const financialClauses = [
    "Revenue/Profit Sharing",
    "Price Restrictions",
    "Minimum Commitment",
    "Liquidated Damages",
    "Cap On Liability",
  ];

  const found = clauses.filter((c) =>
    financialClauses.includes(c.type) && c.present
  );

  score += found.length * 1.5;                    // Each financial clause adds 1.5
  if (hasVariablePricing(clauses)) score += 2;     // Variable pricing adds complexity
  if (hasMultipleCurrencies(clauses)) score += 1;  // Multi-currency adds complexity
  if (hasEscalationClauses(clauses)) score += 1;   // Price escalation

  return Math.min(score, 10);
}
```

---

## Tier Routing Logic

```
┌──────────────────────────────────────────────────────────────┐
│                    Document Uploaded                         │
└────────────────────────┬─────────────────────────────────────┘
                         │
                         ▼
              ┌──────────────────┐
              │  Tier 1 (LLaMA)  │
              │  Quick Analysis  │
              └────────┬─────────┘
                       │
                       ▼
              ┌──────────────────┐
              │   Complexity     │
              │   Score Calc     │
              └────────┬─────────┘
                       │
              ┌────────┴────────┐
              │                 │
         Score < 6         Score ≥ 6
              │                 │
              ▼                 ▼
     ┌──────────────┐  ┌──────────────────┐
     │ Return Tier 1 │  │ Tier 2 Multi-    │
     │ Results       │  │ Agent Analysis   │
     └──────────────┘  └──────────────────┘
```

### Routing Implementation

```typescript
// convex/actions/routeAnalysis.ts

export const routeAnalysis = action({
  args: {
    documentId: v.id("documents"),
    content: v.string(),
    userId: v.string(),
    forceDeep: v.optional(v.boolean()),
  },
  handler: async (ctx, args) => {
    // Step 1: Always run Tier 1
    const tier1Results = await runTier1Analysis(args.content);

    // Step 2: Calculate complexity
    const complexityScore = calculateComplexityScore(
      extractComplexityFactors(tier1Results, args.content)
    );

    // Step 3: Route decision
    const shouldRunTier2 = args.forceDeep || complexityScore >= 6.0;

    if (shouldRunTier2) {
      // Store Tier 1 results and trigger Tier 2
      await ctx.runMutation(internal.analyses.createTier1Analysis, {
        documentId: args.documentId,
        results: tier1Results,
        complexityScore,
        routedToTier2: true,
      });

      // Trigger async Tier 2 analysis
      await ctx.scheduler.runAfter(0, internal.agents.orchestrator.runMultiAgentAnalysis, {
        documentId: args.documentId,
        tier1Results,
      });

      return {
        tier: 2,
        tier1Results,
        complexityScore,
        message: "Complex contract detected. Deep multi-agent analysis started.",
      };
    }

    // Tier 1 only
    await ctx.runMutation(internal.analyses.createTier1Analysis, {
      documentId: args.documentId,
      results: tier1Results,
      complexityScore,
      routedToTier2: false,
    });

    return {
      tier: 1,
      tier1Results,
      complexityScore,
      message: "Standard contract. Tier 1 analysis complete.",
    };
  },
});
```

---

## User Override

Users can force Tier 2 regardless of complexity score:

```typescript
// Frontend - analysis request
const requestAnalysis = async (documentId: string, forceDeep: boolean) => {
  return await convex.action(api.actions.routeAnalysis.routeAnalysis, {
    documentId,
    content: documentContent,
    userId: user.id,
    forceDeep,  // User can toggle "Deep Analysis" in UI
  });
};
```

| Override | Behavior |
|----------|----------|
| Auto (default) | Score ≥ 6 → Tier 2, Score < 6 → Tier 1 |
| Force Deep | Always runs Tier 2 regardless of score |
| Force Quick | Always uses only Tier 1 |

---

## Calibration & Tuning

### Threshold Validation

Track routing decisions to tune the threshold:

```typescript
interface RoutingMetric {
  documentId: string;
  complexityScore: number;
  routedToTier2: boolean;
  userOverride: boolean;
  tier1Accuracy: number;        // Post-hoc evaluation
  tier2AddedValue: number;      // How much Tier 2 improved results
  userSatisfaction: number;     // 1-5 rating
}
```

### Adjustment Rules

| Signal | Action |
|--------|--------|
| Tier 2 adds < 5% value for scores 6-7 | Raise threshold to 7 |
| Tier 1 misses critical issues for scores 4-5 | Lower threshold to 5 |
| Users frequently override to force deep | Lower threshold |
| Groq rate limits frequently hit | Raise threshold |

---

## Edge Cases

| Scenario | Handling |
|----------|----------|
| Score exactly 6.0 | Routes to Tier 2 (≥ threshold) |
| LLaMA returns empty results | Score defaults to 8.0 (force Tier 2) |
| Very short document (< 500 words) | Score capped at 4.0 (unlikely needs Tier 2) |
| Non-English contract | Score += 2.0 (higher uncertainty) |
| Encrypted/corrupted PDF | Error → manual review prompt |
| Tier 1 times out | Score defaults to 7.0, Tier 2 starts with raw text |

---

## Related Documentation

- [02_TIER1_LLAMAMODEL.md](02_TIER1_LLAMAMODEL.md) — Tier 1 analysis details
- [03_TIER2_MULTIAGENT_CORE.md](03_TIER2_MULTIAGENT_CORE.md) — Tier 2 orchestration
- [10_COST_OPTIMIZATION.md](10_COST_OPTIMIZATION.md) — Routing cost implications

## Revision History

| Version | Date | Changes |
|---------|------|---------|
| 2.0.0 | 2026-02-09 | Hybrid tier routing system |
| 1.0.0 | 2025-01-01 | Single-tier Gemini analysis |
