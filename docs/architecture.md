# System Architecture

## 1. AI Model Requirements

The system requires four core AI components:

| Component                  | Purpose                                |
| -------------------------- | -------------------------------------- |
| Main Reasoning LLM         | Clause analysis, summarization, Q&A    |
| Embedding Model            | Vector representations for RAG         |
| OCR Engine                 | Text extraction from scanned documents |
| Rule-based Risk Classifier | Deterministic risk scoring signals     |

> **No fine-tuning is required initially.** Focus on strong prompting and clean architecture.

---

## 2. Tech Stack (Cost-Optimized + 8GB RAM Compatible)

### Backend

| Technology | Purpose                             |
| ---------- | ----------------------------------- |
| FastAPI    | API framework (async, fast, Python) |
| PostgreSQL | Primary relational database         |
| SQLAlchemy | ORM layer                           |
| Celery     | Background job processing           |
| Redis      | Caching, message broker for Celery  |

### Frontend

| Technology    | Purpose                     |
| ------------- | --------------------------- |
| Next.js       | React framework with SSR    |
| React         | UI component library        |
| Tailwind CSS  | Utility-first styling       |
| PDF.js        | In-browser PDF rendering    |
| Custom system | Clause highlighting overlay |

### Vector Database

| Option   | Notes                                    |
| -------- | ---------------------------------------- |
| ChromaDB | Lightweight, easy to set up              |
| PGVector | PostgreSQL extension, single DB solution |

---

## 3. AI Pipeline Flow

```
1. Upload file
   ↓
2. Extract text (OCR if scanned)
   ↓
3. Clause segmentation
   ↓
4. Embed chunks
   ↓
5. Store in vector DB
   ↓
6. Run clause analysis (LLM)
   ↓
7. Run risk rule engine
   ↓
8. Generate structured JSON
   ↓
9. Render frontend UI
```

### Pipeline Details

**Step 1-2: Document Ingestion**

- Accept PDF, DOCX, TXT, or pasted text
- Run OCR on scanned documents
- Extract raw text with structural metadata

**Step 3: Clause Segmentation**

- Split document into individual clauses
- Detect section headers, numbering, and hierarchy
- Classify contract type

**Step 4-5: Embedding & Storage**

- Chunk clauses for embedding
- Generate vector embeddings
- Store in ChromaDB or PGVector

**Step 6: LLM Analysis**

- Per-clause analysis with structured prompts
- Risk identification
- Plain English explanation generation

**Step 7: Rule Engine**

- Deterministic risk signals
- Pattern matching for known risky patterns
- Score computation

**Step 8-9: Output**

- Generate structured JSON response
- Render interactive UI with highlights

---

## 4. Dual RAG Architecture

The system implements **two separate knowledge bases** for optimal retrieval:

### A) Static Legal Knowledge Base (Pre-embedded)

**Contents:**

- Indian Contract Act provisions
- Labour law summaries
- Arbitration Act summaries
- Standard clause samples
- Risk pattern library
- Legal Q&A datasets
- Judicial judgments

**Characteristics:**

- Pre-embedded once during ingestion
- Shared across all users
- Updated periodically (monthly/quarterly)
- Persistent storage in vector DB

**Process:** Load datasets → Clean → Chunk (section-level) → Embed → Store in Vector DB

---

### B) Dynamic User Contract RAG

**Contents:**

- User's uploaded contract
- Embedded per session/user
- User-specific and private

**Characteristics:**

- Created on upload
- Session-based or user-scoped
- Deleted after session (Privacy Tier 1) or encrypted (Tier 2/3)
- Temporary vector index

**Process:** Upload → Extract → Chunk (clause-level) → Embed → Store temporarily

---

### Retrieval Flow

When user asks a question:

1. **Retrieve from both knowledge bases**
   - User's contract DB (specific clauses)
   - Legal knowledge DB (relevant laws/judgments)

2. **Merge and rank results**
   - Combine top K chunks from each
   - Rank by relevance score
   - Apply retrieval threshold

3. **Send to LLM with grounded prompt**
   - Include retrieved context only
   - Enforce citation requirements
   - No external knowledge allowed

**For detailed implementation, see [rag-implementation-guide.md](rag-implementation-guide.md)**

---

## 5. High-Level System Diagram

```
┌─────────────┐     ┌──────────────┐     ┌─────────────────┐
│   Frontend   │────▶│   FastAPI     │────▶│  Celery Worker  │
│  (Next.js)   │◀────│   Backend     │◀────│  (Background)   │
└─────────────┘     └──────┬───────┘     └────────┬────────┘
                           │                       │
                    ┌──────┴───────┐        ┌──────┴────────┐
                    │  PostgreSQL  │        │  Redis Cache   │
                    │  + PGVector  │        └───────────────┘
                    └──────────────┘
                           │
                    ┌──────┴───────┐
                    │  Vector DB   │
                    │  (ChromaDB)  │
                    └──────────────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
        ┌─────┴────┐ ┌────┴─────┐ ┌───┴──────┐
        │ Main LLM │ │ Embedder │ │ OCR      │
        │ (Groq/   │ │ (bge/    │ │ Engine   │
        │  Ollama) │ │  nomic)  │ │          │
        └──────────┘ └──────────┘ └──────────┘
```

---

## 6. Data Flow Summary

1. **User uploads** a contract document
2. **Backend** extracts text, segments into clauses
3. **Embeddings** are generated and stored in vector DB
4. **LLM** analyzes each clause with structured prompts
5. **Rule engine** computes risk scores
6. **Structured JSON** is returned to frontend
7. **Frontend** renders interactive analysis with highlights
8. **RAG Q&A** retrieves from contract + legal knowledge base
