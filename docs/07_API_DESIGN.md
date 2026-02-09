# 07_API_DESIGN

**Last Updated:** 2026-02-09  
**Version:** 2.0.0  
**Author:** LawBotics Team

---

## Table of Contents

1. [API Architecture](#api-architecture)
2. [Document Endpoints](#document-endpoints)
3. [Analysis Endpoints](#analysis-endpoints)
4. [Agent Endpoints](#agent-endpoints)
5. [Usage Tracking](#usage-tracking)
6. [Error Handling](#error-handling)

---

## API Architecture

LawBotics uses Convex for all backend operations. All endpoints are type-safe with automatic real-time subscriptions.

| Type | Description | Use Case |
|------|-------------|----------|
| **Query** | Read-only, real-time subscriptions | Fetching documents, analyses, status |
| **Mutation** | Transactional writes | Creating/updating records |
| **Action** | External API calls (Groq, Gemini, LLaMA) | AI analysis, document processing |

---

## Document Endpoints

### Upload Document
```typescript
// convex/documents.ts
export const generateUploadUrl = mutation({
  handler: async (ctx) => {
    return await ctx.storage.generateUploadUrl();
  },
});

export const createDocument = mutation({
  args: {
    title: v.string(),
    storageId: v.id("_storage"),
    userId: v.string(),
    mimeType: v.optional(v.string()),
    fileSize: v.optional(v.number()),
  },
  handler: async (ctx, args) => {
    const fileUrl = await ctx.storage.getUrl(args.storageId);
    return await ctx.db.insert("documents", {
      ...args,
      fileUrl: fileUrl ?? undefined,
      content: "",
      status: "uploaded",
      isLegalDocument: undefined,
      createdAt: Date.now(),
    });
  },
});
```

### Fetch Documents
```typescript
export const getDocuments = query({
  args: { userId: v.string() },
  handler: async (ctx, args) => {
    return await ctx.db
      .query("documents")
      .withIndex("by_userId", (q) => q.eq("userId", args.userId))
      .order("desc")
      .collect();
  },
});

export const getDocument = query({
  args: { documentId: v.id("documents") },
  handler: async (ctx, args) => {
    return await ctx.db.get(args.documentId);
  },
});
```

---

## Analysis Endpoints

### Route Analysis (Tier Decision)
```typescript
// convex/actions/routeAnalysis.ts
export const routeAnalysis = action({
  args: {
    documentId: v.id("documents"),
    content: v.string(),
    userId: v.string(),
    forceDeep: v.optional(v.boolean()),
    partyPerspective: v.optional(v.string()),
    analysisBias: v.optional(v.string()),
  },
  handler: async (ctx, args) => {
    // 1. Run Tier 1
    const tier1 = await runTier1Analysis(args.content);

    // 2. Calculate complexity
    const score = calculateComplexityScore(tier1);

    // 3. Store Tier 1 results
    const tier1Id = await ctx.runMutation(internal.tier1Analyses.create, {
      documentId: args.documentId,
      userId: args.userId,
      clauses: tier1.clauses,
      complexityScore: score,
      routedToTier2: args.forceDeep || score >= 6.0,
    });

    // 4. Route
    if (args.forceDeep || score >= 6.0) {
      await ctx.scheduler.runAfter(0, internal.agents.orchestrator.runMultiAgentAnalysis, {
        documentId: args.documentId,
        tier1Results: tier1,
      });
    }

    return { tier: score >= 6.0 ? 2 : 1, complexityScore: score, tier1Id };
  },
});
```

### Get Analysis Results (Real-Time)
```typescript
// convex/tier1Analyses.ts
export const getForDocument = query({
  args: { documentId: v.id("documents") },
  handler: async (ctx, args) => {
    return await ctx.db
      .query("tier1_analyses")
      .withIndex("by_documentId", (q) => q.eq("documentId", args.documentId))
      .order("desc")
      .first();
  },
});

// convex/tier2Analyses.ts
export const getForDocument = query({
  args: { documentId: v.id("documents") },
  handler: async (ctx, args) => {
    return await ctx.db
      .query("tier2_analyses")
      .withIndex("by_documentId", (q) => q.eq("documentId", args.documentId))
      .order("desc")
      .first();
  },
});

// Real-time agent progress
export const getAgentProgress = query({
  args: { documentId: v.id("documents") },
  handler: async (ctx, args) => {
    const tier2 = await ctx.db
      .query("tier2_analyses")
      .withIndex("by_documentId", (q) => q.eq("documentId", args.documentId))
      .order("desc")
      .first();

    if (!tier2) return null;

    return {
      status: tier2.status,
      activeAgents: tier2.activeAgents ?? [],
      completedAgents: (tier2.agentFindings ?? [])
        .filter((f) => f.status === "completed")
        .map((f) => f.agentId),
      failedAgents: (tier2.agentFindings ?? [])
        .filter((f) => f.status === "failed" || f.status === "timeout")
        .map((f) => f.agentId),
      synthesis: tier2.synthesis ?? null,
      totalProcessingTimeMs: tier2.totalProcessingTimeMs ?? 0,
    };
  },
});
```

---

## Agent Endpoints

### Internal Agent Mutations
```typescript
// convex/tier2Analyses.ts (internal)
export const create = internalMutation({
  args: {
    documentId: v.id("documents"),
    tier1AnalysisId: v.id("tier1_analyses"),
    userId: v.string(),
    activeAgents: v.array(v.string()),
  },
  handler: async (ctx, args) => {
    return await ctx.db.insert("tier2_analyses", {
      ...args,
      status: "pending",
      createdAt: Date.now(),
    });
  },
});

export const updateStatus = internalMutation({
  args: {
    documentId: v.id("documents"),
    status: v.string(),
  },
  handler: async (ctx, args) => {
    const tier2 = await ctx.db
      .query("tier2_analyses")
      .withIndex("by_documentId", (q) => q.eq("documentId", args.documentId))
      .order("desc")
      .first();
    if (tier2) {
      await ctx.db.patch(tier2._id, {
        status: args.status as any,
        updatedAt: Date.now(),
      });
    }
  },
});

export const appendAgentFinding = internalMutation({
  args: {
    documentId: v.id("documents"),
    finding: v.any(),
  },
  handler: async (ctx, args) => {
    const tier2 = await ctx.db
      .query("tier2_analyses")
      .withIndex("by_documentId", (q) => q.eq("documentId", args.documentId))
      .order("desc")
      .first();

    if (tier2) {
      const existing = tier2.agentFindings ?? [];
      await ctx.db.patch(tier2._id, {
        agentFindings: [...existing, args.finding],
        updatedAt: Date.now(),
      });
    }
  },
});

export const completeTier2 = internalMutation({
  args: {
    documentId: v.id("documents"),
    agentFindings: v.any(),
    communicationLog: v.any(),
    synthesis: v.any(),
    totalProcessingTimeMs: v.number(),
    totalCost: v.number(),
  },
  handler: async (ctx, args) => {
    const tier2 = await ctx.db
      .query("tier2_analyses")
      .withIndex("by_documentId", (q) => q.eq("documentId", args.documentId))
      .order("desc")
      .first();

    if (tier2) {
      await ctx.db.patch(tier2._id, {
        status: "completed",
        agentFindings: args.agentFindings,
        communicationLog: args.communicationLog,
        synthesis: args.synthesis,
        totalProcessingTimeMs: args.totalProcessingTimeMs,
        totalCost: args.totalCost,
        updatedAt: Date.now(),
      });

      // Also update document status
      await ctx.db.patch(args.documentId, {
        status: "analyzed",
        updatedAt: Date.now(),
      });
    }
  },
});
```

---

## Usage Tracking

```typescript
// convex/usageLogs.ts
export const logUsage = internalMutation({
  args: {
    userId: v.string(),
    provider: v.string(),
    model: v.string(),
    agentId: v.optional(v.string()),
    documentId: v.optional(v.id("documents")),
    inputTokens: v.number(),
    outputTokens: v.number(),
    cost: v.number(),
    responseTimeMs: v.number(),
    success: v.boolean(),
    errorType: v.optional(v.string()),
    rateLimited: v.boolean(),
  },
  handler: async (ctx, args) => {
    return await ctx.db.insert("usage_logs", {
      ...args,
      totalTokens: args.inputTokens + args.outputTokens,
      timestamp: Date.now(),
    });
  },
});

// Get usage statistics
export const getUsageStats = query({
  args: {
    userId: v.string(),
    since: v.optional(v.number()),
  },
  handler: async (ctx, args) => {
    const logs = await ctx.db
      .query("usage_logs")
      .withIndex("by_userId", (q) => q.eq("userId", args.userId))
      .collect();

    const filtered = args.since
      ? logs.filter((l) => l.timestamp >= args.since!)
      : logs;

    return {
      totalRequests: filtered.length,
      totalTokens: filtered.reduce((sum, l) => sum + l.totalTokens, 0),
      totalCost: filtered.reduce((sum, l) => sum + l.cost, 0),
      byProvider: groupBy(filtered, "provider"),
      rateLimitHits: filtered.filter((l) => l.rateLimited).length,
      errorRate: filtered.filter((l) => !l.success).length / filtered.length,
    };
  },
});
```

---

## Error Handling

### Convex Error Patterns

```typescript
import { ConvexError } from "convex/values";

// Standard error types
const ERRORS = {
  NOT_FOUND: "Document not found",
  UNAUTHORIZED: "Not authorized to access this resource",
  RATE_LIMITED: "API rate limit reached. Please wait.",
  ANALYSIS_FAILED: "Analysis failed after maximum retries",
  INVALID_DOCUMENT: "Document is not a legal contract",
} as const;

// Usage in handlers
handler: async (ctx, args) => {
  const doc = await ctx.db.get(args.documentId);
  if (!doc) throw new ConvexError(ERRORS.NOT_FOUND);
  if (doc.userId !== args.userId) throw new ConvexError(ERRORS.UNAUTHORIZED);
  // ...
};
```

### Retry Strategy

```typescript
async function withRetry<T>(
  fn: () => Promise<T>,
  maxAttempts: number = 3,
  baseDelay: number = 1000
): Promise<T> {
  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
      return await fn();
    } catch (error: any) {
      if (attempt === maxAttempts) throw error;
      if (error?.status === 429 || error?.status === 503) {
        const delay = baseDelay * Math.pow(2, attempt - 1);
        await new Promise((r) => setTimeout(r, delay));
      } else {
        throw error;  // Non-retryable error
      }
    }
  }
  throw new Error("Unreachable");
}
```

---

## Related Documentation

- [06_DATABASE_SCHEMA.md](06_DATABASE_SCHEMA.md) — Schema definitions
- [03_TIER2_MULTIAGENT_CORE.md](03_TIER2_MULTIAGENT_CORE.md) — Agent orchestration actions
- [14_SECURITY.md](14_SECURITY.md) — Authorization checks

## Revision History

| Version | Date | Changes |
|---------|------|---------|
| 2.0.0 | 2026-02-09 | Tier 1/2 analysis endpoints, agent progress queries |
| 1.0.0 | 2025-01-01 | Basic document and analysis APIs |
