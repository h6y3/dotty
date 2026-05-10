# Agent Instructions — Dotty MVP

**Last updated:** May 2026  
**TRD version:** 1.1  
**Status:** Requirements locked. Ready for implementation.

---

## What is Dotty?

Dotty is a SaaS-hosted personal AI assistant with a cheeky-best-friend personality. The MVP frontend is a PWA (React 19 + TypeScript + TailwindCSS + Vite). Key capabilities:
- Multi-account Gmail + Google Calendar per user (work + personal, same provider)
- Web search (Tavily/SerpAPI via Composio)
- File attachment upload + reading
- Long-term memory via Mem0 SaaS
- Real-time streaming chat via WebSocket (Socket.io)
- Daily morning briefing + per-account notification rules (immediate/digest/off)

**Target users:** Busy executives with complex lives — work Gmail + personal Gmail + multiple calendars.

---

## Documents

| Doc | Location | Purpose |
|-----|----------|---------|
| PRD | `docs/requirements/PRD-Dotty.md` | What to build. Non-negotiable decisions locked. Open questions marked `[DECISION NEEDED]`. |
| TRD | `docs/requirements/TRD-Dotty.md` | How to build. Implementation spec. 8 vertical slices. Entry/exit criteria per slice. |
| **You are here** | `docs/requirements/AGENTS.md` | Handoff context. Decisions made. What's done. What's next. |

---

## Architecture Decisions (Locked)

These decisions were made in the Q&A phase and are **not revisitable** without a full team discussion:

| # | Decision | Choice |
|---|----------|--------|
| 1 | Hosting platform | **Fly.io** |
| 2 | Containerization | **Docker from day one** |
| 3 | Object storage | **Cloudflare R2** (zero egress fees) |
| 4 | Redis | **Upstash single instance** |
| 5 | DB filtering | **PostgreSQL RLS** (enforced at DB layer) |
| 6 | Memory system | **Mem0 SaaS** (no custom pgvector) |
| 7 | Retention | **Indefinite** (partition-ready for future caps) |
| 8 | Security model | **Perimeter now, zero-trust later** |
| 9 | API framework | **Fastify** |
| 10 | WebSocket library | **Socket.io** (HTTP long-polling fallback for mobile Safari) |
| 11 | Rate limits | **100 msg/min, 1000 tool calls/min per user** |
| 12 | Offline behavior | **Fail immediately** (no queue-and-retry) |
| 13 | Cron fallback | **Opt-in only** |

**Also locked (from PRD — non-negotiable):**
- pi-agent-core (`@earendil-works/pi-agent-core`) as agent runtime
- Composio for tool integrations (native SDK, not MCP)
- Clerk for auth (Apple Sign-In + email, JWT)
- Multi-account Gmail + Calendar in MVP (not V2)
- Model-agnostic backend via `config/models.json`
- WebSocket for real-time chat

---

## Decisions Made During TRD Review

These were previously `[DECISION NEEDED]` in the PRD. The following were resolved:

| Decision | Resolution | Rationale |
|----------|------------|-----------|
| Error tone | **Stay in character for non-critical, neutral for data-affecting** | Cheeky for `tool_timeout`, `rate_limit`; apologetic neutral for `oauth_token_expired`, `snapshot_write_failed` |
| Disambiguation UI | **Inline pill buttons inside chat bubble** | Consistent with iMessage style; no modal |
| web_search tool | **Builtin from Slice 1, always registered** | Composio `websearch` toolkit included in initial session creation |
| Test accounts | **Gmail sandbox with `@test` suffix** | Real Gmail only in manual testing |
| Builtin tools | **`web_search` + `read_file` from Slice 1** | Additional tools gated behind integrations |
| Artifact types | **`email_draft`, `schedule_table`, `timeline`, `comparison_table`** | `---TYPE:...---` delimiter MVP; JSON artifacts in V1.1 |
| BOOTSTRAP.md | **Template at `src/app/lib/prompts/BOOTSTRAP.md`** | Variables: `{name}`, `{aliases}`, `{tone}` |
| Personality prompt | **Two variants (casual + professional) defined in TRD Section 2.5** | Casual default; professional via slider |
| Tool naming | **`sanitizeToolAlias()` function centralized in `src/lib/sanitize.ts`** | `provider_action_sanitized_alias` format |
| Model config | **Schema in TRD Section 2.9; `getActiveModel()` in `src/lib/model-config.ts`** | `config/models.json` is the single config point |

---

## Slice Status

| Slice | Name | Status | Notes |
|-------|------|--------|-------|
| 1 | The Chat Pipe | **NOT STARTED** | Entry: Clerk webhook + DB migrations done |
| 2 | One Gmail | **NOT STARTED** | Entry: Slice 1 chat pipe verified |
| 3 | Multi-Account | **NOT STARTED** | Entry: Slice 2 Gmail verified |
| 4 | Calendar | **NOT STARTED** | Entry: Slice 3 multi-account verified |
| 5 | Memory | **NOT STARTED** | Entry: Slice 4 Calendar verified |
| 6 | Morning Briefing | **NOT STARTED** | Entry: Slice 5 Memory verified |
| 7 | Attachments & Artifacts | **NOT STARTED** | Entry: Slice 5 Memory verified + R2 configured |
| 8 | Onboarding & Polish | **NOT STARTED** | Entry: Slices 1–7 all verified |

Each slice has **entry criteria** and **exit criteria** in `TRD-Dotty.md`. Do not move to the next slice until all exit criteria for the current slice are verified.

---

## Critical Implementation Notes for the Next Agent

### System Prompt is Mutable, Not Rebuilt From Scratch

A common mistake would be to recreate the `AgentSession` on every turn. **Don't.** The correct pattern is:
```typescript
// On every turn, mutate the existing session's system prompt in place
const session = this.sessions.get(userId)!;
const activeAccounts = await db.getUserIntegrations(userId);
const memories = await mem0.search({ userId, query: currentMessage, limit: 10 });
session.agent.state.setSystemPrompt(buildSystemPrompt(userId, activeAccounts, memories));
session.agent.state.tools = buildMultiAccountTools(userId, activeAccounts);
```

Tools are also reassigned per-session when integrations change (`session.agent.state.tools = ...`). This is how Dotty gains new capabilities mid-session when a user connects a new Gmail account.

### Every Turn: Rebuild System Prompt + Tools

The system prompt is NOT static. It has 5 layers (Section 2.5 of TRD):
1. Base Identity (~500 tokens) — casual or professional variant
2. Memory Injection (~1500 tokens max) — from Mem0
3. Account Context (~500 tokens) — only present when integrations exist
4. Tool Context (~300 tokens) — dynamic based on registered tools
5. Session Context (~100 tokens) — current time, user timezone

Layer 1 changes only when personality slider is adjusted. Layer 2 changes every turn (Mem0 search). Layer 3 changes when integrations are added/removed. Layer 4 changes when tools are reassigned.

### Token Budget Enforcement

Compaction triggers at 90k tokens in the running context window. The compaction function collapses the oldest 20 messages into a 2-sentence summary and replaces that block. This must happen as a **pre-turn step**, not after the turn starts.

### Composio Initialization Includes `websearch` From Day One

```typescript
const session = await client.create(userId, {
  toolkits: ['gmail', 'websearch'], // websearch always included
  multiAccount: { enable: true, maxAccountsPerToolkit: 5 },
});
```

`web_search` tool must work **before** any integrations are connected. This is tested in Slice 1 exit criteria.

### RLS Requires `SET LOCAL` Per Connection

Every PostgreSQL connection must have the user context set before queries run. In Fastify, this is done in a plugin that runs on every request:
```typescript
// After Clerk JWT verified, set RLS context
await db.query(`SET LOCAL app.current_user_id = '${internalUserId}'`);
await db.query(`SET LOCAL app.current_clerk_id = '${clerkUserId}'`);
```

Every table has an RLS policy enforced at the database layer. Never rely on app-level `WHERE user_id = ?` alone — the DB is the backstop.

### Offline Queue Is Designed But Not a Retry Queue

The offline queue (`session:offline_queue:{userId}`) is NOT a message queue for reliable delivery. It stores messages sent while the runtime was unavailable so they can be replayed on reconnect. **The fail-immediately decision means the client is told the send failed.** The queue just prevents message loss for the case where the user's client kept trying while the server was briefly down.

### Stub Markers Must Be Preserved

Each slice leaves known stubs with `// TODO(Slice N)` markers. These are not holes — they are explicit deferrals with a slice number. When working on Slice 3, search for `// TODO(Slice 3)` and resolve them. When adding new work, add `// TODO(Slice N)` for deferred items.

---

## How to Work Through the Slices

### Starting fresh on a new machine
1. `git clone` the repo
2. Copy `.env.example` to `.env`, fill in all secrets (see TRD Section 8.4 for all env vars)
3. `docker compose up` (starts local dev: Postgres, Redis, app)
4. Run migrations: `node scripts/migrate.js` (applies `migrations/001_initial.sql`)
5. `npm run dev` (starts the app)
6. Use Clerk dev instance + Gmail sandbox accounts for testing

### Testing a slice
Each slice has explicit exit criteria in `TRD-Dotty.md`. Work through the checklist before calling a slice done. The checklist should be verifiable by a human clicking through the app OR by running the E2E test suite.

### When you hit a new `[DECISION NEEDED]` not in the TRD
1. Do not just pick a direction and keep building
2. Write up the decision, options, and pros/cons in a GitHub Issue or directly in AGENTS.md
3. Ping the team for sign-off before proceeding

### When you find the TRD is wrong
If implementation reveals something in the TRD doesn't work (a tool name, a protocol detail, a schema choice), do not silently work around it. Update the TRD to reflect reality. The TRD is the second source of truth after the PRD.

---

## Project Structure

```
dotty/
├── Dockerfile
├── fly.toml
├── package.json
├── tsconfig.json
├── vitest.config.ts
├── config/
│   └── models.json                 # LLM provider config (openai, etc.)
├── src/
│   ├── main.ts                     # Fastify boot, Socket.io mount
│   ├── app/
│   │   ├── plugins/
│   │   │   ├── clerk.ts            # JWT verification + webhook handler
│   │   │   ├── composio.ts         # Composio client manager
│   │   │   ├── mem0.ts             # Mem0 wrapper
│   │   │   └── rate-limit.ts       # Redis token bucket
│   │   ├── routes/
│   │   │   ├── auth.ts             # POST /api/webhook/clerk
│   │   │   ├── integrations.ts     # CRUD for connected accounts
│   │   │   ├── conversations.ts    # GET /api/conversations (paginated)
│   │   │   ├── upload.ts           # POST /api/upload (R2 multipart)
│   │   │   ├── memories.ts         # GET/DELETE /api/memories (Mem0 proxy)
│   │   │   ├── triggers.ts         # POST /api/triggers/email (webhook receiver)
│   │   │   └── onboarding.ts       # GET /api/onboarding/status
│   │   ├── services/
│   │   │   ├── agent-runtime.ts    # AgentRuntime + getOrCreateSession
│   │   │   ├── composio-manager.ts # OAuth + multi-account + tool execution
│   │   │   └── mem0-manager.ts     # Mem0: add, search, get_all, delete
│   │   └── lib/
│   │       ├── tools/
│   │       │   ├── builtin.ts      # web_search, read_file (always registered)
│   │       │   ├── gmail.ts        # gmail_search_all, gmail_send_*, gmail_list_*
│   │       │   └── calendar.ts     # calendar_view_all, calendar_create_*
│   │       ├── prompts/
│   │       │   ├── system-prompt.ts # buildSystemPrompt() — 5 layers
│   │       │   ├── token-budget.ts  # estimateTokens(), compactMessages()
│   │       │   └── BOOTSTRAP.md     # First-run template
│   │       ├── model-config.ts     # getActiveModel() from config/models.json
│   │       └── sanitize.ts          # sanitizeToolAlias()
│   └── worker/
│       ├── cron.ts                # node-cron (briefing + trigger fallback)
│       └── briefing.ts            # Briefing composition logic
├── migrations/
│   └── 001_initial.sql             # Full DDL with RLS policies
└── tests/
    ├── unit/                      # Vitest: prompt builders, sanitize, token budget
    ├── integration/               # Mocked Composio + Mem0
    └── e2e/                       # Playwright: full signup → send email flow
```

---

## Key Files and What They Do

| File | Purpose |
|------|---------|
| `src/app/services/agent-runtime.ts` | Core session management. `getOrCreateSession()` is the main entry point. |
| `src/app/lib/prompts/system-prompt.ts` | Builds the 5-layer system prompt. Called on every turn. |
| `src/app/lib/tools/builtin.ts` | `web_search` + `read_file` — always registered from Slice 1. |
| `src/app/lib/tools/gmail.ts` | All Gmail tools. `buildGmailTools()` called when integrations change. |
| `src/app/lib/tools/calendar.ts` | All Calendar tools. `buildCalendarTools()` called when integrations change. |
| `src/app/lib/prompts/BOOTSTRAP.md` | First-run introduction. Loaded on `user.created` webhook. |
| `src/app/routes/auth.ts` | Clerk webhook endpoint. All user lifecycle events handled here. |
| `config/models.json` | The only place to change LLM provider or model. |

---

## Environment Variables

```bash
# Required for all environments
NODE_ENV=production          # or 'development'
PORT=3000

# Clerk
CLERK_PUBLISHABLE_KEY=pk_test_...
CLERK_SECRET_KEY=sk_test_...
CLERK_WEBHOOK_SECRET=whsec_...  # for svix webhook verification

# Database (Neon)
DATABASE_URL=postgresql://...

# Redis (Upstash)
REDIS_URL=rediss://...

# Composio
COMPOSIO_API_KEY=cmp_...
COMPOSIO_WEBHOOK_SECRET=whsec_...

# LLM
OPENAI_API_KEY=sk-...
# Additional providers in config/models.json

# Storage (Cloudflare R2)
R2_ACCOUNT_ID=...
R2_ACCESS_KEY_ID=...
R2_SECRET_ACCESS_KEY=...
R2_BUCKET=dotty-attachments

# Mem0
MEM0_API_KEY=mem0_...

# Observability
LOG_LEVEL=info
```

---

## What's Pending (Not Started)

- [ ] Full database migration script (`migrations/001_initial.sql`) not yet written
- [ ] No code written yet — all 8 slices are greenfield
- [ ] No E2E test suite
- [ ] No unit test suite
- [ ] No CI/CD pipeline (Docker build + Fly deploy not automated)
- [ ] `scripts/migrate.js` not written
- [ ] Composio webhook endpoint (`/api/triggers/email`) not implemented
- [ ] Cron worker not implemented

---

## Notes for Specific Slices

### Slice 1 — The Chat Pipe
This is the hardest slice because it requires getting pi-agent-core streaming integration right. The main technical risk is making `text_delta` events stream character-by-character to the client. Key things to get right:
- pi-agent-core `message_update` events must be captured and forwarded to Socket.io
- Socket.io must use `transports: ['websocket', 'polling']` — not websocket-only
- Clerk webhook must insert a `user` row AND a `user_agents` row AND create a Mem0 user AND insert the BOOTSTRAP.md system message on first `user.created`
- The initial tool set is `builtin.ts` (web_search + read_file), NOT empty

### Slice 2 — One Gmail
The Composio OAuth flow is the main complexity. The Connect Link opens in a popup; the callback must store the `connected_account_id`. Tool execution with the Composio SDK must handle `oauth_token_expired` errors cleanly.

### Slice 3 — Multi-Account
`buildMultiAccountTools()` dynamically generates per-account tool names using `sanitizeToolAlias()`. This function must be used consistently — never hardcode a tool name. The `account_prompt` interception in the Agent Runtime is also subtle: the API Gateway must intercept the tool call BEFORE it reaches Composio, return the `account_prompt` event, suspend the turn, and resume only when `resolve_account` arrives.

### Slice 5 — Memory (Mem0)
Mem0 is an external SaaS. The `mem0-manager.ts` wraps all Mem0 calls. Key: after every completed assistant turn, extract memories via Mem0's add API. Before every turn, inject via Mem0's search API with the current user message as query.

### Slice 6 — Morning Briefing
The cron runs inside the same process via `node-cron`. Every 15 minutes it queries for users whose `notify_digest_hour` is in the current window. The briefing is a system-initiated turn — a special `AgentSession.prompt()` call with a briefing system prompt. If the user is offline (no `ws:sid:{userId}` in Redis), the push is stored in Redis and delivered on next connect (or the user sees it on next app open).

---

## Glossary

| Term | Definition |
|------|------------|
| **AgentSession** | A pi-agent-core session object. One per user. Held in memory (Map). Evicted after 5 min idle. |
| **AgentRuntime** | The class that manages all AgentSessions. Instantiated once in `main.ts`. |
| **Tool call interception** | When the LLM calls a `gmail_send_*` tool with multiple accounts, the API Gateway intercepts the call, emits `account_prompt`, and suspends the turn. |
| **Builtin tools** | `web_search` and `read_file`. Always registered. Not gated behind integrations. |
| **Per-account tools** | Tools generated dynamically: `gmail_send_work_gmail`, `calendar_create_personal_calendar`, etc. |
| **Session snapshot** | Full state written to PostgreSQL after each completed turn. Used for recovery on restart. |
| **RLS** | Row-Level Security. PostgreSQL-level data isolation. Every query is scoped to the current user. |
| **Composio session** | A per-user Composio client instance, cached in Redis for 1 hour. |
| **Mem0** | External SaaS for long-term memory. We store `mem0_id` in our `users` table. |
| **BOOTSTRAP.md** | The first system message injected on a new user's first session. Contains name, aliases, tone. |
| **Layer 2 (Memory)** | The system prompt layer that injects Mem0 search results. Called with the current user message as the Mem0 query. |
| **Compaction** | When context window exceeds 90k tokens, oldest 20 messages are collapsed into a 2-sentence summary. |
| **Socket.io fallback** | HTTP long-polling used when WebSocket connection fails (common on mobile Safari). |

---

*When you make a decision during implementation that isn't already captured in the TRD, update this file and the TRD to reflect what you learned.*