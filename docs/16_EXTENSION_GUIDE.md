# 16_EXTENSION_GUIDE

**Last Updated:** 2026-02-09  
**Version:** 2.0.0  
**Author:** LawBotics Team

---

## Table of Contents

1. [Adding a New Agent](#adding-a-new-agent)
2. [Adding New Clause Types](#adding-new-clause-types)
3. [Adding New Contract Types](#adding-new-contract-types)
4. [Adding a New AI Provider](#adding-a-new-ai-provider)
5. [Upgrading Models](#upgrading-models)

---

## Adding a New Agent

### Step 1: Define Agent Configuration

```typescript
// convex/agents/orchestrator.ts — add to AGENTS record

const AGENTS: Record<string, AgentConfig> = {
  // ... existing agents ...

  real_estate_specialist: {
    id: "real_estate_specialist",
    name: "Real Estate Specialist Agent",
    provider: "groq",
    model: "mixtral-8x7b-32768",
    systemPrompt: REAL_ESTATE_PROMPT,
    activationConditions: (ctx) =>
      ctx.contractType === "lease" ||
      ctx.clauses.some((c: any) =>
        ["Lease Terms", "Property Rights"].includes(c.type)
      ),
    tokenBudget: { input: 5000, output: 2000 },
  },
};
```

### Step 2: Write System Prompt

```typescript
// convex/agents/prompts.ts

export const REAL_ESTATE_PROMPT = `
You are a real estate contract specialist. Analyze lease agreements, 
property purchase contracts, and real estate-related terms.

RESPONSIBILITIES:
1. Evaluate lease terms, rent escalation, and renewal options
2. Assess property condition requirements and maintenance obligations  
3. Review zoning and land use restrictions
4. Identify tenant improvement allowances and build-out terms
5. Check insurance and liability provisions specific to real estate

OUTPUT FORMAT (strict JSON):
{
  "agent_id": "real_estate_specialist",
  "lease_analysis": { ... },
  "property_risks": [ ... ],
  "recommendations": [ ... ],
  "confidence": 0.0-1.0
}
`;
```

### Step 3: Add Activation Detection

```typescript
// Update buildContractContext in orchestrator.ts
hasRealEstateTerms: clauseTypes.some((t: string) =>
  ["Lease Terms", "Property Rights", "Zoning Restrictions"].includes(t)
) || context.contractType === "lease",
```

### Step 4: Update Frontend Labels

```typescript
// src/components/feature-pages/analysis/AgentProgress.tsx
const AGENT_LABELS: Record<string, { label: string; icon: string }> = {
  // ... existing ...
  real_estate_specialist: { label: "Real Estate Analysis", icon: "🏢" },
};
```

### Step 5: Test

```python
# tests/agent_accuracy/test_real_estate_agent.py
def test_lease_term_extraction(real_estate_agent, lease_contracts):
    for contract in lease_contracts:
        result = real_estate_agent.analyze(contract["text"])
        assert result["lease_analysis"] is not None
        assert result["confidence"] >= 0.7
```

### Checklist

- [ ] Agent config added to AGENTS record
- [ ] System prompt written and tested
- [ ] Activation conditions defined
- [ ] Frontend label and icon added
- [ ] Agent accuracy tests written
- [ ] Documentation updated

---

## Adding New Clause Types

### Step 1: Add to CUAD Type List

```python
# ai-model/training/clause_types.py

CLAUSE_TYPES = [
    # ... existing 41 CUAD types ...
    "Force Majeure",           # New clause type
    "Data Processing Terms",    # New clause type
    "Environmental Liability",  # New clause type
]
```

### Step 2: Prepare Training Data

```python
# Label new clause types in training data
# Format: same as CUAD annotation structure
new_annotations = [
    {
        "contract_id": "custom_001",
        "clause_type": "Force Majeure",
        "text": "Neither party shall be liable for...",
        "present": True,
        "section": "Section 12.1"
    },
]
```

### Step 3: Retrain LLaMA Model

```bash
# Fine-tune with expanded clause types
python train.py \
  --base_model="meta-llama/Llama-3.2-3B-Instruct" \
  --data_path="./data/cuad_extended.json" \
  --output_dir="./models/v2.0" \
  --clause_types=44  # Updated count
```

### Step 4: Update Agent Prompts

Add the new clause types to agent system prompts, particularly the Clause Extraction Agent.

### Step 5: Update Schema (if needed)

```typescript
// Add to contract clause type validation if applicable
clauses: v.array(v.object({
  title: v.string(),
  content: v.string(),
  type: v.string(),  // Now supports 44 clause types
})),
```

---

## Adding New Contract Types

### Step 1: Update Schema

```typescript
// convex/schema.ts — update contractType union
contractType: v.union(
  v.literal("service"),
  v.literal("nda"),
  v.literal("employment"),
  v.literal("license"),
  v.literal("partnership"),
  v.literal("vendor"),
  v.literal("lease"),
  v.literal("consulting"),
  v.literal("distribution"),
  v.literal("real_estate"),     // New
  v.literal("insurance"),       // New
  v.literal("other"),
),
```

### Step 2: Add Agent Activation Rules

```typescript
// Update activation conditions for relevant agents
const ACTIVATION_BY_CONTRACT_TYPE: Record<string, string[]> = {
  // ... existing ...
  real_estate: ["clause_extractor", "risk_assessor", "financial_analyst", "real_estate_specialist", "compliance_checker"],
  insurance: ["clause_extractor", "risk_assessor", "financial_analyst", "compliance_checker"],
};
```

### Step 3: Update Frontend Contract Creation

```typescript
// Add to contract type dropdown options
const CONTRACT_TYPES = [
  // ... existing ...
  { value: "real_estate", label: "Real Estate" },
  { value: "insurance", label: "Insurance" },
];
```

---

## Adding a New AI Provider

### Step 1: Create Provider Client

```typescript
// lib/agents/new-provider-client.ts

import OpenAI from "openai";  // Most free providers support OpenAI format

export function createProviderClient(apiKey: string, baseURL: string) {
  return new OpenAI({
    apiKey,
    baseURL,  // e.g., "https://api.together.xyz/v1"
  });
}

export async function callNewProvider(
  client: OpenAI,
  model: string,
  systemPrompt: string,
  userMessage: string
): Promise<string> {
  const response = await client.chat.completions.create({
    model,
    messages: [
      { role: "system", content: systemPrompt },
      { role: "user", content: userMessage },
    ],
    max_tokens: 4096,
    temperature: 0.2,
  });
  return response.choices[0]?.message?.content ?? "";
}
```

### Step 2: Register in Orchestrator

```typescript
// Add provider type
type Provider = "groq" | "gemini" | "together";

// Add to agent config
const newAgent: AgentConfig = {
  id: "new_agent",
  provider: "together",       // New provider
  model: "meta-llama/Llama-3.3-70B-Instruct-Turbo",
  // ...
};
```

### Step 3: Add Rate Limit Config

```typescript
const RATE_LIMITS: Record<string, { rpm: number; rpd: number }> = {
  groq: { rpm: 30, rpd: 14400 },
  gemini: { rpm: 15, rpd: 1500 },
  together: { rpm: 60, rpd: 10000 },  // New provider limits
};
```

### Free Provider Options

| Provider | Free Tier | Best Models | Sign Up |
|----------|-----------|-------------|---------|
| **Groq** | 30 RPM, 14.4K RPD | Llama 3.3-70B, DeepSeek-R1 | console.groq.com |
| **Google Gemini** | 15 RPM, 1.5K RPD | Gemini 2.0 Flash | ai.google.dev |
| **Together AI** | $1 free credit | Llama 3.3-70B-Turbo | together.ai |
| **Ollama** | Unlimited (Google Colab) | Any GGUF model | ollama.com |

---

## Upgrading Models

### When to Upgrade

| Signal | Action |
|--------|--------|
| New LLaMA version released | Retrain Tier 1 with new base |
| Groq adds new model | Test and swap if better |
| Agent accuracy drops below 90% | Investigate and retrain |
| New free provider appears | Evaluate and add |

### Model Swap Process

```typescript
// 1. Update model in agent config
const risk_assessor = {
  ...AGENTS.risk_assessor,
  model: "llama-4-70b-versatile",  // New model
};

// 2. Run benchmark suite
// npm run test:benchmarks

// 3. Compare results
// If accuracy >= previous: deploy
// If accuracy < previous: keep old model

// 4. Update docs
```

---

## Related Documentation

- [04_AGENT_SPECIFICATIONS.md](04_AGENT_SPECIFICATIONS.md) — Existing agent specs
- [02_TIER1_LLAMAMODEL.md](02_TIER1_LLAMAMODEL.md) — LLaMA training pipeline
- [06_DATABASE_SCHEMA.md](06_DATABASE_SCHEMA.md) — Schema modification guide

## Revision History

| Version | Date | Changes |
|---------|------|---------|
| 2.0.0 | 2026-02-09 | Extension guide for hybrid multi-agent system |
| 1.0.0 | 2025-01-01 | Initial extension documentation |
