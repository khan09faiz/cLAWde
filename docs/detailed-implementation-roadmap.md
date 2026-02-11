# Detailed Implementation Roadmap

## Overview

This document provides a **comprehensive, task-by-task breakdown** of the cLAWde implementation, integrating the RAG architecture strategy with the phase-wise plan. Each task includes specific technical requirements, dependencies, and acceptance criteria.

---

## Table of Contents

- [Phase 0: Pre-Development Setup](#phase-0-pre-development-setup)
- [Phase 1: Core Engine (Month 1)](#phase-1-core-engine-month-1)
- [Phase 2: RAG & Q&A (Month 2)](#phase-2-rag--qa-month-2)
- [Phase 3: Dashboard & Privacy (Month 3)](#phase-3-dashboard--privacy-month-3)
- [Phase 4: Differentiation (Month 4-5)](#phase-4-differentiation-month-4-5)
- [Phase 5: Premium & Enterprise (Month 5+)](#phase-5-premium--enterprise-month-5)

---

# Phase 0: Pre-Development Setup

**Timeline:** Week 0 (Before Month 1)
**Goal:** Set up development environment and validate technology choices

## Task 0.1: Development Environment Setup

**Priority:** P0
**Duration:** 1-2 days

### Subtasks

- [ ] **0.1.1** Install Python 3.10+ and verify installation

  ```bash
  python --version  # Should be 3.10+
  ```

- [ ] **0.1.2** Set up virtual environment

  ```bash
  python -m venv venv
  # Windows: venv\Scripts\activate
  # Linux/Mac: source venv/bin/activate
  ```

- [ ] **0.1.3** Install Node.js 18+ and npm/yarn

  ```bash
  node --version  # Should be 18+
  ```

- [ ] **0.1.4** Set up Git repository

  ```bash
  git init
  git branch -M main
  ```

- [ ] **0.1.5** Create project directory structure

  ```
  cLAWde/
  ├── backend/
  │   ├── app/
  │   ├── tests/
  │   └── requirements.txt
  ├── frontend/
  ├── scripts/
  │   ├── data_ingestion/
  │   └── embeddings/
  ├── docs/
  └── data/
      ├── raw/
      ├── processed/
      └── embeddings/
  ```

- [ ] **0.1.6** Set up code editor (VS Code recommended)
  - Install Python extension
  - Install Pylance
  - Install ESLint/Prettier for frontend

**Acceptance Criteria:**

- Python and Node.js versions verified
- Project structure created
- Git initialized

---

## Task 0.2: Technology Validation

**Priority:** P0
**Duration:** 2-3 days

### Subtasks

- [ ] **0.2.1** Test Groq API access

  ```python
  # Test script to verify Groq API
  from groq import Groq
  client = Groq(api_key="your-api-key")
  response = client.chat.completions.create(...)
  ```

- [ ] **0.2.2** Test Ollama local setup (alternative)

  ```bash
  ollama pull llama3:8b
  ollama run llama3:8b "Test message"
  ```

- [ ] **0.2.3** Test embedding model locally

  ```python
  from sentence_transformers import SentenceTransformer
  model = SentenceTransformer('BAAI/bge-small-en-v1.5')
  embedding = model.encode("Test legal clause")
  print(f"Embedding shape: {embedding.shape}")
  ```

- [ ] **0.2.4** Test ChromaDB setup

  ```python
  import chromadb
  client = chromadb.Client()
  collection = client.create_collection("test")
  # Basic insert and query test
  ```

- [ ] **0.2.5** Test OCR engine (Tesseract)
  ```bash
  tesseract --version
  # Test on sample PDF
  ```

**Acceptance Criteria:**

- LLM API (Groq or Ollama) responding correctly
- Embedding model generating vectors
- ChromaDB CRUD operations working
- OCR extracting text from sample PDF

---

## Task 0.3: Dataset Preparation (Initial)

**Priority:** P1
**Duration:** 2-3 days

### Subtasks

- [ ] **0.3.1** Download Indian Laws dataset from HuggingFace

  ```python
  from datasets import load_dataset
  dataset = load_dataset("mratanusarkar/Indian-Laws")
  ```

- [ ] **0.3.2** Download Indian Law Q&A dataset

  ```python
  dataset = load_dataset("viber1/indian-law-dataset")
  ```

- [ ] **0.3.3** Create sample contract PDFs for testing (3-5 files)
  - Employment agreement
  - Service contract
  - NDA
  - Freelance agreement

- [ ] **0.3.4** Store raw datasets in `data/raw/` folder

**Acceptance Criteria:**

- Raw datasets downloaded and stored
- Sample contracts ready for testing
- Dataset structure documented

---

# Phase 1: Core Engine (Month 1)

**Timeline:** Weeks 1-4
**Goal:** Functional contract analyzer that processes documents and provides clause-by-clause analysis

---

## Week 1: Backend Foundation & Document Ingestion

### Task 1.1: FastAPI Project Setup

**Priority:** P0
**Duration:** 2 days

#### Subtasks

- [ ] **1.1.1** Initialize FastAPI project

  ```python
  # backend/app/main.py
  from fastapi import FastAPI
  app = FastAPI(title="cLAWde API", version="0.1.0")
  ```

- [ ] **1.1.2** Install core dependencies

  ```txt
  # requirements.txt
  fastapi==0.104.1
  uvicorn[standard]==0.24.0
  pydantic==2.5.0
  python-multipart==0.0.6
  ```

- [ ] **1.1.3** Set up project structure

  ```
  backend/app/
  ├── main.py
  ├── api/
  │   ├── __init__.py
  │   └── v1/
  │       ├── __init__.py
  │       └── endpoints/
  │           ├── contracts.py
  │           └── health.py
  ├── core/
  │   ├── __init__.py
  │   ├── config.py
  │   └── dependencies.py
  ├── models/
  │   ├── __init__.py
  │   └── schemas.py
  ├── services/
  │   ├── __init__.py
  │   ├── document_processor.py
  │   ├── clause_extractor.py
  │   └── llm_service.py
  └── utils/
      ├── __init__.py
      └── file_utils.py
  ```

- [ ] **1.1.4** Create configuration management

  ```python
  # core/config.py
  from pydantic_settings import BaseSettings

  class Settings(BaseSettings):
      API_V1_STR: str = "/api/v1"
      PROJECT_NAME: str = "cLAWde"
      GROQ_API_KEY: str
      MAX_FILE_SIZE: int = 10_000_000  # 10MB

      class Config:
          env_file = ".env"
  ```

- [ ] **1.1.5** Create health check endpoint

  ```python
  # api/v1/endpoints/health.py
  @router.get("/health")
  async def health_check():
      return {"status": "healthy", "version": "0.1.0"}
  ```

- [ ] **1.1.6** Test FastAPI server startup
  ```bash
  uvicorn app.main:app --reload
  # Visit http://localhost:8000/docs
  ```

**Acceptance Criteria:**

- FastAPI server starts without errors
- `/health` endpoint returns 200
- Swagger docs accessible at `/docs`
- Project structure organized and logical

---

### Task 1.2: File Upload Endpoint

**Priority:** P0
**Duration:** 2 days

#### Subtasks

- [ ] **1.2.1** Create file upload endpoint

  ```python
  # api/v1/endpoints/contracts.py
  from fastapi import UploadFile, File

  @router.post("/contracts/upload")
  async def upload_contract(
      file: UploadFile = File(...)
  ):
      # Validate file type
      # Save temporarily
      # Return file ID
  ```

- [ ] **1.2.2** Implement file type validation
  - Accept: `.pdf`, `.docx`, `.txt`
  - Reject: Other formats
  - Size limit: 10MB

- [ ] **1.2.3** Create file storage utility

  ```python
  # utils/file_utils.py
  import uuid
  from pathlib import Path

  def save_uploaded_file(file: UploadFile) -> str:
      file_id = str(uuid.uuid4())
      file_path = Path(f"uploads/{file_id}_{file.filename}")
      # Save file
      return file_id
  ```

- [ ] **1.2.4** Define Pydantic schemas

  ```python
  # models/schemas.py
  from pydantic import BaseModel

  class FileUploadResponse(BaseModel):
      file_id: str
      filename: str
      size: int
      content_type: str
  ```

- [ ] **1.2.5** Add error handling
  - Invalid file type → 400 error
  - File too large → 413 error
  - Server error → 500 error

- [ ] **1.2.6** Test with Postman/curl
  ```bash
  curl -X POST "http://localhost:8000/api/v1/contracts/upload" \
    -F "file=@sample_contract.pdf"
  ```

**Acceptance Criteria:**

- PDF, DOCX, TXT files upload successfully
- Invalid files rejected with proper error messages
- File saved to disk with unique ID
- Response includes file metadata

---

### Task 1.3: Text Extraction Pipeline

**Priority:** P0
**Duration:** 3 days

#### Subtasks

- [ ] **1.3.1** Install text extraction libraries

  ```txt
  # requirements.txt additions
  pypdf==3.17.0
  python-docx==1.1.0
  pytesseract==0.3.10
  pdf2image==1.16.3
  Pillow==10.1.0
  ```

- [ ] **1.3.2** Create document processor service

  ```python
  # services/document_processor.py
  from typing import Optional

  class DocumentProcessor:
      def extract_text(self, file_path: str, file_type: str) -> str:
          if file_type == "pdf":
              return self._extract_from_pdf(file_path)
          elif file_type == "docx":
              return self._extract_from_docx(file_path)
          elif file_type == "txt":
              return self._extract_from_txt(file_path)
  ```

- [ ] **1.3.3** Implement PDF text extraction

  ```python
  def _extract_from_pdf(self, file_path: str) -> str:
      import pypdf
      text = ""
      with open(file_path, "rb") as f:
          reader = pypdf.PdfReader(f)
          for page in reader.pages:
              text += page.extract_text()
      return text
  ```

- [ ] **1.3.4** Implement DOCX text extraction

  ```python
  def _extract_from_docx(self, file_path: str) -> str:
      from docx import Document
      doc = Document(file_path)
      return "\n".join([para.text for para in doc.paragraphs])
  ```

- [ ] **1.3.5** Implement OCR fallback for scanned PDFs

  ```python
  def _is_scanned_pdf(self, file_path: str) -> bool:
      # Check if PDF has extractable text
      # If text extraction yields < 100 chars, likely scanned
      pass

  def _extract_with_ocr(self, file_path: str) -> str:
      from pdf2image import convert_from_path
      import pytesseract

      images = convert_from_path(file_path)
      text = ""
      for image in images:
          text += pytesseract.image_to_string(image)
      return text
  ```

- [ ] **1.3.6** Add text cleaning utilities

  ```python
  def clean_extracted_text(self, text: str) -> str:
      # Remove excessive whitespace
      # Fix common OCR errors
      # Normalize formatting
      import re
      text = re.sub(r'\n\s*\n', '\n\n', text)
      text = re.sub(r' +', ' ', text)
      return text.strip()
  ```

- [ ] **1.3.7** Update upload endpoint to extract text

  ```python
  @router.post("/contracts/upload")
  async def upload_contract(file: UploadFile = File(...)):
      file_id = save_uploaded_file(file)

      # Extract text
      processor = DocumentProcessor()
      text = processor.extract_text(file_path, file_type)

      # Store text with file_id
      return {"file_id": file_id, "text_length": len(text)}
  ```

- [ ] **1.3.8** Test with various document types
  - Native PDF (digital text)
  - Scanned PDF (OCR required)
  - DOCX with formatting
  - Plain TXT

**Acceptance Criteria:**

- Text extracted from native PDFs correctly
- OCR works on scanned PDFs
- DOCX and TXT extraction functional
- Extracted text cleaned and normalized
- Text length > 0 for all test documents

---

## Week 2: Clause Segmentation & LLM Integration

### Task 1.4: Clause Chunking/Segmentation

**Priority:** P0
**Duration:** 3 days

#### Subtasks

- [ ] **1.4.1** Install NLP libraries

  ```txt
  # requirements.txt additions
  spacy==3.7.2
  ```

  ```bash
  python -m spacy download en_core_web_sm
  ```

- [ ] **1.4.2** Create clause extractor service

  ```python
  # services/clause_extractor.py
  import spacy
  from typing import List, Dict

  class ClauseExtractor:
      def __init__(self):
          self.nlp = spacy.load("en_core_web_sm")

      def extract_clauses(self, text: str) -> List[Dict]:
          # Segment into clauses
          pass
  ```

- [ ] **1.4.3** Implement rule-based clause detection

  ```python
  def _detect_clause_boundaries(self, text: str) -> List[str]:
      # Split by common patterns:
      # - Numbered sections (1., 2., etc.)
      # - Lettered subsections (a), b), etc.)
      # - Headers (WHEREAS, TERMS, etc.)

      import re

      # Pattern for numbered clauses
      pattern = r'(\n\s*\d+\.|\n\s*\([a-z]\)|\n\s*[A-Z\s]{3,}:)'

      sections = re.split(pattern, text)

      # Combine patterns with following text
      clauses = []
      for i in range(0, len(sections)-1, 2):
          if i+1 < len(sections):
              clause = sections[i] + sections[i+1]
              clauses.append(clause.strip())

      return clauses
  ```

- [ ] **1.4.4** Implement semantic clause chunking

  ```python
  def _semantic_chunking(self, text: str) -> List[Dict]:
      doc = self.nlp(text)

      chunks = []
      current_chunk = []

      for sent in doc.sents:
          current_chunk.append(sent.text)

          # Chunk size target: 200-500 tokens
          if len(" ".join(current_chunk).split()) > 400:
              chunks.append({
                  "text": " ".join(current_chunk),
                  "start_char": current_chunk[0][0].idx,
                  "end_char": sent.end_char
              })
              current_chunk = []

      # Add remaining
      if current_chunk:
          chunks.append({"text": " ".join(current_chunk)})

      return chunks
  ```

- [ ] **1.4.5** Add clause metadata extraction

  ```python
  def extract_clauses(self, text: str) -> List[Dict]:
      clauses = self._detect_clause_boundaries(text)

      structured_clauses = []
      for idx, clause in enumerate(clauses):
          structured_clauses.append({
              "id": f"clause_{idx+1}",
              "text": clause,
              "length": len(clause),
              "clause_type": self._detect_clause_type(clause),
              "position": idx + 1
          })

      return structured_clauses
  ```

- [ ] **1.4.6** Implement basic clause type detection

  ```python
  def _detect_clause_type(self, clause_text: str) -> str:
      # Simple keyword matching
      text_lower = clause_text.lower()

      if "indemnif" in text_lower:
          return "Indemnity"
      elif "terminate" in text_lower or "termination" in text_lower:
          return "Termination"
      elif "payment" in text_lower or "fee" in text_lower:
          return "Payment"
      elif "confidential" in text_lower:
          return "Confidentiality"
      elif "liability" in text_lower:
          return "Liability"
      else:
          return "General"
  ```

- [ ] **1.4.7** Test clause extraction on sample contracts

**Acceptance Criteria:**

- Contract text segmented into distinct clauses
- Each clause has unique ID and metadata
- Clause types detected with reasonable accuracy
- Segmentation works for 3+ sample contracts

---

### Task 1.5: LLM Service Integration

**Priority:** P0
**Duration:** 3 days

#### Subtasks

- [ ] **1.5.1** Install LLM client library

  ```txt
  # requirements.txt additions
  groq==0.4.1  # or ollama-python for local
  ```

- [ ] **1.5.2** Create LLM service abstraction

  ```python
  # services/llm_service.py
  from abc import ABC, abstractmethod
  from typing import Dict, Any

  class LLMProvider(ABC):
      @abstractmethod
      def analyze_clause(self, clause: str, context: Dict) -> Dict:
          pass

      @abstractmethod
      def answer_question(self, question: str, context: str) -> str:
          pass
  ```

- [ ] **1.5.3** Implement Groq provider

  ```python
  from groq import Groq

  class GroqProvider(LLMProvider):
      def __init__(self, api_key: str):
          self.client = Groq(api_key=api_key)
          self.model = "llama-3.1-70b-versatile"

      def analyze_clause(self, clause: str, context: Dict) -> Dict:
          prompt = self._build_analysis_prompt(clause, context)

          response = self.client.chat.completions.create(
              model=self.model,
              messages=[
                  {"role": "system", "content": "You are a legal contract analyst."},
                  {"role": "user", "content": prompt}
              ],
              temperature=0.3,
              max_tokens=1000
          )

          return self._parse_response(response.choices[0].message.content)
  ```

- [ ] **1.5.4** Design clause analysis prompt template

  ```python
  def _build_analysis_prompt(self, clause: str, context: Dict) -> str:
      return f"""
  Analyze the following contract clause:

  Contract Type: {context.get('contract_type', 'General')}
  Jurisdiction: {context.get('jurisdiction', 'India')}

  Clause Text:
  {clause}

  Provide a structured analysis in JSON format:
  {{
    "purpose": "Brief purpose of this clause",
    "explanation": "Plain English explanation (2-3 sentences)",
    "beneficiary": "Party A / Party B / Mutual",
    "risk_level": "Low / Medium / High",
    "concerns": ["Specific concern 1", "Specific concern 2"],
    "flags": ["Any red flags or unusual terms"]
  }}

  Return ONLY valid JSON.
  """
  ```

- [ ] **1.5.5** Implement response parsing

  ```python
  import json

  def _parse_response(self, response_text: str) -> Dict:
      try:
          # Extract JSON from response
          import re
          json_match = re.search(r'\{.*\}', response_text, re.DOTALL)
          if json_match:
              return json.loads(json_match.group())
          else:
              return {"error": "Failed to parse LLM response"}
      except json.JSONDecodeError:
          return {"error": "Invalid JSON in LLM response"}
  ```

- [ ] **1.5.6** Add retry logic and error handling

  ```python
  from tenacity import retry, stop_after_attempt, wait_exponential

  @retry(stop=stop_after_attempt(3), wait=wait_exponential(min=1, max=10))
  def analyze_clause(self, clause: str, context: Dict) -> Dict:
      # Analysis logic with automatic retry
      pass
  ```

- [ ] **1.5.7** Test LLM integration with sample clauses

**Acceptance Criteria:**

- LLM provider successfully calls API
- Clause analysis returns structured JSON
- Error handling works for API failures
- Retry logic functional
- Response parsing handles edge cases

---

## Week 3: Risk Classification & Structured Output

### Task 1.6: Risk Classification Engine

**Priority:** P0
**Duration:** 2-3 days

#### Subtasks

- [ ] **1.6.1** Create risk scoring service

  ```python
  # services/risk_scorer.py
  from typing import Dict, List
  from enum import Enum

  class RiskLevel(str, Enum):
      LOW = "Low"
      MEDIUM = "Medium"
      HIGH = "High"
      CRITICAL = "Critical"

  class RiskScorer:
      def calculate_risk(
          self,
          clause: Dict,
          llm_analysis: Dict
      ) -> Dict:
          # Combine rule-based + LLM risk assessment
          pass
  ```

- [ ] **1.6.2** Implement rule-based risk patterns

  ```python
  RISK_PATTERNS = {
      "HIGH": [
          r"unlimited liability",
          r"unconditional obligation",
          r"perpetual",
          r"non-negotiable",
          r"sole discretion",
          r"no recourse"
      ],
      "MEDIUM": [
          r"indemnify",
          r"hold harmless",
          r"terminate.*without.*cause",
          r"exclusive jurisdiction"
      ]
  }

  def _rule_based_risk(self, clause_text: str) -> RiskLevel:
      import re
      text_lower = clause_text.lower()

      for pattern in RISK_PATTERNS["HIGH"]:
          if re.search(pattern, text_lower):
              return RiskLevel.HIGH

      for pattern in RISK_PATTERNS["MEDIUM"]:
          if re.search(pattern, text_lower):
              return RiskLevel.MEDIUM

      return RiskLevel.LOW
  ```

- [ ] **1.6.3** Combine rule-based and LLM risk scores

  ```python
  def calculate_risk(self, clause: Dict, llm_analysis: Dict) -> Dict:
      # Get rule-based risk
      rule_risk = self._rule_based_risk(clause['text'])

      # Get LLM risk
      llm_risk = llm_analysis.get('risk_level', 'Low')

      # Combine (take higher risk)
      risk_levels = ["Low", "Medium", "High", "Critical"]
      final_risk_idx = max(
          risk_levels.index(rule_risk),
          risk_levels.index(llm_risk)
      )

      return {
          "overall_risk": risk_levels[final_risk_idx],
          "rule_based_risk": rule_risk,
          "llm_risk": llm_risk,
          "risk_factors": self._identify_risk_factors(clause, llm_analysis)
      }
  ```

- [ ] **1.6.4** Identify specific risk factors

  ```python
  def _identify_risk_factors(self, clause: Dict, llm_analysis: Dict) -> List[str]:
      factors = []

      # From LLM
      if llm_analysis.get('concerns'):
          factors.extend(llm_analysis['concerns'])

      if llm_analysis.get('flags'):
          factors.extend(llm_analysis['flags'])

      # From rules
      text_lower = clause['text'].lower()
      if "unlimited" in text_lower:
          factors.append("Contains unlimited obligation")
      if "perpetual" in text_lower:
          factors.append("Perpetual terms detected")

      return list(set(factors))  # Remove duplicates
  ```

- [ ] **1.6.5** Calculate document-level risk score

  ```python
  def calculate_document_risk(self, clause_analyses: List[Dict]) -> Dict:
      risk_counts = {"Low": 0, "Medium": 0, "High": 0, "Critical": 0}

      for analysis in clause_analyses:
          risk = analysis['risk_assessment']['overall_risk']
          risk_counts[risk] += 1

      total = len(clause_analyses)

      # Weighted score
      weighted_score = (
          risk_counts["Low"] * 1 +
          risk_counts["Medium"] * 3 +
          risk_counts["High"] * 7 +
          risk_counts["Critical"] * 10
      ) / total if total > 0 else 0

      return {
          "overall_score": weighted_score,
          "risk_distribution": risk_counts,
          "overall_verdict": self._get_verdict(weighted_score)
      }

  def _get_verdict(self, score: float) -> str:
      if score < 2:
          return "Low Risk"
      elif score < 5:
          return "Medium Risk"
      else:
          return "High Risk"
  ```

**Acceptance Criteria:**

- Rule-based risk detection working
- LLM risk scores integrated
- Combined risk calculation functional
- Document-level risk score computed
- Risk factors clearly identified

---

### Task 1.7: Structured JSON Output

**Priority:** P0
**Duration:** 2 days

#### Subtasks

- [ ] **1.7.1** Design output schema

  ```python
  # models/schemas.py
  from pydantic import BaseModel
  from typing import List, Optional
  from datetime import datetime

  class ClauseAnalysis(BaseModel):
      clause_id: str
      text: str
      clause_type: str
      llm_analysis: Dict
      risk_assessment: Dict
      position: int

  class DocumentAnalysis(BaseModel):
      document_id: str
      filename: str
      analyzed_at: datetime
      clauses: List[ClauseAnalysis]
      document_risk: Dict
      metadata: Dict
  ```

- [ ] **1.7.2** Create analysis orchestrator

  ```python
  # services/analysis_orchestrator.py
  class AnalysisOrchestrator:
      def __init__(
          self,
          document_processor: DocumentProcessor,
          clause_extractor: ClauseExtractor,
          llm_service: LLMProvider,
          risk_scorer: RiskScorer
      ):
          self.document_processor = document_processor
          self.clause_extractor = clause_extractor
          self.llm_service = llm_service
          self.risk_scorer = risk_scorer

      async def analyze_document(
          self,
          file_path: str,
          file_type: str
      ) -> DocumentAnalysis:
          # Orchestrate entire analysis pipeline
          pass
  ```

- [ ] **1.7.3** Implement full analysis pipeline

  ```python
  async def analyze_document(self, file_path: str, file_type: str) -> DocumentAnalysis:
      # 1. Extract text
      text = self.document_processor.extract_text(file_path, file_type)

      # 2. Segment into clauses
      clauses = self.clause_extractor.extract_clauses(text)

      # 3. Analyze each clause
      clause_analyses = []
      for clause in clauses:
          # LLM analysis
          llm_result = self.llm_service.analyze_clause(
              clause['text'],
              context={"contract_type": "General", "jurisdiction": "India"}
          )

          # Risk scoring
          risk_result = self.risk_scorer.calculate_risk(clause, llm_result)

          clause_analyses.append(
              ClauseAnalysis(
                  clause_id=clause['id'],
                  text=clause['text'],
                  clause_type=clause['clause_type'],
                  llm_analysis=llm_result,
                  risk_assessment=risk_result,
                  position=clause['position']
              )
          )

      # 4. Calculate document-level risk
      doc_risk = self.risk_scorer.calculate_document_risk(clause_analyses)

      # 5. Return structured output
      return DocumentAnalysis(
          document_id=str(uuid.uuid4()),
          filename=os.path.basename(file_path),
          analyzed_at=datetime.now(),
          clauses=clause_analyses,
          document_risk=doc_risk,
          metadata={"total_clauses": len(clauses)}
      )
  ```

- [ ] **1.7.4** Create analysis endpoint

  ```python
  # api/v1/endpoints/contracts.py
  @router.post("/contracts/{file_id}/analyze", response_model=DocumentAnalysis)
  async def analyze_contract(file_id: str):
      # Get file path from file_id
      file_path = get_file_path(file_id)

      # Run analysis
      orchestrator = AnalysisOrchestrator(...)
      result = await orchestrator.analyze_document(file_path, "pdf")

      return result
  ```

- [ ] **1.7.5** Add progress tracking (optional but recommended)

  ```python
  from fastapi import BackgroundTasks

  @router.post("/contracts/{file_id}/analyze")
  async def analyze_contract(
      file_id: str,
      background_tasks: BackgroundTasks
  ):
      # Run analysis in background
      background_tasks.add_task(run_analysis, file_id)

      return {"status": "processing", "file_id": file_id}

  @router.get("/contracts/{file_id}/status")
  async def get_analysis_status(file_id: str):
      # Return analysis status
      pass
  ```

**Acceptance Criteria:**

- Full analysis pipeline executes end-to-end
- Structured JSON output matches schema
- All clause analyses included in response
- Document-level risk calculated
- `/analyze` endpoint returns proper response

---

## Week 4: Frontend & Integration

### Task 1.8: Basic Frontend Setup

**Priority:** P1
**Duration:** 3 days

#### Subtasks

- [ ] **1.8.1** Initialize Next.js project

  ```bash
  npx create-next-app@latest frontend --typescript --tailwind --app
  cd frontend
  ```

- [ ] **1.8.2** Install dependencies

  ```bash
  npm install axios react-dropzone lucide-react
  ```

- [ ] **1.8.3** Create project structure

  ```
  frontend/
  ├── app/
  │   ├── page.tsx          # Home page
  │   ├── analyze/
  │   │   └── page.tsx      # Analysis page
  │   └── layout.tsx
  ├── components/
  │   ├── FileUpload.tsx
  │   ├── AnalysisResults.tsx
  │   └── ClauseCard.tsx
  ├── lib/
  │   └── api.ts            # API client
  └── types/
      └── analysis.ts       # TypeScript types
  ```

- [ ] **1.8.4** Define TypeScript types

  ```typescript
  // types/analysis.ts
  export interface ClauseAnalysis {
    clause_id: string;
    text: string;
    clause_type: string;
    llm_analysis: {
      purpose: string;
      explanation: string;
      beneficiary: string;
      risk_level: string;
      concerns: string[];
      flags: string[];
    };
    risk_assessment: {
      overall_risk: string;
      risk_factors: string[];
    };
    position: number;
  }

  export interface DocumentAnalysis {
    document_id: string;
    filename: string;
    analyzed_at: string;
    clauses: ClauseAnalysis[];
    document_risk: {
      overall_score: number;
      risk_distribution: Record<string, number>;
      overall_verdict: string;
    };
  }
  ```

- [ ] **1.8.5** Create API client

  ```typescript
  // lib/api.ts
  import axios from "axios";

  const API_BASE =
    process.env.NEXT_PUBLIC_API_URL || "http://localhost:8000/api/v1";

  export const api = {
    uploadContract: async (file: File) => {
      const formData = new FormData();
      formData.append("file", file);

      const response = await axios.post(
        `${API_BASE}/contracts/upload`,
        formData,
      );
      return response.data;
    },

    analyzeContract: async (fileId: string) => {
      const response = await axios.post(
        `${API_BASE}/contracts/${fileId}/analyze`,
      );
      return response.data;
    },
  };
  ```

- [ ] **1.8.6** Create file upload component

  ```typescript
  // components/FileUpload.tsx
  'use client';

  import { useCallback } from 'react';
  import { useDropzone } from 'react-dropzone';
  import { Upload } from 'lucide-react';

  export default function FileUpload({ onUpload }: { onUpload: (file: File) => void }) {
    const onDrop = useCallback((acceptedFiles: File[]) => {
      if (acceptedFiles.length > 0) {
        onUpload(acceptedFiles[0]);
      }
    }, [onUpload]);

    const { getRootProps, getInputProps, isDragActive } = useDropzone({
      onDrop,
      accept: {
        'application/pdf': ['.pdf'],
        'application/vnd.openxmlformats-officedocument.wordprocessingml.document': ['.docx'],
        'text/plain': ['.txt']
      },
      maxSize: 10_000_000, // 10MB
      multiple: false
    });

    return (
      <div
        {...getRootProps()}
        className={`border-2 border-dashed rounded-lg p-12 text-center cursor-pointer transition-colors
          ${isDragActive ? 'border-blue-500 bg-blue-50' : 'border-gray-300 hover:border-gray-400'}`}
      >
        <input {...getInputProps()} />
        <Upload className="mx-auto h-12 w-12 text-gray-400" />
        <p className="mt-2 text-sm text-gray-600">
          {isDragActive ? 'Drop your contract here' : 'Drag & drop contract or click to browse'}
        </p>
        <p className="mt-1 text-xs text-gray-500">PDF, DOCX, or TXT (max 10MB)</p>
      </div>
    );
  }
  ```

**Acceptance Criteria:**

- Next.js project initialized
- File upload component renders
- API client configured
- TypeScript types match backend schema

---

### Task 1.9: Analysis Results UI

**Priority:** P1
**Duration:** 3 days

#### Subtasks

- [ ] **1.9.1** Create clause card component

  ```typescript
  // components/ClauseCard.tsx
  import { ClauseAnalysis } from '@/types/analysis';
  import { AlertTriangle, Info, CheckCircle } from 'lucide-react';

  const getRiskColor = (risk: string) => {
    switch (risk) {
      case 'High': return 'bg-red-100 border-red-300 text-red-800';
      case 'Medium': return 'bg-yellow-100 border-yellow-300 text-yellow-800';
      case 'Low': return 'bg-green-100 border-green-300 text-green-800';
      default: return 'bg-gray-100 border-gray-300 text-gray-800';
    }
  };

  const getRiskIcon = (risk: string) => {
    switch (risk) {
      case 'High': return <AlertTriangle className="h-5 w-5" />;
      case 'Medium': return <Info className="h-5 w-5" />;
      case 'Low': return <CheckCircle className="h-5 w-5" />;
      default: return null;
    }
  };

  export default function ClauseCard({ clause }: { clause: ClauseAnalysis }) {
    const riskLevel = clause.risk_assessment.overall_risk;

    return (
      <div className={`border rounded-lg p-4 ${getRiskColor(riskLevel)}`}>
        <div className="flex items-start justify-between mb-2">
          <div>
            <span className="text-xs font-semibold uppercase">{clause.clause_type}</span>
            <span className="ml-2 text-xs text-gray-600">Clause {clause.position}</span>
          </div>
          <div className="flex items-center gap-1">
            {getRiskIcon(riskLevel)}
            <span className="text-sm font-semibold">{riskLevel} Risk</span>
          </div>
        </div>

        <p className="text-sm mb-3 text-gray-700 italic">"{clause.text.substring(0, 150)}..."</p>

        <div className="bg-white bg-opacity-50 rounded p-3 space-y-2">
          <div>
            <span className="text-xs font-semibold">Explanation:</span>
            <p className="text-sm">{clause.llm_analysis.explanation}</p>
          </div>

          {clause.risk_assessment.risk_factors.length > 0 && (
            <div>
              <span className="text-xs font-semibold">Risk Factors:</span>
              <ul className="text-sm list-disc list-inside">
                {clause.risk_assessment.risk_factors.map((factor, idx) => (
                  <li key={idx}>{factor}</li>
                ))}
              </ul>
            </div>
          )}

          <div>
            <span className="text-xs font-semibold">Beneficiary:</span>
            <span className="text-sm ml-1">{clause.llm_analysis.beneficiary}</span>
          </div>
        </div>
      </div>
    );
  }
  ```

- [ ] **1.9.2** Create analysis results component

  ```typescript
  // components/AnalysisResults.tsx
  import { DocumentAnalysis } from '@/types/analysis';
  import ClauseCard from './ClauseCard';

  export default function AnalysisResults({ analysis }: { analysis: DocumentAnalysis }) {
    return (
      <div className="max-w-4xl mx-auto p-6">
        <div className="bg-white rounded-lg shadow-lg p-6 mb-6">
          <h2 className="text-2xl font-bold mb-2">{analysis.filename}</h2>
          <p className="text-sm text-gray-600 mb-4">
            Analyzed on {new Date(analysis.analyzed_at).toLocaleString()}
          </p>

          <div className="grid grid-cols-3 gap-4 mb-4">
            <div className="bg-gray-50 rounded p-4">
              <p className="text-xs text-gray-600">Overall Verdict</p>
              <p className="text-lg font-bold">{analysis.document_risk.overall_verdict}</p>
            </div>

            <div className="bg-gray-50 rounded p-4">
              <p className="text-xs text-gray-600">Risk Score</p>
              <p className="text-lg font-bold">{analysis.document_risk.overall_score.toFixed(1)}/10</p>
            </div>

            <div className="bg-gray-50 rounded p-4">
              <p className="text-xs text-gray-600">Total Clauses</p>
              <p className="text-lg font-bold">{analysis.clauses.length}</p>
            </div>
          </div>

          <div className="grid grid-cols-4 gap-2 text-center text-sm">
            <div>
              <p className="text-green-600 font-bold">{analysis.document_risk.risk_distribution.Low}</p>
              <p className="text-xs text-gray-600">Low Risk</p>
            </div>
            <div>
              <p className="text-yellow-600 font-bold">{analysis.document_risk.risk_distribution.Medium}</p>
              <p className="text-xs text-gray-600">Medium Risk</p>
            </div>
            <div>
              <p className="text-red-600 font-bold">{analysis.document_risk.risk_distribution.High}</p>
              <p className="text-xs text-gray-600">High Risk</p>
            </div>
            <div>
              <p className="text-red-800 font-bold">{analysis.document_risk.risk_distribution.Critical || 0}</p>
              <p className="text-xs text-gray-600">Critical</p>
            </div>
          </div>
        </div>

        <h3 className="text-xl font-bold mb-4">Clause Analysis</h3>
        <div className="space-y-4">
          {analysis.clauses.map((clause) => (
            <ClauseCard key={clause.clause_id} clause={clause} />
          ))}
        </div>
      </div>
    );
  }
  ```

- [ ] **1.9.3** Create main analysis page

  ```typescript
  // app/page.tsx
  'use client';

  import { useState } from 'react';
  import FileUpload from '@/components/FileUpload';
  import AnalysisResults from '@/components/AnalysisResults';
  import { api } from '@/lib/api';
  import { DocumentAnalysis } from '@/types/analysis';

  export default function Home() {
    const [isAnalyzing, setIsAnalyzing] = useState(false);
    const [analysis, setAnalysis] = useState<DocumentAnalysis | null>(null);
    const [error, setError] = useState<string | null>(null);

    const handleUpload = async (file: File) => {
      setIsAnalyzing(true);
      setError(null);

      try {
        // Upload file
        const uploadResult = await api.uploadContract(file);
        const fileId = uploadResult.file_id;

        // Analyze
        const analysisResult = await api.analyzeContract(fileId);
        setAnalysis(analysisResult);
      } catch (err: any) {
        setError(err.message || 'Analysis failed');
      } finally {
        setIsAnalyzing(false);
      }
    };

    return (
      <main className="min-h-screen bg-gray-50 py-12">
        <div className="max-w-4xl mx-auto px-4">
          <h1 className="text-4xl font-bold text-center mb-2">cLAWde</h1>
          <p className="text-center text-gray-600 mb-8">AI-Powered Contract Analysis</p>

          {!analysis && (
            <FileUpload onUpload={handleUpload} />
          )}

          {isAnalyzing && (
            <div className="text-center py-12">
              <div className="animate-spin rounded-full h-12 w-12 border-b-2 border-blue-600 mx-auto"></div>
              <p className="mt-4 text-gray-600">Analyzing your contract...</p>
            </div>
          )}

          {error && (
            <div className="bg-red-50 border border-red-200 rounded-lg p-4 text-red-800">
              Error: {error}
            </div>
          )}

          {analysis && (
            <>
              <button
                onClick={() => setAnalysis(null)}
                className="mb-4 text-blue-600 hover:underline"
              >
                ← Analyze another contract
              </button>
              <AnalysisResults analysis={analysis} />
            </>
          )}
        </div>
      </main>
    );
  }
  ```

- [ ] **1.9.4** Test frontend with backend integration

**Acceptance Criteria:**

- File upload component functional
- Analysis results display correctly
- Risk colors and icons appropriate
- Responsive design works on mobile/desktop
- Error handling displays properly

---

### Task 1.10: Phase 1 Testing & Documentation

**Priority:** P0
**Duration:** 2 days

#### Subtasks

- [ ] **1.10.1** End-to-end testing
  - Upload PDF → Get analysis
  - Upload DOCX → Get analysis
  - Upload scanned PDF → OCR works
  - Invalid file → Error handling

- [ ] **1.10.2** API documentation
  - Document all endpoints in Swagger
  - Add request/response examples
  - Include error codes

- [ ] **1.10.3** Write README

  ````markdown
  # cLAWde - Phase 1

  ## Setup

  ### Backend

  ```bash
  cd backend
  python -m venv venv
  source venv/bin/activate  # or venv\Scripts\activate on Windows
  pip install -r requirements.txt
  uvicorn app.main:app --reload
  ```
  ````

  ### Frontend

  ```bash
  cd frontend
  npm install
  npm run dev
  ```

  ## Features
  - File upload (PDF, DOCX, TXT)
  - Text extraction with OCR
  - Clause segmentation
  - LLM-powered analysis
  - Risk scoring
  - Clean UI

  ## API Endpoints
  - POST /api/v1/contracts/upload
  - POST /api/v1/contracts/{id}/analyze

  ```

  ```

- [ ] **1.10.4** Performance testing
  - Test with 10-page contract
  - Test with 50-page contract
  - Measure analysis time
  - Check memory usage

- [ ] **1.10.5** Bug fixes and polish

**Acceptance Criteria:**

- All tests passing
- API documentation complete
- README written
- Performance acceptable (<60s for 10-page contract)

---

# Phase 2: RAG & Q&A (Month 2)

**Timeline:** Weeks 5-8
**Goal:** Smart interactive system with contract-specific Q&A and legal knowledge base

---

## Week 5: Vector DB & Embedding Infrastructure

### Task 2.1: Vector Database Setup

**Priority:** P0
**Duration:** 2 days

#### Subtasks

- [ ] **2.1.1** Install ChromaDB

  ```txt
  # requirements.txt additions
  chromadb==0.4.18
  chromadb-client==0.4.18
  ```

- [ ] **2.1.2** Create vector DB service

  ```python
  # services/vector_db_service.py
  import chromadb
  from chromadb.config import Settings
  from typing import List, Dict

  class VectorDBService:
      def __init__(self, persist_directory: str = "./chroma_db"):
          self.client = chromadb.Client(Settings(
              persist_directory=persist_directory,
              anonymized_telemetry=False
          ))

      def get_or_create_collection(self, name: str):
          return self.client.get_or_create_collection(
              name=name,
              metadata={"hnsw:space": "cosine"}
          )
  ```

- [ ] **2.1.3** Create separate collections

  ```python
  def setup_collections(self):
      # Static legal knowledge base
      self.legal_kb_collection = self.get_or_create_collection("legal_knowledge_base")

      # User contracts (will be per-user in Phase 3)
      self.contracts_collection = self.get_or_create_collection("user_contracts")
  ```

- [ ] **2.1.4** Implement add/query methods

  ```python
  def add_documents(
      self,
      collection_name: str,
      documents: List[str],
      metadatas: List[Dict],
      ids: List[str]
  ):
      collection = self.get_or_create_collection(collection_name)
      collection.add(
          documents=documents,
          metadatas=metadatas,
          ids=ids
      )

  def query_documents(
      self,
      collection_name: str,
      query_text: str,
      n_results: int = 5,
      where: Dict = None
  ):
      collection = self.get_or_create_collection(collection_name)
      results = collection.query(
          query_texts=[query_text],
          n_results=n_results,
          where=where
      )
      return results
  ```

- [ ] **2.1.5** Test basic CRUD operations

**Acceptance Criteria:**

- ChromaDB initialized successfully
- Collections created
- Documents can be added
- Queries return relevant results

---

### Task 2.2: Embedding Pipeline

**Priority:** P0
**Duration:** 2 days

#### Subtasks

- [ ] **2.2.1** Install embedding model

  ```txt
  # requirements.txt additions
  sentence-transformers==2.2.2
  torch==2.1.0
  ```

- [ ] **2.2.2** Create embedding service

  ```python
  # services/embedding_service.py
  from sentence_transformers import SentenceTransformer
  from typing import List, Union
  import numpy as np

  class EmbeddingService:
      def __init__(self, model_name: str = "BAAI/bge-small-en-v1.5"):
          self.model = SentenceTransformer(model_name)
          self.model_name = model_name

      def embed_text(self, text: str) -> np.ndarray:
          return self.model.encode(text, normalize_embeddings=True)

      def embed_batch(self, texts: List[str], batch_size: int = 32) -> np.ndarray:
          return self.model.encode(
              texts,
              batch_size=batch_size,
              normalize_embeddings=True,
              show_progress_bar=True
          )
  ```

- [ ] **2.2.3** Integrate embeddings with ChromaDB

  ```python
  # ChromaDB handles embeddings automatically, but we can use custom embedding function

  from chromadb.utils import embedding_functions

  def setup_collections_with_embeddings(self):
      # Create custom embedding function
      embedding_fn = embedding_functions.SentenceTransformerEmbeddingFunction(
          model_name="BAAI/bge-small-en-v1.5"
      )

      self.legal_kb_collection = self.client.get_or_create_collection(
          name="legal_knowledge_base",
          embedding_function=embedding_fn
      )
  ```

- [ ] **2.2.4** Test embedding generation

  ```python
  # Test script
  embedding_service = EmbeddingService()

  text = "This clause governs the indemnity obligations"
  embedding = embedding_service.embed_text(text)

  assert embedding.shape == (384,)  # bge-small-en-v1.5 dimension
  print(f"Embedding generated successfully: shape {embedding.shape}")
  ```

**Acceptance Criteria:**

- Embedding model loads successfully
- Single text embedding works
- Batch embedding functional
- Integration with ChromaDB verified

---

## Week 6: Legal Knowledge Base Ingestion

### Task 2.3: Dataset Cleaning & Chunking

**Priority:** P0
**Duration:** 4 days

#### Subtasks

- [ ] **2.3.1** Create data ingestion script structure

  ```
  scripts/data_ingestion/
  ├── __init__.py
  ├── clean_indian_laws.py
  ├── clean_judgments.py
  ├── clean_qa_dataset.py
  ├── chunk_acts.py
  ├── chunk_judgments.py
  └── ingest_to_vectordb.py
  ```

- [ ] **2.3.2** Clean Indian Laws dataset

  ```python
  # scripts/data_ingestion/clean_indian_laws.py
  from datasets import load_dataset
  import json
  from pathlib import Path

  def clean_indian_laws():
      # Load dataset
      dataset = load_dataset("mratanusarkar/Indian-Laws")

      cleaned_data = []
      for item in dataset['train']:
          cleaned_data.append({
              "act_name": item['act_title'].strip(),
              "section": item['section'].strip() if item['section'] else "",
              "content": item['law'].strip(),
              "chunk_type": "statute"
          })

      # Save as JSONL
      output_path = Path("data/processed/indian_laws.jsonl")
      output_path.parent.mkdir(parents=True, exist_ok=True)

      with open(output_path, 'w', encoding='utf-8') as f:
          for item in cleaned_data:
              f.write(json.dumps(item, ensure_ascii=False) + '\n')

      print(f"Saved {len(cleaned_data)} acts to {output_path}")

  if __name__ == "__main__":
      clean_indian_laws()
  ```

- [ ] **2.3.3** Chunk Acts dataset (section-level)

  ```python
  # scripts/data_ingestion/chunk_acts.py
  import json
  from pathlib import Path
  from typing import List, Dict

  def chunk_acts() -> List[Dict]:
      input_path = Path("data/processed/indian_laws.jsonl")

      chunks = []
      with open(input_path, 'r', encoding='utf-8') as f:
          for line in f:
              item = json.loads(line)

              # Each section is already a chunk
              # If section too large (>1000 tokens), split further
              if len(item['content'].split()) > 1000:
                  # Split into subsections
                  subsection_chunks = split_long_section(item)
                  chunks.extend(subsection_chunks)
              else:
                  # Add metadata
                  chunk = {
                      "id": f"{item['act_name']}_{item['section']}".replace(" ", "_"),
                      "text": f"Act: {item['act_name']}\nSection: {item['section']}\n\n{item['content']}",
                      "metadata": {
                          "act_name": item['act_name'],
                          "section": item['section'],
                          "chunk_type": "statute",
                          "source": "Indian-Laws-HF"
                      }
                  }
                  chunks.append(chunk)

      # Save chunks
      output_path = Path("data/processed/acts_chunks.jsonl")
      with open(output_path, 'w', encoding='utf-8') as f:
          for chunk in chunks:
              f.write(json.dumps(chunk, ensure_ascii=False) + '\n')

      print(f"Created {len(chunks)} chunks from acts")
      return chunks

  def split_long_section(item: Dict) -> List[Dict]:
      # Split by paragraphs or subsections
      content = item['content']
      paragraphs = content.split('\n\n')

      subsection_chunks = []
      current_chunk = []
      current_length = 0

      for para in paragraphs:
          para_length = len(para.split())

          if current_length + para_length > 800 and current_chunk:
              # Save current chunk
              subsection_chunks.append({
                  "id": f"{item['act_name']}_{item['section']}_part{len(subsection_chunks)+1}".replace(" ", "_"),
                  "text": f"Act: {item['act_name']}\nSection: {item['section']}\n\n" + "\n\n".join(current_chunk),
                  "metadata": {
                      "act_name": item['act_name'],
                      "section": item['section'],
                      "chunk_type": "statute",
                      "source": "Indian-Laws-HF"
                  }
              })
              current_chunk = []
              current_length = 0

          current_chunk.append(para)
          current_length += para_length

      # Add remaining
      if current_chunk:
          subsection_chunks.append({
              "id": f"{item['act_name']}_{item['section']}_part{len(subsection_chunks)+1}".replace(" ", "_"),
              "text": f"Act: {item['act_name']}\nSection: {item['section']}\n\n" + "\n\n".join(current_chunk),
              "metadata": {
                  "act_name": item['act_name'],
                  "section": item['section'],
                  "chunk_type": "statute",
                  "source": "Indian-Laws-HF"
              }
          })

      return subsection_chunks

  if __name__ == "__main__":
      chunk_acts()
  ```

- [ ] **2.3.4** Clean and chunk Q&A dataset

  ```python
  # scripts/data_ingestion/clean_qa_dataset.py
  from datasets import load_dataset
  import json
  from pathlib import Path

  def clean_qa_dataset():
      dataset = load_dataset("viber1/indian-law-dataset")

      chunks = []
      for idx, item in enumerate(dataset['train']):
          chunk = {
              "id": f"qa_{idx}",
              "text": f"Question: {item['instruction']}\n\nAnswer: {item['response']}",
              "metadata": {
                  "question": item['instruction'],
                  "answer": item['response'],
                  "chunk_type": "qa_pair",
                  "source": "indian-law-dataset-HF"
              }
          }
          chunks.append(chunk)

      # Save
      output_path = Path("data/processed/qa_chunks.jsonl")
      with open(output_path, 'w', encoding='utf-8') as f:
          for chunk in chunks:
              f.write(json.dumps(chunk, ensure_ascii=False) + '\n')

      print(f"Created {len(chunks)} Q&A chunks")

  if __name__ == "__main__":
      clean_qa_dataset()
  ```

- [ ] **2.3.5** Run all cleaning scripts
  ```bash
  python scripts/data_ingestion/clean_indian_laws.py
  python scripts/data_ingestion/chunk_acts.py
  python scripts/data_ingestion/clean_qa_dataset.py
  ```

**Acceptance Criteria:**

- Raw datasets cleaned and saved as JSONL
- Acts chunked at section level
- Long sections split appropriately
- Q&A dataset processed
- All chunks have proper metadata

---

### Task 2.4: Vector DB Ingestion

**Priority:** P0
**Duration:** 2 days

#### Subtasks

- [ ] **2.4.1** Create ingestion script

  ```python
  # scripts/data_ingestion/ingest_to_vectordb.py
  import json
  from pathlib import Path
  import sys
  sys.path.append("../../backend")

  from app.services.vector_db_service import VectorDBService
  from tqdm import tqdm

  def ingest_chunks(chunk_file: Path, collection_name: str):
      # Load chunks
      chunks = []
      with open(chunk_file, 'r', encoding='utf-8') as f:
          for line in f:
              chunks.append(json.loads(line))

      print(f"Loaded {len(chunks)} chunks from {chunk_file}")

      # Initialize vector DB
      vector_db = VectorDBService()

      # Batch insert
      batch_size = 100
      for i in tqdm(range(0, len(chunks), batch_size)):
          batch = chunks[i:i+batch_size]

          documents = [chunk['text'] for chunk in batch]
          metadatas = [chunk['metadata'] for chunk in batch]
          ids = [chunk['id'] for chunk in batch]

          vector_db.add_documents(
              collection_name=collection_name,
              documents=documents,
              metadatas=metadatas,
              ids=ids
          )

      print(f"Ingested {len(chunks)} chunks into {collection_name}")

  def main():
      # Ingest Acts
      ingest_chunks(
          Path("data/processed/acts_chunks.jsonl"),
          "legal_knowledge_base"
      )

      # Ingest Q&A
      ingest_chunks(
          Path("data/processed/qa_chunks.jsonl"),
          "legal_knowledge_base"
      )

      print("Ingestion complete!")

  if __name__ == "__main__":
      main()
  ```

- [ ] **2.4.2** Run ingestion

  ```bash
  python scripts/data_ingestion/ingest_to_vectordb.py
  ```

- [ ] **2.4.3** Verify ingestion

  ```python
  # Test query
  vector_db = VectorDBService()
  results = vector_db.query_documents(
      collection_name="legal_knowledge_base",
      query_text="What is the remedy for breach of contract?",
      n_results=5
  )

  print("Top 5 results:")
  for doc, metadata in zip(results['documents'][0], results['metadatas'][0]):
      print(f"\n{metadata['chunk_type']}: {doc[:200]}...")
  ```

**Acceptance Criteria:**

- All chunks ingested into vector DB
- No errors during ingestion
- Queries return relevant results
- Metadata preserved correctly

---

## Week 7: Contract-Specific RAG

### Task 2.5: User Contract Embedding

**Priority:** P0
**Duration:** 2 days

#### Subtasks

- [ ] **2.5.1** Update document upload to embed clauses

  ```python
  # services/contract_rag_service.py
  from typing import List, Dict
  import uuid

  class ContractRAGService:
      def __init__(
          self,
          vector_db: VectorDBService,
          clause_extractor: ClauseExtractor
      ):
          self.vector_db = vector_db
          self.clause_extractor = clause_extractor

      def index_contract(
          self,
          contract_id: str,
          contract_text: str
      ) -> Dict:
          # Extract clauses
          clauses = self.clause_extractor.extract_clauses(contract_text)

          # Prepare for embedding
          documents = []
          metadatas = []
          ids = []

          for clause in clauses:
              doc_id = f"{contract_id}_clause_{clause['id']}"

              documents.append(clause['text'])
              metadatas.append({
                  "contract_id": contract_id,
                  "clause_id": clause['id'],
                  "clause_type": clause['clause_type'],
                  "position": clause['position'],
                  "chunk_type": "user_contract"
              })
              ids.append(doc_id)

          # Add to vector DB
          self.vector_db.add_documents(
              collection_name="user_contracts",
              documents=documents,
              metadatas=metadatas,
              ids=ids
          )

          return {
              "contract_id": contract_id,
              "clauses_indexed": len(clauses)
          }
  ```

- [ ] **2.5.2** Update upload endpoint

  ```python
  # api/v1/endpoints/contracts.py
  @router.post("/contracts/upload")
  async def upload_contract(file: UploadFile = File(...)):
      # Save file
      file_id = save_uploaded_file(file)

      # Extract text
      processor = DocumentProcessor()
      text = processor.extract_text(file_path, file_type)

      # Index in RAG
      rag_service = ContractRAGService(vector_db, clause_extractor)
      index_result = rag_service.index_contract(file_id, text)

      return {
          "file_id": file_id,
          "clauses_indexed": index_result['clauses_indexed']
      }
  ```

- [ ] **2.5.3** Test contract indexing

**Acceptance Criteria:**

- Contract clauses embedded on upload
- Each clause stored with metadata
- Contract ID linked to all clauses
- Indexing completes without errors

---

### Task 2.6: Hybrid Retrieval System

**Priority:** P0
**Duration:** 3 days

#### Subtasks

- [ ] **2.6.1** Create RAG retrieval service

  ```python
  # services/rag_retrieval_service.py
  from typing import List, Dict, Tuple
  from enum import Enum

  class QueryType(str, Enum):
      CONTRACT_SPECIFIC = "contract"
      LEGAL_KNOWLEDGE = "legal"
      HYBRID = "hybrid"

  class RAGRetrievalService:
      def __init__(self, vector_db: VectorDBService):
          self.vector_db = vector_db

      def retrieve_context(
          self,
          query: str,
          contract_id: str = None,
          query_type: QueryType = QueryType.HYBRID,
          n_results: int = 5
      ) -> Dict:
          """Retrieve relevant context for a query"""

          if query_type == QueryType.CONTRACT_SPECIFIC:
              return self._retrieve_from_contract(query, contract_id, n_results)

          elif query_type == QueryType.LEGAL_KNOWLEDGE:
              return self._retrieve_from_legal_kb(query, n_results)

          else:  # HYBRID
              return self._hybrid_retrieval(query, contract_id, n_results)
  ```

- [ ] **2.6.2** Implement contract-specific retrieval

  ```python
  def _retrieve_from_contract(
      self,
      query: str,
      contract_id: str,
      n_results: int
  ) -> Dict:
      results = self.vector_db.query_documents(
          collection_name="user_contracts",
          query_text=query,
          n_results=n_results,
          where={"contract_id": contract_id}
      )

      return {
          "source": "user_contract",
          "chunks": self._format_results(results)
      }
  ```

- [ ] **2.6.3** Implement legal knowledge retrieval

  ```python
  def _retrieve_from_legal_kb(
      self,
      query: str,
      n_results: int
  ) -> Dict:
      results = self.vector_db.query_documents(
          collection_name="legal_knowledge_base",
          query_text=query,
          n_results=n_results
      )

      return {
          "source": "legal_knowledge",
          "chunks": self._format_results(results)
      }
  ```

- [ ] **2.6.4** Implement hybrid retrieval

  ```python
  def _hybrid_retrieval(
      self,
      query: str,
      contract_id: str,
      n_results: int
  ) -> Dict:
      # Get from both sources
      contract_results = self._retrieve_from_contract(query, contract_id, n_results)
      legal_results = self._retrieve_from_legal_kb(query, n_results)

      return {
          "source": "hybrid",
          "contract_chunks": contract_results['chunks'],
          "legal_chunks": legal_results['chunks']
      }
  ```

- [ ] **2.6.5** Add query intent classification
  ```python
  def classify_query_intent(self, query: str) -> QueryType:
      """Classify whether query is about contract or legal knowledge"""

      query_lower = query.lower()

      # Contract-specific keywords
      contract_keywords = [
          "this contract", "this clause", "my agreement",
          "in this document", "what does this say"
      ]

      # Legal knowledge keywords
      legal_keywords = [
          "under indian law", "according to", "what is the law",
          "legal definition", "statute", "section"
      ]

      if any(kw in query_lower for kw in contract_keywords):
          return QueryType.CONTRACT_SPECIFIC
      elif any(kw in query_lower for kw in legal_keywords):
          return QueryType.LEGAL_KNOWLEDGE
      else:
          return QueryType.HYBRID
  ```

**Acceptance Criteria:**

- Contract-specific retrieval works
- Legal knowledge retrieval works
- Hybrid retrieval merges both sources
- Query intent classification functional

---

## Week 8: Q&A Interface & Grounded Prompting

### Task 2.7: Q&A Endpoint with Grounded Prompting

**Priority:** P0
**Duration:** 3 days

#### Subtasks

- [ ] **2.7.1** Create Q&A service with hallucination prevention

  ```python
  # services/qa_service.py
  from typing import Dict, Optional

  class QAService:
      def __init__(
          self,
          rag_retrieval: RAGRetrievalService,
          llm_service: LLMProvider
      ):
          self.rag_retrieval = rag_retrieval
          self.llm_service = llm_service

      def answer_question(
          self,
          question: str,
          contract_id: str,
          retrieval_threshold: float = 0.7
      ) -> Dict:
          """Answer question with strict grounding"""

          # Step 1: Classify query intent
          query_type = self.rag_retrieval.classify_query_intent(question)

          # Step 2: Retrieve context
          retrieval_result = self.rag_retrieval.retrieve_context(
              query=question,
              contract_id=contract_id,
              query_type=query_type,
              n_results=5
          )

          # Step 3: Check retrieval quality
          if not self._has_sufficient_context(retrieval_result, retrieval_threshold):
              return {
                  "answer": "Insufficient legal context to answer this question reliably.",
                  "confidence": "N/A",
                  "retrieved_chunks": [],
                  "sources": []
              }

          # Step 4: Build grounded prompt
          prompt = self._build_grounded_prompt(question, retrieval_result)

          # Step 5: Get LLM response
          llm_response = self.llm_service.answer_question(prompt, context={})

          # Step 6: Parse and validate response
          answer = self._parse_qa_response(llm_response)

          return {
              "answer": answer['text'],
              "confidence": answer.get('confidence', 'Medium'),
              "sources": self._extract_sources(retrieval_result),
              "retrieved_chunks": retrieval_result
          }
  ```

- [ ] **2.7.2** Implement grounded prompt template
  ```python
  def _build_grounded_prompt(self, question: str, retrieval_result: Dict) -> str:
      prompt = """You are a legal contract analyst. Answer the question based ONLY on the provided context below.
  ```

CRITICAL RULES:

1. Do NOT use any external legal knowledge not present in the context
2. Do NOT make assumptions beyond what is explicitly stated
3. If the context does not contain sufficient information, say: "The provided context does not contain sufficient information to answer this question."
4. Always cite the specific source (section, clause, or document) when making a statement
5. Indicate your confidence level: High, Medium, or Low

"""

      # Add context from contract (if any)
      if 'contract_chunks' in retrieval_result:
          prompt += "Context from User's Contract:\n"
          for idx, chunk in enumerate(retrieval_result['contract_chunks'][:3]):
              prompt += f"\n[Contract Clause {idx+1}]:\n{chunk['text']}\n"

      # Add context from legal knowledge base (if any)
      if 'legal_chunks' in retrieval_result:
          prompt += "\n\nContext from Legal Knowledge Base:\n"
          for idx, chunk in enumerate(retrieval_result['legal_chunks'][:3]):
              prompt += f"\n[Legal Reference {idx+1}]:\n{chunk['text']}\n"

      prompt += f"""

Question: {question}

Provide your answer in the following JSON format:
{{
  "answer": "Your detailed answer here",
  "sources": ["Source 1", "Source 2"],
  "confidence": "High/Medium/Low",
  "reasoning": "Brief explanation of your reasoning"
}}

Return ONLY valid JSON.
"""

      return prompt

````

- [ ] **2.7.3** Implement retrieval quality check
```python
def _has_sufficient_context(self, retrieval_result: Dict, threshold: float) -> bool:
    """Check if retrieved context is relevant enough"""

    # Check contract chunks
    if 'contract_chunks' in retrieval_result:
        contract_chunks = retrieval_result['contract_chunks']
        if contract_chunks and any(c.get('score', 0) >= threshold for c in contract_chunks):
            return True

    # Check legal chunks
    if 'legal_chunks' in retrieval_result:
        legal_chunks = retrieval_result['legal_chunks']
        if legal_chunks and any(c.get('score', 0) >= threshold for c in legal_chunks):
            return True

    return False
````

- [ ] **2.7.4** Create Q&A endpoint

  ```python
  # api/v1/endpoints/qa.py
  from fastapi import APIRouter

  router = APIRouter()

  @router.post("/qa/ask")
  async def ask_question(
      question: str,
      contract_id: str
  ):
      qa_service = QAService(rag_retrieval, llm_service)

      result = qa_service.answer_question(
          question=question,
          contract_id=contract_id
      )

      return result
  ```

- [ ] **2.7.5** Add citation extraction
  ```python
  def _extract_sources(self, retrieval_result: Dict) -> List[str]:
      sources = []

      # From contract
      if 'contract_chunks' in retrieval_result:
          for chunk in retrieval_result['contract_chunks'][:3]:
              metadata = chunk['metadata']
              source = f"Contract Clause {metadata.get('position', '?')} ({metadata.get('clause_type', 'General')})"
              sources.append(source)

      # From legal KB
      if 'legal_chunks' in retrieval_result:
          for chunk in retrieval_result['legal_chunks'][:3]:
              metadata = chunk['metadata']
              if metadata.get('chunk_type') == 'statute':
                  source = f"{metadata.get('act_name')} - {metadata.get('section')}"
              elif metadata.get('chunk_type') == 'qa_pair':
                  source = f"Legal Q&A: {metadata.get('question', '')[:50]}..."
              else:
                  source = "Legal Reference"
              sources.append(source)

      return sources
  ```

**Acceptance Criteria:**

- Q&A endpoint functional
- Grounded prompting prevents hallucinations
- Retrieval threshold enforced
- Citations included in response
- Low-confidence queries handled gracefully

---

### Task 2.8: Q&A Chat UI

**Priority:** P1
**Duration:** 3 days

#### Subtasks

- [ ] **2.8.1** Create chat component

  ```typescript
  // components/ChatInterface.tsx
  'use client';

  import { useState } from 'react';
  import { Send } from 'lucide-react';
  import { api } from '@/lib/api';

  interface Message {
    role: 'user' | 'assistant';
    content: string;
    sources?: string[];
    confidence?: string;
  }

  export default function ChatInterface({ contractId }: { contractId: string }) {
    const [messages, setMessages] = useState<Message[]>([]);
    const [input, setInput] = useState('');
    const [isLoading, setIsLoading] = useState(false);

    const handleSend = async () => {
      if (!input.trim()) return;

      const userMessage: Message = { role: 'user', content: input };
      setMessages(prev => [...prev, userMessage]);
      setInput('');
      setIsLoading(true);

      try {
        const response = await api.askQuestion(contractId, input);

        const assistantMessage: Message = {
          role: 'assistant',
          content: response.answer,
          sources: response.sources,
          confidence: response.confidence
        };

        setMessages(prev => [...prev, assistantMessage]);
      } catch (error) {
        const errorMessage: Message = {
          role: 'assistant',
          content: 'Sorry, I encountered an error. Please try again.'
        };
        setMessages(prev => [...prev, errorMessage]);
      } finally {
        setIsLoading(false);
      }
    };

    return (
      <div className="flex flex-col h-[600px] bg-white rounded-lg shadow">
        <div className="flex-1 overflow-y-auto p-4 space-y-4">
          {messages.length === 0 && (
            <div className="text-center text-gray-500 mt-8">
              <p className="text-lg font-semibold mb-2">Ask questions about your contract</p>
              <p className="text-sm">Try asking:</p>
              <ul className="text-sm text-left max-w-md mx-auto mt-2 space-y-1">
                <li>• What is the termination notice period?</li>
                <li>• Are there any liability limitations?</li>
                <li>• Is this indemnity clause risky under Indian law?</li>
              </ul>
            </div>
          )}

          {messages.map((msg, idx) => (
            <div
              key={idx}
              className={`flex ${msg.role === 'user' ? 'justify-end' : 'justify-start'}`}
            >
              <div
                className={`max-w-[70%] rounded-lg p-3 ${
                  msg.role === 'user'
                    ? 'bg-blue-600 text-white'
                    : 'bg-gray-100 text-gray-900'
                }`}
              >
                <p className="text-sm">{msg.content}</p>

                {msg.sources && msg.sources.length > 0 && (
                  <div className="mt-2 pt-2 border-t border-gray-300">
                    <p className="text-xs font-semibold mb-1">Sources:</p>
                    <ul className="text-xs space-y-1">
                      {msg.sources.map((source, i) => (
                        <li key={i}>• {source}</li>
                      ))}
                    </ul>
                  </div>
                )}

                {msg.confidence && (
                  <p className="text-xs mt-1 opacity-70">
                    Confidence: {msg.confidence}
                  </p>
                )}
              </div>
            </div>
          ))}

          {isLoading && (
            <div className="flex justify-start">
              <div className="bg-gray-100 rounded-lg p-3">
                <div className="flex space-x-2">
                  <div className="w-2 h-2 bg-gray-400 rounded-full animate-bounce"></div>
                  <div className="w-2 h-2 bg-gray-400 rounded-full animate-bounce delay-100"></div>
                  <div className="w-2 h-2 bg-gray-400 rounded-full animate-bounce delay-200"></div>
                </div>
              </div>
            </div>
          )}
        </div>

        <div className="border-t p-4">
          <div className="flex gap-2">
            <input
              type="text"
              value={input}
              onChange={(e) => setInput(e.target.value)}
              onKeyPress={(e) => e.key === 'Enter' && handleSend()}
              placeholder="Ask a question about your contract..."
              className="flex-1 px-4 py-2 border rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500"
              disabled={isLoading}
            />
            <button
              onClick={handleSend}
              disabled={isLoading || !input.trim()}
              className="px-6 py-2 bg-blue-600 text-white rounded-lg hover:bg-blue-700 disabled:opacity-50 disabled:cursor-not-allowed flex items-center gap-2"
            >
              <Send className="h-4 w-4" />
              Send
            </button>
          </div>
        </div>
      </div>
    );
  }
  ```

- [ ] **2.8.2** Update API client

  ```typescript
  // lib/api.ts
  export const api = {
    // ... existing methods

    askQuestion: async (contractId: string, question: string) => {
      const response = await axios.post(`${API_BASE}/qa/ask`, {
        question,
        contract_id: contractId,
      });
      return response.data;
    },
  };
  ```

- [ ] **2.8.3** Integrate chat into analysis page
  ```typescript
  // Update app/page.tsx to include chat after analysis
  {analysis && (
    <>
      <AnalysisResults analysis={analysis} />

      <div className="mt-8">
        <h3 className="text-xl font-bold mb-4">Ask Questions</h3>
        <ChatInterface contractId={analysis.document_id} />
      </div>
    </>
  )}
  ```

**Acceptance Criteria:**

- Chat interface renders correctly
- Messages sent and received
- Sources displayed with answers
- Confidence indicators shown
- UI responsive and smooth

---

### Task 2.9: Phase 2 Testing

**Priority:** P0
**Duration:** 2 days

#### Subtasks

- [ ] **2.9.1** Test retrieval quality
  - Query: "What is termination clause?" → Should find termination clause
  - Query: "What does Section 73 say?" → Should retrieve from Acts
  - Query: "Is this enforceable?" → Hybrid retrieval

- [ ] **2.9.2** Test hallucination prevention
  - Ask question with no relevant context → Should decline to answer
  - Ask about non-existent clause → Should say not found

- [ ] **2.9.3** Test citation accuracy
  - Verify all sources are real and traceable
  - Check that citations match retrieved chunks

- [ ] **2.9.4** Performance testing
  - Measure retrieval time (<500ms)
  - Measure end-to-end Q&A time (<5s)

**Acceptance Criteria:**

- Retrieval quality good (>80% relevant)
- Hallucination prevention working
- Citations accurate
- Performance acceptable

---

# Phase 3: Dashboard & Privacy (Month 3)

**Timeline:** Weeks 9-12
**Goal:** Production-ready SaaS with user accounts, privacy controls, and dashboard

---

## Week 9: Authentication & User Management

### Task 3.1: User Authentication System

**Priority:** P0
**Duration:** 3 days

#### Subtasks

- [ ] **3.1.1** Install authentication dependencies

  ```txt
  # Backend requirements.txt
  passlib[bcrypt]==1.7.4
  python-jose[cryptography]==3.3.0
  python-multipart==0.0.6
  ```

- [ ] **3.1.2** Set up PostgreSQL (if not already)

  ```bash
  # Install PostgreSQL
  # Create database
  createdb clawde_db
  ```

- [ ] **3.1.3** Install database dependencies

  ```txt
  sqlalchemy==2.0.23
  alembic==1.12.1
  psycopg2-binary==2.9.9
  ```

- [ ] **3.1.4** Create database models

  ```python
  # models/db_models.py
  from sqlalchemy import Column, Integer, String, DateTime, Enum, ForeignKey
  from sqlalchemy.ext.declarative import declarative_base
  from sqlalchemy.orm import relationship
  from datetime import datetime
  import enum

  Base = declarative_base()

  class PrivacyTier(enum.Enum):
      AUTO_DELETE = "auto_delete"
      ENCRYPTED = "encrypted"
      ENTERPRISE = "enterprise"

  class User(Base):
      __tablename__ = "users"

      id = Column(Integer, primary_key=True, index=True)
      email = Column(String, unique=True, index=True, nullable=False)
      hashed_password = Column(String, nullable=False)
      full_name = Column(String)
      privacy_tier = Column(Enum(PrivacyTier), default=PrivacyTier.AUTO_DELETE)
      created_at = Column(DateTime, default=datetime.utcnow)

      # Relationships
      contracts = relationship("Contract", back_populates="owner")

  class Contract(Base):
      __tablename__ = "contracts"

      id = Column(String, primary_key=True)
      user_id = Column(Integer, ForeignKey("users.id"))
      filename = Column(String)
      file_path = Column(String)
      uploaded_at = Column(DateTime, default=datetime.utcnow)
      analyzed_at = Column(DateTime, nullable=True)
      delete_at = Column(DateTime, nullable=True)  # For auto-delete
      privacy_tier = Column(Enum(PrivacyTier))

      # Relationships
      owner = relationship("User", back_populates="contracts")
  ```

- [ ] **3.1.5** Set up Alembic migrations

  ```bash
  alembic init alembic
  # Configure alembic.ini and env.py
  alembic revision --autogenerate -m "Initial migration"
  alembic upgrade head
  ```

- [ ] **3.1.6** Create authentication utilities

  ```python
  # core/security.py
  from datetime import datetime, timedelta
  from jose import JWTError, jwt
  from passlib.context import CryptContext
  from typing import Optional

  pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

  SECRET_KEY = "your-secret-key-change-in-production"
  ALGORITHM = "HS256"
  ACCESS_TOKEN_EXPIRE_MINUTES = 30

  def verify_password(plain_password: str, hashed_password: str) -> bool:
      return pwd_context.verify(plain_password, hashed_password)

  def get_password_hash(password: str) -> str:
      return pwd_context.hash(password)

  def create_access_token(data: dict, expires_delta: Optional[timedelta] = None):
      to_encode = data.copy()
      if expires_delta:
          expire = datetime.utcnow() + expires_delta
      else:
          expire = datetime.utcnow() + timedelta(minutes=15)
      to_encode.update({"exp": expire})
      encoded_jwt = jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)
      return encoded_jwt

  def decode_access_token(token: str):
      try:
          payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
          return payload
      except JWTError:
          return None
  ```

- [ ] **3.1.7** Create auth endpoints

  ```python
  # api/v1/endpoints/auth.py
  from fastapi import APIRouter, Depends, HTTPException, status
  from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm
  from sqlalchemy.orm import Session

  router = APIRouter()
  oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

  @router.post("/register")
  def register(
      email: str,
      password: str,
      full_name: str,
      db: Session = Depends(get_db)
  ):
      # Check if user exists
      existing_user = db.query(User).filter(User.email == email).first()
      if existing_user:
          raise HTTPException(status_code=400, detail="Email already registered")

      # Create user
      hashed_password = get_password_hash(password)
      user = User(
          email=email,
          hashed_password=hashed_password,
          full_name=full_name
      )
      db.add(user)
      db.commit()
      db.refresh(user)

      return {"message": "User created successfully", "user_id": user.id}

  @router.post("/token")
  def login(
      form_data: OAuth2PasswordRequestForm = Depends(),
      db: Session = Depends(get_db)
  ):
      # Authenticate user
      user = db.query(User).filter(User.email == form_data.username).first()
      if not user or not verify_password(form_data.password, user.hashed_password):
          raise HTTPException(
              status_code=status.HTTP_401_UNAUTHORIZED,
              detail="Incorrect email or password"
          )

      # Create access token
      access_token = create_access_token(
          data={"sub": user.email, "user_id": user.id}
      )

      return {"access_token": access_token, "token_type": "bearer"}

  @router.get("/me")
  def get_current_user(
      token: str = Depends(oauth2_scheme),
      db: Session = Depends(get_db)
  ):
      payload = decode_access_token(token)
      if not payload:
          raise HTTPException(status_code=401, detail="Invalid token")

      user = db.query(User).filter(User.email == payload["sub"]).first()
      if not user:
          raise HTTPException(status_code=404, detail="User not found")

      return {
          "id": user.id,
          "email": user.email,
          "full_name": user.full_name,
          "privacy_tier": user.privacy_tier.value
      }
  ```

**Acceptance Criteria:**

- Database schema created
- User registration working
- Login returns JWT token
- `/me` endpoint returns user info
- Password hashing functional

---

### Task 3.2: Privacy Tier Implementation

**Priority:** P0
**Duration:** 3 days

#### Subtasks

- [ ] **3.2.1** Create privacy service

  ```python
  # services/privacy_service.py
  from datetime import datetime, timedelta
  from typing import Optional
  from models.db_models import PrivacyTier, Contract, User
  from sqlalchemy.orm import Session

  class PrivacyService:
      AUTO_DELETE_MINUTES = 30

      def __init__(self, db: Session):
          self.db = db

      def set_contract_privacy(
          self,
          contract_id: str,
          privacy_tier: PrivacyTier
      ):
          """Set privacy tier for contract"""
          contract = self.db.query(Contract).filter(Contract.id == contract_id).first()
          if not contract:
              raise ValueError("Contract not found")

          contract.privacy_tier = privacy_tier

          if privacy_tier == PrivacyTier.AUTO_DELETE:
              # Set delete_at timestamp
              contract.delete_at = datetime.utcnow() + timedelta(minutes=self.AUTO_DELETE_MINUTES)
          else:
              contract.delete_at = None

          self.db.commit()

      def get_contracts_to_delete(self) -> list:
          """Get contracts that should be auto-deleted"""
          now = datetime.utcnow()
          contracts = self.db.query(Contract).filter(
              Contract.privacy_tier == PrivacyTier.AUTO_DELETE,
              Contract.delete_at <= now
          ).all()
          return contracts

      def delete_contract(self, contract_id: str):
          """Delete contract and all associated data"""
          contract = self.db.query(Contract).filter(Contract.id == contract_id).first()
          if not contract:
              return

          # Delete file from disk
          import os
          if os.path.exists(contract.file_path):
              os.remove(contract.file_path)

          # Delete from vector DB
          from services.vector_db_service import VectorDBService
          vector_db = VectorDBService()
          # Delete all chunks with this contract_id
          # (ChromaDB deletion by metadata filter)

          # Delete from database
          self.db.delete(contract)
          self.db.commit()
  ```

- [ ] **3.2.2** Create auto-delete scheduler

  ```python
  # services/scheduler.py
  from apscheduler.schedulers.background import BackgroundScheduler
  from services.privacy_service import PrivacyService
  from database import SessionLocal
  import logging

  logger = logging.getLogger(__name__)

  def auto_delete_job():
      """Job to periodically delete expired contracts"""
      db = SessionLocal()
      try:
          privacy_service = PrivacyService(db)
          contracts_to_delete = privacy_service.get_contracts_to_delete()

          for contract in contracts_to_delete:
              logger.info(f"Auto-deleting contract {contract.id}")
              privacy_service.delete_contract(contract.id)
              logger.info(f"Deleted contract {contract.id}")
      except Exception as e:
          logger.error(f"Auto-delete job failed: {e}")
      finally:
          db.close()

  def start_scheduler():
      scheduler = BackgroundScheduler()
      # Run every 5 minutes
      scheduler.add_job(auto_delete_job, 'interval', minutes=5)
      scheduler.start()
      logger.info("Auto-delete scheduler started")
  ```

- [ ] **3.2.3** Start scheduler in main app

  ```python
  # main.py
  from fastapi import FastAPI
  from services.scheduler import start_scheduler

  app = FastAPI()

  @app.on_event("startup")
  def startup_event():
      start_scheduler()
  ```

- [ ] **3.2.4** Add encryption for stored contracts

  ```python
  # services/encryption_service.py
  from cryptography.fernet import Fernet
  from cryptography.hazmat.primitives import hashes
  from cryptography.hazmat.primitives.kdf.pbkdf2 import PBKDF2
  import base64
  import os

  class EncryptionService:
      def __init__(self, encryption_key: str = None):
          if encryption_key is None:
              encryption_key = os.getenv("ENCRYPTION_KEY")

          # Derive key from password
          kdf = PBKDF2(
              algorithm=hashes.SHA256(),
              length=32,
              salt=b'salt_'  # Use proper salt in production
          )
          key = base64.urlsafe_b64encode(kdf.derive(encryption_key.encode()))
          self.cipher = Fernet(key)

      def encrypt_file(self, file_path: str) -> str:
          """Encrypt file and save as .enc"""
          with open(file_path, 'rb') as f:
              data = f.read()

          encrypted_data = self.cipher.encrypt(data)

          encrypted_path = file_path + '.enc'
          with open(encrypted_path, 'wb') as f:
              f.write(encrypted_data)

          # Delete original
          os.remove(file_path)

          return encrypted_path

      def decrypt_file(self, encrypted_path: str) -> bytes:
          """Decrypt file and return contents"""
          with open(encrypted_path, 'rb') as f:
              encrypted_data = f.read()

          return self.cipher.decrypt(encrypted_data)
  ```

- [ ] **3.2.5** Update upload endpoint with privacy handling
  ```python
  @router.post("/contracts/upload")
  async def upload_contract(
      file: UploadFile = File(...),
      privacy_tier: str = "auto_delete",
      current_user: User = Depends(get_current_user),
      db: Session = Depends(get_db)
  ):
      # Save file
      file_id = str(uuid.uuid4())
      file_path = save_uploaded_file(file, file_id)

      # Apply privacy tier
      tier = PrivacyTier(privacy_tier)

      if tier == PrivacyTier.ENCRYPTED:
          encryption_service = EncryptionService()
          file_path = encryption_service.encrypt_file(file_path)

      # Create contract record
      contract = Contract(
          id=file_id,
          user_id=current_user.id,
          filename=file.filename,
          file_path=file_path,
          privacy_tier=tier
      )

      if tier == PrivacyTier.AUTO_DELETE:
          contract.delete_at = datetime.utcnow() + timedelta(minutes=30)

      db.add(contract)
      db.commit()

      # Extract and index...

      return {
          "file_id": file_id,
          "privacy_tier": tier.value,
          "delete_at": contract.delete_at.isoformat() if contract.delete_at else None
      }
  ```

**Acceptance Criteria:**

- Privacy tiers enforced on upload
- Auto-delete scheduler running
- Contracts deleted after 30 minutes (auto-delete mode)
- Encryption working for encrypted tier
- Delete timestamps tracked correctly

---

## Week 10: User Dashboard

### Task 3.3: Dashboard UI & History

**Priority:** P1
**Duration:** 4 days

#### Subtasks

- [ ] **3.3.1** Create dashboard endpoint

  ```python
  # api/v1/endpoints/dashboard.py
  @router.get("/dashboard/contracts")
  def get_user_contracts(
      current_user: User = Depends(get_current_user),
      db: Session = Depends(get_db)
  ):
      contracts = db.query(Contract).filter(
          Contract.user_id == current_user.id
      ).order_by(Contract.uploaded_at.desc()).all()

      return [
          {
              "id": c.id,
              "filename": c.filename,
              "uploaded_at": c.uploaded_at.isoformat(),
              "analyzed_at": c.analyzed_at.isoformat() if c.analyzed_at else None,
              "privacy_tier": c.privacy_tier.value,
              "delete_at": c.delete_at.isoformat() if c.delete_at else None,
              "time_remaining": (c.delete_at - datetime.utcnow()).seconds if c.delete_at else None
          }
          for c in contracts
      ]
  ```

- [ ] **3.3.2** Create dashboard page (frontend)

  ```typescript
  // app/dashboard/page.tsx
  'use client';

  import { useEffect, useState } from 'react';
  import { api } from '@/lib/api';
  import { Clock, Trash2, FileText } from 'lucide-react';

  interface Contract {
    id: string;
    filename: string;
    uploaded_at: string;
    analyzed_at: string | null;
    privacy_tier: string;
    delete_at: string | null;
    time_remaining: number | null;
  }

  export default function Dashboard() {
    const [contracts, setContracts] = useState<Contract[]>([]);

    useEffect(() => {
      loadContracts();
    }, []);

    const loadContracts = async () => {
      const data = await api.getUserContracts();
      setContracts(data);
    };

    const formatTimeRemaining = (seconds: number | null) => {
      if (!seconds) return null;
      const minutes = Math.floor(seconds / 60);
      return `${minutes} min`;
    };

    return (
      <div className="max-w-6xl mx-auto p-6">
        <h1 className="text-3xl font-bold mb-6">Your Contracts</h1>

        <div className="grid gap-4">
          {contracts.map((contract) => (
            <div key={contract.id} className="bg-white rounded-lg shadow p-6">
              <div className="flex items-start justify-between">
                <div className="flex items-start gap-4">
                  <FileText className="h-8 w-8 text-blue-600 mt-1" />
                  <div>
                    <h3 className="font-semibold text-lg">{contract.filename}</h3>
                    <p className="text-sm text-gray-600">
                      Uploaded {new Date(contract.uploaded_at).toLocaleString()}
                    </p>

                    {contract.analyzed_at && (
                      <p className="text-sm text-gray-600">
                        Analyzed {new Date(contract.analyzed_at).toLocaleString()}
                      </p>
                    )}

                    <div className="mt-2">
                      <span className={`text-xs px-2 py-1 rounded ${
                        contract.privacy_tier === 'auto_delete'
                          ? 'bg-orange-100 text-orange-800'
                          : contract.privacy_tier === 'encrypted'
                          ? 'bg-green-100 text-green-800'
                          : 'bg-blue-100 text-blue-800'
                      }`}>
                        {contract.privacy_tier.replace('_', ' ').toUpperCase()}
                      </span>
                    </div>
                  </div>
                </div>

                <div className="flex flex-col items-end gap-2">
                  {contract.time_remaining && (
                    <div className="flex items-center gap-2 text-orange-600">
                      <Clock className="h-4 w-4" />
                      <span className="text-sm font-semibold">
                        Deletes in {formatTimeRemaining(contract.time_remaining)}
                      </span>
                    </div>
                  )}

                  <div className="flex gap-2">
                    <button
                      onClick={() => window.location.href = `/analyze/${contract.id}`}
                      className="px-4 py-2 bg-blue-600 text-white rounded hover:bg-blue-700"
                    >
                      View Analysis
                    </button>

                    <button
                      onClick={() => deleteContract(contract.id)}
                      className="px-4 py-2 border border-red-600 text-red-600 rounded hover:bg-red-50"
                    >
                      <Trash2 className="h-4 w-4" />
                    </button>
                  </div>
                </div>
              </div>
            </div>
          ))}

          {contracts.length === 0 && (
            <div className="text-center py-12 text-gray-500">
              <FileText className="h-16 w-16 mx-auto mb-4 opacity-50" />
              <p>No contracts yet. Upload your first contract to get started.</p>
            </div>
          )}
        </div>
      </div>
    );
  }
  ```

- [ ] **3.3.3** Add countdown timer component

  ```typescript
  // components/CountdownTimer.tsx
  'use client';

  import { useState, useEffect } from 'react';
  import { Clock } from 'lucide-react';

  export default function CountdownTimer({ deleteAt }: { deleteAt: string }) {
    const [timeRemaining, setTimeRemaining] = useState('');

    useEffect(() => {
      const interval = setInterval(() => {
        const now = new Date().getTime();
        const deleteTime = new Date(deleteAt).getTime();
        const diff = deleteTime - now;

        if (diff <= 0) {
          setTimeRemaining('Expired');
          clearInterval(interval);
        } else {
          const minutes = Math.floor(diff / 1000 / 60);
          const seconds = Math.floor((diff / 1000) % 60);
          setTimeRemaining(`${minutes}:${seconds.toString().padStart(2, '0')}`);
        }
      }, 1000);

      return () => clearInterval(interval);
    }, [deleteAt]);

    return (
      <div className="flex items-center gap-2 text-orange-600">
        <Clock className="h-4 w-4 animate-pulse" />
        <span className="font-mono text-sm font-semibold">{timeRemaining}</span>
      </div>
    );
  }
  ```

**Acceptance Criteria:**

- Dashboard shows all user contracts
- Privacy tier badges displayed
- Countdown timers functional
- View/delete actions working
- Empty state shown when no contracts

---

## Week 11-12: Deadline Tracking & Trust UX

### Task 3.4: Deadline Extraction

**Priority:** P1
**Duration:** 3 days

#### Subtasks

- [ ] **3.4.1** Create deadline extraction service

  ```python
  # services/deadline_extractor.py
  import re
  from datetime import datetime, timedelta
  from typing import List, Dict
  from dateutil import parser

  class DeadlineExtractor:
      def extract_deadlines(self, contract_text: str) -> List[Dict]:
          """Extract deadline-related clauses"""

          deadlines = []

          # Pattern matching for dates
          date_patterns = [
              r'\b\d{1,2}[/-]\d{1,2}[/-]\d{2,4}\b',  # DD/MM/YYYY
              r'\b\d{4}[/-]\d{1,2}[/-]\d{1,2}\b',     # YYYY-MM-DD
              r'\b(?:Jan|Feb|Mar|Apr|May|Jun|Jul|Aug|Sep|Oct|Nov|Dec)[a-z]* \d{1,2},? \d{4}\b'  # Month DD, YYYY
          ]

          # Pattern matching for relative deadlines
          relative_patterns = [
              r'within (\d+) (?:days?|weeks?|months?|years?)',
              r'(\d+) (?:days?|weeks?|months?|years?) (?:notice|prior)',
              r'no (?:less|later) than (\d+) (?:days?|weeks?|months?)'
          ]

          # Find explicit dates
          for pattern in date_patterns:
              matches = re.finditer(pattern, contract_text)
              for match in matches:
                  try:
                      date_obj = parser.parse(match.group())
                      context = self._get_context(contract_text, match.start(), match.end())

                      deadlines.append({
                          "type": "explicit_date",
                          "date": date_obj.isoformat(),
                          "text": match.group(),
                          "context": context
                      })
                  except:
                      pass

          # Find relative deadlines
          for pattern in relative_patterns:
              matches = re.finditer(pattern, contract_text, re.IGNORECASE)
              for match in matches:
                  context = self._get_context(contract_text, match.start(), match.end())

                  deadlines.append({
                      "type": "relative_deadline",
                      "text": match.group(),
                      "context": context
                  })

          return deadlines

      def _get_context(self, text: str, start: int, end: int, window: int = 100) -> str:
          """Get surrounding context for found deadline"""
          context_start = max(0, start - window)
          context_end = min(len(text), end + window)
          return text[context_start:context_end].strip()
  ```

- [ ] **3.4.2** Integrate deadline extraction into analysis

  ```python
  # Update analysis_orchestrator.py
  async def analyze_document(self, file_path: str, file_type: str) -> DocumentAnalysis:
      # ... existing analysis ...

      # Extract deadlines
      deadline_extractor = DeadlineExtractor()
      deadlines = deadline_extractor.extract_deadlines(text)

      # Add to metadata
      metadata["deadlines"] = deadlines

      return DocumentAnalysis(...)
  ```

- [ ] **3.4.3** Create deadline calendar component

  ```typescript
  // components/DeadlineCalendar.tsx
  import { Calendar, AlertCircle } from 'lucide-react';

  interface Deadline {
    type: string;
    date?: string;
    text: string;
    context: string;
  }

  export default function DeadlineCalendar({ deadlines }: { deadlines: Deadline[] }) {
    const sortedDeadlines = deadlines
      .filter(d => d.date)
      .sort((a, b) => new Date(a.date!).getTime() - new Date(b.date!).getTime());

    return (
      <div className="bg-white rounded-lg shadow p-6">
        <div className="flex items-center gap-2 mb-4">
          <Calendar className="h-5 w-5 text-blue-600" />
          <h3 className="font-semibold text-lg">Important Deadlines</h3>
        </div>

        {sortedDeadlines.length === 0 ? (
          <p className="text-gray-500 text-sm">No explicit deadlines found</p>
        ) : (
          <div className="space-y-3">
            {sortedDeadlines.map((deadline, idx) => {
              const date = new Date(deadline.date!);
              const daysUntil = Math.floor((date.getTime() - Date.now()) / (1000 * 60 * 60 * 24));
              const isUrgent = daysUntil < 30 && daysUntil >= 0;

              return (
                <div key={idx} className={`border-l-4 pl-4 py-2 ${isUrgent ? 'border-orange-500' : 'border-blue-500'}`}>
                  <div className="flex items-start justify-between">
                    <div>
                      <p className="font-semibold text-sm">{date.toLocaleDateString()}</p>
                      <p className="text-xs text-gray-600 mt-1">{deadline.context.substring(0, 100)}...</p>
                    </div>

                    {isUrgent && (
                      <div className="flex items-center gap-1 text-orange-600 text-xs">
                        <AlertCircle className="h-3 w-3" />
                        <span>{daysUntil} days</span>
                      </div>
                    )}
                  </div>
                </div>
              );
            })}
          </div>
        )}

        {deadlines.filter(d => !d.date).length > 0 && (
          <div className="mt-4 pt-4 border-t">
            <p className="text-xs font-semibold text-gray-600 mb-2">Relative Deadlines:</p>
            <ul className="text-xs text-gray-600 space-y-1">
              {deadlines.filter(d => !d.date).map((deadline, idx) => (
                <li key={idx}>• {deadline.text}</li>
              ))}
            </ul>
          </div>
        )}
      </div>
    );
  }
  ```

**Acceptance Criteria:**

- Explicit dates extracted correctly
- Relative deadlines identified
- Calendar component displays deadlines
- Urgent deadlines highlighted
- Context provided for each deadline

---

### Task 3.5: Trust UX Elements

**Priority:** P0
**Duration:** 2 days

#### Subtasks

- [ ] **3.5.1** Create privacy badge component

  ```typescript
  // components/PrivacyBadge.tsx
  import { Shield, Lock, Clock } from 'lucide-react';

  export default function PrivacyBadge({ tier }: { tier: string }) {
    const badges = {
      auto_delete: {
        icon: <Clock className="h-4 w-4" />,
        text: "Auto-Delete in 30 min",
        color: "bg-orange-100 text-orange-800 border-orange-300"
      },
      encrypted: {
        icon: <Lock className="h-4 w-4" />,
        text: "Encrypted Storage",
        color: "bg-green-100 text-green-800 border-green-300"
      },
      enterprise: {
        icon: <Shield className="h-4 w-4" />,
        text: "Enterprise Security",
        color: "bg-blue-100 text-blue-800 border-blue-300"
      }
    };

    const badge = badges[tier as keyof typeof badges];

    return (
      <div className={`inline-flex items-center gap-2 px-3 py-1 rounded-full border ${badge.color}`}>
        {badge.icon}
        <span className="text-xs font-semibold">{badge.text}</span>
      </div>
    );
  }
  ```

- [ ] **3.5.2** Add trust messaging

  ```typescript
  // components/TrustBanner.tsx
  import { Shield, Trash2, Eye } from 'lucide-react';

  export default function TrustBanner() {
    return (
      <div className="bg-blue-50 border border-blue-200 rounded-lg p-4 mb-6">
        <div className="flex items-center gap-2 mb-3">
          <Shield className="h-5 w-5 text-blue-600" />
          <h4 className="font-semibold text-blue-900">Your Privacy is Our Priority</h4>
        </div>

        <div className="grid md:grid-cols-3 gap-4 text-sm">
          <div className="flex gap-2">
            <Trash2 className="h-4 w-4 text-blue-600 flex-shrink-0 mt-0.5" />
            <p className="text-blue-800">
              <strong>Auto-Delete:</strong> Documents deleted in 30 minutes by default
            </p>
          </div>

          <div className="flex gap-2">
            <Eye className="h-4 w-4 text-blue-600 flex-shrink-0 mt-0.5" />
            <p className="text-blue-800">
              <strong>No Training:</strong> We never use your documents to train AI models
            </p>
          </div>

          <div className="flex gap-2">
            <Shield className="h-4 w-4 text-blue-600 flex-shrink-0 mt-0.5" />
            <p className="text-blue-800">
              <strong>Encrypted:</strong> All data encrypted at rest and in transit
            </p>
          </div>
        </div>
      </div>
    );
  }
  ```

- [ ] **3.5.3** Add visible delete button
  ```typescript
  // On analysis page
  <button
    onClick={() => deleteContractNow()}
    className="flex items-center gap-2 px-4 py-2 bg-red-600 text-white rounded hover:bg-red-700"
  >
    <Trash2 className="h-4 w-4" />
    Delete Now
  </button>
  ```

**Acceptance Criteria:**

- Privacy badges visible on all relevant pages
- Trust banner prominent on homepage
- Delete button easily accessible
- Privacy messaging clear and reassuring
- Auto-delete countdown visible

---

# Phase 4: Differentiation (Month 4-5)

**Timeline:** Weeks 13-20
**Goal:** Unique market positioning with Indian-law awareness and negotiation support

I'll continue with Phase 4 and 5 if needed, but this gives you a comprehensive starting point for Phases 0-3 with detailed, actionable tasks that can be followed step-by-step.

---

**Document Continues...**

_(Due to length constraints, Phase 4 and 5 can be detailed separately or referenced from the phase-implementation.md document with specific technical tasks added as implementation progresses)_

---

# Implementation Notes

## General Principles

1. **Start with Phase 0** - Validate all technology choices before writing production code
2. **Test continuously** - Write tests as you build, not after
3. **Document as you go** - Update API docs and README with each feature
4. **Commit frequently** - Small, atomic commits with clear messages
5. **Review regularly** - Weekly review of progress vs plan

## Dependencies Between Tasks

- Authentication (3.1) must complete before Dashboard (3.3)
- Vector DB (2.1) must complete before RAG (2.5)
- Clause extraction (1.4) must complete before Analysis (1.7)
- Basic frontend (1.8) must exist before adding Q&A UI (2.8)

## Resource Requirements

- **Development Machine**: 8GB RAM minimum, 16GB recommended
- **Time Commitment**: Full-time (40 hrs/week) for timeline accuracy
- **API Costs**: Budget ~$50-100/month for Groq/OpenAI during development

## Success Metrics

Track these weekly:

- **Phase 1**: Average analysis time, accuracy of clause detection
- **Phase 2**: Retrieval relevance score, Q&A response quality
- **Phase 3**: User registrations, auto-delete success rate
- **Overall**: Test coverage %, API response times, error rates
