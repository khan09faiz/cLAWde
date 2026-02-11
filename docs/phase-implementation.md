# Phase-wise Implementation Plan

## Overview

| Phase | Timeline | Goal |
|-------|----------|------|
| Phase 1 | Month 1 | Functional analyzer (Core Engine) |
| Phase 2 | Month 2 | Smart interactive system (RAG & Q&A) |
| Phase 3 | Month 3 | Production-ready SaaS (Dashboard & Privacy) |
| Phase 4 | Month 4–5 | Unique market positioning (Differentiation) |
| Phase 5 | Month 5+ | Revenue scale (Premium & Enterprise) |

---

## Phase 1 — Core Engine (Month 1)

**Goal:** Functional contract analyzer that can process and analyze documents.

### Deliverables

| # | Task | Priority |
|---|------|----------|
| 1 | File upload endpoint (PDF, DOCX, TXT) | P0 |
| 2 | Text extraction pipeline (+ OCR for scans) | P0 |
| 3 | Clause chunking / segmentation | P0 |
| 4 | Basic LLM clause explanation | P0 |
| 5 | Risk classification (rule-based + LLM) | P0 |
| 6 | Structured JSON output | P0 |
| 7 | Basic frontend upload + results view | P1 |

### Technical Milestones
- [ ] FastAPI project scaffolded with proper structure
- [ ] Document ingestion pipeline working (PDF → text → clauses)
- [ ] LLM integration working (Groq or Ollama)
- [ ] Clause analysis returning structured JSON
- [ ] Basic risk scoring functional
- [ ] API endpoints documented
- [ ] Basic Next.js frontend rendering results

### Success Criteria
- Can upload a PDF and get clause-by-clause analysis
- Risk scores generated for each clause
- Structured JSON output for all analysis

---

## Phase 2 — RAG & Q&A (Month 2)

**Goal:** Smart interactive system with contract-specific and legal Q&A.

### Deliverables

| # | Task | Priority |
|---|------|----------|
| 1 | Vector DB integration (ChromaDB or PGVector) | P0 |
| 2 | Contract-specific Q&A (internal RAG) | P0 |
| 3 | Basic legal knowledge base (Indian laws) | P0 |
| 4 | Hybrid retrieval routing | P1 |
| 5 | Q&A chat interface in frontend | P1 |
| 6 | Source citation in responses | P1 |

### Technical Milestones
- [ ] Vector DB set up and tested
- [ ] Contract embeddings pipeline working
- [ ] Legal knowledge base chunked and embedded
- [ ] RAG retrieval returning relevant context
- [ ] LLM generating grounded answers
- [ ] Chat UI functional
- [ ] Hybrid routing (contract vs legal) working

### Success Criteria
- User can ask questions about their contract
- System provides sourced answers
- Legal context retrieved when relevant

---

## Phase 3 — Dashboard & Privacy (Month 3)

**Goal:** Production-ready SaaS with privacy controls and user accounts.

### Deliverables

| # | Task | Priority |
|---|------|----------|
| 1 | Auto-delete mode implementation | P0 |
| 2 | Encrypted storage (AES-256) | P0 |
| 3 | User authentication & accounts | P0 |
| 4 | Analysis history / dashboard | P1 |
| 5 | Deadline tracking & calendar | P1 |
| 6 | Privacy mode selector UI | P0 |
| 7 | Trust UX elements (timer, badges, delete) | P0 |

### Technical Milestones
- [ ] Authentication system (JWT) implemented
- [ ] AES-256 encryption for stored contracts
- [ ] Auto-delete scheduler running
- [ ] User dashboard with contract history
- [ ] Deadline extraction and countdown timers
- [ ] Privacy mode toggle working end-to-end
- [ ] Trust UX elements visible and functional

### Success Criteria
- Users can create accounts and save analyses
- Auto-delete works reliably within 30 minutes
- Privacy mode clearly communicated in UI
- Deadlines extracted and displayed

---

## Phase 4 — Differentiation (Month 4–5)

**Goal:** Unique market positioning with Indian-law awareness and negotiation support.

### Deliverables

| # | Task | Priority |
|---|------|----------|
| 1 | Indian-law-specific flags and warnings | P0 |
| 2 | Negotiation suggestions engine | P1 |
| 3 | Full risk scoring framework (5 dimensions) | P0 |
| 4 | Clause comparison (industry standard vs current) | P1 |
| 5 | Legal glossary pop-ups | P2 |
| 6 | "Should I Sign?" decision engine | P0 |
| 7 | Export PDF report | P1 |

### Technical Milestones
- [ ] Indian law flags integrated into analysis
- [ ] Negotiation suggestion prompts refined
- [ ] Risk radar with 5 dimensions functional
- [ ] Clause comparison logic implemented
- [ ] Glossary database built and integrated
- [ ] Decision engine returning verdicts
- [ ] PDF export generating clean reports

### Success Criteria
- System flags Indian-law-specific issues
- Risk radar provides comprehensive scoring
- "Should I Sign?" gives actionable verdicts
- Reports exportable as PDF

---

## Phase 5 — Premium & Enterprise (Month 5+)

**Goal:** Revenue scale with premium features and enterprise capabilities.

### Deliverables

| # | Task | Priority |
|---|------|----------|
| 1 | Local inference mode (Ollama integration) | P0 |
| 2 | Team workspace / collaboration | P1 |
| 3 | API access for integrations | P1 |
| 4 | Compliance checker modules | P2 |
| 5 | On-prem deployment option | P2 |
| 6 | Version comparison (contract diffs) | P2 |
| 7 | Multilingual support (Hindi) | P2 |

### Success Criteria
- Premium users can run fully local
- Teams can collaborate on contract reviews
- API available for third-party integrations
- Enterprise deployment documentation complete

---

## Implementation Strategy Notes

### Step 1: Build Core Intelligence First
**DO NOT** start with:
- Collaboration features
- Enterprise security
- Complex UI animations

**START WITH:**
- Best clause-level AI engine
- Clean API design
- Solid document processing pipeline

### Step 2: Focus on Trust
Build trust visibly from Day 1:
- Auto-delete timer visible
- "We do not train on your data" messaging
- Delete button accessible
- Privacy badge prominent

### Step 3: Validate With Freelancers First

| User Type | Why First? |
|-----------|-----------|
| Freelancers | Fast feedback, lower trust barrier, faster iteration |
| Law Firms | Requires compliance audits, long sales cycle — wait until Phase 5 |

---

## Risk Mitigation

| Risk | Mitigation |
|------|-----------|
| LLM quality issues | Strong prompting, fallback models, human-in-loop for premium |
| OCR accuracy | Multiple OCR engines, user text correction |
| Scope creep | Strict phase adherence, MVP mindset |
| Privacy concerns | Privacy-first architecture from Day 1 |
| Performance (8GB RAM) | Optimize model selection, use quantized models |
