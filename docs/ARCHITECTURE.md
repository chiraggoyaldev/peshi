# Architecture

How Peshi fits together as a system. High-level flow, module boundaries, and external integrations.

> Detailed scope and features live in [`../PRD.md`](../PRD.md). The "why" behind any architectural choice lives in [`../DECISIONS.md`](../DECISIONS.md).

---

## High-level diagram

_Mermaid diagram to be added — modules and data flow._

```
[Browser]
   │
   ▼
[Vercel: Next.js app + API routes]
   │
   ├─── reads/writes ───▶ [Supabase: Postgres + Auth + Storage]
   │
   └─── triggers / enqueues ──▶ [Inngest: jobs + cron]
                                    │
                                    ├─▶ eCourtsIndia.com API (HTTPS)
                                    ├─▶ SC portal via Playwright
                                    ├─▶ GPT-4o + Whisper (OpenAI)
                                    ├─▶ Twilio (WhatsApp sandbox)
                                    └─▶ Resend (email)
```

## Module boundaries

| Module | Responsibility | Notes |
|---|---|---|
| `app/` | Next.js routes — pages, API handlers | Server components by default |
| `lib/ecourts/` | eCourtsIndia.com client | Single retry/cache layer. Feature code never calls eCourtsIndia directly. |
| `lib/sc/` | SC portal scraper (Playwright) | Runs only inside Inngest jobs, never in API routes. |
| `lib/ai/` | GPT-4o + Whisper wrappers | Structured output via JSON mode. |
| `lib/whatsapp/` | Twilio client | Sandbox mode for v1. |
| `lib/email/` | Resend client | Transactional only. |
| `lib/notifications/` | Trigger logic + channel dispatch | Hearing-type classification lives here. |
| `inngest/` | Job definitions | Nightly refresh, prep brief generator, reschedule detector. |
| `db/` | Schema + migrations | Schema file is the source of truth for tables/columns. |

## Data flow walkthroughs

### Adding a CNR (single case)

1. Lawyer pastes CNR into the add-case form (`app/cases/new`)
2. API route validates format → inserts a `cases` row with status=`pending_fetch`
3. API route triggers an Inngest event `case.added`
4. Inngest job `fetchEcourts` runs: calls `lib/ecourts/client.ts:fetchByCnr()`
5. On success: rows inserted into `hearings`, `orders`, `parties`; `cases.status` → `active`
6. On failure: `cases.last_fetch_error` set; dashboard shows stale flag

### Bulk import

Same as above, but the API route enqueues N Inngest events in parallel. Inngest's concurrency controls cap parallel external API calls.

### Nightly refresh

Inngest cron fires at a configured time (TBD). For each `active` case, enqueues a `case.refresh` event. Workers fetch, diff against stored data, and write `HearingDateChange` entries on reschedule.

### Prep brief generation

Inngest cron fires at 20:00 IST. For each case with a hearing the next day:
1. Pulls last 3 hearings + latest order PDF + private notes
2. Calls GPT-4o with a structured prompt
3. Stores the brief in `prep_briefs` table
4. Sends email via Resend to the lawyer

### WhatsApp message dispatch

Two paths:
- **Auto-send** (date reminders, reschedule alerts) — Inngest job calls Twilio directly
- **Draft + approve** (hearing summaries, new orders) — Inngest job generates draft, surfaces in app, lawyer taps approve → Twilio call fires

## External integrations

### eCourtsIndia.com

- **What we use it for:** District-court case data by CNR
- **Auth:** API key in env (`ECOURTS_API_KEY`)
- **Rate limit:** Pay-per-lookup. Free credits cover build phase.
- **Failure mode:** Returns errors / times out. Cache last successful response, flag the case as stale on the dashboard.
- **Fallback if it dies:** Documented in `DECISIONS.md` — in-house Playwright scraper.

### SC portal (direct scrape via Playwright)

- **What we use it for:** Supreme Court case data by SLP / Civil Appeal number
- **Auth:** None (public portal)
- **Rate limit:** Self-imposed, ~1 req/sec to avoid being blocked
- **Failure mode:** Page structure changes break the scraper. Flag case as stale, log to `scraper_errors`.
- **Where it runs:** Inside Inngest functions only — Playwright in Next.js API routes hits Vercel timeout limits.

### OpenAI (GPT-4o + Whisper)

- **What we use it for:** Prep brief generation, order PDF summarisation, citation extraction (GPT-4o); voice note transcription (Whisper)
- **Auth:** `OPENAI_API_KEY`
- **Failure mode:** Rate limits → Inngest auto-retries with backoff. Hard failures logged, lawyer sees "AI brief unavailable for this hearing" but app continues.

### Twilio (WhatsApp sandbox)

- **What we use it for:** WhatsApp messages to litigants
- **Auth:** `TWILIO_ACCOUNT_SID`, `TWILIO_AUTH_TOKEN`, `TWILIO_WHATSAPP_FROM`
- **Sandbox limit:** Only sandbox-joined numbers can receive. Acceptable for portfolio demo (only demo number needs to receive).
- **Failure mode:** Delivery failure logged. App marks message as "send failed" so lawyer can retry.

### Resend (email)

- **What we use it for:** Prep brief email, fetch-failed alerts, daily summaries
- **Auth:** `RESEND_API_KEY`
- **Free tier:** 3000 emails/month — comfortable for portfolio scale.

### Inngest

- **What we use it for:** Cron + event-driven jobs + retries + long-running Playwright execution
- **Auth:** `INNGEST_EVENT_KEY`, `INNGEST_SIGNING_KEY`
- **Failure mode:** Inngest itself rarely fails; if it does, jobs queue up and resume.

### Supabase

- **What we use it for:** Postgres database, authentication, file storage (voice notes + downloaded order PDFs)
- **Auth:** Service role key on the server, anon key in browser. Row-level security on all tables.

---

## Failure isolation

| If X dies | What breaks | What still works |
|---|---|---|
| eCourtsIndia.com | District court refreshes fail | SC cases, all cached data, AI, WhatsApp |
| SC portal | SC scrapes fail | District court, all cached data, AI, WhatsApp |
| OpenAI | New prep briefs and summaries fail | All data, all reads, notifications |
| Twilio | WhatsApp messages fail | Everything else — lawyer can copy/paste manually |
| Resend | Email notifications fail | In-app notifications continue |
| Inngest | All jobs / cron stop | Reads work; writes work but no auto-refresh or AI |
| Supabase | Whole app down | Nothing — Supabase is the data plane |

Stale data is always flagged visibly. Never displayed as fresh.
