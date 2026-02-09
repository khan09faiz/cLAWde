# 12_DEPLOYMENT

**Last Updated:** 2026-02-09  
**Version:** 2.0.0  
**Author:** LawBotics Team

---

## Table of Contents

1. [Deployment Architecture](#deployment-architecture)
2. [Environment Setup](#environment-setup)
3. [Frontend Deployment](#frontend-deployment)
4. [Backend (Convex)](#backend-convex)
5. [Tier 1 Model Hosting](#tier-1-model-hosting)
6. [Free API Configuration](#free-api-configuration)
7. [Monitoring](#monitoring)
8. [CI/CD Pipeline](#cicd-pipeline)

---

## Deployment Architecture

```
┌──────────────────────────────────────────────────────────┐
│                    Production Stack                       │
│                                                          │
│  ┌──────────┐    ┌───────────┐    ┌──────────────────┐  │
│  │ Vercel   │    │ Convex    │    │ External APIs    │  │
│  │ (Next.js)│───▶│ (Backend) │───▶│                  │  │
│  │ Frontend │    │ DB+Actions│    │ • Groq (free)    │  │
│  └──────────┘    └─────┬─────┘    │ • Gemini (free)  │  │
│                        │          │ • Clerk (auth)   │  │
│                        ▼          └──────────────────┘  │
│                  ┌───────────┐                           │
│                  │ LLaMA 3B  │                           │
│                  │(Colab+    │                           │
│                  │ Ollama)   │                           │
│                  └───────────┘                           │
└──────────────────────────────────────────────────────────┘
```

| Component | Platform | Cost |
|-----------|----------|------|
| Frontend | Vercel (Free) | Free |
| Backend + DB | Convex | Free tier available |
| Auth | Clerk | Free tier (10K MAU) |
| Tier 1 LLaMA | Google Colab (Ollama) | Free |
| Tier 2 Agents | Groq API | Free |
| Coordinator/Synthesis | Google Gemini API | Free |

---

## Environment Setup

### Required Environment Variables

```bash
# .env.local (Next.js frontend)
NEXT_PUBLIC_CONVEX_URL=https://your-project.convex.cloud
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=pk_live_xxx
CLERK_SECRET_KEY=sk_live_xxx

# Convex environment variables (set via dashboard or CLI)
# npx convex env set GOOGLE_API_KEY=xxx
# npx convex env set GROQ_API_KEY=xxx
# npx convex env set LLAMA_API_URL=https://your-llama-endpoint.com

GOOGLE_API_KEY=AIza...                    # Gemini (already have)
GROQ_API_KEY=gsk_...                      # Free from console.groq.com
LLAMA_API_URL=https://your-colab-ngrok-url/v1  # Google Colab Ollama endpoint (via ngrok)
CLERK_WEBHOOK_SECRET=whsec_...            # Clerk webhook signing
```

### Getting Free API Keys

1. **Groq**: Sign up at [console.groq.com](https://console.groq.com) → API Keys → Create. Free, no credit card.
2. **Google Gemini**: Already configured. Uses existing `GOOGLE_API_KEY`.
3. **Clerk**: Already configured. Free tier covers 10,000 MAU.

---

## Frontend Deployment

### Vercel (Recommended)

```bash
# Install Vercel CLI
npm i -g vercel

# Deploy from apps/web
cd apps/web
vercel

# Production deployment
vercel --prod
```

### Vercel Configuration

```json
// apps/web/vercel.json
{
  "framework": "nextjs",
  "buildCommand": "next build",
  "outputDirectory": ".next",
  "installCommand": "npm install",
  "env": {
    "NEXT_PUBLIC_CONVEX_URL": "@convex-url",
    "NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY": "@clerk-publishable-key"
  }
}
```

---

## Backend (Convex)

### Deploy Convex

```bash
# Development
npx convex dev

# Production deployment
npx convex deploy

# Set environment variables
npx convex env set GOOGLE_API_KEY "AIza..."
npx convex env set GROQ_API_KEY "gsk_..."
npx convex env set LLAMA_API_URL "https://your-colab-ngrok-url"
```

### Convex Node Dependencies

```json
// apps/web/convex/package.json (for Convex actions)
{
  "dependencies": {
    "groq-sdk": "^0.9.0",
    "@google/generative-ai": "^0.24.1",
    "axios": "^1.7.9",
    "zod": "^3.25.0"
  }
}
```

---

## Tier 1 Model Hosting

### Google Colab + Ollama (Free)

Since the local system cannot handle GPU inference, we use Google Colab's free T4 GPU to run the fine-tuned LLaMA model via Ollama. This provides an OpenAI-compatible API endpoint accessible via ngrok tunneling.

```python
# Run in Google Colab (free GPU)
# Step 1: Install Ollama
!curl -fsSL https://ollama.com/install.sh | sh

# Step 2: Start Ollama server in background
import subprocess
subprocess.Popen(["ollama", "serve"])
import time; time.sleep(5)  # Wait for server to start

# Step 3: Load the fine-tuned LLaMA model
# Option A: Pull base model + apply fine-tuned weights
!ollama pull llama3.2:3b

# Option B: Create custom model with Modelfile
modelfile_content = """
FROM llama3.2:3b
SYSTEM You are a legal contract analysis AI trained on the CUAD dataset.
PARAMETER temperature 0.1
PARAMETER num_ctx 4096
"""
with open("Modelfile", "w") as f:
    f.write(modelfile_content)
!ollama create lawbotics-legal -f Modelfile

# Step 4: Expose via ngrok for external access
!pip install pyngrok
from pyngrok import ngrok
ngrok.set_auth_token("your-ngrok-token")  # Free at ngrok.com
public_url = ngrok.connect(11434)
print(f"Ollama API URL: {public_url}")

# Step 5: Test the endpoint
import requests
response = requests.post(f"{public_url}/api/chat", json={
    "model": "lawbotics-legal",
    "messages": [{"role": "user", "content": "Extract clauses from this contract..."}],
    "stream": False
})
print(response.json())
```

**Important Notes:**
- Google Colab free tier provides T4 GPU sessions (up to ~12 hours)
- Use ngrok to expose the Ollama endpoint to your Convex backend
- Set the `LLAMA_API_URL` env var to the ngrok public URL
- Colab sessions may disconnect after idle periods — reconnect as needed
- For persistent hosting, consider Colab Pro ($10/mo) for longer sessions

---

## Free API Configuration

### Groq Setup in Convex Actions

```typescript
// convex/agents/providers.ts
"use node";

import Groq from "groq-sdk";
import { GoogleGenerativeAI } from "@google/generative-ai";

// Groq client — uses GROQ_API_KEY env var automatically
export const groq = new Groq({
  apiKey: process.env.GROQ_API_KEY,
});

// Gemini client — already configured
export const genAI = new GoogleGenerativeAI(process.env.GOOGLE_API_KEY!);

// LLaMA client — Google Colab (Ollama), OpenAI-compatible API
export async function callLlama(prompt: string): Promise<string> {
  const response = await fetch(`${process.env.LLAMA_API_URL}/api/chat`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      model: "lawbotics-legal",
      messages: [{ role: "user", content: prompt }],
      stream: false,
    }),
  });
  const data = await response.json();
  return data.message.content;
}
```

---

## Monitoring

### Health Checks

```typescript
// convex/internal.ts

export const healthCheck = action({
  handler: async () => {
    const checks: Record<string, boolean> = {};

    // Check Groq
    try {
      const groq = new Groq();
      await groq.chat.completions.create({
        model: "llama-3.1-8b-instant",
        messages: [{ role: "user", content: "ping" }],
        max_tokens: 5,
      });
      checks.groq = true;
    } catch {
      checks.groq = false;
    }

    // Check Gemini
    try {
      const model = genAI.getGenerativeModel({ model: "gemini-2.0-flash" });
      await model.generateContent("ping");
      checks.gemini = true;
    } catch {
      checks.gemini = false;
    }

    // Check LLaMA endpoint
    try {
      const res = await fetch(`${process.env.LLAMA_API_URL}/health`);
      checks.llama = res.ok;
    } catch {
      checks.llama = false;
    }

    return checks;
  },
});
```

### Key Metrics to Monitor

| Metric | Source | Alert Threshold |
|--------|--------|----------------|
| Groq RPM usage | usage_logs | > 25 RPM |
| Gemini RPM usage | usage_logs | > 12 RPM |
| Agent error rate | usage_logs | > 5% |
| Analysis latency | tier2_analyses | > 60s |
| LLaMA endpoint health | health check | Down > 2 min |
| Daily token usage | usage_logs | > 400K tokens |

---

## CI/CD Pipeline

```yaml
# .github/workflows/deploy.yml

name: Deploy LawBotics
on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20 }
      - run: cd apps/web && npm ci
      - run: cd apps/web && npm run lint
      - run: cd apps/web && npm run test

  deploy-convex:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20 }
      - run: cd apps/web && npm ci
      - run: npx convex deploy
        env:
          CONVEX_DEPLOY_KEY: ${{ secrets.CONVEX_DEPLOY_KEY }}

  deploy-frontend:
    needs: [test, deploy-convex]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: amondnet/vercel-action@v25
        with:
          vercel-token: ${{ secrets.VERCEL_TOKEN }}
          vercel-org-id: ${{ secrets.VERCEL_ORG_ID }}
          vercel-project-id: ${{ secrets.VERCEL_PROJECT_ID }}
          working-directory: apps/web
```

---

## Related Documentation

- [01_ARCHITECTURE_OVERVIEW.md](01_ARCHITECTURE_OVERVIEW.md) — System architecture
- [14_SECURITY.md](14_SECURITY.md) — API key security
- [10_COST_OPTIMIZATION.md](10_COST_OPTIMIZATION.md) — Cost management

## Revision History

| Version | Date | Changes |
|---------|------|---------|
| 2.0.0 | 2026-02-09 | Groq/Gemini free tier deployment, LLaMA hosting |
| 1.0.0 | 2025-01-01 | Single Gemini API deployment |
