# 06_DATABASE_SCHEMA

**Last Updated:** 2026-02-09  
**Version:** 2.0.0  
**Author:** LawBotics Team

---

## Table of Contents

1. [Schema Overview](#schema-overview)
2. [Existing Tables](#existing-tables)
3. [New Tables for Hybrid System](#new-tables-for-hybrid-system)
4. [Table Relationships](#table-relationships)
5. [Indexes](#indexes)
6. [Migration Guide](#migration-guide)

---

## Schema Overview

```
┌──────────────┐     ┌──────────────┐     ┌──────────────────┐
│    users      │     │  documents   │     │    analyses       │
│ (Clerk sync)  │────▶│ (uploads)    │────▶│ (Tier 1 results)  │
└──────────────┘     └──────┬───────┘     └──────────────────┘
                            │
                    ┌───────┴───────┐
                    │               │
            ┌───────▼──────┐ ┌─────▼──────────┐
            │tier2_analyses│ │extractedParties │
            │(multi-agent) │ └────────────────┘
            └──────────────┘
```

---

## Existing Tables

### users
```typescript
users: defineTable({
  userId: v.string(),          // Clerk user ID
  email: v.string(),
  name: v.string(),
  role: v.union(v.literal("admin"), v.literal("user")),
  profileImage: v.optional(v.string()),
  createdAt: v.number(),
  lastActive: v.optional(v.number()),
}).index("by_userId", ["userId"])
  .index("by_email", ["email"])
  .index("by_role", ["role"]),
```

### documents
```typescript
documents: defineTable({
  userId: v.string(),
  title: v.string(),
  content: v.optional(v.string()),
  fileUrl: v.optional(v.string()),
  storageId: v.optional(v.id("_storage")),
  mimeType: v.optional(v.string()),
  ownerId: v.optional(v.string()),
  fileSize: v.optional(v.number()),
  vectorEmbedding: v.optional(v.array(v.float64())),
  isLegalDocument: v.optional(v.boolean()),
  status: v.union(
    v.literal("uploaded"),
    v.literal("processing"),
    v.literal("analyzed"),
    v.literal("error")
  ),
  createdAt: v.number(),
  updatedAt: v.optional(v.number()),
}).index("by_userId", ["userId"])
  .index("by_status", ["status"]),
```

### analyses (Tier 1 results — extended)
```typescript
analyses: defineTable({
  documentId: v.id("documents"),
  userId: v.string(),
  status: v.union(
    v.literal("pending"),
    v.literal("in-progress"),
    v.literal("completed"),
    v.literal("error")
  ),
  keyClauses: v.optional(v.array(v.object({
    clause: v.string(),
    explanation: v.string(),
    risk_level: v.string(),
  }))),
  negotiableTerms: v.optional(v.array(v.object({
    term: v.string(),
    current_position: v.string(),
    suggested_revision: v.string(),
    potential_impact: v.string(),
  }))),
  redFlags: v.optional(v.array(v.string())),
  recommendations: v.optional(v.array(v.string())),
  overallImpression: v.optional(v.string()),
  riskScore: v.optional(v.number()),
  metadata: v.optional(v.object({
    modelUsed: v.optional(v.string()),
    tokensUsed: v.optional(v.number()),
    processingTimeMs: v.optional(v.number()),
    contractType: v.optional(v.string()),
    partyPerspective: v.optional(v.string()),
    analysisBias: v.optional(v.string()),
    analysisDepth: v.optional(v.string()),
    error: v.optional(v.string()),
    attempts: v.optional(v.number()),
  })),
  createdAt: v.number(),
  updatedAt: v.optional(v.number()),
}).index("by_documentId", ["documentId"])
  .index("by_userId", ["userId"])
  .index("by_status", ["status"]),
```

### extractedParties
```typescript
extractedParties: defineTable({
  documentId: v.id("documents"),
  parties: v.array(v.object({
    name: v.string(),
    role: v.string(),
    description: v.optional(v.string()),
  })),
  extractedAt: v.number(),
}).index("by_documentId", ["documentId"]),
```

### contracts
```typescript
contracts: defineTable({
  userId: v.string(),
  title: v.string(),
  description: v.optional(v.string()),
  status: v.union(
    v.literal("draft"), v.literal("pending_review"),
    v.literal("under_negotiation"), v.literal("approved"),
    v.literal("active"), v.literal("expired"),
    v.literal("terminated"), v.literal("archived")
  ),
  contractType: v.union(
    v.literal("service"), v.literal("nda"),
    v.literal("employment"), v.literal("license"),
    v.literal("partnership"), v.literal("vendor"),
    v.literal("lease"), v.literal("consulting"),
    v.literal("distribution"), v.literal("other")
  ),
  parties: v.array(v.object({
    name: v.string(),
    role: v.string(),
    email: v.optional(v.string()),
  })),
  clauses: v.array(v.object({
    title: v.string(),
    content: v.string(),
    type: v.string(),
  })),
  startDate: v.optional(v.string()),
  endDate: v.optional(v.string()),
  value: v.optional(v.number()),
  currency: v.optional(v.string()),
  autoRenewal: v.optional(v.boolean()),
  renewalTerms: v.optional(v.string()),
  jurisdiction: v.optional(v.string()),
  tags: v.optional(v.array(v.string())),
  documentId: v.optional(v.id("documents")),
  createdAt: v.number(),
  updatedAt: v.number(),
}).index("by_userId", ["userId"])
  .index("by_status", ["status"])
  .index("by_contractType", ["contractType"]),
```

### contract_analytics
```typescript
contract_analytics: defineTable({
  userId: v.string(),
  contractId: v.id("contracts"),
  metricType: v.string(),
  metricValue: v.number(),
  period: v.optional(v.string()),
  calculatedAt: v.number(),
}).index("by_contractId", ["contractId"])
  .index("by_userId", ["userId"]),
```

### clause_library
```typescript
clause_library: defineTable({
  userId: v.string(),
  title: v.string(),
  content: v.string(),
  clauseType: v.string(),
  description: v.optional(v.string()),
  tags: v.optional(v.array(v.string())),
  isPublic: v.optional(v.boolean()),
  usageCount: v.optional(v.number()),
  createdAt: v.number(),
  updatedAt: v.number(),
}).index("by_userId", ["userId"])
  .index("by_clauseType", ["clauseType"]),
```

---

## New Tables for Hybrid System

### tier1_analyses

Stores fine-grained Tier 1 (LLaMA) results and the complexity score for routing.

```typescript
tier1_analyses: defineTable({
  documentId: v.id("documents"),
  userId: v.string(),
  status: v.union(
    v.literal("pending"),
    v.literal("processing"),
    v.literal("completed"),
    v.literal("error")
  ),
  // LLaMA clause extraction results
  clauses: v.optional(v.array(v.object({
    clauseType: v.string(),
    present: v.boolean(),
    text: v.optional(v.string()),
    confidence: v.number(),
    section: v.optional(v.string()),
  }))),
  // Complexity scoring
  complexityScore: v.optional(v.number()),           // 0-10
  complexityFactors: v.optional(v.object({
    clauseCount: v.number(),
    ambiguousClauseRatio: v.number(),
    uniqueClauseTypes: v.number(),
    contractLength: v.number(),
    financialComplexity: v.number(),
    crossJurisdiction: v.boolean(),
    multiParty: v.boolean(),
    hasUnusualProvisions: v.boolean(),
    tier1Confidence: v.number(),
    regulatoryDomains: v.number(),
  })),
  routedToTier2: v.boolean(),
  // Performance tracking
  modelVersion: v.optional(v.string()),
  processingTimeMs: v.optional(v.number()),
  createdAt: v.number(),
}).index("by_documentId", ["documentId"])
  .index("by_userId", ["userId"])
  .index("by_routedToTier2", ["routedToTier2"]),
```

### tier2_analyses

Stores multi-agent Tier 2 analysis results, communication logs, and synthesis.

```typescript
tier2_analyses: defineTable({
  documentId: v.id("documents"),
  tier1AnalysisId: v.id("tier1_analyses"),
  userId: v.string(),
  status: v.union(
    v.literal("pending"),
    v.literal("agents_running"),
    v.literal("peer_discussion"),
    v.literal("synthesis"),
    v.literal("completed"),
    v.literal("error"),
    v.literal("partial")    // Some agents failed, partial results
  ),
  // Active agents for this analysis
  activeAgents: v.optional(v.array(v.string())),
  // Per-agent findings
  agentFindings: v.optional(v.array(v.object({
    agentId: v.string(),
    model: v.string(),
    provider: v.string(),
    status: v.union(
      v.literal("running"),
      v.literal("completed"),
      v.literal("failed"),
      v.literal("timeout")
    ),
    findings: v.optional(v.any()),    // Agent-specific structured output
    confidence: v.optional(v.number()),
    processingTimeMs: v.optional(v.number()),
    tokenUsage: v.optional(v.object({
      input: v.number(),
      output: v.number(),
    })),
    error: v.optional(v.string()),
    completedAt: v.optional(v.number()),
  }))),
  // Peer discussion log
  communicationLog: v.optional(v.array(v.object({
    round: v.number(),
    sender: v.string(),
    recipients: v.array(v.string()),
    messageType: v.union(
      v.literal("finding"),
      v.literal("question"),
      v.literal("clarification"),
      v.literal("agreement"),
      v.literal("disagreement")
    ),
    content: v.string(),
    timestamp: v.number(),
  }))),
  // Synthesized final report
  synthesis: v.optional(v.object({
    executiveSummary: v.string(),
    overallRiskScore: v.number(),
    riskLevel: v.string(),
    recommendation: v.string(),
    priorityIssues: v.array(v.object({
      rank: v.number(),
      issue: v.string(),
      agentsFlagged: v.array(v.string()),
      severity: v.string(),
      recommendation: v.string(),
      negotiationStrategy: v.optional(v.string()),
    })),
    confidence: v.number(),
    agentAgreementLevel: v.optional(v.number()),
  })),
  // Performance & cost
  totalProcessingTimeMs: v.optional(v.number()),
  totalCost: v.optional(v.number()),       // $0 for free tiers
  createdAt: v.number(),
  updatedAt: v.optional(v.number()),
}).index("by_documentId", ["documentId"])
  .index("by_userId", ["userId"])
  .index("by_status", ["status"])
  .index("by_tier1AnalysisId", ["tier1AnalysisId"]),
```

### usage_logs

Tracks API usage across Groq and Gemini free tiers for rate limit management.

```typescript
usage_logs: defineTable({
  userId: v.string(),
  provider: v.union(
    v.literal("groq"),
    v.literal("gemini"),
    v.literal("llama_colab")
  ),
  model: v.string(),
  agentId: v.optional(v.string()),
  documentId: v.optional(v.id("documents")),
  // Token usage
  inputTokens: v.number(),
  outputTokens: v.number(),
  totalTokens: v.number(),
  // Cost ($0 for free tier)
  cost: v.number(),
  // Performance
  responseTimeMs: v.number(),
  success: v.boolean(),
  errorType: v.optional(v.string()),
  // Rate limit tracking
  rateLimited: v.boolean(),
  retryCount: v.optional(v.number()),
  timestamp: v.number(),
}).index("by_userId", ["userId"])
  .index("by_provider", ["provider"])
  .index("by_timestamp", ["timestamp"])
  .index("by_model", ["model"]),
```

---

## Table Relationships

```
users (1) ──────────────────> (N) documents
documents (1) ──────────────> (N) analyses (Tier 1 legacy)
documents (1) ──────────────> (N) tier1_analyses
documents (1) ──────────────> (1) extractedParties
documents (1) ──────────────> (N) tier2_analyses
tier1_analyses (1) ─────────> (1) tier2_analyses
users (1) ──────────────────> (N) contracts
contracts (1) ──────────────> (N) contract_analytics
users (1) ──────────────────> (N) usage_logs
documents (1) ──────────────> (N) usage_logs
```

---

## Indexes

### Query Patterns & Index Usage

| Query | Table | Index |
|-------|-------|-------|
| Get user's documents | documents | `by_userId` |
| Get Tier 1 results for a doc | tier1_analyses | `by_documentId` |
| Get Tier 2 results for a doc | tier2_analyses | `by_documentId` |
| List in-progress Tier 2 analyses | tier2_analyses | `by_status` |
| Get usage by provider | usage_logs | `by_provider` |
| Get recent usage | usage_logs | `by_timestamp` |
| Track rate limits per model | usage_logs | `by_model` |

---

## Migration Guide

### Adding New Tables to Existing Schema

Add these table definitions to the existing `schema.ts`:

```typescript
// convex/schema.ts — additions

import { defineSchema, defineTable } from "convex/server";
import { v } from "convex/values";

export default defineSchema({
  // ... existing tables unchanged ...

  // NEW: Tier 1 analysis tracking
  tier1_analyses: defineTable({ /* as above */ }),

  // NEW: Tier 2 multi-agent results
  tier2_analyses: defineTable({ /* as above */ }),

  // NEW: API usage tracking
  usage_logs: defineTable({ /* as above */ }),
});
```

### Data Migration Steps

1. Deploy schema changes (Convex handles additive changes automatically)
2. Existing `analyses` table remains untouched for backward compatibility
3. New analyses use `tier1_analyses` + `tier2_analyses` pipeline
4. Set up usage_logs recording in all agent call paths

---

## Related Documentation

- [07_API_DESIGN.md](07_API_DESIGN.md) — Queries and mutations for these tables
- [03_TIER2_MULTIAGENT_CORE.md](03_TIER2_MULTIAGENT_CORE.md) — How tier2_analyses data is populated
- [14_SECURITY.md](14_SECURITY.md) — Row-level security policies

## Revision History

| Version | Date | Changes |
|---------|------|---------|
| 2.0.0 | 2026-02-09 | Added tier1_analyses, tier2_analyses, usage_logs |
| 1.0.0 | 2025-01-01 | Original schema with analyses table |
