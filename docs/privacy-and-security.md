# Privacy & Security Architecture

> **This is the key differentiator of cLAWde.**

---

## 1. Privacy Modes

### Mode 1: Analyze & Auto Delete (Default)

| Step | Action |
|------|--------|
| 1 | Contract uploaded and processed |
| 2 | Vector embeddings generated |
| 3 | Analysis completed and returned |
| 4 | Raw text **deleted within 30 minutes** |

**Guarantees:**
- ❌ No permanent storage
- ❌ No training on user data
- ❌ No logging of contract content
- ✅ Only analysis results shown to user

---

### Mode 2: Save to Dashboard

| Feature | Detail |
|---------|--------|
| Storage | Encrypted (AES-256) |
| Deletion | User-controlled, delete anytime |
| Vector DB | Encrypted at rest |
| Training | ❌ Never trained on user data |

**Use case:** Users who want to revisit analysis, track deadlines, maintain contract portfolio.

---

### Mode 3: Private Mode (Premium)

| Feature | Detail |
|---------|--------|
| LLM | Local via Ollama |
| Embeddings | Generated locally |
| Cloud calls | ❌ Zero cloud API calls |
| Data | Never leaves the user's machine |

**Use case:** Law firms, sensitive corporate contracts, compliance-heavy environments.

---

## 2. Security Measures

### Transport Security
- HTTPS enforced (TLS 1.3)
- Secure API gateways
- CORS properly configured

### Storage Security
- AES-256 encryption at rest
- Encrypted vector database
- Encrypted database backups

### Access Control
- Role-based access control (RBAC)
- JWT authentication
- Session management with secure cookies
- API key management for integrations

### Data Handling
- No LLM data retention (cloud providers)
- Minimal logging (no contract content in logs)
- Audit logs for enterprise tier
- Data isolation between users

### Compliance Readiness
- GDPR-compatible data handling
- Right to deletion support
- Data export capability
- Consent management

---

## 3. Trust UX (Critical for Adoption)

Show these elements **visibly** in the UI:

| Element | Purpose |
|---------|---------|
| Auto-delete timer | Visible countdown showing when data will be purged |
| Privacy badge | "We do not train on your data" prominently displayed |
| Delete button | Instant, user-controlled data deletion |
| Privacy mode indicator | Clear display of which privacy mode is active |
| Encryption indicator | Visual confirmation that data is encrypted |

> **Trust = Retention.** Make privacy visible, not hidden in a policy page.

---

## 4. Data Lifecycle

```
Upload → Process → Analyze → [Auto-Delete / Encrypt-Store]
                                    ↓
                            User Deletes → Permanent Removal
                                    ↓
                            30-day Auto-Purge (Mode 1)
```

### Auto-Delete Mode Timeline
```
T+0:00  — File uploaded
T+0:01  — Text extraction begins
T+0:05  — Clause analysis complete
T+0:10  — Results returned to user
T+30:00 — Raw text permanently deleted
T+30:00 — Temporary embeddings purged
```

---

## 5. Enterprise Security (Premium)

| Feature | Detail |
|---------|--------|
| SSO Integration | SAML / OAuth2 |
| Audit Logs | Full activity trail |
| On-Prem Deployment | Complete air-gapped option |
| Custom Retention | Configurable data policies |
| VPN / Private VNet | Azure private endpoints |
| SOC 2 Readiness | Compliance documentation |

---

## 6. Security Dos and Don'ts

### ✅ DO
- Encrypt all data at rest and in transit
- Implement proper key rotation
- Log access events (not content)
- Provide user data export
- Support right to deletion
- Use parameterized queries (prevent SQL injection)
- Sanitize all file uploads
- Implement rate limiting

### ❌ DON'T
- Never log contract content
- Never train on user data
- Never store raw text longer than needed
- Never share data between users
- Never use default credentials
- Never expose internal errors to users
- Never skip input validation
