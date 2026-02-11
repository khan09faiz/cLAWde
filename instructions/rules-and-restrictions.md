# Rules & Restrictions — Dos and Don'ts

## ⚠️ CRITICAL RULE — Read First

> **DO NOT create a markdown (.md) file to document or summarize work after completing each task.**
> This is a strict rule. No summary files, no changelog files, no "what was done" documents.
> Document through code comments, git commits, and these docs only.

---

## 1. Architecture Rules

### ✅ DO
- Keep backend and frontend as separate, independent projects
- Use environment variables for all configuration
- Design API-first — frontend consumes backend APIs
- Use Pydantic models for all request/response validation
- Implement proper error handling at every layer
- Use async operations for I/O-bound tasks
- Design the LLM layer to be provider-agnostic (switchable)
- Use database migrations (Alembic) for all schema changes

### ❌ DON'T
- Don't couple frontend and backend logic
- Don't hardcode API keys, URLs, secrets, or credentials anywhere
- Don't store LLM provider-specific code in business logic
- Don't skip database migrations — never modify DB schema manually
- Don't create monolithic files — split by responsibility
- Don't use global mutable state
- Don't bypass the API layer for data access

---

## 2. AI / LLM Rules

### ✅ DO
- Always use structured output (JSON matching Pydantic models)
- Always ground LLM responses in source text
- Always include confidence indicators in analysis
- Use temperature 0 for analysis tasks
- Implement retry logic with fallback (max 2 retries)
- Validate LLM output against schema before returning to user
- Use RAG for legal context — never rely on LLM's training data for legal facts
- Include disclaimer: "This is AI analysis, not legal advice"
- Process clauses independently to prevent context bleed
- Cache embeddings to avoid recomputation

### ❌ DON'T
- Don't present LLM output as legal advice
- Don't allow unvalidated LLM output to reach the user
- Don't use high temperature for analysis (causes inconsistency)
- Don't let the LLM fabricate legal citations — always verify
- Don't use LLM-generated content without post-processing
- Don't assume LLM output is correct — always validate
- Don't train on user data — ever
- Don't log contract content for debugging (log metadata only)
- Don't fine-tune models in early phases — focus on prompting first

---

## 3. Security Rules

### ✅ DO
- Encrypt all stored data at rest (AES-256)
- Use HTTPS/TLS 1.3 for all communication
- Implement proper authentication (JWT with refresh tokens)
- Use parameterized queries (prevent SQL injection)
- Sanitize all file uploads (check type, size, content)
- Implement rate limiting on all API endpoints
- Log access events (who accessed what, when)
- Support user data deletion (right to be forgotten)
- Implement auto-delete timers for Mode 1
- Use secure cookie settings (HttpOnly, Secure, SameSite)

### ❌ DON'T
- Don't log contract text content in server logs
- Don't store passwords in plain text (use bcrypt/argon2)
- Don't expose internal error details to users
- Don't skip input validation on any endpoint
- Don't use default credentials or secrets
- Don't share data between users — strict isolation
- Don't keep raw contract text longer than needed (Mode 1)
- Don't trust client-side validation alone — validate server-side too
- Don't store API keys in version control

---

## 4. Frontend Rules

### ✅ DO
- Use TypeScript strictly — no `any` types
- Implement proper loading, error, and empty states for every view
- Show privacy mode and trust indicators at all times
- Make the app responsive (desktop, tablet, mobile)
- Use semantic HTML elements
- Follow WCAG 2.1 AA accessibility standards
- Use proper form validation with clear error messages
- Implement optimistic UI where appropriate
- Use skeleton screens instead of spinners for initial loads

### ❌ DON'T
- Don't use `any` type in TypeScript
- Don't ignore error states — every async action needs error handling
- Don't use color alone to convey information (accessibility)
- Don't create components larger than ~200 lines
- Don't use inline styles — use Tailwind utility classes
- Don't fetch data in components — use hooks or server components
- Don't ignore mobile layouts
- Don't use alert() or confirm() — use custom modals/toasts

---

## 5. Data Handling Rules

### ✅ DO
- Implement all three privacy modes from the start
- Auto-delete raw text within 30 minutes (Mode 1)
- Encrypt stored contracts (Mode 2)
- Support user-initiated deletion at any time
- Use database transactions for multi-step operations
- Validate file types before processing
- Limit upload file size (reasonable maximum)
- Handle encoding issues gracefully (UTF-8 normalization)

### ❌ DON'T
- Don't store raw contract text in logs, caches, or temp files beyond the processing window
- Don't share vectors or embeddings between users
- Don't retain data after user deletes their account
- Don't process files without type validation
- Don't assume PDF text extraction will always succeed

---

## 6. Development Workflow Rules

### ✅ DO
- Write tests for all new features
- Use feature branches and pull requests
- Write meaningful commit messages (conventional commits)
- Review code before merging
- Keep dependencies updated (security patches)
- Use `.env` files for local development (never commit them)
- Run linters and formatters before committing
- Document API changes in OpenAPI spec

### ❌ DON'T
- **Don't create summary/changelog markdown files after each task** (CRITICAL)
- Don't commit directly to main branch
- Don't skip tests for "quick fixes"
- Don't leave TODO comments without tracking issues
- Don't commit node_modules, __pycache__, .env files
- Don't ignore linter warnings
- Don't merge PRs with failing tests
- Don't refactor and add features in the same PR

---

## 7. Performance Rules

### ✅ DO
- Cache LLM responses for identical inputs
- Use background jobs (Celery) for heavy processing
- Implement pagination for all list endpoints
- Use database indexes on frequently queried columns
- Optimize vector DB queries with proper indexing
- Lazy-load frontend components where possible
- Use CDN for static assets
- Monitor API response times

### ❌ DON'T
- Don't process entire contracts synchronously in the request cycle
- Don't load all contracts into memory at once
- Don't skip database indexing
- Don't ignore memory usage on 8GB RAM setups
- Don't embed the same text multiple times (cache embeddings)
- Don't return more data than the frontend needs

---

## 8. Project Scope Rules

### ✅ DO
- Follow the phase plan strictly
- Build MVP first, validate, then expand
- Focus on clause-level analysis quality before adding features
- Test with freelancers first (fastest feedback loop)
- Ship iteratively — working features over perfect features

### ❌ DON'T
- Don't build enterprise features before MVP validation
- Don't add collaboration before core analysis works
- Don't build complex UI before the API is stable
- Don't optimize prematurely — make it work, then make it fast
- Don't add features not in the current phase
- Don't gold-plate — good enough beats perfect but late

---

## 9. Quick Reference — Most Important Rules

| Priority | Rule |
|----------|------|
| 🔴 | Never create summary .md files after tasks |
| 🔴 | Never log contract content |
| 🔴 | Never train on user data |
| 🔴 | Never present analysis as legal advice |
| 🔴 | Never skip LLM output validation |
| 🟡 | Always encrypt stored data |
| 🟡 | Always use structured LLM output |
| 🟡 | Always implement privacy modes |
| 🟡 | Always validate inputs server-side |
| 🟢 | Follow phase plan |
| 🟢 | Write tests |
| 🟢 | Use feature branches |
