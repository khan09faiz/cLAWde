# Feature Specification

## Feature Layers Overview

Features are structured in three layers:
1. **Core MVP Features** — Must-have for launch
2. **V1 Features** — After MVP validation
3. **Advanced / Premium Features** — Revenue & differentiation

---

## 🧩 Core MVP Features

### 1. Document Processing

#### Upload Options
- PDF
- DOCX
- TXT
- Paste clause manually

#### Processing Pipeline
- OCR for scanned docs
- Clause segmentation
- Section detection
- Contract type classification

---

### 2. Clause-Level Analysis

For **each clause**, the system must provide:

| Output | Description |
|--------|-------------|
| Purpose explanation | Why this clause exists |
| Plain English summary | Jargon-free rewrite |
| Who benefits? | Identifies the favored party |
| Risk severity | Low / Medium / High |
| What could go wrong? | Worst-case scenario |
| Location reference | Where in the document |

---

### 3. Risk Radar Scoring

**Individual Scores (0–10):**
- Financial Risk
- Legal Liability
- Exit Difficulty
- Data Privacy Risk
- Term Flexibility

**Overall Safety Score (0–100)**

**Hybrid Scoring System:**
- Rule-based signals (deterministic)
- LLM explanation layer (reasoning)

---

### 4. Three-Tier Summary

| Tier | Description |
|------|-------------|
| Ultra-short | 1–2 line summary |
| Key terms | Bullet list of important terms |
| Risk highlights | Categorized risk breakdown |

---

### 5. Interactive Q&A (RAG)

Users can ask natural language questions:
- "Can I terminate early?"
- "Is this enforceable in India?"
- "Who owns the IP?"

**Hybrid RAG approach:**
- Contract-specific retrieval (user's document)
- Indian legal knowledge base retrieval (external datasets)

---

### 6. "Should I Sign?" Decision Engine

**Output categories:**
- ✅ Safe
- ⚠️ Caution
- 🚫 Do Not Sign

**Includes:**
- Deal breakers identified
- Negotiable clauses highlighted
- Acceptable risks listed
- Plain English explanation

---

### 7. Deadline Extraction

**Extract:**
- Effective date
- Expiry date
- Auto-renewal clauses
- Notice period
- Payment deadlines

**UI:** Countdown timers in dashboard.

---

### 8. Key Entity Extraction

**Extract:**
- Parties involved
- Jurisdiction
- Governing law
- Monetary values
- Contract type

---

## 🌱 V1 Features (Post-MVP Validation)

- Negotiation email templates
- Clause comparison (industry standard vs current)
- Export PDF report
- Dashboard history
- Full-text search across contracts
- Glossary pop-ups for legal terms

---

## 🚀 Advanced / Premium Features

- Version comparison (diff between contract revisions)
- Team collaboration workspace
- Custom risk playbooks
- AI redlining (suggest edits inline)
- API access for integrations
- On-prem deployment option
- Multilingual support (Hindi explanations)
- Compliance checker modules
