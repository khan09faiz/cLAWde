# LLM Models & AI Strategy

## 1. Model Selection Overview

No fine-tuning is required initially. The strategy relies on:
- Strong prompting (structured, few-shot)
- Good chunking (clause-level segmentation)
- Hybrid RAG (contract + legal knowledge base)

---

## 2. Deployment Tiers

### 💰 Free Tier / Low Budget — Cloud API

| Component | Model | Notes |
|-----------|-------|-------|
| **Main LLM** | Groq API (LLaMA 3 70B) | Fast, free tier available |
| **Main LLM** (alt) | Mixtral 8x7B | Good reasoning, cost-effective |
| **Embedding** | bge-large-en | Run locally on CPU |
| **Embedding** (alt) | nomic-embed | Lightweight alternative |

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

| Component | Model | RAM Requirement |
|-----------|-------|-----------------|
| **Main LLM** | LLaMA 3 8B | ~5-6 GB |
| **Main LLM** (alt) | Mistral 7B | ~5-6 GB |
| **Main LLM** (alt) | Phi-3-mini | ~3-4 GB |
| **Embedding** | bge-small | Minimal |
| **Embedding** (alt) | nomic-embed-text | Minimal |

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

| Component | Model | Infrastructure |
|-----------|-------|---------------|
| **Main LLM** | Azure OpenAI GPT-4 class | Private VNet deployment |
| **Storage** | S3 encrypted bucket | AES-256 encryption |
| **Compute** | Dedicated inference cluster | Auto-scaling |

**Pros:**
- Highest quality reasoning
- Enterprise SLAs
- Full compliance support

---

## 3. Embedding Model Comparison

| Model | Size | Quality | Speed | Use Case |
|-------|------|---------|-------|----------|
| bge-large-en | 335M | High | Moderate | Production (cloud) |
| bge-small-en | 33M | Good | Fast | Local / 8GB RAM |
| nomic-embed-text | 137M | Good | Fast | Balanced option |

**Recommendation:**
- Use `bge-small-en` for local development and 8GB RAM setups
- Use `bge-large-en` or `nomic-embed-text` for cloud / production

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

## 6. Model Switching Architecture

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

## 7. Vector DB Strategy

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
