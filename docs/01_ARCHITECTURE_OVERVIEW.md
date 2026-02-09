# 01_ARCHITECTURE_OVERVIEW

**Last Updated:** 2026-02-09  
**Version:** 2.0.0  
**Author:** LawBotics Team

---

## Table of Contents

1. [System Architecture](#system-architecture)
2. [Data Flow](#data-flow)
3. [Technology Stack](#technology-stack)
4. [Design Decisions](#design-decisions)
5. [Component Interactions](#component-interactions)
6. [State Management](#state-management)
7. [Error Propagation](#error-propagation)
8. [Scalability](#scalability)

---

## System Architecture

LawBotics v2.0 is a hybrid multi-agent legal contract analyzer. Tier 1 provides rapid clause extraction via fine-tuned LLaMA 3.2-3B. Tier 2 deploys specialized AI agents (via Groq and Google Gemini free tiers) for deep analysis of complex contracts.

```
┌─────────────────────────────────────────────────────────────────┐
│                      CONTRACT INPUT                             │
│              (PDF/DOCX upload via Next.js frontend)             │
└────────────────────┬────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│          DOCUMENT PROCESSING PIPELINE                           │
│  LangChain PDFLoader → Text Extraction → Chunking → Embeddings │
│  Google Embedding API (embedding-001) → Convex Storage          │
└────────────────────┬────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│               TIER 1: RAPID TRIAGE                              │
│         Fine-tuned LLaMA 3.2-3B (LoRA on CUAD)                 │
│                                                                 │
│  • Extract 41 clause types (5-8 seconds)                        │
│  • Calculate baseline risk score (0-100)                        │
│  • Detect complexity indicators (0-10 scale)                    │
│  • Identify contract type & domain                              │
│  • Extract parties, dates, values                               │
│                                                                 │
│         Complexity Score: 0-10                                  │
│              ↓                    ↓                              │
│         Score < 6            Score ≥ 6                           │
│    (70% of contracts)    (30% of contracts)                     │
└──────────┬───────────────────────────┬──────────────────────────┘
           │                           │
           ▼                           ▼
    ┌──────────┐          ┌────────────────────────────────────┐
    │  RETURN  │          │   TIER 2: DEEP ANALYSIS            │
    │ Tier 1   │          │   Multi-Agent Collaborative System  │
    │ RESULTS  │          │   (Groq + Google Gemini Free APIs)  │
    └──────────┘          │                                    │
                          │  ┌─────────────────────────────┐  │
                          │  │  Coordinator Agent          │  │
                          │  │  (Gemini 2.0 Flash)         │  │
                          │  │  • Activates specialized     │  │
                          │  │    agents based on context   │  │
                          │  │  • Manages communication     │  │
                          │  └──────────┬──────────────────┘  │
                          │             │                      │
                          │  ┌──────────┴──────────┐          │
                          │  │                     │          │
                          │  ▼                     ▼          │
                          │ ┌────────┐  ┌──────────────────┐  │
                          │ │Clause  │  │Risk Assessment   │  │
                          │ │Agent   │◄─┤Agent             │  │
                          │ │(Groq   │  │(Groq DeepSeek    │  │
                          │ │ 8B)    │  │ R1-70B)          │  │
                          │ └───┬────┘  └────┬─────────────┘  │
                          │     │            │                │
                          │     ▼            ▼                │
                          │ ┌────────┐  ┌──────────────────┐  │
                          │ │Finance │  │IP/Licensing      │  │
                          │ │Agent   │◄─┤Agent             │  │
                          │ │(Groq   │  │(Groq Llama       │  │
                          │ │ 70B)   │  │ 3.3-70B)         │  │
                          │ └───┬────┘  └────┬─────────────┘  │
                          │     │            │                │
                          │     └─────┬──────┘                │
                          │           ▼                       │
                          │  ┌─────────────────────────┐     │
                          │  │ Domain-Specific Agents  │     │
                          │  │ (Conditional Activation)│     │
                          │  │ • Employment Specialist │     │
                          │  │ • M&A Specialist        │     │
                          │  │ • SaaS/Tech Specialist  │     │
                          │  │ • Compliance Checker    │     │
                          │  │ (Groq Mixtral-8x7B)     │     │
                          │  └──────────┬──────────────┘     │
                          │             │                    │
                          │             ▼                    │
                          │  ┌─────────────────────────┐    │
                          │  │  Synthesis Agent        │    │
                          │  │  (Gemini 2.0 Flash)     │    │
                          │  │  • Final analysis       │    │
                          │  │  • Resolve conflicts    │    │
                          │  │  • Strategic advice     │    │
                          │  └─────────────────────────┘    │
                          │                                 │
                          └─────────────────────────────────┘
                                       │
                                       ▼
                          ┌────────────────────────────┐
                          │  COMPREHENSIVE RESULTS     │
                          │  Stored in Convex DB       │
                          │  Real-time UI via WebSocket│
                          └────────────────────────────┘
```

---

## Data Flow

```
User Upload → Clerk Auth Check → Convex Mutation (documents.create)
  → File Storage (Convex _storage) → Process Document Action
     → PDF/Text Extraction (LangChain PDFLoader)
     → Text Chunking (RecursiveCharacterTextSplitter, 6000 chars)
     → Vector Embedding (Google embedding-001)
     → Legal Document Validation (Gemini Flash)
     → Store Content + Embeddings in Convex
     → Trigger Tier 1 Analysis
        → LLaMA 3.2-3B Inference (clause extraction + risk score)
        → Calculate Complexity Score (0-10)
        → IF score < 6: Store Tier 1 results → Return
        → IF score ≥ 6: Trigger Tier 2 Multi-Agent
           → Coordinator selects active agents
           → Parallel agent execution (Groq + Gemini)
           → Inter-agent peer discussion (max 2 rounds)
           → Synthesis agent final review
           → Store comprehensive results → Return
```

---

## Technology Stack

| Layer | Technology | Version | Justification |
|-------|-----------|---------|---------------|
| **Frontend** | Next.js | 15.3.4 | App Router, React Server Components, streaming |
| **UI Framework** | React | 19.0.0 | Concurrent rendering, `use` hook, Suspense |
| **Styling** | Tailwind CSS | 4.x | Utility-first, zero-runtime, CSS-in-JS alternative |
| **Components** | Radix UI + Shadcn | Latest | Accessible, unstyled primitives |
| **State** | Zustand | 5.0.5 | Minimal, no boilerplate, TypeScript-first |
| **Auth** | Clerk | 6.24.0 | Drop-in auth, Convex integration, webhooks |
| **Database** | Convex | 1.25.0 | Real-time subscriptions, serverless, type-safe |
| **AI - Tier 1** | LLaMA 3.2-3B | Fine-tuned | CUAD-trained, Ollama on Google Colab |
| **AI - Tier 2 Agents** | Groq API | Free tier | Llama 3.3-70B, DeepSeek-R1-70B, Mixtral-8x7B |
| **AI - Coordinator/Synthesis** | Google Gemini | 2.0 Flash | Already integrated, free tier, strong reasoning |
| **AI - Embeddings** | Google GenAI | embedding-001 | Already integrated, free tier |
| **Document Processing** | LangChain | 0.3.29 | PDF loading, text splitting, embeddings |
| **Validation** | Zod | 3.25.76 | Runtime type validation, schema-first |
| **Charts** | Recharts | 2.15.4 | React-native charting, responsive |
| **Rich Text** | Tiptap | 3.0.7 | Headless editor, contract clause editing |
| **PDF** | react-pdf + jspdf | 9.2.1 / 3.0.1 | View + generate PDFs |
| **Language** | TypeScript | 5.x | Strict mode, full type safety |
| **AI Training** | Unsloth + PEFT | Latest | LoRA fine-tuning, 4-bit quantization |
| **Training Framework** | PyTorch + Transformers | Latest | Model training and inference |

### Why These Free AI Providers?

**Groq (Tier 2 Agents):**
- Free tier: 30 RPM, 14,400 requests/day for most models
- LPU inference: 10x faster than GPU-based inference
- Supports Llama 3.3-70B, DeepSeek-R1-Distill-70B, Mixtral-8x7B
- OpenAI-compatible API (easy migration path)
- No credit card required

**Google Gemini (Coordinator + Synthesis):**
- Already integrated in the codebase (Google API key exists)
- Free tier: 15 RPM, 1 million tokens/day for Gemini 2.0 Flash
- Strong reasoning capabilities for synthesis
- Multimodal (can process PDFs directly if needed)

---

## Design Decisions

### Why Hybrid Over Pure Multi-Agent?

| Concern | Pure Multi-Agent | Hybrid (LawBotics v2) |
|---------|-----------------|----------------------|
| Cost | $0.50+ per contract | $0.004 for 70% of contracts |
| Latency | 40-90s always | 5-8s for simple contracts |
| API Dependency | 100% reliant on external APIs | 70% handled by LLaMA on Colab |
| Accuracy (simple) | 97% (overkill) | 93% (sufficient) |
| Accuracy (complex) | 97% | 97% (Tier 2 activates) |

### Why Convex Over REST API + PostgreSQL?

1. **Real-time by default** — Analysis progress streams to UI without polling
2. **Serverless** — No server management, auto-scaling
3. **Type-safe** — Schema generates TypeScript types automatically
4. **Actions** — Long-running AI tasks with built-in scheduling
5. **Transactions** — ACID guarantees for multi-step analysis workflows
6. **Already integrated** — Existing codebase uses Convex extensively

### Why Groq Over Other Agent Providers?

1. **Free** — No API costs for agent inference
2. **Fast** — LPU delivers tokens 10x faster than GPU
3. **Multiple models** — Can assign different models per agent role
4. **Reliable** — 99.9% uptime, managed infrastructure
5. **No local GPU required** — All inference handled in cloud

### Why LLaMA 3.2-3B for Tier 1?

1. **Speed** — 3B params = fast inference on Colab GPU
2. **Fine-tuned on CUAD** — 13,000+ labeled legal examples
3. **Free hosting** — Google Colab provides free GPU via Ollama
4. **Quantized** — 4-bit = runs on Colab's T4 GPU easily
5. **Sufficient** — 93% accuracy for clause extraction

---

## Component Interactions

```typescript
// Core type definitions for the hybrid system

interface AnalysisRequest {
  documentId: Id<"documents">;
  tier: "tier1" | "tier2" | "auto";
  forceDeepAnalysis?: boolean;
  partyPerspective?: string;
  analysisBias?: "neutral" | "favorable" | "risk";
  userContext?: {
    contractType?: string;
    focusAreas?: string[];
    jurisdiction?: string;
  };
}

interface AnalysisResult {
  tier: "tier1" | "tier2";
  processingTime: number;
  confidence: number;
  tier1Findings: Tier1Findings;
  tier2Findings?: Tier2Findings;
  costBreakdown: {
    tier1Cost: number;
    tier2Cost?: number;
    totalCost: number;
  };
}

interface Tier1Findings {
  clauses: ExtractedClause[];
  complexityScore: number;
  contractType: string;
  baselineRiskScore: number;
  parties: string[];
  jurisdiction?: string;
  contractValue?: number;
  redFlags: string[];
}

interface Tier2Findings {
  activeAgents: string[];
  agentFindings: AgentFinding[];
  communicationLog: AgentMessage[];
  synthesis: SynthesisResult;
  totalTokensUsed: number;
}
```

---

## State Management

### Real-Time Analysis Tracking

Convex provides real-time subscriptions via WebSocket. The UI reactively updates as analysis progresses:

```typescript
// Frontend: Real-time subscription to analysis progress
const analysisProgress = useQuery(api.analyses.subscribeToProgress, {
  documentId,
});

// Returns reactive state:
// {
//   documentStatus: "tier1_processing" | "tier2_processing" | ...
//   currentStage: "clause_extraction" | "agents_running" | "synthesis"
//   progress: 0.65
//   activeAgents: ["risk_assessor", "financial_analyst"]
//   completedAgents: ["clause_extractor"]
// }
```

### Zustand Store (Client State)

```typescript
// store/analysis.ts
interface AnalysisStore {
  selectedDocumentId: Id<"documents"> | null;
  viewMode: "tier1" | "tier2" | "comparison";
  expandedAgents: Set<string>;
  setSelectedDocument: (id: Id<"documents">) => void;
  toggleAgentExpanded: (agentId: string) => void;
}
```

---

## Error Propagation

```
Level 1: AI Model Error (LLaMA inference fails)
  → Retry with smaller chunk size (3 attempts)
  → If still fails: Mark Tier 1 as failed, skip to Tier 2
  → Log error to analytics table

Level 2: Agent Error (Groq/Gemini API fails)
  → Retry with exponential backoff (max 3 attempts)
  → If single agent fails: Continue with remaining agents
  → If >50% agents fail: Mark Tier 2 as degraded
  → Synthesis agent works with available findings

Level 3: Synthesis Error
  → Fall back to confidence-weighted aggregation
  → Return partial results with warning flag

Level 4: System Error (Convex mutation fails)
  → Convex automatically retries mutations
  → Document status set to "failed"
  → User notified via contract_notifications table
```

---

## Scalability

### Horizontal Scaling

| Component | Scaling Strategy |
|-----------|-----------------|
| Frontend (Vercel) | Auto-scales edge nodes globally |
| Convex Backend | Auto-scales (serverless) |
| LLaMA Tier 1 | Multiple GPU instances behind load balancer |
| Groq Agents | API-based, no infrastructure to scale |
| Gemini Coordinator | API-based, free tier rate limits apply |

### Rate Limit Management

```typescript
// Groq free tier: 30 RPM per model
// Strategy: Round-robin across models + request queuing

const GROQ_RATE_LIMITS = {
  "llama-3.3-70b-versatile": { rpm: 30, rpd: 14400 },
  "deepseek-r1-distill-llama-70b": { rpm: 30, rpd: 14400 },
  "llama-3.1-8b-instant": { rpm: 30, rpd: 14400 },
  "mixtral-8x7b-32768": { rpm: 30, rpd: 14400 },
};

// Google Gemini free tier: 15 RPM
const GEMINI_RATE_LIMITS = {
  "gemini-2.0-flash": { rpm: 15, rpd: 1500 },
};
```

### Throughput Estimates

| Scenario | Tier 1 Only | Tier 1 + Tier 2 |
|----------|-------------|-----------------|
| Single contract | 5-8s | 45-90s |
| 10 contracts/hour | ✅ Easy | ✅ Within limits |
| 50 contracts/hour | ✅ Easy | ⚠️ Near Groq limits |
| 100+ contracts/hour | ✅ Easy | ❌ Exceeds free tier limits |

---

## Related Documentation

- [02_TIER1_LLAMAMODEL.md](02_TIER1_LLAMAMODEL.md) — LLaMA model details
- [03_TIER2_MULTIAGENT_CORE.md](03_TIER2_MULTIAGENT_CORE.md) — Multi-agent orchestration
- [06_DATABASE_SCHEMA.md](06_DATABASE_SCHEMA.md) — Convex schema
- [07_API_DESIGN.md](07_API_DESIGN.md) — API endpoints

## Revision History

| Version | Date | Changes |
|---------|------|---------|
| 2.0.0 | 2026-02-09 | Hybrid multi-agent architecture with free APIs |
| 1.0.0 | 2025-01-01 | Initial single-model architecture |
