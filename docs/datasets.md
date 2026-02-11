# Datasets Reference

## Overview

The project leverages multiple datasets for different purposes:

- **Contract content retrieval** — User's own document (internal RAG)
- **Legal context retrieval** — External Indian law datasets
- **Clause classification / model training** — Labeled clause datasets

---

## 1. Indian-Laws Dataset (HuggingFace)

**Source:** [mratanusarkar/Indian-Laws](https://huggingface.co/datasets/mratanusarkar/Indian-Laws)

**How to Load:**

```python
from datasets import load_dataset

ds = load_dataset("mratanusarkar/Indian-Laws")
```

| Property | Detail                    |
| -------- | ------------------------- |
| Type     | Statutes text (bare Acts) |
| Format   | Parquet                   |
| Language | English                   |
| Size     | ~34,244 rows              |
| License  | Check source              |

**Schema:**
| Field | Description |
|-------|-------------|
| `act_title` | Name of the Act (e.g., "Aadhaar Act, 2016") |
| `section` | Section number or heading |
| `law` | Actual text of the section/provision |

**Use cases:**

- Legal context retrieval for RAG
- Retrieve exact statutory text for user queries
- Support "Is this enforceable under Indian law?" Q&A

**Limitations:**

- Contains statutes but NOT judicial interpretations or case summaries

---

## 2. Indian Law Q&A Dataset (HuggingFace)

**Source:** [viber1/indian-law-dataset](https://huggingface.co/datasets/viber1/indian-law-dataset)

**How to Load:**

```python
from datasets import load_dataset

ds = load_dataset("viber1/indian-law-dataset")
```

| Property | Detail                                 |
| -------- | -------------------------------------- |
| Type     | Legal Q&A pairs (instruction-response) |
| Format   | JSON                                   |
| Size     | ~24,600 entries                        |
| License  | Apache-2.0                             |

**Schema:**
| Field | Description |
|-------|-------------|
| `instruction` | A legal question |
| `response` | Explanation answering the question |

**Use cases:**

- Fast retrieval and summarization of legal Q&A
- Training a lightweight legal Q&A model
- Supplementing contract context with legal definitions

**Limitations:**

- Predetermined responses — may not cover contract-specific edge cases
- Focused on general law Q&A, not contract clause risk patterns

---

## 3. SaaS Golden Clauses Dataset (Kaggle)

**Source:** [Kaggle - SaaS Risk App](https://www.kaggle.com/datasets/vrushtibg/saas-risk-app-github-import?select=golden_clauses.csv)

| Property | Detail                   |
| -------- | ------------------------ |
| Type     | Contract clause examples |
| Format   | CSV                      |
| Focus    | SaaS contracts           |

**Expected fields:**

- Clause text
- Clause category
- Manual annotations
- Source contract identifiers

**Use cases:**

- Clause classification training
- Benchmarking standard vs non-standard clauses
- Supplementing clause library for risk detection

**Limitations:**

- SaaS-focused, not general legal texts
- No legal interpretations or enforceability info

---

## 4. LawBotics / CUAD Dataset (GitHub)

**Source:** [LawBotics GitHub](https://github.com/hasnaintypes/lawbotics/tree/master/ai-model/data-set)

| Property  | Detail                                               |
| --------- | ---------------------------------------------------- |
| Type      | Contract clauses + annotations                       |
| Key Files | CUAD_v1.json, master_clauses.csv, full_contract_pdf/ |
| Size      | ~13,000 labeled examples across ~510 contracts       |

**Contains:**

- CUAD dataset — annotated contract clauses
- `master_clauses.csv` — clause-to-category mappings
- `full_contract_pdf/` — original contract PDFs

**Use cases:**

- Contract clause classification
- Fine-tuning LLM to recognize clause categories
- Benchmarking clause extraction output

**Limitations:**

- Commercial contract examples — may not reflect Indian-law patterns
- Requires preprocessing and labeling consistency checks

---

## 5. Mendeley Judgments Dataset

**Source:** [Mendeley Data](https://data.mendeley.com/datasets/gf8n8cnmvc/2)

| Property | Detail                                   |
| -------- | ---------------------------------------- |
| Type     | Indian legal judgments (court decisions) |
| Content  | Judgment text, case metadata, citations  |

**Use cases:**

- Retrieve relevant judgments for clause interpretation
- Supplement statutory context with judicial reasoning
- Answer enforceability questions

**Limitations:**

- May need extra preprocessing
- Text cleaning and metadata extraction required

---

## 6. Google Drive Clauses Folder

**Source:** [Google Drive](https://drive.google.com/drive/folders/1vk8-3sld8WfFk-Zl1Y4e4OiydQEoaD6i)

| Property | Detail                   |
| -------- | ------------------------ |
| Type     | Mixed clause samples     |
| Format   | CSV, JSON, DOCX (varies) |

**Use cases:**

- Expand clause library
- Create custom contract patterns
- Supplement SaaS clauses with Indian contract examples

**Limitations:**

- Not standardized
- Requires manual inspection and preprocessing

---

## 7. Dataset Usage Matrix

| Dataset              | RAG Context | Q&A Training | Clause Classification | Risk Scoring |
| -------------------- | :---------: | :----------: | :-------------------: | :----------: |
| Indian-Laws (HF)     |     ✅      |      —       |           —           |      —       |
| Indian Law Q&A (HF)  |     ✅      |      ✅      |           —           |      —       |
| Mendeley Judgments   |     ✅      |      —       |           —           |      —       |
| SaaS Golden Clauses  |      —      |      —       |          ✅           |      ✅      |
| LawBotics/CUAD       |      —      |      —       |          ✅           |      ✅      |
| Google Drive Clauses |      —      |      —       |          ✅           |      —       |

---

## 8. Dataset Ingestion Workflow

### Development / Ingestion Phase

**You should download datasets locally for:**

1. **Loading from sources**
   - HuggingFace datasets
   - Kaggle downloads
   - Mendeley data
   - Keep raw files for versioning

2. **Cleaning and normalization**
   - Remove duplicates
   - Fix encoding issues
   - Standardize formatting
   - Save as JSONL or Parquet

3. **Embedding generation**
   - Process through chunking pipeline
   - Generate embeddings
   - Store in vector DB with metadata

**After ingestion:** Raw datasets can be archived offline.

---

### Production Runtime

**Production should NOT:**

- ❌ Call HuggingFace APIs
- ❌ Load raw datasets
- ❌ Re-embed corpus on startup

**Production should ONLY:**

- ✅ Query pre-populated vector DB
- ✅ Load pre-computed embeddings

---

### Recommended Workflow

```
┌──────────────┐
│ Raw Dataset  │ (Download from HuggingFace/Kaggle)
└──────┬───────┘
       ↓
┌──────────────┐
│ Clean Script │ (Jupyter Notebook / Python script)
└──────┬───────┘
       ↓
┌──────────────┐
│ JSONL Files  │ (Normalized, structured)
└──────┬───────┘
       ↓
┌──────────────┐
│ Ingest Script│ (Chunking + Embedding pipeline)
└──────┬───────┘
       ↓
┌──────────────┐
│  Vector DB   │ (ChromaDB / PGVector - persistent)
└──────────────┘
       ↑
       │ Query only
┌──────────────┐
│ Production   │ (FastAPI backend)
│     App      │
└──────────────┘
```

---

## 9. Dataset Chunking Strategies

### Critical Principle

**Legal text ≠ normal text**

Legal documents are highly structured. Wrong chunking destroys:

- Section hierarchy
- Legal citations
- Context dependencies

---

### 🔥 Acts Dataset Chunking

**Structure:**

```
Act → Chapter → Section → Subsection
```

**✅ Correct Approach:**

- Chunk at **section level** (ideal)
- If section > 1000 tokens → chunk at **subsection level**
- Never split mid-sentence

**Required metadata per chunk:**

```json
{
  "act_name": "Indian Contract Act, 1872",
  "chapter": "Chapter V",
  "section": "Section 73",
  "title": "Compensation for loss or damage caused by breach",
  "content": "Full section text...",
  "chunk_type": "statute"
}
```

**Why this matters:**

- Legal retrieval depends on section numbers
- Users ask questions like "What does Section 73 say?"
- Section titles contain important keywords

---

### 🔥 Judgments Dataset Chunking

**Challenge:** Judgments are long (50+ pages)

**✅ Strategy:**

**If pre-structured:**

- Split by logical sections: Facts, Issue, Arguments, Decision, Ratio

**If unstructured:**

- Chunk into 800–1200 tokens
- Use 150–200 token overlap
- Preserve paragraph boundaries

**Required metadata:**

```json
{
  "case_name": "ABC Ltd. v. XYZ Corp.",
  "court": "Supreme Court of India",
  "year": 2020,
  "citation": "(2020) 5 SCC 123",
  "section": "Decision",
  "content": "...",
  "chunk_type": "judgment"
}
```

---

### 🔥 Clause Dataset Chunking

**Rule: One clause = One chunk**

Do NOT mix multiple clauses.

**Example:**

```json
{
  "clause_type": "Indemnity",
  "risk_level": "High",
  "party_favored": "Party A",
  "text": "The Vendor shall indemnify...",
  "chunk_type": "clause_library"
}
```

**Why:**

- User asks: "Is my indemnity clause risky?"
- System retrieves similar indemnity clauses
- Comparison requires clean clause boundaries

---

### 🔥 Q&A Dataset Chunking

**Strategy: Chunk per Q&A pair**

```json
{
  "question": "What is remedy for breach under Indian law?",
  "answer": "Under Section 73...",
  "chunk_type": "qa_pair"
}
```

**Usage:**

- Semantic similarity matching
- Few-shot retrieval examples
- Direct answer retrieval

---

## 10. RAG Architecture with Datasets

### Internal RAG (Contract-Specific)

- User's uploaded contract is chunked and embedded
- Stored in user-specific vector index
- Used for contract-specific Q&A

### External RAG (Legal Knowledge Base)

- Indian-Laws statutes → embedded provisions (section-level chunks)
- Indian Law Q&A → embedded question-answer pairs
- Mendeley Judgments → embedded judicial reasoning (logical sections)

### Retrieval Routing

```
User Question
     ↓
┌────────────────┐
│ Intent Router   │
├────────────────┤
│ Contract Q?  ──→ Internal RAG (user's contract)
│ Legal Q?     ──→ External RAG (legal knowledge base)
│ Both?        ──→ Hybrid retrieval (merge results)
└────────────────┘
```

**For complete implementation details, see [rag-implementation-guide.md](rag-implementation-guide.md)**
