# LLMGateway

A production-ready middleware that puts **one standardized request/response schema** in front of OpenAI, Anthropic, Google, Ollama, and any OpenAI-compatible endpoint. Any app or AI agent can call a single URL — the gateway transforms the request for the chosen provider and normalizes the response back.

Includes:

- **Dashboard UI** (dark/light mode) to manage providers, workflows, apps, and logs — no config files needed.
- **Visual workflow builder** (React Flow, n8n-style) for multi-step pipelines: input → transforms → LLM calls → branches → output.
- **Per-app workflow binding** — App A can use Workflow 1, App B can use Workflow 2, with independent API keys.
- **Full request/response logging** with step-by-step traversal.
- **Encrypted provider credentials** at rest (AES-256-GCM).
- **Built-in playground** to send test requests.

## Tech stack (cost-efficient, production-ready)

| Layer    | Choice                          | Why                                    |
| -------- | ------------------------------- | -------------------------------------- |
| Framework| Next.js 15 (App Router)         | Single deploy for UI + API             |
| Language | TypeScript                      | Type safety across adapters            |
| Database | SQLite via Prisma (swap to Postgres for HA) | Zero hosting cost; great perf |
| UI       | Tailwind CSS + shadcn/ui        | Accessible, themeable, no vendor lock  |
| Workflow | React Flow (@xyflow/react)      | MIT-licensed visual graph editor       |
| Adapters | Official SDKs where available   | Stay current with provider features    |

Runs on any Node host: Vercel, Fly.io, Railway, Docker on a cheap VPS. SQLite needs a persistent volume; for serverless Vercel use Postgres or LibSQL.

## Quick start

```bash
cd llmgateway
cp .env.example .env
# edit .env: ADMIN_PASSWORD, ENCRYPTION_KEY (openssl rand -hex 32)

npm install
npx prisma migrate dev --name init
npm run db:seed        # creates "default-passthrough" workflow + demo app
npm run dev
```

Open http://localhost:3000, sign in with the admin password (default `admin123`), then:

1. **Providers** → add OpenAI / Anthropic / Google / Ollama / custom. Click **Test** to verify.
2. **Workflows** → edit `default-passthrough`, pick the provider on the LLM node, save.
3. **Apps** → the seeded `demo-app` has an API key (printed by `db:seed`). Rotate it if needed.
4. **Playground** → pick the app, send a test request.
5. Any external client can now call `POST /api/v1/chat` with `Authorization: Bearer <apiKey>`.

## API

### Request

```http
POST /api/v1/chat
Authorization: Bearer lmg_xxxxxxxx
Content-Type: application/json

{
  "messages": [
    { "role": "system", "content": "You are concise." },
    { "role": "user",   "content": "What is the capital of France?" }
  ],
  "temperature": 0.7,
  "maxTokens": 200,
  "workflow": "default-passthrough"   // optional override
}
```

### Response

```json
{
  "id": "chatcmpl-…",
  "model": "gpt-4o-mini",
  "provider": "openai",
  "createdAt": "2026-04-23T12:00:00.000Z",
  "message": { "role": "assistant", "content": "Paris." },
  "finishReason": "stop",
  "usage": { "promptTokens": 18, "completionTokens": 2, "totalTokens": 20 },
  "requestId": "uuid-…"
}
```

Same shape regardless of provider. Switch providers in the dashboard, clients don't change.

## Workflow nodes

- **input** — entry point (exactly one).
- **llm** — call a provider. Supports optional `systemPrompt` and `promptTemplate` with `{{input}}`, `{{last}}`, `{{steps.<id>.output}}` placeholders.
- **transform** — prefix/suffix the running text.
- **branch** — route based on whether the previous output contains a string (case-insensitive). Connect two outgoing edges with `sourceHandle` `true` and `false`.
- **output** — terminal (exactly one).

## Deployment

**Docker**:

```bash
docker build -t llmgateway .
docker run -p 3000:3000 \
  -v $(pwd)/data:/app/prisma \
  -e DATABASE_URL="file:/app/prisma/prod.db" \
  -e ENCRYPTION_KEY="$(openssl rand -hex 32)" \
  -e ADMIN_PASSWORD="change-me" \
  -e SESSION_SECRET="$(openssl rand -hex 32)" \
  llmgateway
```

**Vercel / Fly / Railway**: set the env vars, attach a persistent disk for SQLite (or point `DATABASE_URL` at Neon / Supabase Postgres).

## Security notes

- Provider API keys are AES-256-GCM encrypted with a key derived from `ENCRYPTION_KEY`. Rotate `ENCRYPTION_KEY` and re-enter provider keys to rotate at rest.
- Admin console is gated by `ADMIN_PASSWORD`. Put it behind your IdP for multi-user deployments.
- Gateway API keys are random 24-byte hex with `lmg_` prefix. Rotate them from the Apps page.
