<!-- BEGIN:nextjs-agent-rules -->

## This is NOT the Next.js you know

This version has breaking changes — APIs, conventions, and file structure may all differ from your training data. Read the relevant guide in `node_modules/next/dist/docs/` (resolved from this file's directory; in monorepos the `next` package may not be visible from the repo root) before writing any code. Heed deprecation notices.

This block is written and re-added by `next dev` — verify at `node_modules/next/dist/server/lib/generate-agent-files.js`. Removing it from a diff only re-creates the uncommitted change; committing it with your work keeps the tree clean.

<!-- END:nextjs-agent-rules -->

# AGENTS.md

## Stack
- Next.js (App Router) + TypeScript, Tailwind CSS
- SQLite via better-sqlite3, zod, openai (OpenAI-compatible client)
- Node.js 20.9+, npm

## Hard constraints (SPEC.md §2)
- No Docker, Redis, external queue, separate backend, or managed DB.
- One Next.js app does everything (dashboard, API, webhook, worker).
- Plain SQL via better-sqlite3; no ORM unless justified.
- Runs with `npm install`, `npm run dev`, `npm run build && npm start`.
- Long-lived Node process; serverless not supported.
- Dependencies added only when needed and justified.
- Secrets in `.env.local` only; never committed.
- No Instagram/AI calls from client-side code; all server-side.

## Commands
- dev: `npm run dev`
- build: `npm run build`
- lint: `npm run lint`

## Rules
- Read SPEC.md before starting any task.
- No new dependencies without justification.
- Never commit secrets.
