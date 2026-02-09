# 13_CODE_STANDARDS

**Last Updated:** 2026-02-09  
**Version:** 2.0.0  
**Author:** LawBotics Team

---

## Table of Contents

1. [TypeScript Conventions](#typescript-conventions)
2. [React/Next.js Conventions](#reactnextjs-conventions)
3. [Convex Conventions](#convex-conventions)
4. [Python Conventions](#python-conventions)
5. [File & Folder Naming](#file--folder-naming)
6. [Git Workflow](#git-workflow)

---

## TypeScript Conventions

### General Rules

```typescript
// ✅ Use explicit types for function parameters and return values
function calculateComplexity(factors: ComplexityFactors): number { ... }

// ✅ Use interfaces for objects, types for unions
interface AgentConfig {
  id: string;
  model: GroqModel;
  provider: "groq" | "gemini";
}

type AnalysisTier = 1 | 2;
type AgentStatus = "running" | "completed" | "failed" | "timeout";

// ✅ Use const assertions for static config
const AGENT_TIMEOUT_MS = 30_000 as const;

// ✅ Use nullish coalescing and optional chaining
const score = tier1Results?.complexityScore ?? 0;

// ❌ Avoid `any` — use `unknown` + type guards
function parseResponse(data: unknown): AgentFinding {
  if (!isAgentFinding(data)) throw new Error("Invalid response");
  return data;
}
```

### Error Handling

```typescript
// ✅ Use typed errors
class AgentError extends Error {
  constructor(
    public agentId: string,
    public errorType: "timeout" | "rate_limit" | "parse_error" | "api_error",
    message: string
  ) {
    super(message);
    this.name = "AgentError";
  }
}

// ✅ Use try/catch with specific handling
try {
  const result = await callGroqAgent(request);
} catch (error) {
  if (error instanceof AgentError && error.errorType === "rate_limit") {
    await delay(60_000);  // Wait for rate limit reset
    return retry();
  }
  throw error;
}
```

### Imports

```typescript
// ✅ Order: external → internal → relative → types
import { v } from "convex/values";                    // External
import { internal } from "../_generated/api";          // Generated
import { callGroqAgent } from "../../lib/agents/groq"; // Internal
import type { AgentConfig } from "./types";            // Types (last)
```

---

## React/Next.js Conventions

### Component Structure

```tsx
// ✅ Standard component template
"use client";  // Only when needed (hooks, interactivity)

import { useState } from "react";
import { useQuery } from "convex/react";
import { api } from "../../../../convex/_generated/api";
import { cn } from "@/lib/utils";

interface AgentCardProps {
  agentId: string;
  status: "running" | "completed" | "failed";
  className?: string;
}

export function AgentCard({ agentId, status, className }: AgentCardProps) {
  // 1. Hooks first
  const [expanded, setExpanded] = useState(false);

  // 2. Derived state
  const isActive = status === "running";

  // 3. Event handlers
  const handleToggle = () => setExpanded((prev) => !prev);

  // 4. Render
  return (
    <div className={cn("rounded-lg border p-4", className)}>
      {/* ... */}
    </div>
  );
}
```

### Key Rules

- Server Components by default, `"use client"` only when necessary
- Use Shadcn/ui components from `@/components/ui/`
- Use `cn()` utility for conditional classNames
- Use Convex `useQuery()` for real-time data, `useMutation()` for writes
- Never fetch in `useEffect` — use Convex queries or Next.js data fetching

---

## Convex Conventions

### File Organization

```
convex/
├── schema.ts              # Single source of truth for DB schema
├── documents.ts           # Document queries + mutations
├── analyses.ts            # Legacy analysis operations
├── tier1Analyses.ts       # Tier 1 analysis operations
├── tier2Analyses.ts       # Tier 2 analysis operations
├── usageLogs.ts           # Usage tracking
├── users.ts               # User management
├── internal.ts            # Internal-only functions
├── http.ts                # HTTP routes (webhooks)
├── actions/
│   ├── analyzeDocument.ts # Main analysis action
│   ├── routeAnalysis.ts   # Tier routing action
│   ├── processDocument.ts # PDF processing
│   └── ...
└── agents/
    ├── orchestrator.ts    # Multi-agent orchestrator
    ├── providers.ts       # Groq + Gemini client setup
    └── prompts.ts         # Agent system prompts
```

### Naming Conventions

```typescript
// Queries: get*, list*, search*
export const getDocument = query({ ... });
export const listByUser = query({ ... });

// Mutations: create*, update*, delete*, patch*
export const createAnalysis = mutation({ ... });
export const updateStatus = mutation({ ... });

// Actions: run*, process*, analyze*
export const runMultiAgentAnalysis = action({ ... });
export const processDocument = action({ ... });

// Internal: use internalQuery, internalMutation, internalAction
export const logUsage = internalMutation({ ... });
```

### Validation

```typescript
// ✅ Always validate with Convex's v validator
args: {
  documentId: v.id("documents"),
  userId: v.string(),
  forceDeep: v.optional(v.boolean()),
}

// ✅ Use Zod for complex runtime validation of AI responses
const AgentResponseSchema = z.object({
  agent_id: z.string(),
  confidence: z.number().min(0).max(1),
  findings: z.record(z.unknown()),
});
```

---

## Python Conventions

### Model Training Code

```python
# ✅ Type hints
def train_model(
    dataset_path: str,
    output_dir: str,
    epochs: int = 3,
    learning_rate: float = 2e-4,
) -> dict[str, float]:
    """Train LLaMA model with LoRA on CUAD dataset.
    
    Args:
        dataset_path: Path to preprocessed CUAD data
        output_dir: Path to save trained model
        epochs: Training epochs (default 3)
        learning_rate: Peak learning rate (default 2e-4)
    
    Returns:
        Dictionary of training metrics
    """
    ...

# ✅ Use dataclasses for structured data
@dataclass
class TrainingConfig:
    model_name: str = "meta-llama/Llama-3.2-3B-Instruct"
    lora_r: int = 16
    lora_alpha: int = 32
    batch_size: int = 4
    max_seq_length: int = 4096
    
# ✅ Use pytest for testing
def test_clause_extraction_accuracy():
    assert accuracy >= 0.93
```

---

## File & Folder Naming

| Type | Convention | Example |
|------|-----------|---------|
| React components | PascalCase | `AgentProgress.tsx` |
| Hooks | camelCase with `use` prefix | `useAnalysis.ts` |
| Utilities | camelCase | `parseResponse.ts` |
| Convex functions | camelCase | `analyzeDocument.ts` |
| Constants | UPPER_SNAKE_CASE | `MAX_RETRIES` |
| CSS modules | kebab-case | `agent-card.module.css` |
| Docs | UPPER_SNAKE with numeric prefix | `03_TIER2_MULTIAGENT_CORE.md` |
| Python | snake_case | `fine_tuning_code.py` |

---

## Git Workflow

### Branch Naming

```
feature/multi-agent-orchestrator
fix/groq-rate-limit-handling
docs/update-agent-specs
refactor/complexity-scoring
```

### Commit Messages

```
feat: add Groq multi-agent orchestrator
fix: handle Groq 429 rate limit with exponential backoff
docs: update agent specifications for free tier models
test: add conflict detection unit tests
refactor: extract agent provider into shared module
```

### PR Template

```markdown
## Summary
Brief description of changes.

## Type
- [ ] Feature
- [ ] Bug fix
- [ ] Documentation
- [ ] Refactor

## Testing
- [ ] Unit tests pass
- [ ] Agent accuracy benchmarks unchanged
- [ ] Rate limits tested
```

---

## Related Documentation

- [11_TESTING_STRATEGY.md](11_TESTING_STRATEGY.md) — Testing standards
- [14_SECURITY.md](14_SECURITY.md) — Security coding practices

## Revision History

| Version | Date | Changes |
|---------|------|---------|
| 2.0.0 | 2026-02-09 | Convex conventions, agent code standards |
| 1.0.0 | 2025-01-01 | Initial code standards |
