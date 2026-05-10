# Dotty

**A SaaS-hosted personal AI assistant with a cheeky-best-friend personality.**

Dotty connects to your Gmail and Google Calendar accounts (work + personal, multiple of each) and acts as your proactive executive assistant — reading emails, sending drafts, scheduling meetings, remembering everything, and delivering a daily morning briefing.

MVP frontend is a PWA (React 19 + TypeScript + TailwindCSS + Vite). Native iOS/macOS target V1.1 using the same WebSocket API contract.

---

## Quick Start

```bash
# 1. Clone and install
npm install

# 2. Configure environment
cp .env.example .env
# Fill in all values in .env (see Configuration below)

# 3. Start infrastructure (Postgres + Redis)
docker compose up

# 4. Apply database migrations
node scripts/migrate.js

# 5. Start the app
npm run dev
```

Open `http://localhost:3000`. Use Clerk dev instance + Gmail sandbox accounts for testing.

---

## Features

- **Multi-account Gmail + Calendar** — Work and personal, same provider. Unified search across all accounts. Conflict detection for scheduling.
- **Real-time streaming chat** — WebSocket (Socket.io) with character-by-character text deltas and typing indicators.
- **Long-term memory** — Mem0 SaaS extracts facts from conversations and injects them into the system prompt every turn.
- **Morning briefing** — Daily summary at your configured time: today's calendar + priority emails.
- **Rich artifacts** — Email drafts, schedule tables, timelines, comparison tables rendered inline in chat.
- **Attachment handling** — Upload images/PDFs; Dotty reads them via `read_file` tool.
- **Per-account notification rules** — Immediate push, digest at a chosen hour, or off per integration.

---

## Architecture

```
Frontend (PWA, React 19)
    ↕ WebSocket + HTTP (Socket.io)
Backend (Node.js 22, Fastify)
    ├── Agent Runtime (pi-agent-core)
    ├── Composio (Gmail/Calendar OAuth + tool execution)
    ├── Mem0 (memory)
    └── Cron (node-cron, briefings + trigger fallback)
         ↕
    Neon PostgreSQL  |  Upstash Redis  |  Cloudflare R2  |  Mem0 SaaS
```

**Stack:**
- Fastify + Socket.io
- pi-agent-core (`@earendil-works/pi-agent-core`)
- Composio (native SDK, not MCP)
- Clerk (auth: Apple Sign-In + email)
- Mem0 SaaS (memory)
- Cloudflare R2 (attachments)
- Neon PostgreSQL + Upstash Redis

**Hosting:** Fly.io with Docker. Single process for MVP (0–1,000 users).

---

## Configuration

All configuration via environment variables. Copy `.env.example` → `.env`:

```bash
# Clerk
CLERK_PUBLISHABLE_KEY=pk_test_...
CLERK_SECRET_KEY=sk_test_...
CLERK_WEBHOOK_SECRET=whsec_...

# Database (Neon)
DATABASE_URL=postgresql://...

# Redis (Upstash)
REDIS_URL=rediss://...

# Composio
COMPOSIO_API_KEY=cmp_...
COMPOSIO_WEBHOOK_SECRET=whsec_...

# LLM (OpenAI-compatible, configurable via config/models.json)
OPENAI_API_KEY=sk-...

# Storage (Cloudflare R2)
R2_ACCOUNT_ID=...
R2_ACCESS_KEY_ID=...
R2_SECRET_ACCESS_KEY=...
R2_BUCKET=dotty-attachments

# Mem0
MEM0_API_KEY=mem0_...
```

**Model config:** `config/models.json` is the single place to change LLM provider or model. No code changes required to switch models.

---

## Project Structure

```
dotty/
├── Dockerfile
├── fly.toml
├── docker-compose.yml          # local dev: Postgres + Redis
├── package.json
├── tsconfig.json
├── vitest.config.ts
├── config/
│   └── models.json             # LLM provider config
├── src/
│   ├── main.ts                 # Fastify boot + Socket.io mount
│   ├── app/
│   │   ├── plugins/            # clerk, composio, mem0, rate-limit
│   │   ├── routes/             # auth, integrations, conversations, upload, memories, triggers, onboarding
│   │   ├── services/           # agent-runtime, composio-manager, mem0-manager
│   │   └── lib/
│   │       ├── tools/          # builtin, gmail, calendar
│   │       ├── prompts/        # system-prompt, token-budget, BOOTSTRAP.md
│   │       ├── model-config.ts
│   │       └── sanitize.ts
│   └── worker/                 # cron, briefing
├── migrations/
│   └── 001_initial.sql         # Full DDL with RLS policies
├── scripts/
│   └── migrate.js              # Apply migrations
└── tests/
    ├── unit/
    ├── integration/
    └── e2e/
```

---

## Requirements Documents

| Document | Purpose |
|----------|---------|
| `docs/requirements/PRD-Dotty.md` | What to build. Non-negotiable decisions, component inventory, blocking questions. |
| `docs/requirements/TRD-Dotty.md` | How to build. Implementation spec. 8 vertical slices with entry/exit criteria. |
| `AGENTS.md` | Agent handoff context. Locked decisions, slice status, critical implementation notes. |

---

## Development

### Running locally

```bash
npm run dev          # Start app (requires Docker compose up)
npm run build        # TypeScript build
npm test             # Unit tests
npm run test:e2e     # E2E tests (Playwright)
```

### Adding a new tool

1. Add to `src/app/lib/tools/builtin.ts` (if always available) or `gmail.ts`/`calendar.ts` (if integration-gated).
2. Use `sanitizeToolAlias()` for per-account tool names.
3. Register in `buildMultiAccountTools()` in `agent-runtime.ts`.
4. Add tests in `tests/integration/`.

---

## License

Internal — pre-funding.