# LLM Models & AI Strategy

## 1. Model Selection Overview

No fine-tuning is required initially. The strategy relies on:

- Strong prompting (structured, few-shot)
- Good chunking (clause-level segmentation)
- Hybrid RAG (contract + legal knowledge base)

---

## 2. Deployment Tiers

### 💰 Free Tier / Low Budget — Cloud API

| Component           | Model                  | Notes                          |
| ------------------- | ---------------------- | ------------------------------ |
| **Main LLM**        | Groq API (LLaMA 3 70B) | Fast, free tier available      |
| **Main LLM** (alt)  | Mixtral 8x7B           | Good reasoning, cost-effective |
| **Embedding**       | bge-large-en           | Run locally on CPU             |
| **Embedding** (alt) | nomic-embed            | Lightweight alternative        |

**Pros:**

- Cheap or free
- Fast inference (Groq is extremely fast)
- Good reasoning quality

**Considerations:**

- Data leaves your machine (use with privacy Mode 1: auto-delete)
- Rate limits on free tier

---

### 🏠 Local Setup — 8GB RAM Compatible

Uses **Ollama** for local inference.

| Component           | Model            | RAM Requirement |
| ------------------- | ---------------- | --------------- |
| **Main LLM**        | LLaMA 3 8B       | ~5-6 GB         |
| **Main LLM** (alt)  | Mistral 7B       | ~5-6 GB         |
| **Main LLM** (alt)  | Phi-3-mini       | ~3-4 GB         |
| **Embedding**       | bge-small        | Minimal         |
| **Embedding** (alt) | nomic-embed-text | Minimal         |

**Compatibility:**

- Works on 8GB RAM (with swap space)
- CPU only (slower but functional)
- 16GB RAM is ideal for comfortable operation

**Pros:**

- Complete data privacy
- No API costs
- No internet required

**Cons:**

- Slower inference
- Lower reasoning quality vs 70B models

---

### 🚀 Enterprise Mode

| Component    | Model                       | Infrastructure          |
| ------------ | --------------------------- | ----------------------- |
| **Main LLM** | Azure OpenAI GPT-4 class    | Private VNet deployment |
| **Storage**  | S3 encrypted bucket         | AES-256 encryption      |
| **Compute**  | Dedicated inference cluster | Auto-scaling            |

**Pros:**

- Highest quality reasoning
- Enterprise SLAs
- Full compliance support

---

## 3. Embedding Model Comparison

| Model                                    | Size | Quality  | Speed     | Use Case                      |
| ---------------------------------------- | ---- | -------- | --------- | ----------------------------- |
| `BAAI/bge-large-en-v1.5`                 | 335M | High     | Moderate  | Production (cloud, 16GB+ RAM) |
| `BAAI/bge-small-en-v1.5`                 | 33M  | Good     | Fast      | **Recommended for 8GB RAM**   |
| `intfloat/e5-small-v2`                   | 33M  | Good     | Fast      | Alternative for 8GB RAM       |
| `sentence-transformers/all-MiniLM-L6-v2` | 22M  | Moderate | Very Fast | Fallback option               |
| `nomic-embed-text`                       | 137M | Good     | Fast      | Balanced (requires 12GB+ RAM) |

---

### ✅ Recommended for 8GB RAM: `BAAI/bge-small-en-v1.5`

**Why this model:**

- Small size (33M parameters)
- Good retrieval quality for legal text
- Fast CPU inference
- Used in production RAG systems
- Memory footprint: ~100-200 MB

**Installation:**

```bash
pip install sentence-transformers
```

**Implementation:**

```python
from sentence_transformers import SentenceTransformer

# Load model (runs on CPU by default)
model = SentenceTransformer('BAAI/bge-small-en-v1.5')

# Embed a single document
embedding = model.encode("This is a legal clause about indemnity.")

# Embed multiple documents (batched for efficiency)
texts = ["Clause 1", "Clause 2", "Clause 3"]
embeddings = model.encode(texts, batch_size=8, show_progress_bar=True)
```

---

### ❌ Do NOT Use These on 8GB RAM

**Avoid:**

- `bge-large-en` (1.3 GB memory, slow on CPU)
- `gte-large` (large memory footprint)
- `OpenAI text-embedding-ada-002` (requires API calls, not local)

**Why:**

- Will slow down inference significantly
- Risk of out-of-memory errors
- Poor user experience (long wait times)

---

### Model Selection by Tier

| Tier                 | Embedding Model     | Reason                           |
| -------------------- | ------------------- | -------------------------------- |
| **Free / 8GB RAM**   | `bge-small-en-v1.5` | Lightweight, CPU-friendly        |
| **Cloud / 16GB RAM** | `bge-large-en-v1.5` | Better quality, acceptable speed |
| **Enterprise**       | `nomic-embed-text`  | High quality, dedicated hardware |

---

### Embedding Best Practices

1. **Batch processing during ingestion**

   ```python
   # Process in batches to avoid memory issues
   batch_size = 32  # Adjust based on RAM
   for i in range(0, len(texts), batch_size):
       batch = texts[i:i+batch_size]
       embeddings = model.encode(batch)
       # Store embeddings in vector DB
   ```

2. **Normalize embeddings**

   ```python
   # BGE models benefit from normalization
   embeddings = model.encode(texts, normalize_embeddings=True)
   ```

3. **Cache embeddings**
   - Pre-compute embeddings during ingestion
   - Store in vector DB
   - Never re-embed in production runtime

---

## 4. OCR Engine

For scanned document text extraction:

- **Tesseract OCR** — Open source, widely supported
- **EasyOCR** — Python-native, good accuracy
- **PaddleOCR** — High accuracy, multi-language

**Recommendation:** Start with Tesseract, upgrade if accuracy is insufficient.

---

## 5. Prompting Strategy

### Clause Analysis Prompt Structure

```
System: You are a legal contract analyst. Analyze the following clause.
Context: Contract type: {type}, Jurisdiction: {jurisdiction}
Clause: {clause_text}

Provide:
1. Purpose of this clause
2. Plain English explanation
3. Who benefits (Party A or Party B)
4. Risk severity (Low/Medium/High)
5. What could go wrong
6. Specific concerns
```

### Key Prompting Principles

- Always provide contract type context
- Use structured output format (JSON)
- Include few-shot examples for consistency
- Chain analysis prompts for complex clauses
- Keep prompts modular and composable

---

## 6. Hallucination Reduction Strategies

**Critical for legal applications** — hallucinations can lead to incorrect legal advice.

### Rule 1: Retrieval First

**If no relevant context is retrieved:**

```python
if len(retrieved_chunks) == 0 or max_relevance_score < 0.7:
    return {
        "answer": "Insufficient legal context to answer this question.",
        "confidence": "N/A"
    }
```

---

### Rule 2: Strict Grounded Prompting

**Template:**

```
You are a legal contract analyst. Answer based ONLY on the provided context.

CRITICAL RULES:
1. Do NOT use external legal knowledge
2. Do NOT make assumptions beyond the context
3. If context is insufficient, say: "Insufficient legal context"
4. Always cite specific section/clause when making statements

Context:
{retrieved_context}

Question:
{user_question}

Answer (with citations):
```

---

### Rule 3: Citation Enforcement

**Require model to cite sources:**

```
Required format:
1. Answer: [Your response]
2. Source: [Section X of Act Y / Clause Z]
3. Confidence: [High/Medium/Low]
```

---

### Rule 4: Confidence Thresholding

**Multi-layer confidence:**

1. Retrieval score (cosine similarity)
2. LLM self-reported confidence
3. Combined threshold

**Implementation:**

```python
if retrieval_score < 0.7 or llm_confidence == "Low":
    return "Low confidence answer. Please consult a legal expert."
```

---

### Rule 5: Hybrid Approach (LLM + Rule Engine)

**Combine semantic understanding with deterministic checks:**

```python
# LLM analysis
llm_result = llm.analyze_clause(clause_text)

# Rule-based check
rule_result = rule_engine.check_patterns(clause_text)

# Cross-validate
if llm_result.risk != rule_result.risk:
    return {
        "risk": "Requires review",
        "note": "LLM and rule engine disagree"
    }
```

**For complete hallucination reduction strategies, see [rag-implementation-guide.md](rag-implementation-guide.md)**

---

## 7. Model Switching Architecture

Design the system to be **model-agnostic**:

```python
# Abstract interface
class LLMProvider:
    def analyze_clause(self, clause: str, context: dict) -> dict: ...
    def answer_question(self, question: str, context: str) -> str: ...
    def generate_summary(self, text: str) -> str: ...

# Implementations
class GroqProvider(LLMProvider): ...    # Cloud free tier
class OllamaProvider(LLMProvider): ...  # Local inference
class AzureProvider(LLMProvider): ...   # Enterprise
```

This allows seamless switching between tiers without changing application logic.

---

## 8. Vector DB Strategy

### ChromaDB (Recommended for MVP)

- Simple setup
- In-memory or persistent
- Good for single-user / small scale

### PGVector (Recommended for Production)

- Lives inside PostgreSQL
- Single database for everything
- Better for multi-user, production scale

### Chunking Strategy

- Chunk at clause boundaries (not arbitrary token windows)
- Include section headers as metadata
- Overlap: 1-2 sentences between chunks
- Target chunk size: 200-500 tokens
