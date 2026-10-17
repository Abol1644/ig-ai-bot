# ig-ai-bot

**A self-hosted AI assistant for Instagram DMs and comments, with a Next.js control dashboard.**

Connect an Instagram Business account, pick any OpenAI-compatible model, and let the bot reply to messages and comments while you stay in control: approve drafts, take over any conversation, switch models on the fly, or hit the kill switch.

![Status](https://img.shields.io/badge/status-in%20development-orange)
![Next.js](https://img.shields.io/badge/Next.js-App%20Router-black)
![TypeScript](https://img.shields.io/badge/TypeScript-strict-3178c6)
![SQLite](https://img.shields.io/badge/database-SQLite-003b57)
![License](https://img.shields.io/badge/license-MIT-green)

> **Status:** early development. The architecture and feature set below are the target design; see the [Roadmap](#roadmap) for what is built so far.

---

## Features

**AI layer**
- Works with any OpenAI-compatible API (hosted providers, gateways, or local servers)
- Add custom providers and models from the dashboard, no code changes
- Switch the active model at runtime, separately for DM replies, comment replies, and a fallback
- Automatic fallback model on errors or timeouts
- Per-provider connection test and a Playground to try any model
- Editable system prompts and a plain-text knowledge base (FAQ, pricing, hours)
- Token, latency, and error tracking per model

**DM bot**
- AI replies with conversation history as context
- Three modes: Off, Approval (you approve drafts), Auto
- Human takeover per conversation, plus handoff keywords
- Respects Instagram's messaging window and never replies to itself

**Comment bot**
- Rules by keyword, any comment, or specific post
- Public replies, private DM replies, or both
- Skips your own comments and spam matching your blocklist

**Control and safety**
- Global kill switch and dry-run mode
- Duplicate-reply protection and per-user and global rate limits
- Full event, job, and AI request logs

**Dashboard**
- Overview, Inbox, Comments, Rules, AI settings, Playground, Usage, Logs, Settings
- Single admin login with a signed session cookie

## Design principles

- **Minimal stack.** One Next.js app, one SQLite file. No Docker, no Redis, no external queue.
- **Self-hosted.** Your tokens, messages, and API keys stay on your machine.
- **Human in the loop.** Approval mode, takeover, and a kill switch are first-class features.

## Architecture

```
Instagram ──webhook──▶ /api/webhooks/instagram
                          │  verify signature, store event, enqueue job
                          ▼
                      SQLite (events, jobs, messages, settings)
                          ▲
                          │  claim job, load context
                   In-process worker ──▶ OpenAI-compatible provider
                          │
                          ▼
                 Instagram API (send DM / reply to comment)

Next.js dashboard ──▶ API routes ──▶ SQLite
```

The webhook handler responds immediately and hands the real work to an in-process worker with retries and exponential backoff. This keeps responses fast and avoids a separate queue service.

## Tech stack

| Area | Choice |
| --- | --- |
| Framework | Next.js (App Router) + TypeScript |
| Styling | Tailwind CSS |
| Database | SQLite via `better-sqlite3` |
| Validation | `zod` |
| AI client | `openai` package with a configurable `baseURL` |
| Background jobs | In-process worker started from `instrumentation.ts` |

## Requirements

- Node.js 20.9 or newer
- An Instagram **Business or Professional** account
- A Meta developer app with the Instagram product enabled
- An OpenAI-compatible API endpoint and key
- A public HTTPS URL for webhooks (a tunnel such as Cloudflare Tunnel or ngrok works for development)
- A long-running host for production (your own machine or a VPS). Serverless platforms are not supported because the worker needs a persistent process.

## Getting started

```bash
git clone https://github.com/<your-username>/ig-ai-bot.git
cd ig-ai-bot
npm install
cp .env.example .env.local   # on Windows PowerShell: Copy-Item .env.example .env.local
```

Fill in `.env.local` (see [Configuration](#configuration)), then:

```bash
npm run dev
```

Open <http://localhost:3000> and sign in with your `ADMIN_PASSWORD`.

### Production

```bash
npm run build
npm start
```

Run it behind a process manager of your choice so it restarts on failure.

## Configuration

All settings live in `.env.local`. Never commit this file.

| Variable | Description |
| --- | --- |
| `APP_BASE_URL` | Public base URL of the app (used for OAuth and webhook URLs) |
| `ADMIN_PASSWORD` | Password for the dashboard |
| `SESSION_SECRET` | Long random string used to sign session cookies |
| `ENCRYPTION_KEY` | Key used to encrypt stored tokens and API keys |
| `DATABASE_PATH` | Path to the SQLite file |
| `META_APP_ID` | Meta app ID |
| `META_APP_SECRET` | Meta app secret (also used to verify webhook signatures) |
| `WEBHOOK_VERIFY_TOKEN` | Token you enter in Meta's webhook setup |
| `AI_BASE_URL` | Base URL of your first OpenAI-compatible provider |
| `AI_API_KEY` | API key for that provider |
| `AI_DEFAULT_MODEL` | Model ID to use initially |

Additional providers and models are managed in the dashboard after first launch.

## Connecting Instagram

1. Switch your Instagram account to a Business or Professional account.
2. Create an app in the [Meta for Developers](https://developers.facebook.com/) console and add the Instagram product.
3. Add your Instagram account as a tester while the app is in development mode.
4. Set the webhook callback URL to `https://<your-domain>/api/webhooks/instagram` and use your `WEBHOOK_VERIFY_TOKEN`.
5. Subscribe to the message and comment webhook fields.
6. In the dashboard, open **Settings** and connect your account.

> In development mode the bot only works for accounts with a role on your Meta app. Public use requires Meta's App Review.

## Roadmap

- [x] Project specification
- [ ] **M0** Scaffold: Next.js app, SQLite, env handling, admin login
- [ ] **M1** AI layer: providers, models, runtime switching, Playground
- [ ] **M2** Instagram connect, webhook receiver, job worker
- [ ] **M3** DM bot, inbox, takeover
- [ ] **M4** Comment bot and rules
- [ ] **M5** Hardening: rate limits, kill switch, dry-run, usage and logs, token refresh

The full design is in [`SPEC.md`](./SPEC.md).

## Security

- Secrets are loaded from environment variables and never exposed to the browser.
- Instagram tokens and provider API keys are encrypted at rest.
- Webhook payloads are verified with the `X-Hub-Signature-256` HMAC before processing.
- Set a strong `ADMIN_PASSWORD` and serve the dashboard over HTTPS.

Found a vulnerability? Please open a private security advisory on GitHub rather than a public issue.

## Responsible use

This project automates messaging on your behalf. You are responsible for complying with the [Meta Platform Terms](https://developers.facebook.com/terms/) and Instagram's policies, local laws, and your audience's expectations. Offer people a way to reach a human, do not send unsolicited messages, and be transparent that replies may be automated.

## Contributing

Issues and pull requests are welcome. For larger changes, please open an issue first to discuss the approach. Keep the dependency footprint small.

## License

Released under the [MIT License](./LICENSE).

## Disclaimer

This project is not affiliated with, endorsed by, or sponsored by Meta Platforms, Inc. or Instagram. Instagram and Meta are trademarks of their respective owners.
