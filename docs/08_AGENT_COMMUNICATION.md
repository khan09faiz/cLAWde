# 08_AGENT_COMMUNICATION

**Last Updated:** 2026-02-09  
**Version:** 2.0.0  
**Author:** LawBotics Team

---

## Table of Contents

1. [Communication Protocol](#communication-protocol)
2. [Message Schema](#message-schema)
3. [Communication Flow](#communication-flow)
4. [Conflict Detection](#conflict-detection)
5. [Consensus Algorithm](#consensus-algorithm)
6. [Communication Limits](#communication-limits)

---

## Communication Protocol

Agent communication follows a structured protocol:

1. **Phase 1 — Independent Analysis:** Agents analyze in parallel with no communication
2. **Phase 2 — Finding Broadcast:** Each agent publishes findings
3. **Phase 3 — Conflict Detection:** Coordinator identifies disagreements
4. **Phase 4 — Peer Discussion:** Conflicting agents exchange clarifications (max 2 rounds)
5. **Phase 5 — Consensus/Vote:** Remaining conflicts resolved by confidence-weighted voting

```
Phase 1          Phase 2          Phase 3          Phase 4          Phase 5
┌──────┐        ┌──────┐        ┌──────┐        ┌──────┐        ┌──────┐
│ Solo │───────▶│Broad-│───────▶│Detect│───────▶│Peer  │───────▶│Vote/ │
│ Work │        │ cast │        │Clash │        │Talk  │        │Merge │
└──────┘        └──────┘        └──────┘        └──────┘        └──────┘
```

---

## Message Schema

```typescript
interface AgentMessage {
  id: string;                    // Unique message ID
  round: number;                 // Discussion round (1 or 2)
  sender: string;                // Agent ID
  recipients: string[];          // Target agent IDs
  messageType: MessageType;
  content: string;               // JSON-serialized content
  timestamp: number;
  replyTo?: string;              // ID of message being responded to
}

type MessageType =
  | "finding"           // Initial finding broadcast
  | "question"          // Question to another agent
  | "clarification"     // Response to a question
  | "agreement"         // Agent agrees with peer's finding
  | "disagreement"      // Agent disagrees with reasoning
  | "revision"          // Agent revises its own finding
  | "evidence"          // Agent provides supporting evidence

interface Conflict {
  id: string;
  type: ConflictType;
  involvedAgents: string[];
  topic: string;
  positions: Array<{
    agentId: string;
    position: string;
    confidence: number;
  }>;
  resolved: boolean;
  resolution?: string;
}

type ConflictType =
  | "risk_score_disagreement"     // >20 point spread
  | "clause_presence_disagreement" // One found, one didn't
  | "severity_disagreement"       // Different severity for same issue
  | "recommendation_conflict"     // Opposing recommendations
  | "factual_disagreement";       // Different factual interpretations
```

---

## Communication Flow

### Phase 2 — Finding Broadcast

```typescript
function broadcastFindings(findings: AgentFinding[]): AgentMessage[] {
  return findings.map((f) => ({
    id: generateId(),
    round: 0,
    sender: f.agentId,
    recipients: ["coordinator"],
    messageType: "finding" as const,
    content: JSON.stringify({
      summary: f.findings?.executive_summary ?? "",
      riskScore: f.findings?.overall_risk_score,
      criticalIssues: f.findings?.critical_findings?.length ?? 0,
      confidence: f.confidence,
    }),
    timestamp: Date.now(),
  }));
}
```

### Phase 3 — Conflict Detection

```typescript
function detectAllConflicts(findings: AgentFinding[]): Conflict[] {
  const conflicts: Conflict[] = [];

  // 1. Risk score spread > 20 points
  const riskScores = findings
    .filter((f) => f.findings?.overall_risk_score !== undefined)
    .map((f) => ({
      agentId: f.agentId,
      score: f.findings.overall_risk_score,
      confidence: f.confidence,
    }));

  if (riskScores.length >= 2) {
    const max = Math.max(...riskScores.map((r) => r.score));
    const min = Math.min(...riskScores.map((r) => r.score));
    if (max - min > 20) {
      conflicts.push({
        id: generateId(),
        type: "risk_score_disagreement",
        involvedAgents: riskScores.map((r) => r.agentId),
        topic: `Risk score spread: ${min}-${max}`,
        positions: riskScores.map((r) => ({
          agentId: r.agentId,
          position: `Risk score: ${r.score}`,
          confidence: r.confidence,
        })),
        resolved: false,
      });
    }
  }

  // 2. Clause presence disagreements
  const clauseAgent = findings.find((f) => f.agentId === "clause_extractor");
  if (clauseAgent) {
    for (const otherAgent of findings) {
      if (otherAgent.agentId === "clause_extractor") continue;
      const referenced = otherAgent.findings?.affected_clauses ?? [];
      const extracted = clauseAgent.findings?.clauses
        ?.filter((c: any) => c.present)
        .map((c: any) => c.type) ?? [];

      for (const clause of referenced) {
        if (!extracted.includes(clause)) {
          conflicts.push({
            id: generateId(),
            type: "clause_presence_disagreement",
            involvedAgents: ["clause_extractor", otherAgent.agentId],
            topic: `Clause "${clause}" presence`,
            positions: [
              { agentId: "clause_extractor", position: "not found", confidence: clauseAgent.confidence },
              { agentId: otherAgent.agentId, position: "referenced as present", confidence: otherAgent.confidence },
            ],
            resolved: false,
          });
        }
      }
    }
  }

  // 3. Recommendation conflicts
  const recommendations = findings
    .filter((f) => f.findings?.recommendation)
    .map((f) => ({
      agentId: f.agentId,
      rec: f.findings.recommendation,
      confidence: f.confidence,
    }));

  const uniqueRecs = new Set(recommendations.map((r) => r.rec));
  if (uniqueRecs.size > 1) {
    conflicts.push({
      id: generateId(),
      type: "recommendation_conflict",
      involvedAgents: recommendations.map((r) => r.agentId),
      topic: "Different signing recommendations",
      positions: recommendations.map((r) => ({
        agentId: r.agentId,
        position: r.rec,
        confidence: r.confidence,
      })),
      resolved: false,
    });
  }

  return conflicts;
}
```

### Phase 4 — Peer Discussion

```typescript
async function conductPeerDiscussion(
  conflicts: Conflict[],
  findings: AgentFinding[],
  maxRounds: number = 2
): Promise<{ messages: AgentMessage[]; resolvedConflicts: Conflict[] }> {
  const allMessages: AgentMessage[] = [];

  for (let round = 1; round <= maxRounds; round++) {
    const unresolvedConflicts = conflicts.filter((c) => !c.resolved);
    if (unresolvedConflicts.length === 0) break;

    for (const conflict of unresolvedConflicts) {
      // Ask each involved agent to respond to the conflict
      const responses = await Promise.all(
        conflict.involvedAgents.map(async (agentId) => {
          const agent = AGENTS[agentId];
          if (!agent) return null;

          const otherPositions = conflict.positions
            .filter((p) => p.agentId !== agentId)
            .map((p) => `${p.agentId}: "${p.position}" (conf: ${p.confidence})`)
            .join("\n");

          const myPosition = conflict.positions
            .find((p) => p.agentId === agentId)?.position ?? "unknown";

          const prompt = `
CONFLICT: ${conflict.topic}
YOUR POSITION: ${myPosition}
OTHER POSITIONS:
${otherPositions}

Respond in JSON:
{
  "revised_position": "your updated or maintained position",
  "confidence": 0.0-1.0,
  "agrees_with_peer": true/false,
  "reasoning": "brief reasoning"
}`;

          const response = await callGroqWithRateLimit({
            model: agent.model as any,
            systemPrompt: agent.systemPrompt + "\n\nYou are in peer review mode.",
            userMessage: prompt,
            maxTokens: 500,
            responseFormat: "json_object",
          });

          return {
            agentId,
            response: JSON.parse(response),
          };
        })
      );

      // Check if conflict resolved
      const validResponses = responses.filter(Boolean);
      const agreements = validResponses.filter((r) => r!.response.agrees_with_peer);

      if (agreements.length >= validResponses.length - 1) {
        conflict.resolved = true;
        conflict.resolution = "Consensus reached through peer discussion";
      }

      // Log messages
      for (const resp of validResponses) {
        if (!resp) continue;
        allMessages.push({
          id: generateId(),
          round,
          sender: resp.agentId,
          recipients: conflict.involvedAgents.filter((id) => id !== resp.agentId),
          messageType: resp.response.agrees_with_peer ? "agreement" : "disagreement",
          content: JSON.stringify(resp.response),
          timestamp: Date.now(),
        });
      }
    }
  }

  return { messages: allMessages, resolvedConflicts: conflicts };
}
```

---

## Consensus Algorithm

For unresolved conflicts after peer discussion, use confidence-weighted voting:

```typescript
function resolveByVoting(conflict: Conflict): string {
  // Weight each position by agent confidence
  const weightedVotes = conflict.positions.map((p) => ({
    position: p.position,
    weight: p.confidence,
    agentId: p.agentId,
  }));

  // Group by position
  const grouped = groupBy(weightedVotes, "position");

  // Find position with highest total weight
  let bestPosition = "";
  let bestWeight = 0;

  for (const [position, votes] of Object.entries(grouped)) {
    const totalWeight = votes.reduce((sum, v) => sum + v.weight, 0);
    if (totalWeight > bestWeight) {
      bestWeight = totalWeight;
      bestPosition = position;
    }
  }

  return bestPosition;
}
```

---

## Communication Limits

| Limit | Value | Reason |
|-------|-------|--------|
| Max discussion rounds | 2 | Diminishing returns after 2 |
| Max messages per round | 10 | Rate limit protection |
| Max tokens per message | 500 | Cost control |
| Message timeout | 10s | Prevent blocking |
| Max conflicts per analysis | 5 | Focus on critical issues |

### Rate Limit Budget for Communication

| Phase | Groq Calls | Gemini Calls | Est. Time |
|-------|-----------|-------------|-----------|
| Agent analysis (5 agents) | 5 | 0 | 5-15s |
| Peer discussion (2 rounds, 3 conflicts) | 6 | 0 | 5-10s |
| Synthesis | 0 | 1 | 5-10s |
| **Total** | **11** | **1** | **15-35s** |

Within Groq free tier (30 RPM) and Gemini free tier (15 RPM).

---

## Related Documentation

- [03_TIER2_MULTIAGENT_CORE.md](03_TIER2_MULTIAGENT_CORE.md) — Orchestration engine
- [04_AGENT_SPECIFICATIONS.md](04_AGENT_SPECIFICATIONS.md) — Agent system prompts
- [10_COST_OPTIMIZATION.md](10_COST_OPTIMIZATION.md) — Token budgets

## Revision History

| Version | Date | Changes |
|---------|------|---------|
| 2.0.0 | 2026-02-09 | Multi-agent peer communication protocol |
| 1.0.0 | 2025-01-01 | Single-model, no inter-agent communication |
