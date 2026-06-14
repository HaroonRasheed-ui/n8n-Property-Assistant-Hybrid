# 🏠 Property Assistant — Hybrid AI Chatbot (n8n)

An intelligent real estate chat assistant built on [n8n](https://n8n.io), combining a deterministic state machine with LLM-powered intent extraction. It remembers user preferences across turns, searches a Postgres property database, caches results, paginates listings, reports live weather, and keeps working even when the AI provider is down.

Built as a hybrid of two architectures: persistent session state (Postgres) for reliability + conversational context (chat history fed to the LLM) for natural understanding.

## ✨ Features

- **Conversational property search** — extracts property type, city, price range, and bedrooms from natural language using Groq (Llama 3.3 70B) with `temperature: 0` and forced JSON output
- **Persistent per-session memory** — filters, results, chat history (last 12 turns), and pagination offset stored in Postgres; survives restarts and is fully multi-user safe
- **Zero-token fast path** — number selections (`1`/`2`/`3`), greetings, `reset`, and `more` are handled in pure JavaScript without any LLM call
- **AI fallback ("Quick mode")** — if the Groq API fails or returns malformed JSON, a regex-based keyword extractor takes over so the bot never goes down
- **City-level result cache** — 30-minute TTL cache per city stored in the session, cutting repeat DB queries
- **Pagination** — fetches up to 10 matches, shows 3 per page; `more` advances the page and selections stay index-correct via a stored offset
- **Reset command** — `reset` / `start over` wipes filters and results while keeping the still-fresh city cache
- **Weather lookup** — OpenWeatherMap integration with graceful error handling for unknown cities, plus a reminder of the active property search
- **SQL-injection safe** — every query is parameterized (`$1`, `$2`, …); JSONB payloads are passed as Base64 and decoded server-side (`convert_from(decode($n,'base64'),'UTF8')::jsonb`) to avoid n8n's comma-splitting in `queryReplacement`

## 🧭 Architecture

```
Chat Trigger
   │
Parse Input ──► fast-path detection (1/2/3, greet, reset, more → skip AI)
   │
Read Session (Postgres: state, results, cache, history, offset)
   │
Build Prompt (previous state + last 6 chat turns)
   │
Skip AI? ──no──► Groq (Llama 3.3 70B) ──► Parse + Merge State ◄── regex fallback on failure
   │ yes                                        │
   └────────────────────────────────────────────┤
                                          Save Session
                                                │
                                          Intent Router
   ┌──────────┬──────────┬──────────┬───────────┼──────────┬──────────┐
 WEATHER    SEARCH     GATHER     DETAIL      GREET      RESET     SHOW MORE
   │      cache→DB        │          │           │          │          │
   └──────────┴──────────┴──────────┴───────────┴──────────┴──────────┘
                                                │
                                   Finalize Turn (history append)
                                                │
                                   Save Turn (single UPDATE)
                                                │
                                        Respond to Chat
```

## 📋 Prerequisites

- n8n (self-hosted or cloud) with the **Chat Trigger** node available
- PostgreSQL database
- [Groq API key](https://console.groq.com) (Header Auth credential: `Authorization: Bearer <key>`)
- [OpenWeatherMap API key](https://openweathermap.org/api)

## 🗄️ Database Setup

```sql
CREATE TABLE IF NOT EXISTS properties (
  id SERIAL PRIMARY KEY,
  property_type TEXT NOT NULL,      -- Apartment | Villa | Studio | Condo | House
  price NUMERIC NOT NULL,           -- monthly price
  city TEXT NOT NULL,
  state TEXT,
  bedrooms INT,                     -- 0 = Studio
  amenities JSONB DEFAULT '[]'::jsonb,
  availability BOOLEAN DEFAULT true
);

CREATE TABLE IF NOT EXISTS sessions (
  session_id TEXT PRIMARY KEY,
  state JSONB DEFAULT '{}'::jsonb,
  last_results JSONB DEFAULT '[]'::jsonb,
  city_cache JSONB DEFAULT '{}'::jsonb,
  chat_history JSONB DEFAULT '[]'::jsonb,
  result_offset INT DEFAULT 0,
  updated_at TIMESTAMPTZ DEFAULT NOW()
);
```

## 🚀 Installation

1. Import `property-assistant-hybrid.json` into n8n (**Workflows → Import from File**)
2. Create credentials and attach them:
   - **Postgres** — on nodes `03`, `08`, `15`, `31`
   - **Header Auth (Groq)** — on node `06 | Groq Call`
3. Set your OpenWeatherMap API key in node `10 | Weather API` (recommended: move it to a credential or environment variable instead of leaving it inline)
4. Enable **"Always Output Data"** on node `15 | DB Query` so empty result sets fall through to the "No matches found" message
5. Activate the workflow and open the public chat URL from the Chat Trigger node

## 💬 Usage Examples

| You say | Bot does |
|---|---|
| `Hi, how are you` | Greets instantly (no AI call), recalls any active search |
| `I want a 2 bedroom apartment in Lahore under 50000` | Extracts filters, queries DB, lists top 3 by price |
| `2` | Shows full details of result #2 (no AI call) |
| `more` | Next page of results (no AI call) |
| `what about the weather there?` | Live weather for the tracked city |
| `actually make it a villa` | Updates only `property_type`, keeps city and budget |
| `reset` | Clears all filters, fresh start (no AI call) |

## ⚙️ Key Design Decisions

- **State lives in Postgres, not in the LLM.** The model only extracts deltas; merging is deterministic JavaScript. The model can never "forget" a filter.
- **Intent escalation is conservative.** `search_property` is only forced when intent is `gather_info` and both `property_type` + `city` are known — greetings and detail selections are never hijacked into a search.
- **Cheap before smart.** Anything resolvable with string matching skips the LLM entirely.
- **Base64 for JSONB parameters.** n8n's `queryReplacement` splits values on commas; JSON contains commas, Base64 does not.

## 🔧 Troubleshooting

| Symptom | Cause / Fix |
|---|---|
| Branch dies silently after DB query | Enable **Always Output Data** on node `15` |
| Zero results but data exists | Check exact-case `property_type` values, price units, `availability` flag, and `bedrooms` NULL vs `0` for studios |
| Greeting routed to search | Ensure node `07` only escalates intent when it equals `gather_info` |
| "(Quick mode)" suffix in replies | Groq call failed; regex fallback handled the turn — check Groq credential/quota |

## 📄 License

MIT
