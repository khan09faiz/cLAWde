# Coding Principles

## 1. General Principles

### Clean Architecture
- Separate concerns: API layer, business logic, data access, AI pipeline
- Use dependency injection for LLM providers, DB connections, etc.
- Keep functions small and single-purpose
- Write self-documenting code with clear naming

### Code Organization
```
backend/
├── app/
│   ├── api/            # FastAPI route handlers
│   ├── core/           # Config, security, constants
│   ├── models/         # SQLAlchemy / Pydantic models
│   ├── services/       # Business logic layer
│   ├── ai/             # LLM, embedding, RAG logic
│   │   ├── providers/  # LLM provider implementations
│   │   ├── prompts/    # Prompt templates
│   │   ├── pipeline/   # Document processing pipeline
│   │   └── rag/        # RAG retrieval logic
│   ├── utils/          # Shared utilities
│   └── tests/          # Test files
├── alembic/            # DB migrations
└── main.py             # Entry point

frontend/
├── src/
│   ├── app/            # Next.js app router pages
│   ├── components/     # Reusable UI components
│   ├── hooks/          # Custom React hooks
│   ├── lib/            # Utilities, API client
│   ├── types/          # TypeScript type definitions
│   └── styles/         # Global styles
└── public/             # Static assets
```

### Naming Conventions
| Language | Convention | Example |
|----------|-----------|---------|
| Python functions | snake_case | `analyze_clause()` |
| Python classes | PascalCase | `ClauseAnalyzer` |
| Python constants | UPPER_SNAKE | `MAX_RETRIES` |
| TypeScript functions | camelCase | `analyzeClause()` |
| TypeScript components | PascalCase | `ClausePanel` |
| TypeScript types | PascalCase | `ClauseAnalysis` |
| CSS classes | kebab-case (Tailwind utilities) | `text-red-500` |
| API endpoints | kebab-case | `/api/analyze-contract` |
| DB tables | snake_case | `contract_analyses` |

---

## 2. Python / Backend Principles

### Type Hints Everywhere
```python
# ✅ Good
def analyze_clause(clause_text: str, context: AnalysisContext) -> ClauseResult:
    ...

# ❌ Bad
def analyze_clause(clause_text, context):
    ...
```

### Pydantic Models for All Data Boundaries
```python
# Request/response validation
class ContractUploadRequest(BaseModel):
    file_type: Literal["pdf", "docx", "txt"]
    privacy_mode: PrivacyMode = PrivacyMode.AUTO_DELETE

# Structured LLM output
class ClauseAnalysis(BaseModel):
    purpose: str
    plain_english: str
    who_benefits: str
    risk_severity: Literal["low", "medium", "high"]
    concerns: list[str]
```

### Async Where Possible
- Use `async def` for API endpoints
- Use async DB drivers (asyncpg)
- Use async HTTP clients (httpx) for LLM API calls
- Celery for truly background tasks (not blocking request)

### Error Handling
```python
# ✅ Specific exceptions with context
class DocumentProcessingError(Exception):
    def __init__(self, message: str, document_id: str):
        self.document_id = document_id
        super().__init__(message)

# ✅ Graceful degradation
try:
    result = await llm_provider.analyze(clause)
except LLMTimeoutError:
    result = fallback_rule_based_analysis(clause)
```

### Configuration Management
- Use environment variables for secrets
- Use Pydantic Settings for config validation
- Never hardcode API keys, URLs, or credentials
- Separate configs for dev / staging / production

---

## 3. Frontend / TypeScript Principles

### Component Design
- Small, focused components (single responsibility)
- Separate container (logic) from presentational components
- Use custom hooks for shared logic
- Prop types always defined with TypeScript interfaces

### State Management
- Use React hooks for local state
- Server state with React Query / SWR
- Avoid unnecessary global state
- Keep state as close to where it's used as possible

### API Integration
- Centralized API client with proper error handling
- TypeScript types matching backend Pydantic models
- Loading / error / success states for all async operations
- Retry logic for transient failures

---

## 4. Testing Principles

### Backend Testing
- Unit tests for business logic and utilities
- Integration tests for API endpoints
- Mock LLM calls in tests (deterministic)
- Test with sample contracts (not real user data)

### Frontend Testing
- Component tests with React Testing Library
- Integration tests for critical flows (upload → analysis)
- Visual regression tests for key pages (optional)

### Test Coverage Targets
| Layer | Target |
|-------|--------|
| Core business logic | 90%+ |
| API endpoints | 80%+ |
| AI pipeline | 70%+ (mock LLM) |
| Frontend components | 70%+ |

---

## 5. Git Practices

- Feature branches: `feature/clause-analysis`
- Bug fixes: `fix/ocr-encoding`
- Conventional commits: `feat:`, `fix:`, `docs:`, `refactor:`
- Pull requests with description and test evidence
- No direct commits to main
- Squash merge for clean history

---

## 6. Documentation

- Docstrings on all public functions
- README in each major directory
- API documentation auto-generated (FastAPI OpenAPI)
- Architecture decision records for major choices
- Keep docs close to code (not in separate wiki)
