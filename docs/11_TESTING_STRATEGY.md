# 11_TESTING_STRATEGY

**Last Updated:** 2026-02-09  
**Version:** 2.0.0  
**Author:** LawBotics Team

---

## Table of Contents

1. [Testing Layers](#testing-layers)
2. [Agent Accuracy Tests](#agent-accuracy-tests)
3. [Integration Tests](#integration-tests)
4. [Frontend Tests](#frontend-tests)
5. [Load & Rate Limit Tests](#load--rate-limit-tests)
6. [CUAD Benchmark Tests](#cuad-benchmark-tests)

---

## Testing Layers

```
┌─────────────────────────────────────────────────────────┐
│ E2E Tests (Playwright)                                  │
│ Upload → Tier 1 → Tier 2 → Results Display              │
├─────────────────────────────────────────────────────────┤
│ Integration Tests (Vitest)                              │
│ API routes, Convex actions, agent orchestration         │
├─────────────────────────────────────────────────────────┤
│ Agent Accuracy Tests (Python/pytest)                    │
│ Per-agent precision/recall on CUAD benchmark            │
├─────────────────────────────────────────────────────────┤
│ Unit Tests (Vitest)                                     │
│ Complexity scoring, conflict detection, parsing         │
└─────────────────────────────────────────────────────────┘
```

| Layer | Framework | Count Target | Coverage Target |
|-------|-----------|-------------|-----------------|
| Unit | Vitest | 100+ | 80% |
| Agent Accuracy | pytest | 41 clause types × 9 agents | 90%+ per agent |
| Integration | Vitest | 30+ | Critical paths |
| E2E | Playwright | 10+ | Happy paths |

---

## Agent Accuracy Tests

### Per-Agent Benchmark Suite

```python
# tests/agent_accuracy/test_clause_agent.py

import pytest
from lawbotics.agents import ClauseExtractionAgent
from lawbotics.data import load_cuad_test_set

@pytest.fixture
def clause_agent():
    return ClauseExtractionAgent(
        model="llama-3.1-8b-instant",
        provider="groq"
    )

@pytest.fixture
def cuad_test_data():
    return load_cuad_test_set()  # 51 contracts held out for testing

class TestClauseExtractionAccuracy:
    """Test clause extraction against CUAD ground truth."""
    
    @pytest.mark.parametrize("clause_type", [
        "Document Name", "Parties", "Agreement Date",
        "Expiration Date", "Renewal Term", "Notice Period To Terminate Renewal",
        "Governing Law", "Most Favored Nation", "Non-Compete",
        "Exclusivity", "No-Solicit Of Customers", "Competitive Restriction Exception",
        "Non-Disparagement", "Termination For Convenience", "Rofr/Rofo/Rofn",
        "Change Of Control", "Anti-Assignment", "Revenue/Profit Sharing",
        "Price Restrictions", "Minimum Commitment", "Volume Restriction",
        "IP Ownership Assignment", "Joint IP Ownership", "License Grant",
        "Non-Transferable License", "Affiliate License-Loss Of Control",
        "Cap On Liability", "Liquidated Damages", "Warranty Duration",
        "Insurance", "Covenant Not To Sue", "Third Party Beneficiary",
        "Audit Rights", "Uncapped Liability", "Post-Termination Services",
        "Irrevocable Or Perpetual License", "Source Code Escrow",
    ])
    def test_clause_detection(self, clause_agent, cuad_test_data, clause_type):
        """Each clause type should achieve >= 90% F1 score."""
        true_positives = 0
        false_positives = 0
        false_negatives = 0
        
        for contract in cuad_test_data:
            result = clause_agent.extract(contract["text"])
            predicted = any(c["type"] == clause_type and c["present"] for c in result["clauses"])
            actual = clause_type in contract["ground_truth_clauses"]
            
            if predicted and actual: true_positives += 1
            elif predicted and not actual: false_positives += 1
            elif not predicted and actual: false_negatives += 1
        
        precision = true_positives / max(true_positives + false_positives, 1)
        recall = true_positives / max(true_positives + false_negatives, 1)
        f1 = 2 * precision * recall / max(precision + recall, 0.001)
        
        assert f1 >= 0.90, f"{clause_type}: F1={f1:.3f} (P={precision:.3f}, R={recall:.3f})"

    def test_overall_accuracy(self, clause_agent, cuad_test_data):
        """Overall accuracy should be >= 93%."""
        correct = 0
        total = 0
        
        for contract in cuad_test_data:
            result = clause_agent.extract(contract["text"])
            for clause_type in contract["all_clause_types"]:
                predicted = any(c["type"] == clause_type and c["present"] for c in result["clauses"])
                actual = clause_type in contract["ground_truth_clauses"]
                if predicted == actual: correct += 1
                total += 1
        
        accuracy = correct / total
        assert accuracy >= 0.93, f"Overall accuracy: {accuracy:.3f}"
```

### Risk Assessment Agent Tests

```python
# tests/agent_accuracy/test_risk_agent.py

class TestRiskAssessmentAccuracy:
    
    def test_risk_score_consistency(self, risk_agent):
        """Same contract should produce same score ±3 points."""
        contract = load_sample_contract("service_agreement_01.txt")
        scores = [risk_agent.assess(contract)["overall_risk_score"] for _ in range(5)]
        spread = max(scores) - min(scores)
        assert spread <= 3, f"Score spread: {spread} (scores: {scores})"
    
    def test_high_risk_detection(self, risk_agent, high_risk_contracts):
        """Should flag all known high-risk contracts as HIGH or CRITICAL."""
        for contract in high_risk_contracts:
            result = risk_agent.assess(contract["text"])
            assert result["risk_level"] in ["HIGH", "CRITICAL"], \
                f"High-risk contract scored as {result['risk_level']}"
    
    def test_missing_protections(self, risk_agent):
        """Should detect contracts missing standard protective provisions."""
        contract = load_sample_contract("no_liability_cap.txt")
        result = risk_agent.assess(contract)
        missing = [p["provision"] for p in result["missing_protections"]]
        assert "Cap On Liability" in missing
```

---

## Integration Tests

### Tier Routing Tests

```typescript
// tests/integration/routing.test.ts

import { describe, it, expect } from "vitest";
import { calculateComplexityScore } from "../convex/agents/complexity";

describe("Tier Routing", () => {
  it("routes simple NDA to Tier 1 only", () => {
    const factors = {
      clauseCount: 8,
      ambiguousClauseRatio: 0.1,
      uniqueClauseTypes: 5,
      contractLength: 2000,
      financialComplexity: 0,
      crossJurisdiction: false,
      multiParty: false,
      hasUnusualProvisions: false,
      tier1Confidence: 0.92,
      contractValue: null,
      regulatoryDomains: 0,
    };

    const score = calculateComplexityScore(factors);
    expect(score).toBeLessThan(6);
  });

  it("routes complex M&A agreement to Tier 2", () => {
    const factors = {
      clauseCount: 35,
      ambiguousClauseRatio: 0.4,
      uniqueClauseTypes: 20,
      contractLength: 15000,
      financialComplexity: 8,
      crossJurisdiction: true,
      multiParty: true,
      hasUnusualProvisions: true,
      tier1Confidence: 0.65,
      contractValue: 5000000,
      regulatoryDomains: 3,
    };

    const score = calculateComplexityScore(factors);
    expect(score).toBeGreaterThanOrEqual(6);
  });
});
```

### Conflict Detection Tests

```typescript
describe("Conflict Detection", () => {
  it("detects risk score spread > 20 points", () => {
    const findings = [
      { agentId: "risk_assessor", findings: { overall_risk_score: 75 }, confidence: 0.85 },
      { agentId: "financial_analyst", findings: { overall_risk_score: 40 }, confidence: 0.80 },
    ];

    const conflicts = detectConflicts(findings as any);
    expect(conflicts).toHaveLength(1);
    expect(conflicts[0].type).toBe("risk_score_disagreement");
  });

  it("detects no conflict when scores are close", () => {
    const findings = [
      { agentId: "risk_assessor", findings: { overall_risk_score: 65 }, confidence: 0.9 },
      { agentId: "financial_analyst", findings: { overall_risk_score: 55 }, confidence: 0.85 },
    ];

    const conflicts = detectConflicts(findings as any);
    const riskConflicts = conflicts.filter((c) => c.type === "risk_score_disagreement");
    expect(riskConflicts).toHaveLength(0);
  });
});
```

---

## Frontend Tests

### Component Tests

```typescript
// tests/components/AgentProgress.test.tsx

import { render, screen } from "@testing-library/react";
import { AgentProgress } from "@/components/feature-pages/analysis/AgentProgress";

// Mock Convex useQuery
vi.mock("convex/react", () => ({
  useQuery: vi.fn(() => ({
    status: "agents_running",
    activeAgents: ["clause_extractor", "risk_assessor", "financial_analyst"],
    completedAgents: ["clause_extractor"],
    failedAgents: [],
    totalProcessingTimeMs: 5000,
  })),
}));

describe("AgentProgress", () => {
  it("shows active agents with correct status", () => {
    render(<AgentProgress documentId={"test-id" as any} />);
    
    expect(screen.getByText("Clause Extraction")).toBeInTheDocument();
    expect(screen.getByText("Risk Assessment")).toBeInTheDocument();
    expect(screen.getByText("Financial Analysis")).toBeInTheDocument();
  });

  it("shows processing time", () => {
    render(<AgentProgress documentId={"test-id" as any} />);
    expect(screen.getByText(/5\.0s elapsed/)).toBeInTheDocument();
  });
});
```

---

## Load & Rate Limit Tests

```typescript
describe("Rate Limit Handling", () => {
  it("stays within Groq 30 RPM limit", async () => {
    const startTime = Date.now();
    const calls: number[] = [];

    for (let i = 0; i < 35; i++) {
      await callGroqWithRateLimit({
        model: "llama-3.1-8b-instant",
        systemPrompt: "test",
        userMessage: "test",
      });
      calls.push(Date.now());
    }

    // Check that no 60-second window has > 30 calls
    for (let i = 0; i < calls.length; i++) {
      const windowEnd = calls[i] + 60000;
      const callsInWindow = calls.filter((t) => t >= calls[i] && t < windowEnd);
      expect(callsInWindow.length).toBeLessThanOrEqual(30);
    }
  });
});
```

---

## CUAD Benchmark Tests

### Benchmark Runner

```bash
# Run CUAD benchmark suite
python -m pytest tests/agent_accuracy/ \
  --benchmark \
  --report-file=benchmarks/cuad_results.json \
  -v

# Expected output:
# clause_extractor: F1=0.962, Precision=0.971, Recall=0.953
# risk_assessor:    Consistency=±2.5 points
# Overall accuracy: 93.2%
```

---

## Related Documentation

- [02_TIER1_LLAMAMODEL.md](02_TIER1_LLAMAMODEL.md) — Model accuracy metrics
- [04_AGENT_SPECIFICATIONS.md](04_AGENT_SPECIFICATIONS.md) — Agent behavior specs
- [15_PERFORMANCE_BENCHMARKS.md](15_PERFORMANCE_BENCHMARKS.md) — Full benchmarks

## Revision History

| Version | Date | Changes |
|---------|------|---------|
| 2.0.0 | 2026-02-09 | Multi-agent testing, conflict detection tests |
| 1.0.0 | 2025-01-01 | Basic document analysis tests |
