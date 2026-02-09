# 03_TIER2_MULTIAGENT_CORE

**Last Updated:** 2026-02-09  
**Version:** 2.0.0  
**Author:** LawBotics Team

---

## Table of Contents

1. [Agent Orchestrator](#agent-orchestrator)
2. [Free AI Provider Integration](#free-ai-provider-integration)
3. [Coordinator Logic](#coordinator-logic)
4. [Parallel Execution](#parallel-execution)
5. [Peer Discussion](#peer-discussion)
6. [Conflict Resolution](#conflict-resolution)
7. [Synthesis](#synthesis)
8. [Timeout Handling](#timeout-handling)
9. [Partial Result Streaming](#partial-result-streaming)

---

## Agent Orchestrator

The orchestrator manages the lifecycle of a Tier 2 multi-agent analysis: agent selection → parallel execution → peer discussion → synthesis.

### Agent Model Assignment (Free Tiers)

| Agent Role | Provider | Model | Why |
|-----------|----------|-------|-----|
| Clause Extraction | Groq | `llama-3.1-8b-instant` | Fast, lightweight task |
| Risk Assessment | Groq | `deepseek-r1-distill-llama-70b` | Strong legal reasoning |
| Financial Terms | Groq | `llama-3.3-70b-versatile` | Structured analysis |
| IP/Licensing | Groq | `llama-3.3-70b-versatile` | Broad legal knowledge |
| Employment Specialist | Groq | `mixtral-8x7b-32768` | Good domain context |
| M&A Specialist | Groq | `mixtral-8x7b-32768` | Good domain context |
| SaaS/Tech Specialist | Groq | `mixtral-8x7b-32768` | Good domain context |
| Compliance Checker | Groq | `llama-3.3-70b-versatile` | Regulatory knowledge |
| Coordinator | Google Gemini | `gemini-2.0-flash` | Already integrated, strong orchestration |
| Synthesis | Google Gemini | `gemini-2.0-flash` | Best free reasoning |

---

## Free AI Provider Integration

### Groq Client Setup

```typescript
// lib/agents/groq-client.ts

import Groq from "groq-sdk";

const groq = new Groq({
  apiKey: process.env.GROQ_API_KEY,  // Free API key from console.groq.com
});

interface AgentRequest {
  model: GroqModel;
  systemPrompt: string;
  userMessage: string;
  maxTokens?: number;
  temperature?: number;
  responseFormat?: "json_object" | "text";
}

type GroqModel =
  | "llama-3.3-70b-versatile"
  | "deepseek-r1-distill-llama-70b"
  | "llama-3.1-8b-instant"
  | "mixtral-8x7b-32768";

export async function callGroqAgent(request: AgentRequest): Promise<string> {
  const response = await groq.chat.completions.create({
    model: request.model,
    messages: [
      { role: "system", content: request.systemPrompt },
      { role: "user", content: request.userMessage },
    ],
    max_tokens: request.maxTokens ?? 4096,
    temperature: request.temperature ?? 0.2,
    response_format: request.responseFormat
      ? { type: request.responseFormat }
      : undefined,
  });

  return response.choices[0]?.message?.content ?? "";
}

// Rate limit tracking
const requestCounts = new Map<string, { count: number; resetAt: number }>();

export async function callGroqWithRateLimit(
  request: AgentRequest
): Promise<string> {
  const key = request.model;
  const now = Date.now();
  const limit = requestCounts.get(key);

  if (limit && now < limit.resetAt && limit.count >= 28) {
    // Near rate limit (30 RPM), wait until reset
    const waitMs = limit.resetAt - now;
    await new Promise((resolve) => setTimeout(resolve, waitMs));
  }

  // Update counter
  if (!limit || now >= limit.resetAt) {
    requestCounts.set(key, { count: 1, resetAt: now + 60000 });
  } else {
    limit.count++;
  }

  return callGroqAgent(request);
}
```

### Google Gemini Client (Existing)

```typescript
// lib/agents/gemini-client.ts

import { GoogleGenerativeAI } from "@google/generative-ai";

const genAI = new GoogleGenerativeAI(process.env.GOOGLE_API_KEY!);

interface GeminiAgentRequest {
  systemPrompt: string;
  userMessage: string;
  maxTokens?: number;
}

export async function callGeminiAgent(
  request: GeminiAgentRequest
): Promise<string> {
  const model = genAI.getGenerativeModel({
    model: "gemini-2.0-flash",
    systemInstruction: request.systemPrompt,
  });

  const result = await model.generateContent(request.userMessage);
  const response = await result.response;
  return response.text();
}
```

---

## Coordinator Logic

```typescript
// convex/agents/orchestrator.ts

"use node";

import { action } from "../_generated/server";
import { v } from "convex/values";
import { internal } from "../_generated/api";
import { callGroqWithRateLimit } from "../../lib/agents/groq-client";
import { callGeminiAgent } from "../../lib/agents/gemini-client";

interface AgentConfig {
  id: string;
  name: string;
  provider: "groq" | "gemini";
  model: string;
  systemPrompt: string;
  activationConditions: (context: ContractContext) => boolean;
  tokenBudget: { input: number; output: number };
}

interface ContractContext {
  contractType: string;
  clauses: any[];
  complexityScore: number;
  hasFinancialTerms: boolean;
  hasIPTerms: boolean;
  hasEmploymentTerms: boolean;
  hasMergerTerms: boolean;
  hasSaaSTerms: boolean;
  hasComplianceTerms: boolean;
  contractValue: number | null;
  jurisdiction: string | null;
}

const AGENTS: Record<string, AgentConfig> = {
  clause_extractor: {
    id: "clause_extractor",
    name: "Clause Extraction Agent",
    provider: "groq",
    model: "llama-3.1-8b-instant",
    systemPrompt: CLAUSE_EXTRACTION_PROMPT,
    activationConditions: () => true,  // Always active for Tier 2
    tokenBudget: { input: 4000, output: 2000 },
  },
  risk_assessor: {
    id: "risk_assessor",
    name: "Risk Assessment Agent",
    provider: "groq",
    model: "deepseek-r1-distill-llama-70b",
    systemPrompt: RISK_ASSESSMENT_PROMPT,
    activationConditions: () => true,  // Always active
    tokenBudget: { input: 6000, output: 3000 },
  },
  financial_analyst: {
    id: "financial_analyst",
    name: "Financial Terms Agent",
    provider: "groq",
    model: "llama-3.3-70b-versatile",
    systemPrompt: FINANCIAL_ANALYSIS_PROMPT,
    activationConditions: (ctx) => ctx.hasFinancialTerms || (ctx.contractValue ?? 0) > 100000,
    tokenBudget: { input: 5000, output: 2500 },
  },
  ip_specialist: {
    id: "ip_specialist",
    name: "IP/Licensing Agent",
    provider: "groq",
    model: "llama-3.3-70b-versatile",
    systemPrompt: IP_LICENSING_PROMPT,
    activationConditions: (ctx) => ctx.hasIPTerms,
    tokenBudget: { input: 5000, output: 2500 },
  },
  employment_specialist: {
    id: "employment_specialist",
    name: "Employment Specialist Agent",
    provider: "groq",
    model: "mixtral-8x7b-32768",
    systemPrompt: EMPLOYMENT_PROMPT,
    activationConditions: (ctx) => ctx.hasEmploymentTerms,
    tokenBudget: { input: 5000, output: 2000 },
  },
  ma_specialist: {
    id: "ma_specialist",
    name: "M&A Specialist Agent",
    provider: "groq",
    model: "mixtral-8x7b-32768",
    systemPrompt: MA_SPECIALIST_PROMPT,
    activationConditions: (ctx) => ctx.hasMergerTerms,
    tokenBudget: { input: 5000, output: 2000 },
  },
  compliance_checker: {
    id: "compliance_checker",
    name: "Compliance Agent",
    provider: "groq",
    model: "llama-3.3-70b-versatile",
    systemPrompt: COMPLIANCE_PROMPT,
    activationConditions: (ctx) => ctx.hasComplianceTerms,
    tokenBudget: { input: 5000, output: 2000 },
  },
};

export const runMultiAgentAnalysis = action({
  args: {
    documentId: v.id("documents"),
    tier1Results: v.any(),
  },
  handler: async (ctx, args) => {
    const startTime = Date.now();

    // 1. Build contract context from Tier 1 results
    const contractContext = buildContractContext(args.tier1Results);

    // 2. Determine active agents
    const activeAgents = Object.values(AGENTS).filter(
      (agent) => agent.activationConditions(contractContext)
    );

    console.log(
      `Tier 2: Activating ${activeAgents.length} agents:`,
      activeAgents.map((a) => a.id)
    );

    // 3. Update status
    await ctx.runMutation(internal.analyses.updateTier2Status, {
      documentId: args.documentId,
      status: "agents_running",
      activeAgents: activeAgents.map((a) => a.id),
    });

    // 4. Run agents in parallel (respecting rate limits)
    const agentFindings = await executeAgentsParallel(
      activeAgents,
      contractContext,
      args.tier1Results
    );

    // 5. Update status
    await ctx.runMutation(internal.analyses.updateTier2Status, {
      documentId: args.documentId,
      status: "peer_discussion",
    });

    // 6. Peer discussion rounds (max 2)
    const communicationLog = await runPeerDiscussion(
      agentFindings,
      contractContext
    );

    // 7. Update status
    await ctx.runMutation(internal.analyses.updateTier2Status, {
      documentId: args.documentId,
      status: "synthesis",
    });

    // 8. Synthesis (Gemini 2.0 Flash)
    const synthesis = await synthesizeFindings(
      agentFindings,
      communicationLog,
      contractContext
    );

    // 9. Store final results
    const totalTime = Date.now() - startTime;
    await ctx.runMutation(internal.analyses.completeTier2, {
      documentId: args.documentId,
      agentFindings,
      communicationLog,
      synthesis,
      totalProcessingTimeMs: totalTime,
      totalCost: 0,  // Free tier = $0
    });

    return synthesis;
  },
});
```

---

## Parallel Execution

```typescript
async function executeAgentsParallel(
  agents: AgentConfig[],
  context: ContractContext,
  tier1Results: any
): Promise<AgentFinding[]> {
  // Split into batches of 3 to respect Groq rate limits
  const BATCH_SIZE = 3;
  const findings: AgentFinding[] = [];

  for (let i = 0; i < agents.length; i += BATCH_SIZE) {
    const batch = agents.slice(i, i + BATCH_SIZE);
    
    const batchResults = await Promise.allSettled(
      batch.map(async (agent) => {
        const start = Date.now();
        
        const userMessage = buildAgentPrompt(agent.id, context, tier1Results);
        
        let response: string;
        
        if (agent.provider === "groq") {
          response = await callGroqWithRateLimit({
            model: agent.model as any,
            systemPrompt: agent.systemPrompt,
            userMessage,
            maxTokens: agent.tokenBudget.output,
            temperature: 0.2,
            responseFormat: "json_object",
          });
        } else {
          response = await callGeminiAgent({
            systemPrompt: agent.systemPrompt,
            userMessage,
            maxTokens: agent.tokenBudget.output,
          });
        }

        const parsed = parseAgentResponse(response);

        return {
          agentId: agent.id,
          model: agent.model,
          provider: agent.provider,
          findings: parsed,
          confidence: parsed.confidence ?? 0.8,
          processingTimeMs: Date.now() - start,
          tokenUsage: {
            input: estimateTokens(userMessage),
            output: estimateTokens(response),
          },
          completedAt: Date.now(),
        } as AgentFinding;
      })
    );

    // Collect successful results, log failures
    for (const result of batchResults) {
      if (result.status === "fulfilled") {
        findings.push(result.value);
      } else {
        console.error("Agent failed:", result.reason);
        // Continue with remaining agents
      }
    }

    // Small delay between batches for rate limiting
    if (i + BATCH_SIZE < agents.length) {
      await new Promise((r) => setTimeout(r, 2000));
    }
  }

  return findings;
}
```

---

## Peer Discussion

Inter-agent communication resolves conflicts and validates findings across 1-2 rounds.

```typescript
async function runPeerDiscussion(
  findings: AgentFinding[],
  context: ContractContext
): Promise<AgentMessage[]> {
  const messages: AgentMessage[] = [];
  const maxRounds = 2;

  for (let round = 0; round < maxRounds; round++) {
    // Detect conflicts between agent findings
    const conflicts = detectConflicts(findings);

    if (conflicts.length === 0) {
      console.log(`Round ${round + 1}: No conflicts, consensus reached`);
      break;
    }

    console.log(
      `Round ${round + 1}: ${conflicts.length} conflicts detected`
    );

    // Ask conflicting agents to reconcile
    for (const conflict of conflicts) {
      const clarifications = await Promise.all(
        conflict.involvedAgents.map(async (agentId) => {
          const agent = AGENTS[agentId];
          if (!agent) return null;

          const prompt = `
Another agent found: "${conflict.counterFinding}"
Your finding was: "${conflict.yourFinding}"

Please clarify or revise your position. If you disagree, explain why 
with evidence from the contract text. Respond in JSON:
{
  "revised_position": "your updated or maintained position",
  "confidence": 0.0-1.0,
  "reasoning": "why you hold this position",
  "agrees_with_peer": true/false
}`;

          const response = await callGroqWithRateLimit({
            model: agent.model as any,
            systemPrompt: `${agent.systemPrompt}\n\nYou are in peer review mode.`,
            userMessage: prompt,
            maxTokens: 1000,
            responseFormat: "json_object",
          });

          return {
            round: round + 1,
            sender: agentId,
            recipients: conflict.involvedAgents.filter((id) => id !== agentId),
            messageType: "clarification" as const,
            content: response,
            timestamp: Date.now(),
          };
        })
      );

      messages.push(
        ...(clarifications.filter(Boolean) as AgentMessage[])
      );
    }
  }

  return messages;
}

function detectConflicts(findings: AgentFinding[]): Conflict[] {
  const conflicts: Conflict[] = [];

  // Compare risk scores across agents
  const riskFindings = findings
    .filter((f) => f.findings?.overall_risk_score !== undefined)
    .map((f) => ({
      agentId: f.agentId,
      score: f.findings.overall_risk_score,
    }));

  if (riskFindings.length >= 2) {
    const scores = riskFindings.map((r) => r.score);
    const spread = Math.max(...scores) - Math.min(...scores);

    // Score spread > 20 points = conflict
    if (spread > 20) {
      conflicts.push({
        type: "risk_score_disagreement",
        involvedAgents: riskFindings.map((r) => r.agentId),
        yourFinding: `Risk score: ${scores[0]}`,
        counterFinding: `Risk score: ${scores[1]}`,
      });
    }
  }

  // Compare clause detection findings
  const clauseAgentFindings = findings.find(
    (f) => f.agentId === "clause_extractor"
  );
  const riskAgentFindings = findings.find(
    (f) => f.agentId === "risk_assessor"
  );

  if (clauseAgentFindings && riskAgentFindings) {
    // Check if risk agent flags clauses that clause agent missed
    const riskClauses = riskAgentFindings.findings?.affected_clauses ?? [];
    const extractedClauses =
      clauseAgentFindings.findings?.clauses?.map((c: any) => c.type) ?? [];

    for (const riskClause of riskClauses) {
      if (!extractedClauses.includes(riskClause)) {
        conflicts.push({
          type: "missing_clause_disagreement",
          involvedAgents: ["clause_extractor", "risk_assessor"],
          yourFinding: `Clause "${riskClause}" not detected`,
          counterFinding: `Clause "${riskClause}" flagged as risk`,
        });
      }
    }
  }

  return conflicts;
}
```

---

## Synthesis

The Synthesis Agent (Gemini 2.0 Flash) produces the final comprehensive report.

```typescript
async function synthesizeFindings(
  agentFindings: AgentFinding[],
  communicationLog: AgentMessage[],
  context: ContractContext
): Promise<SynthesisResult> {
  const synthesisPrompt = `
You are the Synthesis Agent for a legal contract analysis system.

You have received findings from ${agentFindings.length} specialized agents.
Your job is to produce a final, unified analysis that:
1. Reconciles any conflicting findings
2. Prioritizes issues by severity and business impact
3. Provides a clear recommendation
4. Includes negotiation strategies

AGENT FINDINGS:
${agentFindings.map((f) => `
--- ${f.agentId} (confidence: ${f.confidence}) ---
${JSON.stringify(f.findings, null, 2)}
`).join("\n")}

PEER DISCUSSION LOG:
${communicationLog.map((m) => `
[${m.sender} → ${m.recipients.join(", ")}]: ${m.content}
`).join("\n")}

CONTRACT CONTEXT:
- Type: ${context.contractType}
- Complexity Score: ${context.complexityScore}/10
- Value: ${context.contractValue ?? "Not specified"}
- Jurisdiction: ${context.jurisdiction ?? "Not specified"}

Return a JSON object with this structure:
{
  "executiveSummary": "2-3 paragraph summary for a CEO",
  "overallRiskScore": 0-100,
  "riskLevel": "LOW" | "MEDIUM" | "HIGH" | "CRITICAL",
  "priorityIssues": [
    {
      "rank": 1,
      "issue": "clear issue description",
      "agentsFlagged": ["agent_ids"],
      "severity": "critical/high/medium/low",
      "recommendation": "specific action to take",
      "negotiationStrategy": "how to negotiate this term"
    }
  ],
  "recommendation": "SIGN_AS_IS" | "SIGN_WITH_MINOR_REVISIONS" | "NEGOTIATE_KEY_TERMS" | "DO_NOT_SIGN",
  "confidence": 0.0-1.0
}`;

  const response = await callGeminiAgent({
    systemPrompt:
      "You are an expert legal synthesis agent. You unify findings from multiple specialized legal AI agents into a coherent, actionable report. Always respond in valid JSON.",
    userMessage: synthesisPrompt,
  });

  const parsed = JSON.parse(
    response.match(/\{[\s\S]*\}/)?.[0] ?? "{}"
  );

  return {
    executiveSummary: parsed.executiveSummary ?? "",
    overallRiskScore: parsed.overallRiskScore ?? 50,
    riskLevel: parsed.riskLevel ?? "MEDIUM",
    priorityIssues: parsed.priorityIssues ?? [],
    recommendation: parsed.recommendation ?? "NEGOTIATE_KEY_TERMS",
    confidence: parsed.confidence ?? 0.7,
  };
}
```

---

## Timeout Handling

```typescript
const AGENT_TIMEOUT_MS = 30000;  // 30 seconds per agent

async function callAgentWithTimeout<T>(
  agentFn: () => Promise<T>,
  agentId: string
): Promise<T | null> {
  try {
    const result = await Promise.race([
      agentFn(),
      new Promise<never>((_, reject) =>
        setTimeout(
          () => reject(new Error(`Agent ${agentId} timed out`)),
          AGENT_TIMEOUT_MS
        )
      ),
    ]);
    return result;
  } catch (error) {
    console.error(`Agent ${agentId} error:`, error);
    return null;  // Skip this agent, continue with others
  }
}
```

---

## Partial Result Streaming

Convex mutations update analysis state in real-time:

```typescript
// As each agent completes, update the DB (streams to frontend via WebSocket)
for (const finding of agentFindings) {
  await ctx.runMutation(internal.analyses.appendAgentFinding, {
    documentId: args.documentId,
    finding: {
      agentId: finding.agentId,
      status: "complete",
      summary: finding.findings?.executive_summary ?? "",
      confidence: finding.confidence,
      processingTimeMs: finding.processingTimeMs,
    },
  });
}
```

---

## Related Documentation

- [04_AGENT_SPECIFICATIONS.md](04_AGENT_SPECIFICATIONS.md) — Individual agent specs
- [08_AGENT_COMMUNICATION.md](08_AGENT_COMMUNICATION.md) — Message protocol
- [10_COST_OPTIMIZATION.md](10_COST_OPTIMIZATION.md) — Token budgets

## Revision History

| Version | Date | Changes |
|---------|------|---------|
| 2.0.0 | 2026-02-09 | Free API agents (Groq + Gemini) replacing Claude |
| 1.0.0 | 2025-01-01 | Single-model Google Gemini analysis |
