# 09_FRONTEND_IMPLEMENTATION

**Last Updated:** 2026-02-09  
**Version:** 2.0.0  
**Author:** LawBotics Team

---

## Table of Contents

1. [Tech Stack](#tech-stack)
2. [Route Structure](#route-structure)
3. [Multi-Agent Progress UI](#multi-agent-progress-ui)
4. [Analysis Results View](#analysis-results-view)
5. [Real-Time State Management](#real-time-state-management)
6. [Key Components](#key-components)

---

## Tech Stack

| Technology | Version | Purpose |
|-----------|---------|---------|
| Next.js | 15.3.4 | App Router, RSC, SSR |
| React | 19 | UI rendering |
| TypeScript | 5 | Type safety |
| Tailwind CSS | 4 | Styling |
| Radix UI / Shadcn | latest | Accessible components |
| Zustand | 5.0.5 | Client state management |
| Convex React | 1.25.0 | Real-time data subscriptions |
| Tiptap | 3.0.7 | Rich text contract editor |
| react-pdf | 9.2.1 | PDF rendering |
| jspdf | 3.0.1 | PDF export |
| Recharts | 2.15.4 | Analytics charts |

---

## Route Structure

```
src/app/
├── (auth)/                    # Protected routes (Clerk)
│   ├── dashboard/
│   │   └── page.tsx          # Main dashboard
│   ├── documents/
│   │   ├── page.tsx          # Document list
│   │   └── [id]/
│   │       └── page.tsx      # Document detail + analysis
│   ├── contracts/
│   │   ├── page.tsx          # Contract management
│   │   └── [id]/
│   │       └── page.tsx      # Contract detail
│   ├── analytics/
│   │   └── page.tsx          # Usage analytics
│   └── settings/
│       └── page.tsx          # User settings
├── (public)/                  # Public routes
│   ├── page.tsx              # Landing page
│   └── features/
│       └── page.tsx
└── layout.tsx                 # Root layout
```

---

## Multi-Agent Progress UI

### Agent Progress Component

```tsx
// src/components/feature-pages/analysis/AgentProgress.tsx

"use client";

import { useQuery } from "convex/react";
import { api } from "../../../../convex/_generated/api";
import { Id } from "../../../../convex/_generated/dataModel";
import { cn } from "@/lib/utils";
import { Loader2, CheckCircle2, XCircle, Clock } from "lucide-react";

interface AgentProgressProps {
  documentId: Id<"documents">;
}

const AGENT_LABELS: Record<string, { label: string; icon: string }> = {
  clause_extractor: { label: "Clause Extraction", icon: "📋" },
  risk_assessor: { label: "Risk Assessment", icon: "⚠️" },
  financial_analyst: { label: "Financial Analysis", icon: "💰" },
  ip_specialist: { label: "IP/Licensing", icon: "📜" },
  employment_specialist: { label: "Employment Terms", icon: "👔" },
  ma_specialist: { label: "M&A Analysis", icon: "🤝" },
  compliance_checker: { label: "Compliance Check", icon: "✅" },
};

export function AgentProgress({ documentId }: AgentProgressProps) {
  // Real-time subscription — updates automatically via Convex WebSocket
  const progress = useQuery(api.tier2Analyses.getAgentProgress, { documentId });

  if (!progress) return null;

  const phases = [
    { key: "agents_running", label: "Agent Analysis" },
    { key: "peer_discussion", label: "Peer Discussion" },
    { key: "synthesis", label: "Final Synthesis" },
  ];

  const currentPhaseIndex = phases.findIndex((p) => p.key === progress.status);

  return (
    <div className="space-y-6 rounded-xl border bg-card p-6">
      {/* Phase Progress Bar */}
      <div className="flex items-center justify-between">
        {phases.map((phase, i) => (
          <div key={phase.key} className="flex items-center flex-1">
            <div
              className={cn(
                "flex h-8 w-8 items-center justify-center rounded-full text-sm font-medium",
                i < currentPhaseIndex
                  ? "bg-green-500 text-white"
                  : i === currentPhaseIndex
                    ? "bg-blue-500 text-white animate-pulse"
                    : "bg-muted text-muted-foreground"
              )}
            >
              {i + 1}
            </div>
            <span className="ml-2 text-sm font-medium">{phase.label}</span>
            {i < phases.length - 1 && (
              <div
                className={cn(
                  "mx-4 h-0.5 flex-1",
                  i < currentPhaseIndex ? "bg-green-500" : "bg-muted"
                )}
              />
            )}
          </div>
        ))}
      </div>

      {/* Individual Agent Status */}
      <div className="grid grid-cols-2 gap-3 md:grid-cols-3 lg:grid-cols-4">
        {progress.activeAgents.map((agentId) => {
          const info = AGENT_LABELS[agentId] ?? { label: agentId, icon: "🤖" };
          const isComplete = progress.completedAgents.includes(agentId);
          const isFailed = progress.failedAgents.includes(agentId);

          return (
            <div
              key={agentId}
              className={cn(
                "flex items-center gap-2 rounded-lg border p-3 text-sm",
                isComplete && "border-green-200 bg-green-50 dark:bg-green-950/20",
                isFailed && "border-red-200 bg-red-50 dark:bg-red-950/20",
                !isComplete && !isFailed && "border-blue-200 bg-blue-50 dark:bg-blue-950/20"
              )}
            >
              <span>{info.icon}</span>
              <span className="flex-1 font-medium">{info.label}</span>
              {isComplete && <CheckCircle2 className="h-4 w-4 text-green-500" />}
              {isFailed && <XCircle className="h-4 w-4 text-red-500" />}
              {!isComplete && !isFailed && (
                <Loader2 className="h-4 w-4 animate-spin text-blue-500" />
              )}
            </div>
          );
        })}
      </div>

      {/* Processing Time */}
      <div className="flex items-center gap-2 text-sm text-muted-foreground">
        <Clock className="h-4 w-4" />
        <span>
          {progress.totalProcessingTimeMs > 0
            ? `${(progress.totalProcessingTimeMs / 1000).toFixed(1)}s elapsed`
            : "Processing..."}
        </span>
      </div>
    </div>
  );
}
```

---

## Analysis Results View

### Tiered Results Display

```tsx
// src/components/feature-pages/analysis/AnalysisResults.tsx

"use client";

import { useQuery } from "convex/react";
import { api } from "../../../../convex/_generated/api";
import { Id } from "../../../../convex/_generated/dataModel";
import { Badge } from "@/components/ui/badge";
import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card";

interface AnalysisResultsProps {
  documentId: Id<"documents">;
}

export function AnalysisResults({ documentId }: AnalysisResultsProps) {
  const tier1 = useQuery(api.tier1Analyses.getForDocument, { documentId });
  const tier2 = useQuery(api.tier2Analyses.getForDocument, { documentId });

  // Use Tier 2 synthesis if available, otherwise Tier 1
  const hasDeepAnalysis = tier2?.status === "completed" && tier2.synthesis;

  return (
    <div className="space-y-6">
      {/* Tier Badge */}
      <div className="flex items-center gap-3">
        <Badge variant={hasDeepAnalysis ? "default" : "secondary"}>
          {hasDeepAnalysis ? "Deep Analysis (Tier 2)" : "Quick Analysis (Tier 1)"}
        </Badge>
        {tier1 && (
          <span className="text-sm text-muted-foreground">
            Complexity: {tier1.complexityScore}/10
          </span>
        )}
      </div>

      {/* Risk Score */}
      {hasDeepAnalysis && tier2.synthesis && (
        <Card>
          <CardHeader>
            <CardTitle className="flex items-center justify-between">
              <span>Overall Risk Assessment</span>
              <RiskBadge
                score={tier2.synthesis.overallRiskScore}
                level={tier2.synthesis.riskLevel}
              />
            </CardTitle>
          </CardHeader>
          <CardContent>
            <p className="text-sm">{tier2.synthesis.executiveSummary}</p>
            <div className="mt-4">
              <SigningRecommendation
                recommendation={tier2.synthesis.recommendation}
              />
            </div>
          </CardContent>
        </Card>
      )}

      {/* Priority Issues */}
      {hasDeepAnalysis && tier2.synthesis?.priorityIssues && (
        <Card>
          <CardHeader>
            <CardTitle>Priority Issues</CardTitle>
          </CardHeader>
          <CardContent>
            <div className="space-y-3">
              {tier2.synthesis.priorityIssues.map((issue: any, i: number) => (
                <div key={i} className="border-l-4 border-l-red-500 pl-4 py-2">
                  <div className="flex items-center gap-2">
                    <span className="font-mono text-sm">#{issue.rank}</span>
                    <Badge variant="destructive">{issue.severity}</Badge>
                  </div>
                  <p className="font-medium mt-1">{issue.issue}</p>
                  <p className="text-sm text-muted-foreground mt-1">
                    {issue.recommendation}
                  </p>
                  {issue.negotiationStrategy && (
                    <p className="text-sm text-blue-600 mt-1">
                      💡 {issue.negotiationStrategy}
                    </p>
                  )}
                </div>
              ))}
            </div>
          </CardContent>
        </Card>
      )}

      {/* Tier 1 Clause Results (always shown) */}
      {tier1?.clauses && (
        <Card>
          <CardHeader>
            <CardTitle>Detected Clauses</CardTitle>
          </CardHeader>
          <CardContent>
            <div className="grid grid-cols-1 gap-2 md:grid-cols-2">
              {tier1.clauses
                .filter((c: any) => c.present)
                .map((clause: any, i: number) => (
                  <div key={i} className="flex items-center justify-between rounded-lg bg-muted/50 px-3 py-2">
                    <span className="text-sm font-medium">{clause.clauseType}</span>
                    <Badge variant={clause.confidence > 0.7 ? "default" : "outline"}>
                      {(clause.confidence * 100).toFixed(0)}%
                    </Badge>
                  </div>
                ))}
            </div>
          </CardContent>
        </Card>
      )}
    </div>
  );
}

function RiskBadge({ score, level }: { score: number; level: string }) {
  const colors: Record<string, string> = {
    LOW: "bg-green-100 text-green-800",
    MEDIUM: "bg-yellow-100 text-yellow-800",
    HIGH: "bg-orange-100 text-orange-800",
    CRITICAL: "bg-red-100 text-red-800",
  };

  return (
    <span className={cn("rounded-full px-3 py-1 text-sm font-medium", colors[level])}>
      {score}/100 — {level}
    </span>
  );
}

function SigningRecommendation({ recommendation }: { recommendation: string }) {
  const info: Record<string, { label: string; color: string }> = {
    SIGN_AS_IS: { label: "Safe to Sign", color: "text-green-600" },
    SIGN_WITH_MINOR_REVISIONS: { label: "Sign with Minor Revisions", color: "text-yellow-600" },
    NEGOTIATE_KEY_TERMS: { label: "Negotiate Key Terms", color: "text-orange-600" },
    DO_NOT_SIGN: { label: "Do Not Sign", color: "text-red-600" },
  };

  const rec = info[recommendation] ?? { label: recommendation, color: "text-gray-600" };

  return (
    <div className={cn("text-lg font-bold", rec.color)}>
      Recommendation: {rec.label}
    </div>
  );
}
```

---

## Real-Time State Management

### Convex + Zustand Integration

```typescript
// src/store/analysisStore.ts

import { create } from "zustand";

interface AnalysisState {
  selectedDocumentId: string | null;
  forceDeepAnalysis: boolean;
  partyPerspective: string;
  analysisBias: string;
  
  setSelectedDocument: (id: string | null) => void;
  setForceDeep: (force: boolean) => void;
  setPartyPerspective: (party: string) => void;
  setAnalysisBias: (bias: string) => void;
}

export const useAnalysisStore = create<AnalysisState>((set) => ({
  selectedDocumentId: null,
  forceDeepAnalysis: false,
  partyPerspective: "neutral",
  analysisBias: "balanced",
  
  setSelectedDocument: (id) => set({ selectedDocumentId: id }),
  setForceDeep: (force) => set({ forceDeepAnalysis: force }),
  setPartyPerspective: (party) => set({ partyPerspective: party }),
  setAnalysisBias: (bias) => set({ analysisBias: bias }),
}));
```

### Data Flow

```
Convex DB ──(WebSocket)──▶ useQuery() hooks ──▶ Components (auto-rerender)
                                                    │
User Actions ──▶ useMutation() / useAction() ──▶ Convex Backend
                                                    │
                                              Zustand (local UI state)
```

All analysis data flows through Convex real-time queries. Zustand only manages local UI preferences (party perspective, analysis mode, etc.).

---

## Key Components

| Component | Location | Purpose |
|-----------|----------|---------|
| `AgentProgress` | `components/feature-pages/analysis/` | Real-time multi-agent progress |
| `AnalysisResults` | `components/feature-pages/analysis/` | Tiered results display |
| `DocumentUploader` | `components/feature-pages/documents/` | File upload with drag-drop |
| `ContractEditor` | `components/feature-pages/contracts/` | Tiptap rich text editor |
| `RiskDashboard` | `components/feature-pages/analytics/` | Recharts risk visualizations |
| `ClauseLibrary` | `components/feature-pages/clauses/` | Reusable clause templates |

---

## Related Documentation

- [01_ARCHITECTURE_OVERVIEW.md](01_ARCHITECTURE_OVERVIEW.md) — System architecture
- [07_API_DESIGN.md](07_API_DESIGN.md) — API endpoints consumed by frontend
- [13_CODE_STANDARDS.md](13_CODE_STANDARDS.md) — Component conventions

## Revision History

| Version | Date | Changes |
|---------|------|---------|
| 2.0.0 | 2026-02-09 | Multi-agent progress UI, tiered results |
| 1.0.0 | 2025-01-01 | Single-analysis results display |
