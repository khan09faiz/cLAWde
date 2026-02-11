# RAG Implementation Guide for Legal Contract Analysis

## Overview

This guide provides the **production-suitable RAG architecture** for legal contract analysis. It covers exact implementation strategies for building a robust system that minimizes hallucinations and maximizes accuracy for legal text.

---

## 1️⃣ Exact Architecture You Should Follow

### High-Level Design

You will have **two knowledge systems**:

#### A) Static Legal Knowledge Base (Pre-embedded once)

Contains:

- Acts (Indian statutes)
- Judgments (judicial decisions)
- Clause libraries
- SaaS clauses
- Q&A datasets

**Storage:**

- Vector DB (persistent)
- Embedded once during ingestion
- Queried at runtime

**Characteristics:**

- Static — updated periodically (monthly/quarterly)
- Shared across all users
- High quality, curated legal knowledge

---

#### B) Dynamic User Contract RAG

Contains:

- Uploaded user contract
- Embedded temporarily per session
- Deleted after session (or encrypted per-user DB partition)

**Storage:**

- User-specific vector index
- Session-based or user-scoped
- Ephemeral or encrypted

**Characteristics:**

- Dynamic — created on upload
- User-specific and private
- Deleted based on privacy tier

---

### Flow When User Asks Question

**Example Question:**

> "Is this indemnity clause risky under Indian law?"

**System Flow:**

1. **Retrieve relevant chunks from:**
   - User contract DB (the specific indemnity clause)
   - Legal knowledge DB (Indian Contract Act provisions, relevant judgments)

2. **Combine retrieved context:**
   - Merge results from both knowledge bases
   - Rank by relevance score
   - Limit to top K chunks (e.g., top 5 from each)

3. **Send to LLM with strict grounded prompt:**

   ```
   You are a legal analyst. Answer based ONLY on the provided context.

   Context from user's contract:
   {user_contract_chunks}

   Context from legal knowledge base:
   {legal_knowledge_chunks}

   Question: {user_question}

   If the context is insufficient, say: "Insufficient legal context to answer."
   ```

4. **Return structured response:**
   - Risk level (Low/Medium/High)
   - Explanation (plain English)
   - Legal citation (specific Act/Section)
   - Clause references (from user's contract)

---

## 2️⃣ Do You Need to Download All Datasets Locally?

### Short Answer

✔ **YES** — For ingestion phase
❌ **NO** — For runtime/production

---

### During Development / Ingestion Phase

You should:

1. **Load from HuggingFace/Kaggle/Mendeley**
   - Download datasets to local machine
   - Keep raw files for reference

2. **Clean and normalize**
   - Remove duplicates
   - Fix encoding issues
   - Standardize formatting

3. **Save as structured files**
   - Export to JSONL or Parquet
   - Include metadata fields
   - Version control the cleaned data

4. **Run embedding pipeline**
   - Process chunks through embedding model
   - Generate vector representations
   - Store in vector DB with metadata

5. **Store in vector DB**
   - Persist embeddings with IDs
   - Index for fast retrieval
   - Backup the vector DB

**After ingestion:**
You do NOT need raw datasets anymore (except for versioning/auditing).

---

### Production App Behavior

Production should:

❌ **Not call HuggingFace APIs**
❌ **Not load datasets from disk**
❌ **Not re-embed entire corpus on every startup**

It should only:
✅ **Query vector DB** for relevant chunks
✅ **Load pre-computed embeddings**

---

### Recommended Workflow

```
┌──────────────┐
│ Raw Dataset  │ (HuggingFace / Kaggle)
└──────┬───────┘
       ↓
┌──────────────┐
│ Clean Script │ (Jupyter Notebook / Python script)
└──────┬───────┘
       ↓
┌──────────────┐
│ JSONL Files  │ (Structured, normalized)
└──────┬───────┘
       ↓
┌──────────────┐
│ Ingest Script│ (Embedding pipeline)
└──────┬───────┘
       ↓
┌──────────────┐
│  Vector DB   │ (ChromaDB / PGVector)
└──────────────┘
       ↑
       │
┌──────────────┐
│ Production   │ (FastAPI backend)
│     App      │
└──────────────┘
```

After ingestion, dataset files can stay offline or archived.

---

## 3️⃣ Most Important Part: Chunking Strategy for Legal Text

### Why Legal Chunking is Different

Legal text ≠ normal blog posts or documentation.

**If chunking is wrong:**

- Retrieval fails (wrong context returned)
- Hallucinations increase (model invents details)
- Context mismatch happens (section structure lost)

**Legal text characteristics:**

- Highly structured (sections, subsections, clauses)
- Context-dependent (section numbers matter)
- Hierarchical (Act → Chapter → Section → Subsection)

---

### 🔥 Chunking Strategy for Acts Dataset (Indian Laws)

#### Structure of Acts:

```
Act Name
├── Chapter I
│   ├── Section 1 (Title)
│   │   └── Subsection (a), (b), (c)
│   ├── Section 2
│   └── ...
├── Chapter II
│   └── ...
```

#### ❌ Wrong Chunking Approach:

Split every 500 tokens randomly.

**Why it fails:**

- Destroys section structure
- Loses Act name context
- Splits mid-sentence or mid-subsection

---

#### ✅ Correct Chunking for Acts:

**Chunk per:**

- **Section level** (ideal for most sections)
- **Subsection level** (if section is too large, >1000 tokens)

**Example Chunk:**

```json
{
  "act_name": "Indian Contract Act, 1872",
  "chapter": "Chapter V",
  "section": "Section 73",
  "title": "Compensation for loss or damage caused by breach of contract",
  "content": "When a contract has been broken, the party who suffers by such breach is entitled to receive, from the party who has broken the contract, compensation for any loss or damage caused to him thereby, which naturally arose in the usual course of things from such breach, or which the parties knew, when they made the contract, to be likely to result from the breach of it.",
  "chunk_type": "statute"
}
```

#### Each Chunk Must Contain:

1. **Act name** — for legal citation
2. **Section number** — critical for retrieval
3. **Section heading/title** — keyword-rich
4. **Full section body** — complete context

**Why?**

Legal retrieval often depends on:

- Exact section number (e.g., "Section 73 of Indian Contract Act")
- Title keywords (e.g., "compensation for breach")
- Full section context (not partial sentences)

---

### 🔥 Chunking Strategy for Judgments Dataset

#### Challenge:

Judgments are long (often 50+ pages).

❌ **Do NOT embed entire judgment as one chunk.**

---

#### ✅ Correct Strategy:

**If judgment is pre-structured:**

Split into logical sections:

1. **Facts** — what happened
2. **Issue** — legal question
3. **Arguments** — parties' positions
4. **Decision** — court's ruling
5. **Ratio decidendi** — legal reasoning

**If judgment is NOT structured:**

- Chunk into **800–1200 tokens** per chunk
- Use **150–200 token overlap** between chunks
- Preserve paragraph boundaries (don't split mid-paragraph)

---

#### Metadata to Attach:

```json
{
  "case_name": "Satyam Computer Services Ltd. v. Upaid Systems Ltd.",
  "court": "Supreme Court of India",
  "year": 2008,
  "citation": "(2008) 6 SCC 1",
  "section": "Decision",
  "content": "...",
  "chunk_type": "judgment"
}
```

**Why metadata matters:**

- User asks: "What did Supreme Court say about indemnity?"
- System retrieves chunks with `court = "Supreme Court"` + semantic match

---

### 🔥 Chunking Strategy for Clause Dataset

#### Rule:

**One clause = One chunk**

Do NOT mix multiple clauses in one chunk.

---

#### Example Chunk:

```json
{
  "clause_type": "Indemnity",
  "risk_level": "High",
  "party_favored": "Party A",
  "jurisdiction": "India",
  "text": "The Vendor shall indemnify, defend, and hold harmless the Company from and against any and all claims, damages, losses, and expenses (including reasonable attorneys' fees) arising out of or resulting from the Vendor's breach of this Agreement.",
  "chunk_type": "clause_library"
}
```

---

#### Why This Matters:

When user asks:

> "Is my indemnity clause one-sided?"

System retrieves:

1. User's indemnity clause (from their contract)
2. Standard indemnity clauses (from clause library)
3. Compares and flags if one-sided

If clauses are mixed in chunks, retrieval quality degrades.

---

### 🔥 Chunking Strategy for Q&A Dataset

#### Purpose:

- Semantic similarity examples
- Few-shot retrieval support
- Pre-answered legal questions

---

#### Strategy:

**Chunk per Q&A pair:**

```json
{
  "question": "What is the remedy for breach of contract under Indian law?",
  "answer": "Under Section 73 of the Indian Contract Act, the remedy is compensation for loss or damage caused by the breach.",
  "chunk_type": "qa_pair"
}
```

---

#### Usage:

When user asks:

> "What happens if a party breaches the contract?"

System retrieves similar Q&A pairs and uses them to:

- Augment context for LLM
- Provide direct answers if exact match
- Cross-reference with user's contract

---

## 4️⃣ Embedding Strategy (8GB RAM Setup)

### Requirements:

For **8GB RAM** setup, you need:

- **Lightweight embedding model**
- **Good quality** (not sacrificing too much accuracy)
- **CPU-compatible** (no GPU required)

---

### Best Embedding Models for 8GB RAM:

| Model                                    | Size | Quality  | Speed     | Use Case                |
| ---------------------------------------- | ---- | -------- | --------- | ----------------------- |
| `BAAI/bge-small-en-v1.5`                 | 33M  | Good     | Fast      | **Recommended for 8GB** |
| `intfloat/e5-small-v2`                   | 33M  | Good     | Fast      | Alternative             |
| `sentence-transformers/all-MiniLM-L6-v2` | 22M  | Moderate | Very Fast | Fallback option         |

---

### ✅ Recommended: `BAAI/bge-small-en-v1.5`

**Why:**

- Small size (33M parameters)
- Good retrieval quality
- Works well on CPU
- Used in production RAG systems

---

### Implementation:

```python
from sentence_transformers import SentenceTransformer

# Load model (runs on CPU by default)
model = SentenceTransformer('BAAI/bge-small-en-v1.5')

# Embed a single document
embedding = model.encode("This is a legal clause about indemnity.")

# Embed multiple documents (batched)
texts = ["Clause 1", "Clause 2", "Clause 3"]
embeddings = model.encode(texts, batch_size=8)
```

**Memory usage:** ~100-200 MB

---

### ❌ Do NOT Use Large Embedding Models on 8GB RAM

Avoid:

- `bge-large-en` (1.3 GB memory)
- `gte-large` (large memory footprint)
- OpenAI `text-embedding-ada-002` (requires API calls, not local)

These will:

- Slow down inference significantly
- Risk out-of-memory errors
- Hurt user experience

---

## 5️⃣ How to Reduce Hallucination in Legal RAG

This is **critical** for legal applications.

Hallucinations in legal context can:

- Provide incorrect legal advice
- Misinterpret clauses
- Cite non-existent laws

---

### Rule 1: Retrieval First, Generation Second

**Principle:**

If nothing relevant is retrieved → Model must say:

> "Insufficient legal context to answer this question."

**Implementation:**

```python
# After retrieval
if len(retrieved_chunks) == 0 or max_relevance_score < threshold:
    return {
        "answer": "Insufficient legal context to answer.",
        "retrieved_chunks": []
    }

# Otherwise, proceed with LLM
```

---

### Rule 2: Strict Grounded Prompting

**Prompt Template:**

```
You are a legal contract analyst. Your task is to answer questions based ONLY on the provided context below.

CRITICAL RULES:
1. Do NOT use any external legal knowledge
2. Do NOT make assumptions beyond the context
3. If the context does not contain the answer, say: "The provided context does not contain sufficient information to answer this question."
4. Always cite the specific section or clause when making a statement

Context:
{retrieved_context}

Question:
{user_question}

Answer:
```

---

### Rule 3: Citation Enforcement

**Require the model to cite sources:**

```
Answer format:
1. Answer: [Your answer]
2. Source: [Section X of Act Y / Clause Z of Contract]
3. Confidence: [High/Medium/Low]
```

This forces the model to:

- Ground answers in retrieved context
- Provide traceable sources
- Indicate uncertainty

---

### Rule 4: Confidence Thresholding

**Implement confidence scoring:**

1. **Retrieval confidence** — cosine similarity score
2. **LLM confidence** — model's self-reported confidence
3. **Combined threshold** — only show answers above threshold

**Example:**

```python
if retrieval_score < 0.7 or llm_confidence == "Low":
    return "I am not confident in this answer. Please consult a legal expert."
```

---

### Rule 5: Dual-Check with Rule Engine

**Hybrid approach:**

1. **LLM analysis** — semantic understanding
2. **Rule engine** — deterministic checks

**Example for indemnity clause:**

- LLM: "This clause is one-sided"
- Rule engine: Checks for keywords like "shall indemnify", "hold harmless", "defend"
- If both agree → High confidence
- If mismatch → Flag for review

---

### Rule 6: User Feedback Loop

**Allow users to report incorrect answers:**

1. User flags wrong answer
2. System logs the query + retrieved context + LLM response
3. Human reviewer audits
4. Update prompt or fine-tune model

This creates a **continuous improvement loop**.

---

## 6️⃣ Production Deployment Checklist

- [ ] Vector DB is persistent and backed up
- [ ] Embeddings are pre-computed (not generated at runtime)
- [ ] Retrieval threshold is set (minimum relevance score)
- [ ] Prompt includes "no external knowledge" instruction
- [ ] Citation enforcement is enabled
- [ ] Confidence scoring is implemented
- [ ] User feedback mechanism is in place
- [ ] Legal disclaimer is shown to users
- [ ] Privacy tier handling is implemented (auto-delete for Tier 1)

---

## 7️⃣ Summary

| Component         | Strategy                                                  |
| ----------------- | --------------------------------------------------------- |
| **Architecture**  | Dual RAG (static legal + dynamic user contract)           |
| **Datasets**      | Load once → Clean → Embed → Store in Vector DB            |
| **Chunking**      | Section-level for Acts, clause-level for contracts        |
| **Embedding**     | `bge-small-en-v1.5` for 8GB RAM                           |
| **Hallucination** | Retrieval-first + grounded prompts + citation enforcement |

---

## Next Steps

1. Review [architecture.md](architecture.md) for overall system design
2. Review [datasets.md](datasets.md) for dataset details
3. Review [llm-models.md](llm-models.md) for model selection
4. Implement chunking pipeline (start with Acts dataset)
5. Set up vector DB (ChromaDB or PGVector)
6. Build ingestion script
7. Test retrieval quality
8. Implement grounded prompting
9. Deploy and iterate

---

**Remember:** Legal RAG is about **precision**, not just performance. Accuracy and traceability are paramount.
