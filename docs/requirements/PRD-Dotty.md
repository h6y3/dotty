# Dotty — Product Requirements Document

**Version:** 0.2 — Agent-ready for TRD generation
**Date:** May 2026
**Classification:** Internal — pre-funding

**Purpose:** This document is the single source of truth for building Dotty's MVP. It contains architecture decisions, component inventories, blocking questions, and reference specifications.

---

## 1. The Job to Be Done

Build a SaaS-hosted personal AI assistant (name: Dotty) with a cheeky-best-friend personality. The **MVP frontend is a Progressive Web App (PWA)** that users access via mobile Safari. It integrates with **multiple Gmail and Google Calendar accounts per user** (this is MVP-critical, not a V2 feature), performs web search, handles images/attachments, and remembers everything about the user. **Native iOS and macOS apps target V1.1.** They reuse the exact same WebSocket API contract — nothing in the backend changes when we swap the frontend.

**Why this matters:** Our target first users are busy executives with complex lives — work Gmail + personal Gmail + multiple calendars. Email writing help and scheduling only work if the agent sees the whole picture.

---

## 2. Non-Negotiable Architecture Decisions (Already Made)

| Decision | Status | Rationale |
|---|---|---|
| **SaaS-hosted, not local-first** | Fixed | Users sign up on our site. Backend runs in our cloud. Device is dumb client. |
| **pi.dev (`@earendil-works/pi-agent-core`) as agent runtime** | Fixed | Proven by OpenClaw at scale. Event streaming, tool execution, steering. MIT licensed. |
| **Composio for tool integrations** | Fixed | Handles OAuth, token refresh, 1000+ apps. Native multi-account support via `multi_account` mode. |
| **Multi-account Gmail + Calendar in MVP** | Fixed | Work + personal + shared calendars. Unified search, conflict detection, smart defaults. |
| **Model-agnostic backend** | Fixed | Config-driven provider swap via `config/models.json`. OpenAI-compatible endpoints only. |
| **WebSocket for real-time chat** | Fixed | Required for streaming text deltas, typing indicators, steering. |
| **PostgreSQL + Redis for data** | Fixed | Neon for persistence, Upstash for session/active-user caching. |
| **Clerk for auth** | Fixed | Apple Sign-In + email. JWT session tokens. |
| **PWA as MVP frontend** | Fixed | React SPA with service worker. Fastest iteration. Native iOS/macOS target V1.1 using same API contract. |

---

## 3. Component Inventory

Each component below has its dependencies, scaling model, and known unknowns listed.

### 3.1 Web App (MVP Frontend)
**Tech:** React 19 + TypeScript, TailwindCSS, Vite. No Next.js needed (no SSR required — this is an app, not a content site).

**Why a PWA for MVP?**
- Fastest iteration cycle: reload browser, no Xcode builds, no TestFlight
- No Apple provisioning or App Store review gates
- One codebase for mobile Safari + Chrome desktop
- The WebSocket API contract is identical to what native apps will use later

**Trade-offs accepted:**
- iOS Safari PWA push notifications are unreliable (Apple restricts Web Push API). Email digests and in-app alerts will suffice for MVP.
- Camera access is via `<input type="file" capture>` — clunkier than native camera roll but functional.
- "Add to Home Screen" is required for fullscreen feel. Users who skip this get a Safari tab experience.

**Screens to build:**
1. **ChatScreen** — iMessage-style bubbles, typing indicator, attachment upload
2. **OnboardingScreen** — Sign up (via Clerk React SDK), personality setup, Composio Connect Link OAuth
3. **IntegrationManagementScreen** — CRUD for multiple accounts per provider, alias editing, default selection, color coding, per-account notification toggles
4. **MemoryScreen** — "What Dotty Knows About Me" — editable list of extracted facts
5. **SettingsScreen** — Name, timezone, model preference, personality slider (professional ↔ casual)
6. **AttachmentViewer** — Full-screen photo/PDF
7. **SendConfirmationCard** — Inline account selection before sending email (no modals)
   **Confirmation flow:** Frontend UI interception. When LLM calls `gmail_send_*`, the API Gateway intercepts the tool result and returns an `account_prompt` WebSocket message BEFORE sending. User selects account in the UI, which resolves the pending tool call with the chosen account. Agent sees the resolved account and completes the send. This keeps the agent stateless with respect to account selection — the frontend resolves the ambiguity, Dotty adapts to the choice. |

**PWA Requirements:**
- Service worker for offline caching of last 100 messages
- `manifest.json` for "Add to Home Screen" support
- Fullscreen mode on iOS: `viewport-fit=cover`, `apple-mobile-web-app-capable=yes`
- Push notification permission request (works reliably only on Android; on iOS, use it for digest alerts and accept that real-time push may miss)

**Dependencies:** Backend API, WebSocket endpoint, Clerk React SDK.

**Known unknowns:**
- [DECISION NEEDED] **Offline behavior:** Cache last N messages via service worker. What happens to messages sent while offline? Queue-and-retry or fail immediately?
- [DECISION NEEDED] **Attachment size limit:** 10MB? 25MB? Where stored? (S3/R2)
- [DECISION NEEDED] **Push notification strategy:** Per-account rules (immediate/digest/off). What triggers a push vs. what waits for the user to open the app?
- [DECISION NEEDED] **Push reliability on iOS Safari:** Accept degraded push or implement SMS fallback for critical alerts?

### 3.2 iOS App (V1.1)
**Status:** Not in MVP. Build once the agent experience is validated.

**Tech:** SwiftUI, MVVM + Combine, URLSession + WebSocket, SwiftData (local cache), Clerk iOS SDK.

**What changes from the PWA:**
- Same WebSocket API contract — zero backend changes
- APNS for reliable push notifications
- Native camera roll for attachments
- iMessage-style haptics and animations
- App Store distribution for credibility

**Known unknowns (deferred to V1.1):**
- [DECISION NEEDED] **Offline behavior:** Cache last N messages locally. What happens to messages sent while offline? Queue-and-retry or fail immediately?
- [DECISION NEEDED] **Attachment size limit:** 10MB? 25MB? Where stored? (S3/R2)
- [DECISION NEEDED] **Push notification strategy:** Per-account rules (immediate/digest/off). What triggers a push vs. what waits for the user to open the app?

### 3.3 macOS Companion App (V1.1)
**Status:** Not in MVP. Build alongside or shortly after iOS app.

**Tech:** Shared SwiftUI codebase with iOS. Menu bar mini-window + full window. `WKWebView` for rich HTML artifacts (tables, charts, reports).

**Features beyond web:**
- Drag-and-drop files into chat
- `Cmd+Shift+D` global shortcut
- Rich artifact rendering inline in conversation bubbles
- Native file picker for uploads
- NotificationCenter for push (when APNS is available)

**Dependencies:** Same WebSocket API as web app and iOS app. Same account model.

### 3.3 Backend — API Gateway
**Tech:** Node.js (22+), Fastify or Express. Stateless horizontal pods.

**Responsibilities:**
- HTTP REST API for CRUD operations
- WebSocket upgrade + auth (validate Clerk JWT on connection)
- Message routing to Agent Runtime
- File upload handling → S3/R2
- Composio webhook receiver (email triggers)

**Endpoints:**
| Endpoint | Method | Purpose |
|---|---|---|
| `/auth/signup` | POST | Clerk handles this |
| `/auth/login` | POST | Clerk handles this |
| `/api/user/profile` | GET/PUT | Get/update user profile |
| `/api/integrations` | GET | List connected integrations |
| `/api/integrations/{provider}/auth` | POST | Start OAuth flow |
| `/api/integrations/{provider}/confirm` | POST | Complete OAuth, store connected_account_id |
| `/api/integrations/{integration_id}` | DELETE | Disconnect + revoke |
| `/api/memories` | GET/POST/DELETE | CRUD memories |
| `/api/conversations` | GET | Paginated message history |
| `/api/upload` | POST | Attachment upload (returns S3 URL) |
| `/api/triggers/email` | POST | Composio webhook receiver |

**Known unknowns:**
- [DECISION NEEDED] **Rate limiting strategy:** Fixed window per user? Token bucket? What are the limits? (Impacts Redis schema)
- [DECISION NEEDED] **Webhook verification:** How do we verify Composio webhooks are authentic? (Signature validation, secret token?)
- [DECISION NEEDED] **API Gateway WebSocket:** Do we use a library (Socket.io, ws) or built-in? Socket.io adds presence rooms and fallback to HTTP long-polling — worth it for mobile reliability.

### 3.4 Backend — Agent Runtime (Single Process for MVP)
**Tech:** Node.js 22, `@earendil-works/pi-agent-core`, `@earendil-works/pi-ai`.

**Responsibilities:**
- Hold `AgentSession` objects in memory for active users
- Rebuild `AgentSession` from PostgreSQL snapshot on first message (or on pod restart)
- Snapshot after every completed turn
- Execute tools via Composio
- Stream events back via WebSocket

**Data flow per message:**
```
1. Receive message from API Gateway (via internal queue or direct call)
2. Load/restore AgentSession
3. Fetch user's active integrations from PostgreSQL
4. Rebuild tools (unified + per-account) based on connected accounts
5. Rebuild system prompt with: base identity + memories + account context
6. Run AgentSession.prompt(message)
7. LLM streams response → forward text deltas to WebSocket
8. Tool calls execute → forward tool_start/tool_end events
9. Turn completes → extract new memories → write snapshot to PostgreSQL
10. Close or keep session resident (idle eviction after N minutes)
```

**Internal IPC (API Gateway ↔ Agent Runtime):**
MVP: HTTP/1.1 POST from API Gateway to Agent Runtime pod (same process, localhost)
Post-MVP (>1000 users): Redis pub/sub queue — API Gateway publishes `{userId, message}`, Agent Runtime subscribes and processes

**Why HTTP for MVP:**
- Simplest possible stack for 0–1000 users
- Single deploy unit (one pod) means no network hop
- Upgrades to Redis pub/sub without architectural change — only the transport layer changes

**Post-MVP scaling path:**
```
API Gateway                    Agent Runtime Pods
[pod-1] ←→ Redis pub/sub ←→ [pod-2]
[pod-3]                        [pod-N]
  ↑                               ↑
  └── Redis pub/sub channel: "agent:queue:{userId}"
```
Each userId maps deterministically to a runtime pod (consistent hashing via Redis). No message loss — Redis persists queue items until consumed.

**Key implementation details (from pi.dev docs):**
- `Agent` class emits events: `agent_start`, `turn_start`, `message_start`, `message_update` (with `text_delta`), `tool_execution_start`, `tool_execution_end`, `message_end`, `turn_end`, `agent_end`
- `session.navigateTree()` for branching if we add it later
- `session.agent.state.systemPrompt` is mutable between turns
- Tools are registered per-session via `agent.state.tools = [...]`
- Parallel vs sequential tool execution configurable per-tool
- Steering: `session.steer("Stop!")` interrupts after current tool batch

**Tool architecture for multi-account:**
For a user with Work Gmail + Personal Gmail + Work Calendar + Personal Calendar, the runtime registers:

```
gmail_search_all          → searches ALL Gmail accounts, merges + tags results
gmail_send_work_gmail     → sends from Work Gmail
gmail_send_personal_gmail → sends from Personal Gmail
gmail_list_work_gmail     → lists emails from Work Gmail
gmail_list_personal_gmail → lists emails from Personal Gmail
calendar_view_all         → checks ALL calendars, merges events
calendar_create_work_cal  → creates event in Work Calendar
calendar_create_personal_cal → creates event in Personal Calendar
web_search                → web search (Tavily / SerpAPI)
read_file                 → read uploaded attachments
```

Per-account tool names are generated from alias: `gmail_send_${sanitize(alias)}` where `sanitize` lowercases and replaces spaces with underscores.

**Example multi-account runtime code:**
```typescript
class AgentRuntime {
  private sessions: Map<string, AgentSession> = new Map();

  async getOrCreateSession(userId: string): Promise<AgentSession> {
    if (this.sessions.has(userId)) return this.sessions.get(userId)!;

    const snapshot = await db.getLatestSnapshot(userId);
    const activeAccounts = await db.getUserIntegrations(userId);
    const session = await createAgentSession({
      initialState: {
        systemPrompt: buildSystemPrompt(userId, activeAccounts),
        model: getModelFromConfig(),
        messages: snapshot?.messages ?? [],
        tools: await this.buildMultiAccountTools(userId, activeAccounts),
      },
      sessionManager: SessionManager.inMemory(),
    });

    this.sessions.set(userId, session);
    return session;
  }

  private async buildMultiAccountTools(
    userId: string,
    accounts: Integration[]
  ): Promise<AgentTool[]> {
    const tools: AgentTool[] = [...builtinTools];
    const gmailAccounts = accounts.filter(a => a.provider === 'gmail' && a.status === 'active');
    const calAccounts = accounts.filter(a => a.provider === 'googlecalendar' && a.status === 'active');

    if (gmailAccounts.length > 0) {
      // Unified search tool
      tools.push(defineTool({
        name: 'gmail_search_all',
        label: 'Search All Gmail Accounts',
        description: `Search across all connected Gmail accounts. Use when user doesn't specify which account.`,
        parameters: Type.Object({
          query: Type.String(),
          maxResults: Type.Integer({ default: 10 }),
        }),
        execute: async (id, params) => {
          const results = await Promise.all(
            gmailAccounts.map(async (acct) => {
              const composioSession = await this.getComposioSession(userId, acct.connected_account_id);
              return composioSession.execute('GMAIL_SEARCH_EMAILS', { ...params, account: acct.alias });
            })
          );
          return { content: mergeAndTagResults(results, gmailAccounts) };
        },
      }));

      // Per-account send tools
      for (const acct of gmailAccounts) {
        const toolName = `gmail_send_${acct.alias.toLowerCase().replace(/\s+/g, '_').replace(/[^a-z0-9_]/g, '')}`;
        tools.push(defineTool({
          name: toolName,
          label: `Send Email from ${acct.alias}`,
          description: `Send an email from ${acct.email_address || acct.alias}.`,
          parameters: Type.Object({ to: Type.String(), subject: Type.String(), body: Type.String() }),
          execute: async (id, params) => {
            const composioSession = await this.getComposioSession(userId, acct.connected_account_id);
            return composioSession.execute('GMAIL_SEND_EMAIL', params);
          },
        }));
      }
    }

    if (calAccounts.length > 0) {
      tools.push(defineTool({
        name: 'calendar_view_all',
        label: 'View All Calendars',
        description: `Check events across all calendars.`,
        parameters: Type.Object({ start: Type.String({ format: 'date-time' }), end: Type.Optional(Type.String({ format: 'date-time' })) }),
        execute: async (id, params) => {
          const results = await Promise.all(
            calAccounts.map(async (acct) => {
              const composioSession = await this.getComposioSession(userId, acct.connected_account_id);
              return composioSession.execute('GOOGLECALENDAR_FIND_FREE_SLOTS', params);
            })
          );
          return { content: mergeCalendarResults(results, calAccounts) };
        },
      }));

      for (const acct of calAccounts) {
        const toolName = `calendar_create_${acct.alias.toLowerCase().replace(/\s+/g, '_').replace(/[^a-z0-9_]/g, '')}`;
        tools.push(defineTool({
          name: toolName,
          label: `Create Event in ${acct.alias}`,
          description: `Create a calendar event in ${acct.alias}.`,
          parameters: Type.Object({ title: Type.String(), start: Type.String({ format: 'date-time' }), end: Type.String({ format: 'date-time' }), attendees: Type.Optional(Type.Array(Type.String())) }),
          execute: async (id, params) => {
            const composioSession = await this.getComposioSession(userId, acct.connected_account_id);
            return composioSession.execute('GOOGLECALENDAR_CREATE_EVENT', params);
          },
        }));
      }
    }

    return tools;
  }
}
```

**Resolved:**
- Session eviction: 5 minutes idle → evict from memory (Redis key `session:active:{userId}` has 5-min TTL)
- Snapshot granularity: after every completed turn (simpler than batching for MVP)
- Tool timeout: 30 seconds — surfaces as `tool_timeout` error code (see Error Code Catalog)
- What happens when one account's token expires: tool fails with `oauth_token_expired` → Dotty informs user → re-link prompt shown inline

### 3.5 Backend — Composio Integration Layer
**Responsibilities:**
- Manage Composio sessions per user
- Handle OAuth Connect Links for new accounts
- Create per-account triggers for email webhooks
- Execute tools with correct `connected_account_id`

**Multi-account setup:**
```typescript
const composioSession = await composio.create(userId, {
  toolkits: ['gmail', 'googlecalendar'],
  multiAccount: { enable: true, maxAccountsPerToolkit: 5 },
});

const workAuth = await composioSession.authorize('gmail', { alias: 'Work Gmail' });
const personalAuth = await composioSession.authorize('gmail', { alias: 'Personal Gmail' });
```

**Composio MCP vs Native Tools decision:**
| Aspect | Native Tools (chosen for MVP) | MCP |
|---|---|---|
| Tool naming control | Full — we define exact names like `gmail_send_work_gmail` | Limited — Composio assigns names like `composio_gmail_send_email` |
| Description control | Full — we write precise descriptions for the LLM | Inherited from Composio; may be imprecise |
| Per-account routing | Explicit in our tool execute function | Requires MCP server config per account |
| Code complexity | More code (we wire each tool) | Less code (MCP handles tool bridge) |
| Debugging | Transparent — we see exact args/responses | Opaque — MCP serializes/deserialize |
| Composio version risk | We control failures | MCP failures are harder to trace |

**Resolution: Native tools for MVP.** The multi-account routing logic requires per-account `connected_account_id` injection that MCP's generic tool bindings don't expose cleanly. We can migrate to MCP post-MVP once Composio's MCP server supports per-account session routing.

**Trigger architecture:**
- One Composio trigger per `connected_account_id`
- Webhook payload includes `connected_account_id` → lookup user + alias + notify rules
- Cron fallback polls all `active` integrations every 5 minutes

**Webhook verification:**
- Composio signs payloads with `X-Composio-Signature` header (HMAC-SHA256)
- API Gateway verifies: `crypto.timingSafeEqual(computedSig, receivedSig)`
- Secret stored in environment variable `COMPOSIO_WEBHOOK_SECRET`
- If verification fails → return HTTP 401, log to monitoring

**Known unknowns:**
- [DECISION NEEDED] **Trigger reliability:** If Composio's Gmail trigger misses emails, our cron fallback is the safety net. Do we run cron for ALL users or only users who opt into it? (Cost/Composio quota impact)

### 3.6 Backend — Memory System
**Dual layer:**
1. **Short-term:** pi-agent-core's context window (conversation history + injected memories as text)
2. **Long-term:** PostgreSQL `memories` table (extracted facts)

**Fact extraction pipeline:**
- After every completed assistant turn, run a cheap LLM (Claude Haiku / GPT-4o-mini) to extract new facts
- Format: `{type: "fact"|"preference"|"event"|"relationship", content: string, source_integration_id?: UUID}`
- Insert into `memories` table
- Deduplicate by semantic similarity (or exact match for MVP)

**Memory retrieval — injected into system prompt every turn:**
- Keyword match against `memories.content` (MVP — no pgvector yet)
- Include top 10 most recent memories (sorted by `created_at DESC` for MVP)
- Also include recent conversation summary (last 20 messages)

**Memory embedding pipeline (for when pgvector is added):**
| Parameter | Value |
|---|---|
| Embedding model | `text-embedding-3-small` (OpenAI, 1536 dims) or `claude-embed-v3` |
| Chunk size | 512 tokens max per chunk |
| Overlap | 64 tokens between chunks |
| Upsert trigger | On every new memory insert |
| Delete strategy | Hard delete from both table and vector index |

**Memory types:**
| Type | Example |
|---|---|
| `fact` | "User is vegetarian" |
| `preference` | "User hates morning meetings" |
| `event` | "User's anniversary is June 15" |
| `relationship` | "User's sister is Kate" |

**Known unknowns:**
- [DECISION NEEDED] **Memory store: custom PostgreSQL vs. Mem0 SaaS:** Mem0 is $50+/month at scale but gives semantic search + automatic decay out of the box. Custom is cheaper but requires building vector search (pgvector) + deduplication + decay logic. What's the budget and timeline?
- [DECISION NEEDED] **Memory decay:** Do old memories fade? Does "User had a headache last Tuesday" get automatically forgotten? If yes, what's the TTL?
- [DECISION NEEDED] **Memory source tracking:** Should memories include `source_integration_id` so Dotty can say "I remember this from your work calendar"?

### 3.7 Database — PostgreSQL (Neon)

**Tables:**

| Table | Key Fields | Purpose |
|---|---|---|
| `users` | id, email, name, timezone | Auth identity |
| `user_agents` | user_id, agent_name, personality_config, model_provider, model_id | Per-user agent settings |
| `integrations` | user_id, provider, connected_account_id, alias, email_address, is_default, sort_order, status, notify_mode, notify_digest_hour | Multi-account tracking |
| `integration_preferences` | integration_id, pref_type, pref_value | Per-account filters |
| `session_snapshots` | user_id, messages (JSONB), tool_state, model_ref | Agent state recovery |
| `messages` | user_id, role, content, tool_calls, tool_results, attachments | Conversation history |
| `memories` | user_id, memory_type, content, confidence, source_message_id, source_integration_id | Extracted facts |
| `usage_events` | user_id, event_type, provider, model, tokens, cost_cents | Billing/observability |

**Known unknowns:**
- [DECISION NEEDED] **Row-level security:** Use PostgreSQL RLS or application-level filtering? RLS is safer but adds complexity. Application-level is simpler but riskier.
- [DECISION NEEDED] **pgvector for semantic memory:** If building custom memory, we need a `vector` column on `memories`. Do we add it now or migrate later?

### 3.8 Redis (Upstash)

| Key | TTL | Purpose |
|---|---|---|
| `session:active:{user_id}` | 5 min | Session is resident in runtime |
| `composio:session:{user_id}` | 1 hour | Cached Composio session ID |
| `ws:connection:{user_id}` | Session | Which WebSocket connection belongs to user |
| `rate_limit:{user_id}` | 1 min | Request rate limiting |
| `trigger_queue:{integration_id}` | 1 hour | Queued email notifications for digest mode |

**Known unknowns:**
- [DECISION NEEDED] **Redis cluster vs. single instance:** Upstash offers both. For MVP, single instance is fine. When do we need cluster? (Defines scaling threshold)

### 3.7.1 SQL DDL (PostgreSQL / Neon)

```sql
-- Enable UUID extension
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- Users (Clerk owns this table; we add our own fields)
CREATE TABLE users (
  id          UUID        PRIMARY KEY DEFAULT uuid_generate_v4(),
  clerk_id    TEXT        UNIQUE NOT NULL,           -- Clerk's user ID
  email       TEXT        UNIQUE NOT NULL,
  name        TEXT,
  timezone    TEXT        DEFAULT 'America/New_York',
  created_at  TIMESTAMPTZ DEFAULT NOW(),
  updated_at  TIMESTAMPTZ DEFAULT NOW()
);

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

-- Connected accounts (Gmail, Calendar, etc.)
CREATE TABLE integrations (
  id                    UUID  PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id               UUID  NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  provider              TEXT  NOT NULL,              -- 'gmail', 'googlecalendar'
  connected_account_id  TEXT  NOT NULL,              -- Composio's connected account ID
  alias                 TEXT  NOT NULL,               -- 'Work Gmail', 'Personal'
  email_address         TEXT,
  is_default            BOOLEAN DEFAULT FALSE,
  sort_order            INTEGER DEFAULT 0,
  status                TEXT  DEFAULT 'active',      -- 'active', 'expired', 'disconnected'
  notify_mode           TEXT  DEFAULT 'immediate',    -- 'immediate', 'digest', 'off'
  notify_digest_hour   INTEGER DEFAULT 9,
  created_at            TIMESTAMPTZ DEFAULT NOW(),
  updated_at            TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(user_id, provider, alias)
);
CREATE INDEX idx_integrations_user_id      ON integrations(user_id);
CREATE INDEX idx_integrations_provider    ON integrations(provider);
CREATE INDEX idx_integrations_status      ON integrations(status);

-- Per-account preference filters
CREATE TABLE integration_preferences (
  id              UUID  PRIMARY KEY DEFAULT uuid_generate_v4(),
  integration_id  UUID  NOT NULL REFERENCES integrations(id) ON DELETE CASCADE,
  pref_type       TEXT  NOT NULL,              -- 'filter_label', 'sync_range', 'exclude_domain'
  pref_value      TEXT  NOT NULL,
  UNIQUE(integration_id, pref_type)
);

-- Agent session snapshots (state recovery on restart)
CREATE TABLE session_snapshots (
  id          UUID  PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id     UUID  NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  messages    JSONB NOT NULL,                  -- Full message tree as JSON
  tool_state  JSONB,                            -- In-flight tool state (if any)
  model_ref   TEXT,                             -- Model used for this session
  created_at  TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(user_id)                               -- Only keep latest snapshot per user
);
CREATE INDEX idx_session_snapshots_user_id  ON session_snapshots(user_id);
CREATE INDEX idx_session_snapshots_created   ON session_snapshots(created_at DESC);

-- Conversation messages
CREATE TABLE messages (
  id           UUID  PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id      UUID  NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  role         TEXT  NOT NULL,               -- 'user', 'assistant', 'system'
  content      TEXT,
  tool_calls   JSONB,                        -- [{name, args, result}]
  tool_results JSONB,
  attachments  JSONB,                        -- [{id, url, mimeType, sizeBytes, filename}]
  model_used   TEXT,
  tokens_used  INTEGER,
  created_at   TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_messages_user_id     ON messages(user_id);
CREATE INDEX idx_messages_created_at  ON messages(created_at DESC);
CREATE INDEX idx_messages_user_recent  ON messages(user_id, created_at DESC) WHERE role = 'user';

-- Long-term memories (extracted facts)
CREATE TABLE memories (
  id                  UUID  PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id             UUID  NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  memory_type         TEXT  NOT NULL,      -- 'fact', 'preference', 'event', 'relationship'
  content             TEXT  NOT NULL,
  confidence          REAL  DEFAULT 1.0,   -- 0.0–1.0
  source_message_id   UUID REFERENCES messages(id) ON DELETE SET NULL,
  source_integration_id UUID REFERENCES integrations(id) ON DELETE SET NULL,
  created_at          TIMESTAMPTZ DEFAULT NOW(),
  updated_at          TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_memories_user_id    ON memories(user_id);
CREATE INDEX idx_memories_type       ON memories(memory_type);
CREATE INDEX idx_memories_confidence ON memories(confidence DESC);
-- Vector column added only when pgvector migration is triggered (see Known Unknowns)
-- ALTER TABLE memories ADD COLUMN embedding vector(1536);

-- Usage / billing events
CREATE TABLE usage_events (
  id           UUID  PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id      UUID  NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  event_type   TEXT  NOT NULL,   -- 'turn', 'tool_call', 'memory_extraction'
  provider     TEXT,
  model        TEXT,
  tokens       INTEGER,
  cost_cents   REAL,
  created_at   TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_usage_user_id    ON usage_events(user_id);
CREATE INDEX idx_usage_created_at ON usage_events(created_at DESC);

-- Row-Level Security (optional — see Known Unknowns)
-- For MVP, application-level filtering is used (all queries include WHERE user_id = :auth_user_id)
```

**Scaling notes:**
- `session_snapshots.messages` JSONB can grow large — compaction/resummarization before snapshot write keeps it bounded
- `memories` table: add `vector` column when pgvector migration is triggered (not in MVP)
- `messages` table: partition by month when > 1M rows (Neon supports native partitioning)

---

## 4. Personality & Prompt Architecture

**Name:** Dotty
**Persona:** Cheeky best friend — warm, informal, gently teasing, honest about AI limitations.

**System prompt layers (injected every turn):**
```
LAYER 1: Base Identity (static)
"You are Dotty, a personal AI assistant. You are warm, cheeky, and helpful.
You speak like a smart friend texting. You remember facts about the user
and reference them naturally."

LAYER 2: Memory Injection (dynamic)
"Facts about {user_name}: {relevant_memories}"

LAYER 3: Account Context (dynamic)
"Your Connected Accounts:
- Gmail: Work Gmail (sarah@acme.com), Personal Gmail (sarah.smith@gmail.com)
- Calendar: Work Calendar, Personal Calendar
When searching emails or events, check ALL accounts unless user specifies one.
When sending emails or creating events, ask which account if unclear."

LAYER 4: Tool Context (dynamic)
"You have access to: {registered_tools}. Use them proactively when helpful."

LAYER 5: Session Context (dynamic)
Current time: {ISO8601}
User timezone: {tz}
```

**Constraint:** Total system prompt < ~1500 tokens to leave room for conversation history.

**Personality slider:** Settings offers a "professional ↔ casual" slider that adjusts the base identity layer:
- Professional: "You are Dotty, a professional AI assistant. You are efficient, warm, and concise."
- Casual: "You are Dotty, a cheeky best friend AI. You speak like you're texting a close friend."

**Known unknowns:**
- [DECISION NEEDED] **Tone calibration:** How do we measure if Dotty's tone is hitting the mark? A/B test? Manual review? User thumbs up/down on each message?
- [DECISION NEEDED] **Error tone:** When tools fail or the LLM errors, does Dotty stay in character ("Oof, my Gmail connection is being moody") or switch to neutral? (Impacts user trust)

---

## 5. Multi-Account Deep Dive

This is the highest-complexity feature. Every table below represents a decision an agent needs to make when writing the TRD.

### 5.1 Account Disambiguation Logic

When the user says something ambiguous, how does Dotty pick the right account?

| Scenario | Rule |
|---|---|
| "What's new in my email?" | Search ALL Gmail accounts, merge by date, tag with alias |
| "Reply to Sarah" | Check Work contacts first (heuristic: "Sarah" from work context). If found in both, ask. |
| "Email Mom" | Default to Personal Gmail (heuristic: "Mom" = personal relationship) |
| "Schedule a meeting" | Default to Work Calendar (heuristic: "meeting" = work) |
| "Add 'lunch with Mom'" | Default to Personal Calendar (heuristic: "Mom" = personal) |
| "Send an email to the CEO" | Default to Work Gmail. If CEO also in personal contacts, ask. |
| Explicit: "from Work Gmail" | Use Work Gmail directly |
| Ambiguous + high stakes (send email) | Always ask user for confirmation before sending |

**Known unknowns:**
- [DECISION NEEDED] **Contact correlation:** How do we know "Sarah" is the work Sarah vs. personal Sarah? We don't have a contact book API. Do we extract contact relationships from email history and build an implicit contact graph?
- [DECISION NEEDED] **Disambiguation UI:** When Dotty can't decide, what does the response look like? Inline buttons? "Work Sarah or Personal Sarah? [Work] [Personal]"?
- [DECISION NEEDED] **Learning from corrections:** If user corrects Dotty's account choice, does Dotty remember? (e.g., "No, email my personal Sarah" → stores as memory for future turns.)

### 5.2 Conflict Detection

When checking availability or creating events:
- Check ALL calendars BEFORE creating
- If conflict found across ANY calendar: warn user, suggest alternatives
- "You have a conflict! Work wants you at 3pm but Personal Calendar has 'dentist.' Want me to suggest alternatives?"

**Known unknowns:**
- [DECISION NEEDED] **Calendar priority:** If Work Calendar says "busy" and Personal Calendar says "free," is the user actually busy? Or should we respect the "busy" as a hard block? (e.g., "focus time" blocks on one calendar might not block the other)
- [DECISION NEEDED] **Tentative events:** How do we handle "tentative" calendar events? Treat as busy or free?

### 5.3 Trigger Architecture (New Emails)

Each connected account gets its own Composio webhook trigger.

**Notification rules per account:**
| Mode | Behavior |
|---|---|
| `immediate` | Push/WebSocket as soon as email arrives |
| `digest` | Queue, send batch summary at configured hour (default 9am) |
| `off` | Never push. Still searchable/queryable by user. |

**Smart immediate rules:**
Even in `immediate` mode, apply heuristics:
- From boss/CEO + subject contains "urgent"/"asap" → immediate push even during quiet hours
- From mailing list/newsletter → treat as digest unless user explicitly subscribed to immediate
- Label "Promotions" → digest by default

**Known unknowns:**
- [DECISION NEEDED] **Quiet hours:** Configurable time window where even urgent emails get batched? (e.g., 10pm–7am). Per-user or global?
- [DECISION NEEDED] **Trigger deduplication:** If cron fallback AND webhook fire for the same email, how do we prevent double-notification? (Idempotency key based on email message ID)

---

## 6. WebSocket Protocol

### Client → Server
```json
{ "type": "message", "id": "msg_123", "content": "What's my schedule?", "attachments": ["upload_456"] }
{ "type": "typing_indicator", "id": "msg_123", "status": "started" }
{ "type": "presence", "status": "online" }
{ "type": "steer", "content": "Wait, stop, check my personal calendar too" }
{ "type": "feedback", "messageId": "msg_124", "rating": 1 }
```

### Server → Client
```json
{ "type": "text_delta", "messageId": "msg_124", "delta": "You have 3 meetings today..." }
{ "type": "tool_start", "toolName": "calendar_view_all", "args": {"start": "2026-05-10T00:00:00Z"}, "accountAlias": null }
{ "type": "tool_end", "toolName": "calendar_view_all", "result": "...", "duration": 340 }
{ "type": "account_prompt", "messageId": "msg_124", "text": "Which Gmail account should I use?", "options": [{"alias": "Work Gmail", "email": "sarah@acme.com"}, {"alias": "Personal Gmail", "email": "sarah.smith@gmail.com"}] }
{ "type": "message_complete", "messageId": "msg_124" }
{ "type": "memory_update", "memories": [{"type": "fact", "content": "User has 3 meetings today"}] }
{ "type": "push", "title": "📧 New email on Work Gmail", "body": "'Q4 numbers' from CEO — might want to look at this first. 🤔" }
{ "type": "error", "code": "tool_timeout", "message": "Gmail connection timed out. Want me to retry?" }
```

**Known unknowns:**
- [DECISION NEEDED] **Reconnection strategy:** WebSocket drops on mobile networks. Auto-reconnect with exponential backoff? Replay missed messages on reconnect? (Requires message sequencing/seq numbers)
- [DECISION NEEDED] **Message ordering:** If user sends two messages rapidly while the agent is still streaming, how are they queued? Steer queue vs. follow-up queue? (pi-agent-core supports both)

---

## 6.1 Attachment Flow

### Upload Pipeline
1. Client uploads file to `/api/upload` (POST, multipart/form-data)
2. API Gateway validates file type (images: JPEG/PNG/GIF/WebP/heic; documents: PDF; max 25MB)
3. File stored in Cloudflare R2 at path `{user_id}/{conversation_id}/{upload_id}.{ext}`
4. API Gateway returns `{ "uploadId": "upload_456", "url": "https://r2.../upload_456", "mimeType": "...", "sizeBytes": N }`
5. Client includes `uploadId` in WebSocket message: `{ "type": "message", "content": "...", "attachments": ["upload_456"] }`
6. Agent Runtime reads attachment via `read_file` tool (blob from R2)

### Storage
- R2 (S3-compatible): user-scoped paths, signed URLs (15-min TTL) for reads
- No raw bytes in WebSocket messages — always reference by ID
- Attachment metadata stored in `messages.attachments` JSONB column: `{ id, url, mimeType, sizeBytes, filename }`

### Constraints
| Parameter | Value |
|---|---|
| Max file size | 25MB |
| Max total per message | 5 attachments |
| Stored formats | JPEG, PNG, GIF, WebP, HEIC, PDF |
| Retention | Same as conversation history |

---

## 7. Error Handling & Edge Cases

### Token Budget & Context Overflow Strategy
When the context window approaches its limit, messages are evicted in this order (last to first, "sacrifice order"):

**Truncation order (evict last = keep longest):**
1. Oldest user messages (user turns) — until `BASE_PROMPT_TOKENS + LAST_N_MESSAGES` fit
2. Oldest assistant messages — only if step 1 is insufficient
3. Memories — oldest/lowest-confidence first, but always keep top 5

**Sacrifice order (evict first = most dispensable):**
1. `relationship` memories — lowest confidence first
2. `event` memories older than 90 days
3. `preference` memories
4. `fact` memories — only if all above exhausted

**Base prompt composition (max tokens):**
| Layer | Max Tokens |
|---|---|
| Base identity | 500 |
| Memory injection | 1500 |
| Account context | 500 |
| Recent messages summary | 2000 |
| Working room for LLM response | 2000 |
| **Total base** | **6500** |

Model context window: 128k (GPT-4o). Soft eviction triggers at 100k tokens. No hard limit enforced — LLM provider handles overflow with error.

**Compaction trigger:** When `systemPromptTokens > 90k`, run compaction: collapse oldest 20 messages into a 2-sentence summary, replace the block in context.

---

### Observability & Logging Strategy
All services emit structured JSON logs to stdout (collected by stdout → CloudWatch/Cloudflare Logs).

**Log levels:**
- `ERROR`: Tool failures, pod crashes, snapshot write failures, auth failures
- `WARN`: Rate limits hit, single-account OAuth expiry, LLM retry
- `INFO`: Turn completions, session creation/destruction, tool executions
- `DEBUG`: Full tool args (redact PII), LLM token counts

**Structured log fields (every log line):**
```json
{ "ts": "ISO8601", "level": "INFO", "service": "agent-runtime", "userId": "usr_xxx", "traceId": "trc_xxx", "msg": "...", "duration_ms": 340 }
```

**Key traces to capture:**
- `turn_start` → `turn_end` (full turn latency)
- `tool_execution_start` → `tool_execution_end` (per-tool latency, tool name, account alias)
- `message_received` → `message_complete` (end-to-end message latency)
- `snapshot_write` success/failure

**Metrics (Prometheus format, /metrics endpoint):**
- `dotty_turns_total{userId}` — counter
- `dotty_turn_duration_seconds{userId}` — histogram
- `dotty_tool_duration_seconds{toolName}` — histogram
- `dotty_active_sessions` — gauge
- `dotty_tokens_used{model}` — counter
- `dotty_cost_cents` — counter
- `dotty_tool_errors_total{toolName, errorCode}` — counter

**SLOs:**
| SLO | Target | Alert if |
|---|---|---|
| Turn completion latency (p95) | < 10s | > 15s for 5min |
| Tool execution success rate | > 95% | < 90% for 5min |
| Session restore success rate | > 99% | < 95% for 5min |
| Snapshot write success rate | > 99.5% | < 99% for 5min |

### Test Strategy

**Unit tests (Jest/Vitest):**
- All pure functions: prompt builders, tool name sanitizers, memory type classifiers, message serializers
- Deduplication logic for memory insertion
- Token budget calculations

**Integration tests (Composio mock):**
- Full turn flow with mocked Composio responses (no real API calls)
- Multi-account routing: verify correct `connected_account_id` used per tool call
- Session snapshot save/restore cycle
- WebSocket message parsing (both directions)
- OAuth token expiry flow (mock expired token → verify correct error code)

**E2E tests (Playwright + real Gmail/Calendar in staging):**
- Full signup → connect Gmail → send email flow
- Multi-account switching: work vs personal send
- Morning briefing delivery end-to-end
- Attachment upload → send with attachment
- Push notification delivery (staging APNS)

**Composio mock approach:**
```typescript
// Mock Composio so tool calls return deterministic data without real API calls
jest.mock('@composio/core', () => ({
  Composio: jest.fn().mockImplementation(() => ({
    create: jest.fn().mockResolvedValue({ sessionId: 'mock' }),
    execute: jest.fn().mockResolvedValue({
      content: JSON.stringify({ emails: [...mockEmails] }),
    }),
  })),
}));
```

**Known unknowns:**
- [DECISION NEEDED] **Test accounts:** Do we use real Gmail test accounts or Gmail sandbox? (Sandbox requires `@test` suffix on addresses)

### Morning Briefing / Scheduled Task Architecture

Dotty sends a daily summary ("Good morning, here's your day") at a user-configured time (default 7am local time).

**Trigger mechanism:**
- User sets `notify_digest_hour` (0–23) in their profile
- Cron job runs every 15 minutes, queries users whose `notify_digest_hour` falls within the current 15-min window
- For each qualifying user: enqueue a briefing job to Redis (sorted set by timestamp)
- Agent Runtime dequeues and executes briefing as a special "system-initiated" turn

**Briefing composition:**
1. Fetch all calendar events for today across all accounts
2. Fetch unread emails from priority senders (boss, urgent labels) — last 24h
3. Inject memories tagged `event` with dates in next 7 days
4. Dotty writes briefing in her voice (2–4 sentences, bullet points for schedule)
5. Delivered via WebSocket `push` message type if user is online; stored for pickup if offline

**Cron fallback for Composio webhooks:**
- If Composio misses an email, the 15-min cron polls all `active` integrations
- Deduplicates using email `Message-ID` as idempotency key stored in Redis (`seen_emails:{messageId}` with 24h TTL)

### Integration Failures
| Scenario | Expected Behavior |
|---|---|
| One Gmail account's OAuth token expires | Tool call fails with specific error. Dotty informs user: "My connection to Work Gmail expired — want me to re-link it?" Other accounts continue working. |
| All Gmail accounts disconnected | Dotty replies: "I can't check your email right now (my connections are down). Want me to try again or just search the web?" |
| Composio API down | All tool calls fail. Dotty handles gracefully: "My tools are having a moment. Here's what I know from memory..." |
| LLM API rate limited | pi-agent-core auto-retries with backoff. User sees extended typing indicator. |
| LLM API completely down | Return error to user: "My brain is taking a quick nap. Try again in a minute?" |

### Data Safety
| Scenario | Expected Behavior |
|---|---|
| Agent runtime pod crashes | In-progress turn is lost (user sees truncated message). Last completed turn is recoverable from snapshot. On reconnect, session restores and continues from last completed turn. |
| Database write fails during snapshot | Retry 3 times with backoff. If still failing, log to monitoring but don't crash the agent runtime. Accept that one snapshot is lost. |
| User deletes account | GDPR deletion: remove from `users`, `integrations`, `session_snapshots`, `messages`, `memories`. Composio connected accounts revoked via API. |

### Error Code Catalog
All errors are `code` + `message` pairs sent via WebSocket `error` type. HTTP equivalents are for API Gateway logging.

| Code | HTTP Status | Description | Client Remediation |
|---|---|---|---|
| `invalid_attachment` | 400 | Attachment file type not allowed or exceeds 25MB | Strip attachment, retry |
| `attachment_too_large` | 413 | Combined attachments exceed 5 per message | Reduce attachment count |
| `upload_not_found` | 404 | Referenced upload ID does not exist or expired | Re-upload file |
| `rate_limit_exceeded` | 429 | User hit rate limit | Auto-retry after `retryAfter` seconds |
| `auth_expired` | 401 | Clerk JWT expired or invalid | Redirect to login |
| `oauth_token_expired` | 401 | One Gmail/Calendar account OAuth expired | Show "re-link account" prompt inline |
| `oauth_all_expired` | 401 | All accounts for a provider are expired | Show reconnect flow |
| `tool_timeout` | 504 | Tool call (Gmail/Calendar search) timed out after 30s | Retry or try again later |
| `tool_execution_failed` | 502 | Composio tool execution failed (non-timeout) | Retry; if persistent, surface to Dotty |
| `session_not_found` | 404 | Session restore attempted but no snapshot exists | Start fresh session, inform user |
| `snapshot_write_failed` | 500 | Snapshot could not be written | Retry; log to monitoring; continue without snapshot |
| `llm_rate_limited` | 429 | LLM provider (OpenAI) rate limited | Auto-retry with exponential backoff (pi-agent-core handles) |
| `llm_unavailable` | 503 | LLM provider completely down | Surface "brain nap" message, suggest retry |
| `composio_down` | 502 | Composio API unreachable | Surface graceful degradation message |
| `webhook_verification_failed` | 401 | Incoming Composio webhook signature invalid | Log and reject |
| `pod_crash` | 500 | Agent runtime pod crashed mid-turn | Session restores on restart; user sees truncated message |

---

## 8. Security Model

- **Auth:** Clerk handles JWT issuance/validation. No custom auth code.
- **Data isolation:** Every query scoped by `user_id`. Never return data for user A when user B asks.
- **OAuth tokens:** Never stored in our database. Composio holds tokens. We only store `connected_account_id`.
- **Session snapshots:** Encrypted at rest (Neon). No PII in snapshot other than conversation content.
- **Attachments:** Uploaded to S3/R2 with user-scoped paths. Signed URLs for access.
- **Rate limiting:** Redis-based. Per-user, per-endpoint. Prevents abuse and runaway costs.
- **Webhook security:** Composio webhooks validated via `X-Composio-Signature` HMAC-SHA256 header. Secret in `COMPOSIO_WEBHOOK_SECRET` env var.

**Environment / secret management:**
- `dotenv` for local dev (`.env` file, never committed)
- Production secrets in Fly.io secrets / Railway env vars / Render secret store
- All secrets: `CLERK_SECRET_KEY`, `COMPOSIO_API_KEY`, `OPENAI_API_KEY`, `COMPOSIO_WEBHOOK_SECRET`, `DATABASE_URL`, `REDIS_URL`, `R2_*`
- No secrets in code, no hardcoded credentials

**Known unknowns:**
- [DECISION NEEDED] **Zero-trust vs. perimeter:** Do we run the agent runtime in a network-isolated environment (VPC) or is public internet access fine? (Impacts hosting choice)
- [DECISION NEEDED] **Audit log retention:** How long do we keep `usage_events` and `messages`? 30 days? 1 year? Forever? (Compliance + cost impact)

---

## 7.1 Onboarding UI Flow (Integrations-First)

Onboarding is integrations-first (not chat-first) to ensure Dotty has Gmail/Calendar access before the user sends their first message.

**Onboarding steps:**
1. **SplashScreen** — "Meet Dotty" + sign up / sign in (Clerk)
2. **NameCapture** — "What should Dotty call you?" (stores in `users.name`)
3. **IntegrationConnectScreen** — Connect Gmail (1+ accounts) + Calendar via Composio Connect Links
4. **DefaultAccountSelection** — Which Gmail/Calendar should Dotty use by default? (sets `is_default = true`)
5. **InductionPrompt** — Dotty sends first message: "Hey {name}! 👋 Nice to officially meet you. I've got access to your {account aliases}. What do you want to do first?" — generated from `BOOTSTRAP.md` on first run
6. **ChatScreen** — Empty state with suggested prompts

**BOOTSTRAP.md first-run ritual:**
`BOOTSTRAP.md` loaded once per new user (inserted as system message on first session creation). Contains user name, connected account aliases, personality reminder, "Introduce yourself naturally."

---

## 7.2 Feedback Mechanism

**Thumbs down on a message:**
- Client sends `{ "type": "feedback", "messageId": "msg_124", "rating": 1 }`
- Stored in `usage_events` as `event_type: 'feedback'` with message ID reference
- NOT used for automatic regeneration
- Dotty does not acknowledge the feedback directly
- Reviewed by the team weekly; persistent patterns trigger system prompt adjustments

---

## 7.3 Rate Limit Behavior

Redis token bucket: `rate_limit:{user_id}` — 100 messages/minute, 1000 tool calls/minute.
- HTTP 429 with `{ "code": "rate_limit_exceeded", "retryAfter": 30 }`
- WebSocket: server sends `{ "type": "error", "code": "rate_limit_exceeded", "message": "Slow down! Try again in 30s." }`
- Client auto-retries after `retryAfter` seconds with exponential backoff (cap: 5 retries)

---

## 7.4 Conversation Branching

`session.navigateTree()` is already in pi-agent-core. For MVP, this is the preserved mechanism for future branching but is not actively used.

**MVP behavior:** All messages go into a single linear conversation. `navigateTree()` exists for future use when we add message edit/regenerate, branching into sub-conversations, or exploring alternatives. Tree state stored in `session_snapshots.messages` as flat array.

---

## 7.5 Session Snapshot Recovery Race Condition

**Scenario:** Pod crashes mid-turn. User sends new message. New pod instance starts and tries to restore session.

**Handling:**
- On pod restart: Redis `session:active:{userId}` is cleared (TTL-expired or crashed key)
- New message arrives → `getOrCreateSession()` finds NO active session → loads from PostgreSQL snapshot
- If snapshot is from before the crashed turn: user sees message "Hey, sorry about that — my connection hiccuped. Let me catch up." and the previous completed turn is restored
- If no snapshot exists: fresh session starts, no history
- **Queued messages on restart:** messages sent while pod was down are queued in Redis at `session:offline_queue:{userId}` (1-hour TTL). On session restore, API Gateway replays the queue before processing new message.

---

## 7.6 Rich Artifact Pipeline

Dotty returns formatted content beyond plain text: tables, timelines, draft emails.

**Artifact types:**

| Type | Rendered as |
|---|---|
| `schedule_table` | Markdown table in chat bubble |
| `email_draft` | Formatted email with to/subject/body |
| `timeline` | Vertical timeline with dates |
| `comparison_table` | Multi-column table |

**Delivery:** Agent emits `text_delta` with markdown + type hint embedded: `---TYPE:schedule_table---`. Frontend detects type hint and renders in appropriate component. macOS uses `WKWebView` for rich HTML.

---

## 7.7 Multi-Device Session State

**Each device has its own WebSocket connection.** No cross-device message relay.

**Per-device behavior:**
- Chat history fetched from PostgreSQL on connect (paginated)
- New messages on Device A appear in Device B's history on next refresh/poll
- No real-time sync between devices via WebSocket — client polls `/api/conversations` on visibility change
- Active session state is per-runtime-pod; Device B may briefly see stale session state after Device A sends a message

**Trade-off:** Multi-device users may see a 1–5 second delay before new messages appear in secondary device history. No cross-device WebSocket push in MVP.

---

## 7.8 Rollback Strategy for Failed Tool Calls

**Immediate retry (pi-agent-core handles):**
- Tool timeout: 30s → retry once automatically
- Composio rate limit: exponential backoff (1s, 2s, 4s) up to 3 retries

**User-surfaced vs silently retried:**
- `tool_timeout` after retry: surface to user
- `oauth_token_expired`: surface immediately with re-link prompt
- `tool_execution_failed` (non-timeout): log and retry silently up to 2 times, then surface

**Backoff schedule:**
| Attempt | Delay |
|---|---|
| 1st retry | 1 second |
| 2nd retry | 2 seconds |
| 3rd retry | 4 seconds |
| Then surface to user | — |

---

## 7.9 OAuth Token Lifecycle

Composio handles OAuth token refresh automatically for all connected accounts.

- `connected_account_id` stored in our DB — the Composio stable ID
- We do NOT store OAuth access tokens — Composio holds them
- `status: 'active'` in `integrations` means Composio reports account as connected
- If Composio reports token expired (via `/connected_accounts/{id}/status` or webhook), we update `integrations.status = 'expired'` and surface re-link prompt

Composio auto-refreshes tokens automatically before expiry. We implement no own OAuth refresh logic.

---

## 9. Hosting & Scaling

### MVP: "One Box" (0–1000 users)
- Single Node.js process (API + Runtime + Trigger handler)
- Neon PostgreSQL, Upstash Redis, Cloudflare R2
- Deployed on Fly.io, Railway, or Render
- No Kubernetes complexity

### When to Scale (1000–10,000 users)
- Separate API Gateway (stateless pods) from Agent Runtime (stateful pods)
- Redis session routing between runtime pods
- Composio trigger handler as separate service
- Neon read replicas for message history queries

### When to Scale (10,000+)
- Multi-region deployment
- Dedicated Composio webhook queue (SQS / RabbitMQ)
- pgvector for semantic memory at scale
- Potential migration to Mem0 for memory if custom becomes burdensome

**Known unknowns:**
- [DECISION NEEDED] **Initial hosting platform:** Fly.io vs. Railway vs. Render? (Fly has better WebSocket support. Railway is simpler. Render is cheapest.)
- [DECISION NEEDED] **Containerization:** Docker from day one, or deploy Node.js directly? (Docker adds ops overhead but portability)

---

## 10. Browser Automation Future-Proofing (Not in MVP)

We are NOT building this now, but the architecture must not prevent it.

**How it would work post-MVP:**
1. Add `browser_*` tools to the tool registry: `browser_navigate`, `browser_click`, `browser_type`, `browser_screenshot`
2. Tool executes in a Playwright/Chromium container (separate microservice)
3. Returns HTML/screenshot as artifact
4. macOS app's existing WebView renders the result

**MVP requirements for future-proofing:**
- `AgentTool` system can handle arbitrary new tools (verified: pi-agent-core supports this)
- macOS WebView can render HTML artifacts (already in MVP scope)
- Database doesn't need migration (browser actions are just tool calls in `messages.tool_calls`)

---

## 11. Success Metrics (MVP)

| Metric | Target | How Measured |
|---|---|---|
| Day-1 retention | > 60% | Users who return within 24h of signup |
| Week-1 retention | > 40% | Users who return within 7 days |
| Gmail connect rate | > 50% | % of signups who connect Gmail |
| Multi-account connect rate | > 30% | % of signups who connect 2+ accounts |
| Avg messages/day | > 5 | Active users only |
| Tool usage rate | > 30% | % of turns that use at least one tool |
| Memory recall accuracy | > 80% | Manual sampling: does Dotty reference memories correctly? |
| CSAT | > 4.0/5 | In-app NPS survey |
| COGS/user/month | <$5 | Usage tracking table |

---

## 12. The Blocking Questions (Summary)

These are the questions that, if unanswered, prevent writing a good TRD. They are grouped by which component they block.

### Blocks Web App Design
1. Offline behavior: queue-and-retry or fail immediately?
2. Attachment size limit and storage?
3. Push notification strategy per account?
4. Artifact interaction: read-only or interactive?
5. Disambiguation UI: inline buttons or modal?
6. Reconnection strategy: auto-reconnect with replay?
7. Push reliability on iOS Safari: degraded push acceptable or SMS fallback?

### Blocks Backend/API Design
8. Rate limiting: fixed window or token bucket? What limits?
9. WebSocket library: Socket.io or ws?
10. Composio integration: MCP or native SDK?
11. Webhook verification method?
12. API Gateway framework: Fastify or Express?
13. Containerization: Docker from day one?

### Blocks Agent Runtime Design
13. Session eviction: how many minutes of idle?
14. Snapshot granularity: every turn or batched?
15. Tool timeout: how many seconds?
16. System prompt token budget: hard limit?
17. Error behavior: When one account's token expires, how does the runtime handle it?
18. LLM fallback: auto-switch provider on outage or fail gracefully?

### Blocks Memory System Design
19. Memory store: Mem0 SaaS or custom PostgreSQL?
20. Memory decay: TTL or permanent?
21. Memory source tracking: include integration source?

### Blocks Database Design
22. Row-level security: RLS or application-level?
23. pgvector: add `vector` column now or migrate later?

### Blocks Security/Compliance Design
24. Zero-trust vs. perimeter model?
25. Audit log retention period?

### Blocks Notification/Trigger Design
26. Quiet hours: per-user configurable or global?
27. Trigger deduplication: based on email message ID?
28. Cron fallback: all users or opt-in?

### Blocks Product/UX Design
29. Tone calibration: A/B test or manual review?
30. Error tone: stay in character or go neutral?
31. Calendar priority: respect "busy" from all calendars or only same-account?
32. Tentative events: treat as busy or free?
33. Contact correlation: build implicit contact graph from email history?
34. Learning from corrections: store disambiguation choices as memories?

### Blocks Hosting/Infrastructure Design
35. Initial platform: Fly.io, Railway, or Render?
36. Redis: single instance or cluster from day one?
37. Budget: What's the monthly COGS ceiling per active user?

---

## Appendix: Reference Documents

- [pi.dev Docs](https://pi.dev/docs/latest) — pi coding agent SDK, runtime, and extension architecture
- [Composio Multi-Account Docs](https://docs.composio.dev/docs/managing-multiple-connected-accounts) — How Composio handles multiple accounts per toolkit
- [OpenClaw Architecture](https://docs.openclaw.ai/concepts/architecture) — How OpenClaw uses pi.dev as its embedded agent runtime
- [OpenClaw Agent Runtime](https://docs.openclaw.ai/concepts/agent) — Details on session model, bootstrap files, and steering

---

*If you are an AI agent building a TRD from this PRD, please ask clarifying questions on the [DECISION NEEDED] items before proceeding. Every unanswered question is a blocker.*
