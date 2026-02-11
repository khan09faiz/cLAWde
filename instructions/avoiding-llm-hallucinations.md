# Avoiding LLM Hallucinations

## 1. Why This Matters

In a legal contract analysis tool, **hallucinations are dangerous**. A fabricated clause interpretation, a wrong risk score, or an invented legal reference could lead users to make harmful decisions. This document defines strict guardrails.

---

## 2. Core Anti-Hallucination Strategies

### 2.1 Structured Output Enforcement

**Always** force the LLM to return structured JSON matching a Pydantic schema.

```python
# ✅ Force structured output
class ClauseAnalysis(BaseModel):
    purpose: str
    plain_english: str
    risk_severity: Literal["low", "medium", "high"]  # Constrained values
    concerns: list[str]
    source_reference: str  # Must cite the clause text

# ❌ Never accept free-form unstructured text for analysis results
```

**Why:** Structured output constrains the LLM to fill defined fields, reducing open-ended fabrication.

---

### 2.2 Ground Everything in Source Text

- Every analysis point MUST reference the original clause text
- The LLM must quote or paraphrase the source, not invent content
- If the clause doesn't mention something, the analysis must say "Not specified in this clause"

**Prompt pattern:**
```
Analyze ONLY what is present in the following clause text.
If something is not mentioned, respond with "Not specified."
Do NOT infer terms that are not explicitly stated.

Clause: "{clause_text}"
```

---

### 2.3 RAG Grounding (Retrieval-Augmented Generation)

- For legal context questions, ALWAYS retrieve from the knowledge base first
- Pass retrieved context as part of the prompt
- Instruct the LLM to answer ONLY based on provided context

**Prompt pattern:**
```
Answer the user's question using ONLY the following context.
If the answer cannot be found in the context, say "I cannot determine this from the available information."

Context: {retrieved_chunks}
Question: {user_question}
```

---

### 2.4 Confidence Indicators

- For every analysis result, include a confidence level
- If the LLM is uncertain, surface that uncertainty to the user
- Never present uncertain analysis as definitive

```python
class AnalysisResult(BaseModel):
    analysis: str
    confidence: Literal["high", "medium", "low"]
    reasoning: str  # Why this confidence level
```

---

### 2.5 Rule-Based Validation Layer

Apply deterministic checks AFTER LLM analysis:

| Check | Action |
|-------|--------|
| Risk score consistency | If LLM says "high risk" but no concerns listed → flag |
| Entity validation | Verify extracted entities exist in source text |
| Date validation | Check extracted dates are parseable and reasonable |
| Monetary value check | Verify amounts exist in source text |
| Legal citation check | Verify cited Acts/sections exist in knowledge base |

---

## 3. Prompting Best Practices

### 3.1 Be Explicit About Constraints
```
# ✅ Good
"List ONLY the risks that are explicitly mentioned or directly implied by the clause text."

# ❌ Bad
"What are the risks of this clause?"
```

### 3.2 Use Few-Shot Examples
Provide 2-3 examples of correct analysis in the prompt to anchor the LLM's behavior.

### 3.3 Use System Prompts for Role Definition
```
System: You are a contract analysis assistant. You analyze contract clauses
based ONLY on the text provided. You never fabricate legal information.
When uncertain, you clearly state your uncertainty.
```

### 3.4 Temperature Settings
- Use **temperature 0 or near 0** for analysis tasks (deterministic)
- Use slightly higher temperature (0.3-0.5) only for creative tasks like negotiation email drafts

### 3.5 Avoid Leading Questions in Prompts
```
# ✅ Good
"Identify any risks present in this clause."

# ❌ Bad
"This clause appears to have several risks. What are they?"
```

---

## 4. Post-Processing Validation

### 4.1 Output Schema Validation
- Parse LLM output against Pydantic model
- If parsing fails, retry with clearer instructions (max 2 retries)
- If still fails, return error rather than partial/invalid data

### 4.2 Source Text Cross-Reference
- Every quoted text in analysis must be findable in the source clause
- Implement fuzzy matching to verify quotes
- Flag if LLM "quotes" text not in the source

### 4.3 Legal Citation Verification
- If LLM cites a specific Act or section, verify it exists in the legal knowledge base
- If not verifiable, add disclaimer: "This legal reference could not be verified"

### 4.4 Consistency Checks
- Cross-check risk severity across related clauses
- Verify entity extraction is consistent (same party names throughout)
- Check that summary aligns with detailed analysis

---

## 5. User-Facing Safeguards

### 5.1 Disclaimers
- Always display: "This is AI-generated analysis. Consult a legal professional for critical decisions."
- Never position the tool as legal advice

### 5.2 Source Attribution
- Show which clause text each analysis point refers to
- Allow users to click analysis → see source clause highlighted

### 5.3 Uncertainty Display
- If confidence is low, show a warning badge
- Allow users to flag incorrect analysis (feedback loop)

### 5.4 "I Don't Know" is Acceptable
- The system should say "I cannot determine this" rather than guessing
- This is especially critical for legal enforceability questions

---

## 6. Common Hallucination Patterns to Watch For

| Pattern | Example | Mitigation |
|---------|---------|-----------|
| Invented clauses | "The contract also states..." (when it doesn't) | Source text cross-reference |
| Fabricated legal refs | "Under Section 42 of..." (non-existent) | Legal citation verification |
| Over-confident risk | "This will definitely cause..." | Confidence scoring |
| Missing nuance | Ignoring "unless" or "subject to" qualifiers | Explicit prompt instructions |
| Context bleed | Mixing info from different clauses | Process clauses independently |
| Assumed jurisdiction | Assuming Indian law when not specified | Only state jurisdiction if in contract |

---

## 7. Monitoring & Improvement

- Log analysis quality metrics (not contract content)
- Track user flags on incorrect analysis
- Periodically review edge cases
- Update prompts based on failure patterns
- Maintain a test suite of tricky clauses with expected outputs
