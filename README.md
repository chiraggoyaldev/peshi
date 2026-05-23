# Peshi

> Linear for litigators — auto-tracked cases, AI-prepped hearings, WhatsApp-kept clients.

Peshi is a unified case tracker for Indian litigators. Lawyers add a CNR (district court) or SLP/Civil Appeal (Supreme Court) once; Peshi auto-fetches all hearing data, generates an AI prep brief the night before each hearing, and keeps clients informed over WhatsApp.

## Status

In active development. Portfolio project — not for production legal use.

## Demo

- **Live URL:** _coming soon_
- **Demo video (90 sec):** _coming soon_

## Tech stack

Next.js · Supabase (Postgres + Auth + Storage) · Inngest · Playwright · GPT-4o · Whisper · Twilio · Resend · Vercel

See [`PRD.md`](./PRD.md) for the full product spec, including the demo flow v1 is built around.

## Documentation

| File | Purpose |
|---|---|
| [`PRD.md`](./PRD.md) | Product intent and scope |
| [`DECISIONS.md`](./DECISIONS.md) | Architecture decision records (the thinking archive) |
| [`CLAUDE.md`](./CLAUDE.md) | AI/dev entry point — read first on every new session |
| [`docs/ARCHITECTURE.md`](./docs/ARCHITECTURE.md) | Logical flow, modules, external integrations |
| [`docs/FEATURES.md`](./docs/FEATURES.md) | Per-feature reality — what, where, why |
| [`docs/API.md`](./docs/API.md) | Endpoints reference |
| [`docs/CONVENTIONS.md`](./docs/CONVENTIONS.md) | Coding rules and patterns |

Bugs, TODOs, and progress live in GitHub Issues — not in markdown.

## What was hard, how I solved it

_To be filled as the project progresses. Highlights expected: stage-string classification across inconsistent eCourts data, append-only reschedule history without losing current-date UX, fitting prep brief context into the LLM window for long cases, Playwright execution inside Inngest serverless jobs._

## License

To be added.
