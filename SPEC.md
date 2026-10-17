# Instagram AI DM & Comment Bot — Project Specification

## 1. Goal

A self-hosted bot that uses the Instagram Graph API to read incoming DMs and comments on one or more Instagram Business/Professional accounts, generates replies with an OpenAI-compatible AI provider, and sends them back. Everything is managed from a Next.js dashboard.

**Finished means:** a connected Instagram account receives a DM or comment, the bot replies using the currently selected AI model, and the owner can see, pause, take over, and switch models from the dashboard without restarting anything.

## 2. Hard constraints

- **Minimal technology.** No Docker, no Redis, no external queue, no separate backend service, no managed database.
- One Next.js app (App Router, TypeScript) does everything: dashboard, API routes, webhook receiver, background worker.
- SQLite file database. Plain SQL via `better-sqlite3` (no ORM unless the plan shows a clear need).
- Runs with `npm install`, `npm run dev`, and `npm run build && npm start`. Nothing else required.
- Must run on a long-lived Node.js process (own machine or VPS). Serverless hosting (Vercel etc.) is **not** supported because the background worker needs a persistent process.
- Dependencies are added only when needed and justified in the final report of the task that adds them.
- Secrets live in `.env.local` only. Never committed.
- Do not call Instagram or the AI provider from client-side code. All calls are server-side.

## 3. Tech stack (decided)

- Node.js 20.9+ (LTS), npm
- Next.js (App Router) + TypeScript
- Tailwind CSS (styling only; no component library unless later approved)
- SQLite via `better-sqlite3`
- `zod` for validation
- `openai` npm package, used only as an OpenAI-compatible client with a configurable `baseURL`
- Background worker: in-process loop started from Next.js `instrumentation.ts`, jobs stored in SQLite

## 4. Architecture

1. Instagram sends a webhook to `POST /api/webhooks/instagram`.
2. The handler verifies the `X-Hub-Signature-256` HMAC with the app secret, stores the raw event idempotently (unique on message ID / comment ID), enqueues a job, and returns `200` immediately.
3. The in-process worker claims jobs from the SQLite `jobs` table (with locking, retries, and exponential backoff).
4. For each job the worker loads context (conversation history, rules, persona, knowledge base), calls the active AI model, applies safety checks, and sends the reply through the Instagram API.
5. Everything is logged: events, AI requests (model, tokens, latency, error), and sent messages.
6. `GET /api/webhooks/instagram` handles Meta's webhook verification handshake (`hub.verify_token`).

## 5. Instagram integration

Use the **Instagram API with Instagram Login** (Business/Professional accounts, no Facebook Page required). The Worker must verify every detail below against current official Meta documentation before implementing, because API versions, permissions, and limits change.

- OAuth connect flow from the dashboard; store the long-lived token encrypted at rest and refresh it before it expires (about 60 days).
- Permissions expected: `instagram_business_basic`, `instagram_business_manage_messages`, `instagram_business_manage_comments`.
- Webhook fields expected: `messages`, `messaging_postbacks`, `comments`.
- Send DMs through the messages endpoint. Enforce the messaging window (replies allowed only within 24 hours of the user's last message; do not attempt outside it, mark as "window expired").
- Comments: reply publicly under the comment, and/or send a **private reply** (DM) tied to the comment ID (limited time window, one per comment; verify exact limits).
- Ignore the account's own messages and echoes so the bot never replies to itself.
- Development mode works only for accounts with a role on the Meta app. Production use needs App Review for advanced access. Document this in the README.
- Respect Meta Platform Terms: always offer a human handoff path and never send unsolicited messages.

## 6. AI layer (OpenAI-compatible)

- **Providers:** a table of providers, each with name, base URL, API key (encrypted at rest), enabled flag, and optional extra headers.
- **Models:** each provider has models (model ID string, display name, optional context length, enabled flag). Models can be added by:
  - fetching `GET {baseURL}/models` from the provider, or
  - typing a model ID manually (for providers without a models endpoint).
- **Custom provider and model support:** any OpenAI-compatible endpoint can be added from the dashboard with no code changes. The first provider is seeded from env vars (`AI_BASE_URL`, `AI_API_KEY`, `AI_DEFAULT_MODEL`).
- **Model switching at runtime:** the active model is stored in the database, set independently for DM replies, comment replies, and an optional fallback. Changing it in the dashboard takes effect on the next job, no restart.
- **Fallback:** if the primary model errors or times out, retry with the fallback model and log it.
- **Test tools:** "Test connection" button per provider and a Playground page to send a test prompt to any model.
- **Parameters:** temperature, max tokens, and timeout configurable globally and per use case.
- **Prompting:** editable system prompt (persona and rules) per use case, plus a plain-text knowledge base (FAQ, pricing, hours) injected into the prompt. No vector database.
- **Usage log:** prompt/completion tokens, latency, and errors per request, with a simple usage page grouped by model and day.

## 7. Bot behavior features

### DM bot
- Auto-reply to incoming DMs with conversation history as context (last N messages, configurable).
- Modes per account: **Off**, **Approval** (AI drafts, owner approves in dashboard), **Auto**.
- Human takeover: owner can pause the bot for a single conversation and send manual messages from the dashboard; bot resumes only when re-enabled.
- Handoff keywords (for example "human", "agent") pause the bot for that conversation and flag it in the inbox.
- Per-user rate limit and a global cap on replies per hour.
- Ignore rules: blocked users and keyword blocklist.

### Comment bot
- Rules: trigger by keyword, any comment, or specific post; action is public reply, private DM reply, or both.
- Reply templates with AI generation or fixed text per rule.
- Skip the account's own comments, replies to replies (configurable), and spam matching the blocklist.
- Same modes as DM: Off, Approval, Auto.

### Safety
- **Global kill switch** that stops all outgoing messages instantly.
- Dry-run mode: runs the whole pipeline but sends nothing, logs what would have been sent.
- Duplicate protection: never reply twice to the same message or comment.

## 8. Dashboard (Next.js)

Single admin login: password from `ADMIN_PASSWORD` env, signed httpOnly session cookie. No auth library.

Pages:
- **Overview:** bot status, kill switch, today's message counts, error count, active model.
- **Inbox:** conversations list and thread view, take over, send manual message, approve or edit drafts.
- **Comments:** recent comments, status (replied, pending, skipped), approve drafts.
- **Rules:** create and edit comment rules and DM settings.
- **AI:** providers, models, active model selectors for DM, comments, and fallback, parameters, prompts, knowledge base.
- **Playground:** test any model with a prompt.
- **Usage:** tokens and requests by model and day.
- **Logs:** webhook events, jobs, errors, with filters.
- **Settings:** Instagram connection status, token expiry, webhook URL and verify token display, danger zone.

## 9. Data model (SQLite tables, indicative)

`accounts`, `providers`, `models`, `settings` (key/value including active models), `conversations`, `messages`, `comments`, `rules`, `jobs`, `events`, `ai_requests`.

The Worker proposes the exact schema in the plan; migrations are simple numbered `.sql` files applied on startup.

## 10. Configuration (`.env.example` must list all)

`APP_BASE_URL`, `ADMIN_PASSWORD`, `SESSION_SECRET`, `ENCRYPTION_KEY`, `DATABASE_PATH`, `META_APP_ID`, `META_APP_SECRET`, `WEBHOOK_VERIFY_TOKEN`, `AI_BASE_URL`, `AI_API_KEY`, `AI_DEFAULT_MODEL`.

## 11. Milestones

- **M0 — Scaffold:** Next.js app, SQLite setup with migrations, env handling, `AGENTS.md`, README skeleton, admin login.
- **M1 — AI layer:** providers, models, runtime model switching, test connection, Playground.
- **M2 — Instagram connect + webhook:** OAuth connect, webhook verify and signature check, event storage, job queue and worker.
- **M3 — DM bot:** auto-reply, modes, takeover, handoff, inbox page.
- **M4 — Comment bot:** rules, public and private replies, comments page.
- **M5 — Hardening:** rate limits, kill switch, dry-run, usage and logs pages, token refresh, README (setup, Meta app configuration, dev tunnel, deployment).

## 12. Manual prerequisites (owner does these, not the Worker)

- Instagram account switched to Business or Professional.
- Meta developer app created with the Instagram product added, the account added as a tester, and webhook callback configured.
- A public HTTPS URL for the webhook during development (for example a Cloudflare Tunnel or ngrok pointing at `localhost:3000`).

## 13. Out of scope (for now)

Multi-user dashboard roles, Facebook Messenger/WhatsApp, image or voice message understanding, vector search, billing, Docker or cloud deployment automation.
