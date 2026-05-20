# Dotty — Technical Requirements Document

**Version:** 1.1 — Post-gap-analysis additions  
**Date:** May 2026  
**Classification:** Internal — pre-funding  
**Status:** Locked  

This document is the implementation specification for the Dotty MVP. It is organized as **vertical slices**, each representing one integrated, demoable piece of the product. Slices are sequenced by risk: novel and uncertain first, known and cosmetic last.

Every slice includes frontend affordances, backend wiring, database schema, and a demo script. Each slice builds on the previous one; nothing is integrated in the 11th hour.

---

## 1. Locked Architecture Decisions

| # | Decision | Choice | Rationale |
|---|----------|--------|-----------|
| 1 | **Hosting platform** | Fly.io | Best WebSocket support; private networking ready for zero-trust migration later. |
| 2 | **Containerization** | Docker from day one | Local dev matches prod; future multi-region migration is trivial. |
| 3 | **Object storage** | Cloudflare R2 | Zero egress fees; simpler than AWS. |
| 4 | **Redis** | Upstash single instance | Cheapest viable option for 0–1000 users; cluster upgrade path available. |
| 5 | **DB filtering** | PostgreSQL RLS | Enforced at the database layer; no leaking another user's data via a forgotten `WHERE` clause. |
| 6 | **Memory system** | Mem0 SaaS | Outsources extraction, semantic search, deduplication, and decay. No custom pgvector pipeline. |
| 7 | **Retention** | Indefinite (partition-ready) | Tables partitioned by month; soft quotas enforceable later for COGS control. |
| 8 | **Security model** | Perimeter now, zero-trust later | Faster to ship; Fly private networking `.internal` makes migration frictionless. |
| 9 | **API framework** | Fastify | Faster JSON/HTTP; WebSocket backpressure handled well. |
| 10 | **WebSocket library** | Socket.io | HTTP long-polling fallback critical for mobile Safari on patchy networks. |
| 11 | **Rate limits** | 100 msg/min, 1000 tool calls/min per user | Redis token bucket; tunable without migration. |
| 12 | **Offline behavior** | Fail immediately | Simpler client code; no retry queue or deduplication logic. |
| 13 | **Cron fallback** | Opt-in only | Protects Composio quota/COGS. Users not opted in rely on webhooks. |

**Derived implications:**
- No Kubernetes. One Docker container on Fly.io for MVP.
- No pgvector. No `memories` content table. Memory data lives in Mem0.
- No VPC. API Gateway, Agent Runtime, and cron job all run in one process.

---

## 2.5 System Prompt Architecture

The system prompt is assembled from five layers, injected fresh on every turn. This architecture is the single source of truth for how Dotty's personality, context, and capabilities are constructed at runtime.

### Layer 1 — Base Identity (static, ~500 tokens)

Dotty's core personality. The string changes only when the personality slider (Slice 8) is adjusted.

**Casual (default):**
```
You are Dotty, a cheeky best friend AI. You speak like you're texting a close friend.
You are warm, witty, and honest about your limitations. When you don't know something, say so.
You remember facts about the user and reference them naturally. You use occasional emoji tastefully.
Never pretend to be human. Never apologize excessively. Keep messages concise and conversational.
```

**Professional:**
```
You are Dotty, a professional AI assistant. You are efficient, warm, and concise.
You speak in a composed, articulate manner. You remember facts about the user and reference them naturally.
Never pretend to be human. Keep messages focused and actionable.
```

### Layer 2 — Memory Injection (dynamic, ~1500 tokens max)

Results from Mem0 semantic search. Injected as a bulleted list. Always ends with "Now proceed."

```
Facts about {user_name}:
{top_10_memories_formatted}
Now proceed.
```

If no memories: `Facts about {user_name}: (none yet)`. Never skip the layer — the LLM needs the structure even when empty.

### Layer 3 — Account Context (dynamic, ~500 tokens max)

All connected integrations. This layer is absent in Slice 1 (no integrations yet) and grows as accounts are connected.

```
Your connected accounts:
- Gmail: {alias} ({email_address}), default: {is_default}
- Calendar: {alias}, default: {is_default}

When searching emails or events, check ALL accounts unless the user specifies one.
When sending emails or creating events, confirm the account if ambiguous.
```

### Layer 4 — Tool Context (dynamic, ~300 tokens)

Registry of available tools for this session. Built from `session.agent.state.tools` (see Session State below).

```
You have access to:
{tool_name} — {tool_description}
```

### Layer 5 — Session Context (static, ~100 tokens)

```
Current time: {ISO8601}
User timezone: {user.timezone}
Current date: {YYYY-MM-DD}
```

### Token Budget

| Layer | Max Tokens | Notes |
|-------|-----------|-------|
| Layer 1: Base Identity | 500 | Casual or professional variant |
| Layer 2: Memory Injection | 1,500 | Truncate from oldest/lowest-confidence first |
| Layer 3: Account Context | 500 | Grows with integrations; truncate if needed |
| Layer 4: Tool Context | 300 | Dynamic; shrinks when tools are few |
| Layer 5: Session Context | 100 | Static |
| **Base total** | **2,900** | Before conversation history |
| Conversation history | ~2,000 | Last 20 messages; compacted if needed |
| LLM response working room | ~2,000 | Target headroom |
| **Total target** | **~6,900** | Well within 128k context window |

**Compaction trigger:** When system prompt (layers 1–5 + injected memories) exceeds 90k tokens, collapse the oldest 20 messages into a 2-sentence summary and replace that block in context. This is run as a pre-turn step before sending to the LLM.

**Soft eviction:** At 100k tokens in the running context window, begin compacting. The LLM provider (OpenAI) handles overflow with an error; we don't enforce a hard limit — we proactively compact before hitting the wall.

---

## 2.6 Session State Management

The Agent Runtime manages `AgentSession` objects from pi-agent-core. Key rules:

- **`systemPrompt` is mutable.** We rebuild it every turn by calling `session.agent.state.setSystemPrompt(buildSystemPrompt(...))`. We do not recreate the session object — we mutate the existing session's prompt reference.
- **Tools are registered per-session.** After `getOrCreateSession()`, we assign tools via `session.agent.state.tools = buildMultiAccountTools(userId, activeAccounts)`. When a user adds a new Gmail account mid-session, we call `buildMultiAccountTools` again and reassign to `session.agent.state.tools`.
- **Model is assigned at session creation.** Config is read from `config/models.json` via `getActiveModel()` at startup. Switching models mid-session is not supported in MVP.
- **Message history is assigned at session creation.** Snapshot `messages` array is passed to `createAgentSession({ initialState: { messages: snapshot.messages } })`. Between turns, pi-agent-core appends to this array internally. On turn end, we write the full array back to `session_snapshots`.

```typescript
// src/app/services/agent-runtime.ts
async getOrCreateSession(userId: string): Promise<AgentSession> {
  if (this.sessions.has(userId)) {
    const session = this.sessions.get(userId)!;
    // Mutate systemPrompt in place on every turn
    const activeAccounts = await db.getUserIntegrations(userId);
    const memories = await mem0.search({ userId, query: '', limit: 10 });
    session.agent.state.setSystemPrompt(buildSystemPrompt(userId, activeAccounts, memories));
    // Reassign tools if integrations changed (add/remove account)
    session.agent.state.tools = buildMultiAccountTools(userId, activeAccounts);
    return session;
  }
  // ... create new session
}
```

---

## 2.7 pi-agent-core → Socket.io Event Mapping

Every event emitted by the pi-agent-core `Agent` class is mapped to a Socket.io server→client event. This is the canonical mapping:

| pi-agent-core Event | Socket.io Event Emitted | Payload |
|--------------------|-----------------------|---------|
| `agent_start` | *(internal)* | Log `agent_start`, set Redis `session:active:{userId}` |
| `turn_start` | *(internal)* | Log `turn_start` with traceId |
| `message_start` | *(internal)* | Initialize message record in `messages` table |
| `message_update` with `text_delta` | `text_delta` | `{ messageId, delta: string }` |
| `tool_execution_start` | `tool_start` | `{ toolName, args, accountAlias? }` |
| `tool_execution_end` | `tool_end` | `{ toolName, result, durationMs }` |
| `message_end` | `message_complete` | `{ messageId }` |
| `turn_end` | *(internal)* | Trigger memory extraction + snapshot write |
| `agent_end` | *(internal)* | Log; Redis TTL refreshed for active sessions |

**Steering:** When the server receives a `steer` client event, it calls `agentSession.steer(content)` on the pi-agent-core session. Steering interrupts after the current tool batch completes — it cannot interrupt mid-tool. The `steer` event does not emit a response event; the agent simply continues from the next turn with the new steering content prepended.

---

## 2.8 Tool Naming Convention

All dynamically-generated tool names follow this convention:

**Format:** `{provider}_{action}_{sanitized_alias}`

**Sanitize function:**
```typescript
function sanitizeToolAlias(alias: string): string {
  return alias
    .toLowerCase()
    .replace(/\s+/g, '_')        // spaces → underscores
    .replace(/[^a-z0-9_]/g, ''); // strip non-alphanumeric except underscore
}
```

**Examples:**
- `"Work Gmail"` → `gmail_send_work_gmail`
- `"Personal Calendar"` → `calendar_create_personal_calendar`
- `"Sarah's Gmail"` → `gmail_search_sarahs_gmail` (apostrophe stripped)

This applies to all per-account tools generated in `buildMultiAccountTools()`. Builtin tools (`web_search`, `read_file`) use fixed names and do not go through this function.

---

## 2.9 Model Configuration Schema

`config/models.json` is the single place to configure LLM providers. Switching providers requires only a config change, no code.

```json
{
  "defaultProvider": "openai",
  "models": {
    "openai": {
      "id": "gpt-4o",
      "apiKeyEnv": "OPENAI_API_KEY",
      "endpoint": "https://api.openai.com/v1",
      "supportsStreaming": true,
      "supportsVision": true,
      "contextWindow": 128000,
      "costPer1kTokens": 0.005
    }
  }
}
```

```typescript
// src/lib/model-config.ts
import { readFileSync } from 'fs';
import { join } from 'path';

export interface ModelConfig {
  defaultProvider: string;
  models: Record<string, {
    id: string;
    apiKeyEnv: string;
    endpoint: string;
    supportsStreaming: boolean;
    supportsVision: boolean;
    contextWindow: number;
    costPer1kTokens: number;
  }>;
}

let config: ModelConfig | null = null;

export function getModelConfig(): ModelConfig {
  if (!config) {
    config = JSON.parse(readFileSync(join(process.cwd(), 'config/models.json'), 'utf-8'));
  }
  return config;
}

export function getActiveModel() {
  const cfg = getModelConfig();
  const provider = cfg.defaultProvider;
  const model = cfg.models[provider];
  if (!model) throw new Error(`Model provider "${provider}" not configured in config/models.json`);
  return {
    provider,
    id: model.id,
    endpoint: model.endpoint,
    apiKey: process.env[model.apiKeyEnv],
  };
}
```

---

## 2.10 Disambiguation Heuristics

When the user's request is ambiguous about which account to use, Dotty applies these rules in order:

| Scenario | Rule | Rationale |
|----------|------|-----------|
| "What's new in my email?" | Search ALL Gmail accounts, merge by date, tag with alias | User wants the full picture |
| "Reply to Sarah" | Check Work contacts first; if Sarah appears in both, ask inline | Work context is higher-fidelity |
| "Email Mom" | Default to Personal Gmail | Heuristic: "Mom" = personal relationship |
| "Schedule a meeting" | Default to Work Calendar | Heuristic: "meeting" = work context |
| "Add 'lunch with Mom'" | Default to Personal Calendar | Heuristic: "Mom" = personal |
| "Email the CEO" | Default to Work Gmail; if CEO found in both, ask | Work first; confirm if ambiguous |
| Explicit: "from Work Gmail" | Use Work Gmail directly | User has specified |
| Ambiguous + send action | Always emit `account_prompt`, do not auto-select | High stakes; confirm before sending |
| Ambiguous + search/read | Auto-select default account | No irreversible action |

**Learning from corrections:** If user says "No, use Personal," Dotty stores a Mem0 preference: `{"type": "preference", "content": "When sending to [recipient], use Personal Gmail"}`. This is injected via Layer 2 in subsequent turns.

---

## 2.11 Error Tone Mapping

Dotty stays in character for non-critical errors. She switches to neutral + apologetic for data-affecting errors.

| Error Type | Example | Dotty's Response Style |
|-----------|---------|----------------------|
| Non-critical | `tool_timeout`, `rate_limit_exceeded` | Cheeky: "Oof, my Gmail connection is being moody. Want me to try again?" |
| Performance | `llm_unavailable`, `llm_rate_limited` | Neutral: "My brain is taking a quick nap. Try again in a minute?" |
| Data-affecting | `oauth_token_expired`, `snapshot_write_failed` | Neutral + apologetic: "Hmm, I've lost connection to Work Gmail. Want to re-link it?" |
| Auth | `auth_expired` | Neutral: "Your session expired. Let's sign you back in." |

**Rule:** Dotty never lies about being an AI. Never says "I forgot" for a system error. Always attributes failures honestly to technical causes in a conversational way.

---

## 2.12 Clerk Webhook Handler

**Endpoint:** `POST /api/webhook/clerk`

**Verification:** Validate `svix` signature header using Clerk's signing secret (`CLERK_WEBHOOK_SECRET`). Use the `svix` library:

```typescript
import { Webhook } from 'svix';
const webhook = new Webhook(CLERK_WEBHOOK_SECRET);
webhook.verify(rawBody, headers);
```

**Events handled:**

| Event | Action |
|-------|--------|
| `user.created` | Insert `users` row (clerk_id, email, name); insert `user_agents` row with defaults; call `mem0.users.create({ external_id: userId })` → store returned `mem0_id` in `users.mem0_id` |
| `user.updated` | Update `users.email` if changed, `users.name` if changed |
| `user.deleted` | Delete from `users` (CASCADE removes all dependent rows per DDL) |

**On `user.created`, insert the BOOTSTRAP.md system message:** After creating `users` and `user_agents`, insert a system message into `messages` with role `system` and content equal to the BOOTSTRAP.md template (see Section 2.13). This ensures the first session starts with Dotty's introduction.

---

## 2.13 BOOTSTRAP.md

Located at `src/app/lib/prompts/BOOTSTRAP.md`. Loaded once per new user on first session creation.

```markdown
User name: {name}
Connected accounts: {aliases}
Personality: {tone}

You are Dotty. Your user has just signed up and connected their accounts.
Introduce yourself naturally and warmly. Reference their name and accounts.
Give them 2–3 suggested things to try first. Keep it short and enthusiastic.
```

**Variable substitution happens at runtime** (string replace on `{name}`, `{aliases}`, `{tone}`) before the content is inserted as a system message in `messages`.

---

## 2.14 Builtin Tools

These tools are always registered from Slice 1, regardless of integrations:

### `web_search`

```typescript
defineTool({
  name: 'web_search',
  label: 'Search the Web',
  description: 'Search the web for current information, weather, news, or anything that requires up-to-date data not in your training.',
  parameters: Type.Object({
    query: Type.String(),
    maxResults: Type.Integer({ default: 5 }),
  }),
  execute: async (id, params) => {
    // Composio 'websearch' toolkit wraps Tavily or SerpAPI
    const result = await composioSession.execute('WEBSEARCH_SEARCH', params);
    return result;
  },
});
```

Registered in Slice 1 as part of the initial tool set. Composio `websearch` toolkit must be included in the initial Composio session creation (see Slice 2 Composio initialization).

### `read_file`

```typescript
defineTool({
  name: 'read_file',
  label: 'Read Attachment',
  description: 'Read the content of an uploaded file (image or PDF). Use this when the user asks about a file they attached.',
  parameters: Type.Object({
    uploadId: Type.String(),
  }),
  execute: async (id, params) => {
    const upload = await db.getUpload(params.uploadId);
    const signedUrl = await generateR2SignedUrl(upload.r2Key);
    const response = await fetch(signedUrl);
    const buffer = await response.arrayBuffer();
    // For images: base64 encode. For PDFs: extract text via pdf-parse or similar.
    // Return { content: string, mimeType: string }
  },
});
```

Registered in Slice 1 alongside `web_search`. Both are in the initial `tools` array even before integrations are connected.

---

## 2.15 Tool Execution & Retry Schedule

| Parameter | Value |
|-----------|-------|
| Tool timeout | 30 seconds per call |
| Timeout retry | 1 automatic retry on timeout |
| Rate limit backoff | Exponential: 1s → 2s → 4s, max 3 retries |
| After exhausted retries | Surface `tool_timeout` error to client via Socket.io |

```typescript
async function executeToolWithRetry(
  toolFn: () => Promise<ToolResult>,
  toolName: string,
  maxRetries = 3
): Promise<ToolResult> {
  const backoffMs = [1000, 2000, 4000];
  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      return await withTimeout(toolFn(), 30_000);
    } catch (err) {
      if (attempt === maxRetries) throw err;
      if (err.code === 'TIMEOUT') {
        console.warn(`Tool ${toolName} timed out, retry ${attempt + 1}/${maxRetries}`);
        await sleep(backoffMs[attempt]);
        continue;
      }
      if (err.code === 'RATE_LIMIT') {
        console.warn(`Tool ${toolName} rate limited, backoff ${backoffMs[attempt]}ms`);
        await sleep(backoffMs[attempt]);
        continue;
      }
      throw err; // Non-retryable error
    }
  }
  throw new Error(`Tool ${toolName} failed after ${maxRetries} retries`);
}
```

---

## 2.16 Offline Queue Flow

**Enqueueing (when runtime is unavailable):**
1. On message receive, API Gateway checks Redis `session:active:{userId}`.
2. If key is missing (pod was down or session evicted): append message to `session:offline_queue:{userId}` as JSON `{ id, content, attachments, timestamp }`.
3. Return success to client (message is queued).

**Replay (on session restore):**
1. After `getOrCreateSession()` loads snapshot, API Gateway checks `session:offline_queue:{userId}`.
2. Drain queue FIFO. For each queued message, forward to Agent Runtime as a new turn.
3. Clear the queue after successful replay.
4. If a queued message is older than 1 hour (TTL-expiry), discard it and emit `error: offline_messages_lost` to client: "Some messages sent while I was away couldn't be delivered. Sorry about that!"

---

## 2.17 Multi-Device Visibility Polling

No WebSocket relay between devices. Each device polls on visibility change:

```typescript
document.addEventListener('visibilitychange', () => {
  if (document.visibilityState === 'visible') {
    // Refresh conversation history
    fetch('/api/conversations?limit=50')
      .then(r => r.json())
      .then(data => updateMessageList(data.messages));
  }
});
```

On WebSocket reconnect (after disconnect), client also calls `/api/conversations` to refresh. Socket.io's built-in reconnect + backoff is used automatically.

---

## 2.18 Project File Structure

```
dotty/
├── Dockerfile
├── fly.toml
├── package.json
├── tsconfig.json
├── vitest.config.ts
├── config/
│   └── models.json
├── src/
│   ├── main.ts                     # Fastify boot, Socket.io mount, agent runtime init
│   ├── app/
│   │   ├── plugins/
│   │   │   ├── clerk.ts            # Clerk JWT verification + webhook handler
│   │   │   ├── composio.ts         # Composio client + session manager
│   │   │   ├── mem0.ts             # Mem0 client manager
│   │   │   └── rate-limit.ts      # Redis token bucket middleware
│   │   ├── routes/
│   │   │   ├── auth.ts             # Clerk webhook POST /api/webhook/clerk
│   │   │   ├── integrations.ts     # CRUD for integrations
│   │   │   ├── conversations.ts    # GET /api/conversations (paginated)
│   │   │   ├── upload.ts           # POST /api/upload (R2 multipart)
│   │   │   ├── memories.ts         # GET/DELETE /api/memories (Mem0 proxy)
│   │   │   ├── triggers.ts         # POST /api/triggers/email (Composio webhook)
│   │   │   └── onboarding.ts       # GET /api/onboarding/status
│   │   ├── services/
│   │   │   ├── agent-runtime.ts    # AgentRuntime class, session management
│   │   │   ├── composio-manager.ts # OAuth, multi-account session, tool execution
│   │   │   └── mem0-manager.ts     # Mem0 wrapper: add, search, get_all, delete
│   │   └── lib/
│   │       ├── tools/
│   │       │   ├── builtin.ts      # web_search, read_file
│   │       │   ├── gmail.ts        # gmail_search_all, gmail_send_*, gmail_list_*
│   │       │   └── calendar.ts     # calendar_view_all, calendar_create_*
│   │       ├── prompts/
│   │       │   ├── system-prompt.ts # buildSystemPrompt() — 5 layers + token budget
│   │       │   ├── token-budget.ts  # estimateTokens(), compactMessages()
│   │       │   └── BOOTSTRAP.md    # First-run introduction template
│   │       ├── model-config.ts     # getModelConfig(), getActiveModel()
│   │       └── sanitize.ts          # sanitizeToolAlias()
│   └── worker/
│       ├── cron.ts                # node-cron briefing + cron fallback jobs
│       └── briefing.ts            # Briefing composition logic
├── migrations/
│   └── 001_initial.sql             # Full DDL from TRD section 4.1
└── tests/
    ├── unit/
    │   ├── system-prompt.test.ts
    │   ├── sanitize.test.ts
    │   ├── token-budget.test.ts
    │   └── model-config.test.ts
    ├── integration/
    │   ├── agent-runtime.test.ts   # Mock Composio, mock Mem0
    │   ├── composio-manager.test.ts
    │   └── socketio-events.test.ts
    └── e2e/
        └── full-flow.test.ts       # Playwright: signup → connect Gmail → send email
```

---

## 3. Security Model

### 3.1 Authentication

Clerk handles signup/login. The PWA uses `@clerk/clerk-react`. Backend validates JWT via `@clerk/fastify`.

**Middleware flow:**
1. Client sends `Authorization: Bearer <clerk_jwt>` (HTTP) or token via Socket.io auth handshake.
2. Fastify plugin verifies JWT with Clerk JWKS.
3. After verification, `request.userId = clerkUserId`.
4. Lookup internal `users.id` via `users.clerk_id = request.userId`.
5. Set `SET LOCAL app.current_user_id = '<internal_user_id>'` on the PostgreSQL connection.
6. RLS policies enforce every query returns only that user's data.

### 3.2 RLS Policies

Every table has:

```sql
ALTER TABLE {table} ENABLE ROW LEVEL SECURITY;
CREATE POLICY user_isolation ON {table}
  USING (user_id = current_setting('app.current_user_id', true)::UUID);
```

Tables owned by Clerk (`users`) are filtered by `clerk_id` mapping.

### 3.3 OAuth Token Safety

- **No OAuth tokens in our database.** We only store Composio's stable `connected_account_id`.
- Composio handles token storage, refresh, and expiry.
- Expiry surfaced via Composio API error → runtime returns `oauth_token_expired` → frontend shows re-link prompt.

### 3.4 Webhook Verification

Composio signs payloads with `X-Composio-Signature` (HMAC-SHA256). Verified via:

```ts
crypto.timingSafeEqual(
  crypto.createHmac('sha256', COMPOSIO_WEBHOOK_SECRET).update(rawBody).digest('hex'),
  receivedSig
);
```

Failure returns HTTP 401 + structured log.

### 3.5 Attachment Access

- Uploaded to R2 at `{user_id}/{conversation_id}/{upload_id}.{ext}`.
- API Gateway generates signed URLs (15-min TTL) for reads.
- No public bucket listing. No direct client-to-R2 auth.

---

## 4. Database Schema (PostgreSQL / Neon)

### 4.1 DDL

```sql
-- Enable UUID extension
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- Users (Clerk owns identity; we augment)
CREATE TABLE users (
  id          UUID        PRIMARY KEY DEFAULT uuid_generate_v4(),
  clerk_id    TEXT        UNIQUE NOT NULL,
  email       TEXT        UNIQUE NOT NULL,
  name        TEXT,
  timezone    TEXT        DEFAULT 'America/New_York',
  mem0_id     TEXT,                     -- Mem0 external user ID
  created_at  TIMESTAMPTZ DEFAULT NOW(),
  updated_at  TIMESTAMPTZ DEFAULT NOW()
);
ALTER TABLE users ENABLE ROW LEVEL SECURITY;
CREATE POLICY user_isolation ON users
  USING (clerk_id = current_setting('app.current_clerk_id', true));

-- Per-user agent settings
CREATE TABLE user_agents (
  id                UUID  PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id           UUID  NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  agent_name        TEXT  DEFAULT 'Dotty',
  personality_config JSONB DEFAULT '{"tone": "cheeky", "formality": "casual"}',
  model_provider    TEXT  DEFAULT 'openai',
  model_id          TEXT  DEFAULT 'gpt-4o',
  created_at        TIMESTAMPTZ DEFAULT NOW(),
  updated_at        TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(user_id)
);
ALTER TABLE user_agents ENABLE ROW LEVEL SECURITY;
CREATE POLICY user_isolation ON user_agents
  USING (user_id = current_setting('app.current_user_id', true)::UUID);

-- Connected accounts (Gmail, Calendar, etc.)
CREATE TABLE integrations (
  id                    UUID  PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id               UUID  NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  provider              TEXT  NOT NULL,              -- 'gmail', 'googlecalendar'
  connected_account_id  TEXT  NOT NULL,              -- Composio stable ID
  alias                 TEXT  NOT NULL,               -- 'Work Gmail', 'Personal'
  email_address         TEXT,
  is_default            BOOLEAN DEFAULT FALSE,
  sort_order            INTEGER DEFAULT 0,
  status                TEXT  DEFAULT 'active',      -- 'active', 'expired', 'disconnected'
  notify_mode           TEXT  DEFAULT 'immediate',    -- 'immediate', 'digest', 'off'
  notify_digest_hour    INTEGER DEFAULT 9,
  cron_fallback_enabled BOOLEAN DEFAULT FALSE,       -- opt-in only
  created_at            TIMESTAMPTZ DEFAULT NOW(),
  updated_at            TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(user_id, provider, alias)
);
CREATE INDEX idx_integrations_user_id    ON integrations(user_id);
CREATE INDEX idx_integrations_provider   ON integrations(provider);
CREATE INDEX idx_integrations_status     ON integrations(status);
ALTER TABLE integrations ENABLE ROW LEVEL SECURITY;
CREATE POLICY user_isolation ON integrations
  USING (user_id = current_setting('app.current_user_id', true)::UUID);

-- Per-account preference filters
CREATE TABLE integration_preferences (
  id              UUID  PRIMARY KEY DEFAULT uuid_generate_v4(),
  integration_id  UUID  NOT NULL REFERENCES integrations(id) ON DELETE CASCADE,
  pref_type       TEXT  NOT NULL,              -- 'filter_label', 'sync_range', 'exclude_domain'
  pref_value      TEXT  NOT NULL,
  UNIQUE(integration_id, pref_type)
);
ALTER TABLE integration_preferences ENABLE ROW LEVEL SECURITY;
CREATE POLICY user_isolation ON integration_preferences
  USING (EXISTS (
    SELECT 1 FROM integrations i
    WHERE i.id = integration_id
      AND i.user_id = current_setting('app.current_user_id', true)::UUID
  ));

-- Agent session snapshots (state recovery on restart)
CREATE TABLE session_snapshots (
  id          UUID  PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id     UUID  NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  messages    JSONB NOT NULL,                  -- Flat array; tree preserved but linear for MVP
  tool_state  JSONB,                            -- In-flight tool state
  model_ref   TEXT,
  created_at  TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(user_id)
);
CREATE INDEX idx_session_snapshots_user_id  ON session_snapshots(user_id);
CREATE INDEX idx_session_snapshots_created   ON session_snapshots(created_at DESC);
ALTER TABLE session_snapshots ENABLE ROW LEVEL SECURITY;
CREATE POLICY user_isolation ON session_snapshots
  USING (user_id = current_setting('app.current_user_id', true)::UUID);

-- Conversation messages
CREATE TABLE messages (
  id           UUID  PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id      UUID  NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  role         TEXT  NOT NULL,               -- 'user', 'assistant', 'system'
  content      TEXT,
  tool_calls   JSONB,
  tool_results JSONB,
  attachments  JSONB,                        -- [{ id, url, mimeType, sizeBytes, filename }]
  model_used   TEXT,
  tokens_used  INTEGER,
  created_at   TIMESTAMPTZ DEFAULT NOW()
) PARTITION BY RANGE (created_at);
-- Create initial monthly partition; automate later via pg_partman
CREATE TABLE messages_2026_05 PARTITION OF messages
  FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');
CREATE INDEX idx_messages_user_id      ON messages(user_id);
CREATE INDEX idx_messages_created_at   ON messages(created_at DESC);
CREATE INDEX idx_messages_user_recent  ON messages(user_id, created_at DESC) WHERE role = 'user';
ALTER TABLE messages ENABLE ROW LEVEL SECURITY;
CREATE POLICY user_isolation ON messages
  USING (user_id = current_setting('app.current_user_id', true)::UUID);

-- Usage / billing events
CREATE TABLE usage_events (
  id           UUID  PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id      UUID  NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  event_type   TEXT  NOT NULL,   -- 'turn', 'tool_call', 'memory_extraction', 'feedback'
  provider     TEXT,
  model        TEXT,
  tokens       INTEGER,
  cost_cents   REAL,
  created_at   TIMESTAMPTZ DEFAULT NOW()
) PARTITION BY RANGE (created_at);
CREATE TABLE usage_events_2026_05 PARTITION OF usage_events
  FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');
CREATE INDEX idx_usage_user_id     ON usage_events(user_id);
CREATE INDEX idx_usage_created_at  ON usage_events(created_at DESC);
ALTER TABLE usage_events ENABLE ROW LEVEL SECURITY;
CREATE POLICY user_isolation ON usage_events
  USING (user_id = current_setting('app.current_user_id', true)::UUID);

-- Seen email deduplication (trigger idempotency)
CREATE TABLE seen_emails (
  message_id   TEXT  PRIMARY KEY,
  integration_id UUID NOT NULL REFERENCES integrations(id) ON DELETE CASCADE,
  user_id      UUID  NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  created_at   TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_seen_emails_created ON seen_emails(created_at);
ALTER TABLE seen_emails ENABLE ROW LEVEL SECURITY;
CREATE POLICY user_isolation ON seen_emails
  USING (user_id = current_setting('app.current_user_id', true)::UUID);
```

**Partitioning strategy:** `messages` and `usage_events` partitioned by month. Neon supports native partitioning. Add new partitions monthly; drop old partitions when retention policy changes. `session_snapshots` not partitioned (one row per user).

### 4.2 Redis Schema

| Key Pattern | TTL | Purpose |
|-------------|-----|---------|
| `session:active:{user_id}` | 5 min | Session resident in Agent Runtime |
| `composio:session:{user_id}` | 1 hour | Cached Composio session object |
| `ws:sid:{user_id}` | Session | Socket.io socket ID for push |
| `rate_limit:{user_id}` | 1 min | Token bucket counter |
| `trigger_queue:{integration_id}` | 1 hour | Queued digest notifications |
| `seen_email:{message_id}` | 24 hours | Webhook/cron deduplication |
| `session:offline_queue:{user_id}` | 1 hour | Messages sent while runtime was down (for replay on reconnect) |

---

## 5. WebSocket Protocol (Socket.io)

Socket.io namespaces and events replace the raw JSON protocol in the PRD. All messages are typed.

### 5.1 Client → Server Events

| Event | Payload | Purpose |
|-------|---------|---------|
| `message` | `{ id, content, attachments[] }` | User sends a chat message |
| `steer` | `{ content }` | Interrupt or redirect the agent mid-turn |
| `feedback` | `{ messageId, rating }` | Thumbs down on a message |
| `resolve_account` | `{ messageId, alias }` | User selects account after `account_prompt` |
| `presence` | `{ status: 'online' \| 'away' }` | Heartbeat / status update |

### 5.2 Server → Client Events

| Event | Payload | Purpose |
|-------|---------|---------|
| `text_delta` | `{ messageId, delta }` | Streaming LLM token |
| `tool_start` | `{ toolName, args, accountAlias? }` | Agent started using a tool |
| `tool_end` | `{ toolName, result, durationMs }` | Tool finished |
| `account_prompt` | `{ messageId, text, options[] }` | Ask user to disambiguate account |
| `message_complete` | `{ messageId }` | Full turn finished |
| `memory_update` | `{ memories[] }` | New facts extracted this turn |
| `push` | `{ title, body }` | Notification (digest/urgent email) |
| `error` | `{ code, message, retryAfter? }` | Error surfaced in chat |
| `typing` | `{ status: 'start' \| 'stop' }` | Agent typing indicator |

### 5.3 Connection & Auth

```ts
const io = new Server(fastify.server, { transports: ['websocket', 'polling'] });

io.use(async (socket, next) => {
  const token = socket.handshake.auth.token;
  const user = await clerk.verifyToken(token);
  socket.data.userId = user.id;
  next();
});
```

**Reconnection strategy (decided):** Socket.io handles reconnection with exponential backoff automatically. On reconnect, client receives no replay of missed messages (fail-immediately model). Client polls `/api/conversations` on reconnect to refresh history.

---

## 6. Error Code Catalog

All errors emitted via Socket.io `error` event or returned as HTTP JSON. Structured log sent to stdout.

| Code | HTTP | Description | Client Action |
|------|------|-------------|---------------|
| `invalid_attachment` | 400 | File type or size violation | Strip attachment, retry |
| `attachment_too_large` | 413 | >5 attachments per message | Reduce count |
| `upload_not_found` | 404 | Referenced upload ID missing | Re-upload |
| `rate_limit_exceeded` | 429 | Token bucket exhausted | Wait `retryAfter`, then retry |
| `auth_expired` | 401 | Clerk JWT invalid | Redirect to login |
| `oauth_token_expired` | 401 | One account token expired | Show inline re-link prompt |
| `oauth_all_expired` | 401 | All accounts for a provider expired | Show reconnect flow |
| `tool_timeout` | 504 | Tool call >30s after retry | Surface to user; suggest retry |
| `tool_execution_failed` | 502 | Composio non-timeout error | Silent retry x2, then surface |
| `session_not_found` | 404 | No snapshot to restore | Fresh session + inform user |
| `snapshot_write_failed` | 500 | DB write failed | Log; continue without snapshot |
| `llm_rate_limited` | 429 | Provider rate limit | pi-agent-core exponential backoff |
| `llm_unavailable` | 503 | Provider down | "Brain nap" message |
| `composio_down` | 502 | Composio API unreachable | Graceful degradation |
| `webhook_verification_failed` | 401 | Signature mismatch | Log + reject |

---

## 7. Vertical Slices

Each slice is a **vertical integration**: a real UI affordance wired to real backend logic and real data, demoable end-to-end.

### Slice 1: The Chat Pipe (Week 1)

**The one piece to prove:** A user can sign up and chat with Dotty. The full real-time WebSocket stream — text deltas, typing indicator, message history — works end-to-end.

**Why this slice first:**
- **Core:** Without chat, nothing else matters.
- **Small:** One screen. One API endpoint. One WebSocket event.
- **Novel:** pi-agent-core streaming integration is the main technical unknown.

**Entry criteria:** Clerk webhook handler verified working; Neon DB migrations applied; `config/models.json` valid.

**Exit criteria:**
- [ ] User can sign up with Apple Sign-In and land in ChatScreen
- [ ] Streaming text appears character-by-character
- [ ] Typing indicator shows/hides correctly
- [ ] Message persists after page refresh
- [ ] Second device sees same message history
- [ ] `web_search` tool responds to "What's the weather?" without any integrations
- [ ] Snapshot written to `session_snapshots` after each turn
- [ ] Service worker caches last 100 messages (read-only on reconnect)

#### Frontend Affordances

1. **PWA Shell**
   - `manifest.json` for Add to Home Screen.
   - Viewport meta: `viewport-fit=cover`, `apple-mobile-web-app-capable=yes`.
   - Minimal styling (Tailwind utility classes only; no custom CSS tokens yet).

2. **AuthScreen**
   - Clerk `<SignIn />` and `<SignUp />` components (Apple + email).
   - No custom auth screens. Pure Clerk SDK defaults.

3. **ChatScreen**
   - iMessage-style bubbles (raw HTML/CSS; no animations yet).
   - Text input at bottom.
   - Streaming text appears character-by-character via `text_delta`.
   - Typing indicator (three dots) shown on `typing:start`, hidden on `typing:stop`.
   - Messages fetched from `/api/conversations` on mount (paginated, last 50).
   - Visibility polling: on `visibilitychange` to `visible`, refetch `/api/conversations`.

#### Backend Implementation

1. **Fastify Boot Sequence**
   - Register `@clerk/fastify` plugin.
   - Mount Socket.io server on same HTTP port (`transports: ['websocket', 'polling']`).
   - Connect to Neon + Upstash.

2. **HTTP Endpoints**
   - `POST /api/webhook/clerk` — Clerk webhook handler (see Section 2.12). On `user.created`, insert BOOTSTRAP.md system message and create Mem0 user.
   - `GET /api/conversations` — Paginated message history (cursor-based). Returns `{ messages[], nextCursor? }`.
   - `GET /api/user/profile` — Returns `name`, `timezone`, `personality_config` from `users` + `user_agents`.
   - `PUT /api/user/profile` — Updates `name`, `timezone`.

3. **Agent Runtime Boot**
   - Single `AgentRuntime` class instantiated on process start.
   - `createAgentSession()` from pi-agent-core with:
     - `systemPrompt`: Layer 1 (base identity, casual) + Layer 5 (session context).
     - `model`: via `getActiveModel()` from `config/models.json`.
     - `tools`: `buildBuiltinTools()` — returns `[web_search, read_file]` (see Section 2.14).

4. **WebSocket Flow**
   - Client emits `message` → API Gateway calls `agentRuntime.getOrCreateSession(userId).prompt(content)` → pi-agent-core streams `text_delta` events → mapped to Socket.io `text_delta` → client.
   - On `message_end`: write snapshot to `session_snapshots`. Insert assistant message to `messages`.
   - **Steering:** On `steer` event from client, call `agentSession.steer(content)`. Steering takes effect after current tool batch.
   - **Visibility polling:** No cross-device relay. Client polls `/api/conversations` on `visibilitychange` (see Section 2.17).

5. **Offline Queue**
   - On message receive, if Redis `session:active:{userId}` missing: enqueue to `session:offline_queue:{userId}` (see Section 2.16).
   - On session restore: drain queue FIFO before processing new message.

#### Database for This Slice

- `users` (created by Clerk webhook handler).
- `user_agents` (default row inserted on user creation).
- `messages` (append user and assistant turns).
- `session_snapshots` (overwrite after each turn).

#### What's Real
- Sign up/in via Clerk.
- Streaming chat via Socket.io.
- Message persistence to PostgreSQL.
- Personality base identity injected every turn.
- Session snapshot on turn completion.

#### What's Stubbed
- **Zero tools.** Dotty cannot check email or calendar. If asked, she says "I'd love to help, but I don't have access to your accounts yet!"
- **No memory injection.** System prompt has only base identity + session context.
- **No offline caching.** Service worker is a no-op passthrough.
- **No onboarding flow beyond auth.** User lands straight in chat after sign in.

#### Integration Points
- **Clerk** (JWT validation + user sync).
- **OpenAI** (GPT-4o completions).
- **Neon** (messages, snapshots).
- **Upstash** (session active TTL).

#### Demo Script
1. Open PWA in Safari.
2. Sign up with Apple Sign-In.
3. Land on ChatScreen (empty state says "Hi, I'm Dotty! Ask me anything.").
4. Type "What can you do?"
5. See streaming response: "Not much yet, but I'm learning fast! 😄"
6. Refresh page. See previous message in history.
7. Open second device (desktop). Sign in. See same history.
8. Note: no real-time sync between devices (secondary device polls on visibility change).

---

### Slice 2: One Gmail (Week 2)

**The one piece to prove:** A user can connect a Gmail account via Composio, then ask Dotty to read emails and send one.

**Why this slice now:**
- **Core:** Gmail is the primary job-to-be-done.
- **Novel:** Composio tool registration + OAuth flow is the next biggest technical risk after streaming.
- **Small:** One provider. One account. No multi-account logic yet.

**Entry criteria:** Slice 1 chat pipe verified working (streaming, snapshot, history, visibility polling all confirmed).

**Exit criteria:**
- [ ] "Connect Gmail" button opens Composio Connect Link in popup
- [ ] OAuth completes; account appears in IntegrationManagementScreen
- [ ] "What's new in my email?" returns real emails from the connected account
- [ ] `tool_start` / `tool_end` events render as inline tool indicators in chat bubbles
- [ ] Email can be sent via Dotty ("Send an email to myself saying hello")
- [ ] Composio webhook fires for new emails; if user is online, `push` event appears in chat
- [ ] `oauth_token_expired` surfaces a re-link prompt inline in chat
- [ ] E2E test passes: staging Gmail sandbox account sends a real email

**Test account strategy:** Staging uses Gmail sandbox (`user+testN@gmail.com`) — all addresses require `@test` suffix in Gmail sandbox mode. Real Gmail accounts only used in manual testing.

#### Frontend Affordances

1. **IntegrationConnectScreen** (raw affordance)
   - Button: "Connect Gmail".
   - Opens Composio Connect Link in popup/SafariVC.
   - Polls backend for completion (or receives `integration_connected` push event).
   - Shows connected alias + email address.

2. **ChatScreen Enhancements**
   - Tool execution indicators: when Dotty uses `gmail_search` or `gmail_send`, show a small inline spinner with tool name.
   - `tool_start` and `tool_end` events rendered as sub-bubbles (gray, minimal).

3. **SendConfirmationCard** (preparation for later disambiguation)
   - For this slice, with only one account, no disambiguation needed. But render the confirmation inline when `account_prompt` fires (even if only one option, auto-confirm in backend for this slice, or let user tap once to confirm).
   - Actually: skip confirmation for single account in this slice. The card exists as a placeholder div but is auto-resolved. We build the actual UI interaction in Slice 3.

#### Backend Implementation

1. **Composio Integration Layer**
   - On user creation (Clerk webhook `user.created`): initialize Mem0 user (see Slice 5 for full Mem0 wiring).
   - On first integration connection: `POST /api/integrations/gmail/auth` returns Composio Connect Link URL (opens in popup/SafariVC).
   - OAuth callback: `POST /api/integrations/gmail/confirm` receives `connected_account_id`, stores in `integrations`, sets status `active`.

2. **Composio Session Initialization**
   When Composio session is first created for a user:
   ```typescript
   const client = new Composio({ apiKey: process.env.COMPOSIO_API_KEY });
   const session = await client.create(userId, {
     toolkits: ['gmail', 'websearch'],  // gmail for email; websearch for builtin web_search
     multiAccount: { enable: true, maxAccountsPerToolkit: 5 },
   });
   // Cache session ID in Redis: composio:session:{userId} → sessionId (1hr TTL)
   ```
   The `websearch` toolkit is included from the start (not added later) so that `web_search` tool works before any integrations are connected.

3. **Tool Registration (Single Account)**
   When user connects Gmail, `AgentRuntime` rebuilds tools for that user:
   - `gmail_search` — wraps `GMAIL_SEARCH_EMAILS`
   - `gmail_send` — wraps `GMAIL_SEND_EMAIL`
   - `gmail_list` — wraps `GMAIL_LIST_EMAILS`

3. **Webhook Receiver**
   - `POST /api/triggers/email` — receives Composio webhook for new emails.
   - Verify signature.
   - Lookup `integration_id` from `connected_account_id`.
   - If user is online (Redis `ws:sid:{user_id}` exists), emit `push` event.
   - If digest mode, push to `trigger_queue:{integration_id}`.

#### Database for This Slice

- `integrations` (first row per user: provider='gmail').
- `integration_preferences` (empty for now; schema exists).

#### What's Real
- OAuth Connect Link for Gmail.
- Single-account Gmail search, list, and send tools.
- Tool execution streaming (`tool_start`/`tool_end`).
- Composio webhook receiver for new emails (push to online users).
- Error handling for `oauth_token_expired`.

#### What's Stubbed
- **Multi-account:** Only one Gmail allowed per user in this slice. Code has a `// TODO(Slice 3)` for tool arrays.
- **Disambiguation:** `account_prompt` event defined but not emitted (backend auto-selects the only account).
- **Confirmation card:** Hardcoded auto-confirm.
- **Triggers:** Only `immediate` mode. Digest and smart rules deferred.

#### Integration Points
- **Composio** (OAuth + native tool execution).

#### Demo Script
1. On ChatScreen, tap "Connect Gmail".
2. OAuth popup with Google. Approve.
3. Return to app. See "Gmail connected ✓".
4. Ask "What's new in my email?"
5. See `tool_start: gmail_search` → spinner.
6. See `tool_end` → Dotty summarizes 3 latest emails.
7. Ask "Send an email to myself saying hello from Dotty."
8. See email sent. Inbox shows new message.

---

### Slice 3: Multi-Account (Week 3)

**The one piece to prove:** A user connects Work Gmail + Personal Gmail. Dotty merges search results, asks which account to send from, and remembers the choice.

**Why this slice now:**
- **Core:** Multi-account is the PRD's marquee feature. This is the "visibility toggle" of the project.
- **Novel:** Dynamic tool name generation, `account_prompt` interception, per-account routing.
- **Small:** Expands Slice 2's tool array; adds one UI affordance (inline buttons).

**Entry criteria:** Slice 2 Gmail integration verified working (real email sent and received via Composio).

**Exit criteria:**
- [ ] Second Gmail account can be added via "Add Account" button
- [ ] `gmail_search_all` merges results from both accounts, tagged with alias/color
- [ ] "Email the CEO" → `account_prompt` fires → inline pill buttons appear in chat bubble
- [ ] Tapping "Work Gmail" resolves the send and email arrives from the correct address
- [ ] After re-linking expired token, tool resumes correctly
- [ ] Disambiguation UI: inline pill buttons inside the chat bubble (no modal, no separate card)
- [ ] Learning from corrections: "No, use Personal" → stored as Mem0 preference; verified in next similar request

**Disambiguation UI:** Pill-shaped buttons rendered inline inside the chat bubble, beneath the assistant's text, horizontally arranged: `[Work Gmail] [Personal Gmail]`. Buttons use the account's assigned color. Tap emits `resolve_account` with the alias.

#### Frontend Affordances

1. **IntegrationManagementScreen** (raw affordance)
   - List of connected accounts with alias, email address, status.
   - "Add Account" button (opens Connect Link for same provider).
   - Inline editable alias (text field, save on blur).
   - Toggle for `is_default`.
   - Color badge per account (dropdown of 5 colors).
   - Delete button per account.

2. **ChatScreen Enhancements**
   - **Inline account buttons:** When `account_prompt` fires, render two (or more) pill-shaped buttons inside the chat bubble: `[Work Gmail] [Personal Gmail]`.
   - Tapping a button emits `resolve_account` with the chosen alias.
   - Search results tagged with color badge + alias (e.g., small blue dot + "Work").

#### Backend Implementation

1. **Dynamic Tool Generation**
   `buildMultiAccountTools()` generates:
   - `gmail_search_all` — Parallel `GMAIL_SEARCH_EMAILS` across all active Gmail accounts. Merge + tag with alias.
   - `gmail_send_work_gmail`, `gmail_send_personal_gmail` — Per-account send tools.
   - `gmail_list_work_gmail`, `gmail_list_personal_gmail` — Per-account list.
   Tool naming: `gmail_send_${alias.toLowerCase().replace(/\s+/g, '_').replace(/[^a-z0-9_]/g, '')}`.

2. **Account Prompt Interception**
   - When LLM calls a `gmail_send_*` tool:
     a. If multiple Gmail accounts exist and no `from:` context, API Gateway intercepts the tool call.
     b. Returns `account_prompt` event to client.
     c. Suspends turn execution (pi-agent-core supports pending tool states).
     d. Client resolves via `resolve_account`.
     e. Backend maps alias → `connected_account_id`, executes tool, resumes turn.
   - If only one account: auto-execute (backward compatible with Slice 2).

3. **Learning from Corrections**
   - If user corrects account choice (e.g., "No, use Personal"), store a Mem0 memory: `{"type": "preference", "content": "When sending to Mom, use Personal Gmail"}`.
   - Injected into system prompt next turn (Slice 4 wires Mem0; for this slice, store a local stub note in `integration_preferences` as `pref_type='default_for_context'` if needed, or just stub it).

#### Database for This Slice

- `integrations`: now expects 1–5 rows per user per provider.
- `integration_preferences`: add rows for per-account filters (stubbed; schema used later).

#### What's Real
- Multiple Gmail accounts per user.
- `gmail_search_all` merges results.
- `account_prompt` disambiguation flow end-to-end.
- Inline alias editing and default selection.

#### What's Stubbed
- **Calendar:** Not yet connected (`provider` column exists but no UI).
- **Learning deep inference:** We remember the explicit choice but don't yet mine implicit rules (defer to Slice 4/5).
- **Color coding:** Stored in DB but only used as visual badge; no logic tied to color.

#### Integration Points
- **Composio multi-account mode** (already verified; now using `maxAccountsPerToolkit: 5`).

#### Demo Script
1. Go to IntegrationManagementScreen.
2. Tap "Add Account" → connect second Gmail alias "Work".
3. List shows: "Personal Gmail" (default, green), "Work Gmail" (blue).
4. Ask "Check my email."
5. Results merged: "Work: Re: Q4 numbers..." (blue dot), "Personal: Mom's recipe" (green dot).
6. Ask "Email the CEO good luck."
7. Dotty sends `account_prompt`: "Which account? [Work Gmail] [Personal Gmail]"
8. Tap "Work Gmail".
9. Email sent from `sarah@acme.com`.
10. Ask again later: "Email the CEO..." → Dotty defaults to Work (because last choice + Mem0 preference in Slice 4).

---

### Slice 4: Calendar (Week 4)

**The one piece to prove:** Dotty sees Work Calendar + Personal Calendar. Scheduling checks ALL calendars before creating events.

**Entry criteria:** Slice 3 multi-account Gmail verified working (both accounts, disambiguation, learning from corrections all confirmed).

**Exit criteria:**
- [ ] "Connect Calendar" button appears in IntegrationManagementScreen
- [ ] Calendar accounts appear in the UI with alias + color badge
- [ ] "What's my day look like?" returns merged events from all calendars
- [ ] Conflict detection runs before any event creation
- [ ] "Schedule a meeting at 3pm" when 3pm is busy on Personal Calendar → warning bubble + "How about 4pm?" suggestion chip appears
- [ ] Confirming "4pm" creates event in correct calendar
- [ ] Tentative events treated as "busy" (hard block)

#### Frontend Affordances

1. **IntegrationManagementScreen**
   - "Connect Calendar" button alongside Gmail.
   - Calendar accounts listed under separate accordion section.
   - Same alias/default/color affordances.

2. **ChatScreen Enhancements**
   - Conflict warning rendered inline as a special bubble: "⚠️ You have a conflict! Work wants you at 3pm but Personal Calendar has 'dentist.'"
   - Suggested alternatives rendered as clickable chips.

#### Backend Implementation

1. **Calendar Tools**
   - `calendar_view_all` — parallel `GOOGLECALENDAR_FIND_FREE_SLOTS` across all calendars. Merge busy periods.
   - `calendar_create_work_calendar`, `calendar_create_personal_calendar` — per-account create.

2. **Conflict Detection**
   - Before `calendar_create_{alias}` executes, check all calendars for the requested slot.
   - If busy period overlaps on ANY calendar: surface warning + suggest next available slot (naive: next hour block).
   - "Busy" heuristic: respect `busy` status on all calendars as hard block. Tentative events treated as busy for MVP (simplest safest).

#### Database for This Slice

- `integrations`: rows with `provider = 'googlecalendar'`.
- No new tables.

#### What's Real
- Multi-account calendar view and create.
- Conflict detection across all connected calendars.
- Warning + suggestion UI.

#### What's Stubbed
- **Smart suggestions:** Suggests "next hour" not "next available 30-min window". Deferred.
- **Calendar priority rules:** All calendars equal. No "dentist is personal, ignore for work" logic yet.

#### Demo Script
1. Connect Work Calendar + Personal Calendar.
2. Ask "What's my day look like?"
3. Merged list: "Work: All-hands 10am", "Personal: Dentist 3pm".
4. Ask "Schedule a meeting with the team at 3pm tomorrow."
5. Dotty runs conflict detection.
6. Returns: "You have a conflict with 'Dentist' on your Personal Calendar at 3pm. How about 4pm?"
7. Tap "4pm sounds good."
8. Event created in Work Calendar at 4pm.

---

### Slice 5: Memory (Week 5)

**The one piece to prove:** Dotty remembers facts across sessions and displays them in a screen.

**Entry criteria:** Slice 4 Calendar verified working. Mem0 account created on user signup in Slice 1 (stubbed), now wired up end-to-end.

**Exit criteria:**
- [ ] After saying "I hate morning meetings," the fact appears in MemoryScreen under Preferences
- [ ] "Schedule a meeting" → Dotty references the preference: "I know you're not a morning person — how about 2pm?"
- [ ] MemoryScreen sections (Facts, Preferences, Events, Relationships) all display correctly
- [ ] Swipe-to-delete removes a memory; Dotty no longer references it in subsequent turns
- [ ] Pull-to-refresh on MemoryScreen fetches latest Mem0 data
- [ ] ✨ icon appears next to Dotty messages that reference memories

#### Frontend Affordances

1. **MemoryScreen** (raw list)
   - Sectioned list: Facts, Preferences, Events, Relationships.
   - Each row shows content + source note (e.g., "Remembered from chat on May 10").
   - Swipe-to-delete (or tap trash icon).
   - Pull-to-refresh fetches latest from Mem0.

2. **ChatScreen Enhancements**
   - If Dotty references a memory, prefix with a subtle sparkle icon (✨).

#### Backend Implementation

1. **Mem0 Integration**
   - On user creation: `mem0.users.create({ user_id: userId })` → store `mem0_id` in `users`.
   - After every completed assistant turn: extract facts via Mem0's `add()` API (which runs its own extraction LLM, or we pre-extract with a cheap call and send to Mem0).
   - Before every turn: `mem0.search({ user_id, query: userMessage, limit: 10 })` → inject results into LAYER 2 of system prompt.

2. **System Prompt Layer 2 (dynamic)**
   ```
   Facts about {user_name}: {mem0_search_results}
   ```

3. **Memory CRUD**
   - `GET /api/memories` → proxies `mem0.get_all(user_id)`.
   - `DELETE /api/memories/{mem0_memory_id}` → proxies `mem0.delete()`.
   - No local `memories` table; metadata stored in Mem0.

#### Database for This Slice

- `users.mem0_id` added (nullable until first Mem0 call).
- No `memories` table. We rely entirely on Mem0.

#### What's Real
- Automatic extraction after turns (delegated to Mem0).
- Semantic retrieval injected into prompt.
- MemoryScreen displaying user-visible memories.
- Delete memory.

#### What's Stubbed
- **Memory decay:** Mem0 may support TTL; if not, we accept permanent memories for MVP. No custom decay job.
- **Source tracking:** Mem0 metadata stores `source_message_id` if we pass it, but display in UI is basic. No integration source tracking (e.g., "from your work calendar" deferred).
- **Memory editing:** Only delete, not edit. Add via telling Dotty directly.

#### Integration Points
- **Mem0 API** (`add`, `get_all`, `search`, `delete`).

#### Demo Script
1. Chat: "I hate morning meetings."
2. Later: "Schedule a meeting with the team."
3. Dotty: "How about 2pm? I know you're not a morning person. 😄"
4. Open MemoryScreen. See: "Preference: User hates morning meetings."
5. Delete it. Return to chat. Ask again. Dotty forgets (or falls back to generic suggestions).

---

### Slice 6: Morning Briefing & Notifications (Week 6)

**The one piece to prove:** Dotty delivers a daily summary at the user's chosen time. Urgent emails trigger immediate push. Non-urgent are digested.

**Entry criteria:** Slice 5 Memory verified working (Mem0 search and add confirmed). At least one Gmail integration connected.

**Exit criteria:**
- [ ] Briefing push delivered at configured `notify_digest_hour` (or when cron fires if testing manually)
- [ ] Briefing includes today's calendar events + priority emails (2–4 sentences + bullet list)
- [ ] Immediate push fires for new email webhook (if `notify_mode = immediate`)
- [ ] Quiet hours (10pm–7am) suppresses non-urgent pushes; boss-domain + "urgent"/"asap" in subject bypasses quiet hours
- [ ] Digest mode queues notifications; delivers at `notify_digest_hour`
- [ ] Opt-in cron fallback fires for integrations with `cron_fallback_enabled = true`
- [ ] `seen_email` deduplication prevents double-notification from webhook + cron overlap

#### Frontend Affordances

1. **SettingsScreen**
   - Time picker for `notify_digest_hour` (default 9am).
   - Per-account `notify_mode` selector (immediate / digest / off).
   - Quiet hours toggle (global, 10pm–7am).

2. **ChatScreen Enhancements**
   - `push` events rendered as system-style cards (not chat bubbles): "📧 New email on Work Gmail: 'Q4 numbers' from CEO — might want to look at this first. 🤔"

#### Backend Implementation

1. **Cron Job (inside same process)**
   - `node-cron` runs every 15 minutes.
   - Queries: `SELECT user_id, notify_digest_hour FROM users WHERE notify_digest_hour BETWEEN $current_hour AND $current_hour + 0.25`.
   - Enqueues briefing jobs to Redis sorted set `briefing_queue`.
   - Agent Runtime dequeues and executes as a system-initiated turn.

2. **Briefing Composition**
   - Fetch calendar events for today (all accounts).
   - Fetch unread priority emails (last 24h, from known important senders/heuristics).
   - `AgentSession.prompt()` with special briefing system prompt.
   - Deliver via `push` event to `ws:sid:{user_id}` if online; otherwise stored for pickup.

3. **Smart Immediate Rules**
   - If `notify_mode = 'immediate'`: webhook fires → `push` event.
   - If email is from `boss` domain + subject contains "urgent"/"asap": push even during quiet hours.
   - If from mailing list/newsletter or label "Promotions": downgrade to digest regardless of mode.

4. **Cron Fallback (opt-in only)**
   - Every 5 minutes, query `integrations WHERE cron_fallback_enabled = true AND status = 'active'`.
   - Poll Gmail for new emails since last check.
   - Deduplicate using `Message-ID` → Redis `seen_email:{message_id}` (24h TTL).

#### Database for This Slice

- `integrations.notify_mode`, `notify_digest_hour`, `cron_fallback_enabled` (already in schema; now used).

#### What's Real
- Daily briefing at configured time.
- Immediate push for webhooks (if user online).
- Quiet hours respecting (global 10pm–7am).
- Cron fallback for opt-in users.

#### What's Stubbed
- **Push reliability on iOS Safari:** Degraded. We accept Socket.io `push` only works while app is foreground. No background Web Push API (known PWA limitation). No SMS fallback.
- **APNS:** Deferred to V1.1 native app.
- **Smart sender detection:** "Boss" heuristic is hardcoded domain list, not mined from email history.

#### Demo Script
1. Set digest for 9am in Settings.
2. Wait (or force cron). Receive push: "Good morning! You have 3 meetings and 2 unread priority emails."
3. Ask "What's the email about?" → Dotty summarizes.
4. Receive urgent email from CEO at 11pm.
5. Push immediately arrives (if app open): "CEO emailed: 'URGENT: All-hands moved to 8am.'"

---

### Slice 7: Attachments & Artifacts (Week 7)

**The one piece to prove:** Users upload images/PDFs. Dotty reads them. Dotty returns rich artifacts (tables, email drafts) rendered inline.

**Entry criteria:** Slice 5 Memory verified (Mem0 working). R2 bucket configured and accessible.

**Exit criteria:**
- [ ] File picker opens for images/PDF (JPEG, PNG, GIF, WebP, HEIC, PDF accepted)
- [ ] Upload to R2 at `{user_id}/{conversation_id}/{uploadId}.{ext}` succeeds; progress bar shows
- [ ] Thumbnail preview appears in message composer before send
- [ ] "Summarize this PDF" → `read_file` fetches content → Dotty summarizes
- [ ] `---TYPE:comparison_table---` renders as a multi-column HTML table in the chat bubble
- [ ] `---TYPE:email_draft---` renders as a formatted card with to/subject/body fields
- [ ] `---TYPE:timeline---` renders as a vertical list with dates
- [ ] 25MB max enforced; >25MB shows `invalid_attachment` error
- [ ] Max 5 attachments per message enforced; >5 shows `attachment_too_large` error

#### Frontend Affordances

1. **ChatScreen**
   - Attachment button opens native file picker (`<input type="file" accept="image/*,application/pdf">`).
   - Thumbnail preview of selected files (raw grid, no carousel).
   - Upload progress indicator (raw progress bar).

2. **AttachmentViewer**
   - Full-screen overlay for images/PDF. Swipe to dismiss. Minimal styling.

3. **Artifact Rendering**
   - `---TYPE:email_draft---` detected in `text_delta` → render in a bordered card with To/Subject/Body fields.
   - `---TYPE:schedule_table---` → render as HTML `<table>` inside bubble.
   - `---TYPE:timeline---` → render as vertical list of date blocks.
   - `---TYPE:comparison_table---` → render as multi-column table with headers.

   **Adding new artifact types:** To add a new artifact type, update the frontend parser to detect the new `---TYPE:new_type---` delimiter and define a render function for it. The backend agent is instructed to use `---TYPE:...---` delimiters for any structured output; adding a new type requires no backend change — only frontend component registration.

   **Future (V1.1):** Replace `---TYPE:...---` delimiters with structured JSON artifacts `{ type: "email_draft", payload: {...} }` emitted via a dedicated `artifact` Socket.io event. The delimiter approach is MVP-only.

#### Backend Implementation

1. **Upload Pipeline**
   - `POST /api/upload` (multipart/form-data).
   - Validate: size ≤25MB, format ∈ {JPEG, PNG, GIF, WebP, HEIC, PDF}.
   - Stream to R2 at `{user_id}/{conversation_id}/{upload_id}.{ext}`.
   - Return `{ uploadId, url, mimeType, sizeBytes, filename }`.
   - Max 5 attachments per message.

2. **Read Tool**
   - `read_file(uploadId)` downloads blob from R2 via signed URL, returns content (or base64 for images).

3. **Artifact Pipeline**
   - Agent instructed to wrap structured output in `---TYPE:...---` delimiters.
   - No structured JSON output yet (MVP uses markdown hints). Native V1.1 can use richer schemas.

#### Database for This Slice

- `messages.attachments` JSONB column stores metadata.

#### What's Real
- Upload to R2.
- Attachment referenced in chat.
- Agent reads images/PDFs via `read_file`.
- Artifact rendering for tables, drafts, timelines.

#### What's Stubbed
- **HEIC conversion:** R2 stores raw HEIC. Browser may not render. Accept for MVP (user can download).
- **Image analysis quality:** Depends on LLM vision capability. We pass image URL and hope provider supports it.
- **Rich HTML artifacts:** Only markdown-like tables and simple cards. No interactive elements.

#### Demo Script
1. Tap attachment, select a PDF invoice.
2. Upload completes. Thumbnail appears in composer.
3. Send: "Summarize this invoice."
4. Dotty uses `read_file` → returns text.
5. Dotty replies with `---TYPE:comparison_table---`.
6. Chat renders a neat table: Vendor | Amount | Due Date.
7. Ask "Draft an email to the vendor confirming receipt."
8. Dotty returns `---TYPE:email_draft---` card. Tap to expand. Looks like a real email preview.

---

### Slice 8: Onboarding & Polish (Week 8)

**The one piece to prove:** A brand-new user is guided through setup and lands in a functional, credible app. All edge cases and error surfaces handled.

**Entry criteria:** Slices 1–7 all verified working. Rate limiting implemented. Error catalog complete.

**Exit criteria:**
- [ ] New user sees SplashScreen → NameCapture → IntegrationConnectScreen → ChatScreen (complete flow)
- [ ] BOOTSTRAP.md fires on first session; Dotty introduces herself with user name + connected accounts
- [ ] Personality slider updates system prompt in real-time (casual vs professional)
- [ ] `config/models.json` provider swap works (change defaultProvider → restart → verify new model used)
- [ ] Rate limit hit at 101 msg/min → `rate_limit_exceeded` error surfaces with retry affordance
- [ ] `auth_expired` → redirect to Clerk login flow
- [ ] `oauth_token_expired` → inline re-link button appears in chat (not a modal)
- [ ] Service worker caches last 100 messages (read-only on offline)
- [ ] E2E test suite passes: signup → connect Gmail → send email → memory persists → briefing fires

#### Frontend Affordances

1. **Onboarding Flow (integrations-first)**
   - SplashScreen: "Meet Dotty" + sign in.
   - NameCapture: "What should Dotty call you?" → writes `users.name`.
   - IntegrationConnectScreen: "Connect Gmail" (primary CTA), "Connect Calendar" (secondary).
   - DefaultAccountSelection: If multiple accounts connected, pick default. Sets `is_default`.
   - InductionMessage: Dotty sends first chat: "Hey {name}! I've got access to your {aliases}. What do you want to do first?"
   - **Skip logic:** User can skip integrations and land in chat (stubbed behavior from Slice 1 kicks in).

2. **SettingsScreen**
   - Name, timezone.
   - Personality slider (professional ↔ casual) → updates `user_agents.personality_config`.
   - Model preference dropdown (reads `config/models.json`).

3. **PWA Polish**
   - Service worker caches static assets + last 100 messages (read-only cache; no offline send queue).
   - `manifest.json` with icons.
   - Fullscreen mode on iOS.

#### Backend Implementation

1. **Onboarding API**
   - `GET /api/onboarding/status` → returns `{ hasName, hasIntegration, defaultSet }`.
   - Guides frontend through steps.

2. **BOOTSTRAP.md**
   - On first session creation (no snapshot), inject a system message built from BOOTSTRAP.md template:
     ```
     User name: {name}
     Connected accounts: {aliases}
     Personality: {tone}
     Introduce yourself naturally.
     ```

3. **Rate Limiting**
   - Redis token bucket: 100 msg/min, 1000 tool calls/min.
   - Implemented as Fastify preHandler hook.
   - Returns `rate_limit_exceeded` with `retryAfter`.

4. **Error Surface**
   - Every error code in the catalog has a user-facing message in Dotty's voice (cheeky where safe, neutral where trust-critical).
   - `error` event always includes `code` so frontend can show inline recovery affordances (re-link button, retry button).

5. **Conversation History**
   - `/api/conversations` paginated with cursor (not offset, for performance).

#### Database for This Slice

- No schema changes. Uses all existing tables.

#### What's Real
- Full onboarding flow.
- Settings persistence.
- Service worker caching (last 100 messages).
- Rate limiting.
- Complete error catalog handling.
- Paginated history.

#### What's Stubbed
- **Offline send:** Still fails immediately. Service worker only caches for display.
- **Push opt-in UI:** Prompt shown but Safari limitations accepted.
- **Visual polish:** Tailwind defaults used; no custom illustration or animation library.

#### Demo Script
1. Install PWA, open fresh.
2. See SplashScreen. Sign up with Apple.
3. "What should Dotty call you?" → type "Alex".
4. Connect "Work Gmail" then "Personal Gmail".
5. Pick "Work Gmail" as default.
6. Land in Chat. Dotty says: "Hey Alex! I've got access to your Work Gmail and Personal Gmail. What do you want to do first?"
7. Go to Settings, drag personality slider to "Professional".
8. Send "Hi" → Dotty responds formally.
9. Drag to "Casual" → Dotty responds cheekily.
10. Turn on airplane mode. Open app. Last 100 messages visible. Try to send → "Send failed. Check connection."
11. Turn off airplane mode. Pull-to-refresh → history synced. Send again → works.

---

## 8. Cross-Cutting Concerns

### 8.1 Observability

**Structured Logs (stdout → Fly.io Logtail / Cloudflare Logs):**
Every log line is JSON:
```json
{
  "ts": "2026-05-10T09:23:00Z",
  "level": "INFO",
  "service": "agent-runtime",
  "userId": "usr_xxx",
  "traceId": "trc_xxx",
  "msg": "Turn completed",
  "duration_ms": 4200,
  "model": "gpt-4o",
  "tokens": 2450
}
```

**Metrics (Prometheus format, `/metrics` endpoint):**
```
dotty_turns_total{userId="..."}
dotty_turn_duration_seconds_bucket{le="5.0"}
dotty_tool_duration_seconds{toolName="gmail_search_all"}
dotty_active_sessions
dotty_tokens_used{model="gpt-4o"}
dotty_cost_cents_total
dotty_tool_errors_total{toolName="...", errorCode="..."}
```

**SLOs:**
| SLO | Target | Alert Threshold |
|-----|--------|-----------------|
| Turn latency p95 | < 10s | > 15s for 5min |
| Tool success rate | > 95% | < 90% for 5min |
| Session restore | > 99% | < 95% for 5min |
| Snapshot write | > 99.5% | < 99% for 5min |

### 8.2 Session Management

- **Resident memory:** `Map<string, AgentSession>` in the Agent Runtime process.
- **Eviction:** Redis `session:active:{userId}` TTL = 5 min. On TTL expiry, snapshot is written to PostgreSQL (if not already written after last turn).
- **Recovery:** On new message, if not in memory → load latest snapshot from `session_snapshots` → rebuild session.
- **Race condition:** Pod crash mid-turn → in-progress turn lost. Last completed turn recoverable. New pod restores from snapshot.
- **Offline queue:** Redis `session:offline_queue:{userId}` stores messages sent while runtime was down. Replayed FIFO on restore.

### 8.3 Scaling Path

| Users | Change |
|-------|--------|
| 0–1,000 | One box (this TRD). |
| 1,000–10,000 | Separate API Gateway (stateless) from Agent Runtime (stateful). Redis pub/sub routing. Neon read replicas. |
| 10,000+ | Multi-region Fly.io. Dedicated trigger worker machine. Evaluate Mem0 cost vs. custom pgvector return-on-investment. |

### 8.4 Environment Variables

```bash
# Fly.io / Docker
NODE_ENV=production
PORT=3000

# Clerk
CLERK_PUBLISHABLE_KEY=pk_test_...
CLERK_SECRET_KEY=sk_test_...

# Database
DATABASE_URL=postgresql://... neon.tech/...

# Redis
REDIS_URL=rediss://... upstash.io/...

# Composio
COMPOSIO_API_KEY=cmp_...
COMPOSIO_WEBHOOK_SECRET=whsec_...

# LLM
OPENAI_API_KEY=sk-...
# Additional models via config/models.json

# Storage
R2_ACCOUNT_ID=...
R2_ACCESS_KEY_ID=...
R2_SECRET_ACCESS_KEY=...
R2_BUCKET=dotty-attachments

# Mem0
MEM0_API_KEY=mem0_...

# Observability
LOG_LEVEL=info
```

### 8.5 Testing Strategy per Slice

| Type | Scope | Tools |
|------|-------|-------|
| Unit | Prompt builders, tool name sanitizers, token budget calc | Vitest |
| Integration | Mocked Composio (deterministic responses), snapshot save/restore | Vitest + `nock` |
| E2E | Signup → connect Gmail → send email | Playwright + staging Gmail test accounts |

**Composio mock for integration tests:**
```ts
vi.mock('@composio/core', () => ({
  Composio: vi.fn().mockImplementation(() => ({
    create: vi.fn().mockResolvedValue({ sessionId: 'mock' }),
    execute: vi.fn().mockResolvedValue({ content: JSON.stringify({ emails: [] }) }),
  })),
}));
```

---

## Appendix A: Shape Up Checklist for Slicing

- [ ] **Slice 1 is demoable in ~3 days.** (Chat works.)
- [ ] **Each slice has real backend + real frontend.** (No pure-design or pure-API slices.)
- [ ] **Earlier slices do not depend on later slices.** (Calendar does not block attachments.)
- [ ] **Each slice leaves stubs, not holes.** (Slice 2 leaves a `// TODO(Slice 3)` in tool generation — it's a known stub, not a gaping hole.)
- [ ] **Vertical integration validated before horizontal polish.** (Chat pipe works before styling tokens exist.)

---

*End of TRD.*

<!-- BUILD-HTML-PROMPT
{
  "template": "minimal",
  "purpose": "technical",
  "mood": "clean",
  "format": "screen",
  "notes": "Implementation spec for a SaaS AI assistant. Contains architecture diagrams, SQL DDL, TypeScript code samples, tables, flow descriptions. Rendered as a clean technical reference document."
}
-->

# Build HTML from this document

```bash
build-html docs/requirements/TRD-Dotty.md
```

Or ask: `@docs/requirements/TRD-Dotty.md follow the html build instructions`

<!-- Generated by mdprint. Template: minimal. Purpose: technical. Mood: clean. Format: screen. -->
