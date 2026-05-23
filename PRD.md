# Peshi — Product Requirements Document (v3)

> Case tracking & hearing prep for Indian litigators.

This is the product source of truth. Scope and intent live here; locked architectural choices live in [`DECISIONS.md`](./DECISIONS.md); implementation reality lives in [`docs/`](./docs/).

---

## Overview

### What Peshi is

A single dashboard for Indian litigators that auto-tracks every case across eCourts and the Supreme Court, prepares the lawyer for tomorrow's hearings with AI, and keeps clients informed over WhatsApp.

> **Pitch:** *Linear for litigators — auto-tracked cases, AI-prepped hearings, WhatsApp-kept clients.*

### Audience

| Field | Value |
|---|---|
| Primary user | Solo lawyer / 2-person chamber |
| Active caseload sweet spot | 30–100 cases |
| Courts covered | eCourts (district & subordinate) + Supreme Court |
| Litigant experience | WhatsApp only |

### The problem being solved

A litigator with 50 active cases today tracks them across a physical diary, WhatsApp threads with a clerk, and manual checks on multiple court websites. Cases get missed. Rescheduled dates are caught hours before a hearing or not at all. Clients call constantly for status. The lawyer arrives at court each morning having spent the night flipping through paper files trying to recall where a matter stands.

Peshi replaces that entire workflow with one auto-updating dashboard, an AI-generated prep brief the night before each hearing, and automatic client updates over WhatsApp.

### Core scope

| Item | Decision |
|---|---|
| Data sources | eCourts (via eCourtsIndia.com API) + SC direct scrape |
| Case identifier | 1 case = 1 CNR (eCourts) or 1 SLP/Civil Appeal (SC) |
| User roles in v1 | Single user (lawyer) |
| AI headline feature | Hearing prep brief, night before each hearing |
| Client channel | WhatsApp (Twilio sandbox) |
| Build window | 4–6 weeks for a demo-ready v1 |

### What makes Peshi different

Most legal tools in this space do case tracking *or* document AI *or* client communication. Peshi unifies all three around one event: **the next hearing**. Every feature is in service of "the lawyer walks into court tomorrow prepared, and the client already knows what's happening."

---

## Tech stack

All free-tier runnable. Three hosting services, no separate worker process. Detailed reasoning per choice lives in [`DECISIONS.md`](./DECISIONS.md).

### Stack by layer

| Layer | Choice |
|---|---|
| Frontend | Next.js + Tailwind + shadcn/ui |
| Database | Supabase (Postgres + Auth + Storage) |
| Jobs & cron | Inngest (serverless-native) |
| Scraping (SC portal only) | Playwright, runs inside Inngest jobs |
| eCourts data | eCourtsIndia.com API (HTTP, no Playwright) |
| AI — text | GPT-4o |
| AI — voice | Whisper |
| WhatsApp | Twilio (sandbox for v1) |
| Email | Resend |
| Testing | Vitest (unit) + Playwright (e2e) |
| Hosting | Vercel (web) + Supabase + Inngest |

### Hosting topology

```
Vercel        → Next.js app + API routes
Supabase      → Postgres + Auth + Storage (voice notes, PDFs)
Inngest       → Nightly refresh, SC scraper, AI pipelines
```

Three managed services. No standalone worker host. The "where does my Playwright actually run" puzzle is solved by Inngest's serverless execution model.

### Why these choices — quick reasons

| Choice | Reason |
|---|---|
| Next.js + shadcn/ui | SSR, PWA-ready, professional UI without design work |
| Supabase | Auth built-in saves weeks; Storage covers voice notes/PDFs; Postgres is universal |
| Inngest over BullMQ | No separate worker host, cleaner Vercel integration, free tier sufficient |
| Playwright (SC only) | SC portal needs a browser. eCourtsIndia is plain HTTP — no browser needed |
| GPT-4o | Free API access available; portfolio scope means model differences not material |
| Resend | Transactional email free tier (3000/month), required for email notification channel |

---

## Data sources

Two sources, two ingestion techniques. Demonstrates both API consumption and structured scraping without the maintenance burden of 25 high-court scrapers.

### eCourts — District & Subordinate Courts (API)

Accessed via `eCourtsIndia.com` — a paid API that scrapes eCourts and returns clean JSON. ₹200 free credits on signup; pay-per-lookup after. Free credits comfortably cover the build phase. Lawyer enters CNR number once (e.g. `DLST010234`).

| Field | Availability |
|---|---|
| Next hearing date | ✓ available |
| Case stage | ~ partial / inconsistent strings |
| Party & advocate names | ✓ available |
| Full hearing history | ✓ available |
| Order PDFs | ~ patchy by state |

> Paid API was chosen over free alternatives for reliability — free third-party scrapers go inactive without notice, which kills a portfolio demo. README documents production path: in-house Playwright + Tesseract scraper, or NJDG partnership for a real deployment.

### Supreme Court (direct scrape)

Scraped directly from the SC case status portal. No CAPTCHA in the common path. Lawyer enters SLP or Civil Appeal number.

| Field | Availability |
|---|---|
| Case status & next date | ✓ available |
| Hearing history | ✓ available |
| Order PDFs | ✓ widely available — best source for AI summaries |
| Bench details | ✓ available |

### Data refresh strategy

- Nightly Inngest cron refreshes all `active` cases
- New case added → immediate fetch
- Archived and closed cases are excluded from refresh
- If a fetch fails, the dashboard flags the case visibly — stale data is never shown silently

### Reschedule tracking

Courts reschedule matters frequently. When a previously fetched next-date changes, the system writes an append-only `HearingDateChange` log entry — every move is preserved, not just the latest.

> Example: `15 March → 2 April → 22 April`. A matter rescheduled five times tells the lawyer something important about the bench. Showing only the latest date loses that signal.

### Hearing type vs order type

Orders are not passed at every hearing. The platform marks each entry clearly:

- **Interlocutory order passed** → order — PDF linked
- **Final judgment** → order — PDF linked
- **Adjournment only** → no order — date entry only

---

## Dashboard

The first screen — and the one the lawyer returns to fifty times a day. Organised around how a lawyer actually thinks about their work: in time buckets, not as a flat list.

### Time-segmented buckets

| Bucket | Contents |
|---|---|
| **Pinned** | Always at top — max 5 hot matters |
| **Today** | Hearings happening today |
| **Tomorrow** | The prep window — brief is ready |
| **This week** | Next 6 days |
| **Later** | Everything beyond this week |
| **No date set** | Awaiting next fetch, or fetch failed |

Each card shows party name, court, stage, and a "prep brief ready" indicator. Empty buckets collapse — a lawyer with nothing tomorrow doesn't need a "Tomorrow: 0" header.

### Search

Top-bar search across everything that matters when looking for a specific case:

- Party name — e.g. "Sharma"
- Client name — links to client entity
- CNR / SLP number — direct match
- Transcribed voice notes — full-text
- Private hearing notes — full-text

> What this replaces: scrolling through a 50-row dashboard or asking the clerk *"yaar woh Sharma waala case kya tha CNR?"*

### Morning-of view

When the lawyer opens the app on a hearing day, the Today bucket expands into a court-by-court itinerary — sorted by courthouse first (so the lawyer isn't running across the building), then by time.

`Courthouse → Court room + judge → Time → Case + prep brief tap`

Each row shows court room number, judge name, scheduled time, parties, and a one-tap open for the prep brief. Designed to be readable in an elevator on the way up.

---

## Case management

How a lawyer gets 50 cases into the system in one minute, organises them, and keeps them current.

### Bulk import — the onboarding moment

Lawyer pastes a newline-separated list of CNRs or SLP numbers. System previews each row (matched / invalid), then fetches all matched cases in parallel. The dashboard populates within seconds.

> What this replaces: typing CNRs one by one from a physical diary — a 90-minute onboarding session that kills adoption before it starts.

### Client as a first-class entity

A client (litigant) has name, phone, optional email, and many cases. Cases belong to a client. The lawyer creates the client once; WhatsApp routing, message history, and case grouping all flow from this entity.

Common pattern: one client with a criminal case + a parallel civil suit + a writ — all linked to the same person, all messaged on the same number.

### Case status system

Three statuses, no ambiguity:

| Status | Meaning |
|---|---|
| 🟢 Active | Ongoing — fetched nightly — shown on dashboard |
| ⚪ Closed | Disposed by court — no more fetching — fully visible |
| ◽ Archived | Manually parked by lawyer — no fetching — off main dashboard |

### Pin a case

Pinned cases float to the top of the dashboard regardless of next date. Maximum 5 pins. For hot matters needing constant attention.

### Closing a case

Court data cannot tell you whether you won or lost. When the lawyer marks a case as Closed, a prompt asks:

- **Outcome:** Won / Lost / Settled / Withdrawn
- **One-line reason:** free text (optional)

The one-line reason turns the case list into something analyzable later — patterns like "I lose 60% of bail applications on this ground" only emerge if the reason field exists from day one.

### Calendar export

Each lawyer gets a personal subscription URL exposing upcoming hearings as an iCalendar feed. Works in Google Calendar, Apple Calendar, Outlook — no OAuth setup required.

### Scraper / API health panel

Settings page surfaces last-successful-fetch per source, failure rate over 7 days, retry queue depth, and current stale-case count.

### Court data is read-only

Fetched court data cannot be edited — it is the official record. Lawyers add context through private notes alongside any hearing entry.

---

## Hearings & logs

A chronological record of everything that happened in a case — auto-fetched skeleton, AI-generated layer, and lawyer-added context.

### What each hearing entry contains

| Field | Source |
|---|---|
| Hearing number (1st, 2nd, 3rd…) | auto |
| Date & stage | auto |
| Official order text | auto |
| Order PDF link | auto |
| Reschedule flag (if date changed) | auto |
| AI summary of order PDF | AI |
| Cited statutes & case law | AI |
| Private hearing note (text or voice) | manual |
| Action items | manual / AI-extracted |

### Voice notes — two attachment levels

| Level | Purpose |
|---|---|
| Hearing-level | What happened in court today — attached to a specific hearing date |
| Case-level | Strategy, background, client instructions — attached to the case itself |

Both are playable from the app — useful to back-track a case in seconds without reading pages of text.

### Full reschedule timeline

Each hearing shows its complete date-change history, not just the latest move. Implemented as an append-only log so nothing is lost.

---

## AI features

Four practical AI integrations. One headline feature recruiters will remember; three supporting ones that make daily use feel effortless.

### Hearing prep brief — headline feature

The night before each hearing, AI reads the last three hearings on this case, the latest order PDF, and the lawyer's private notes. It generates a one-page brief: where the matter currently stands, what is likely to come up tomorrow, points the lawyer should be ready to argue, documents to carry.

`Last 3 hearings + latest order + private notes → 1-page brief`

> What this replaces: the lawyer sitting at the desk at 11pm flipping through a paper file trying to remember what was argued last time, what the judge had asked, and what was due today.

### Voice note → structured hearing log (Whisper)

Lawyer records a 2–4 minute voice note after a hearing, in any language. Whisper transcribes (~5–10 sec). A second pass structures the transcript into a clean log: stage, what was discussed, next date, action items. Lawyer can edit before saving.

Cost: ~$0.006/min. Negligible for portfolio scale.

### Order PDF summarisation + citation extraction

Platform downloads SC order PDFs automatically, passes to AI, generates a 3–4 line plain-language summary attached to that hearing entry. In the same pass, AI extracts cited statutes (e.g. `CrPC §482`) and prior case law as structured tags — searchable, filterable.

Most valuable for Supreme Court orders that run several paragraphs. District court orders are too thin for summarisation to add value.

---

## Notifications

Sensible defaults the lawyer never has to configure, with a single override for the rare exception. Designed so notifications never become noise.

### Hearing-type defaults

| Stage | Notification timing |
|---|---|
| Arguments | 3 days before — heavy prep |
| For orders | 1 day before — minimal prep |
| Filing / compliance | 2 days before — document needed |
| Evidence / examination | 1 week before — witness prep |
| Unknown stage string | 1 day before — generic fallback |

Stage strings from eCourts are inconsistent — "Arguments", "For arguments", "Final arguments", "Part-arguments" all appear. Peshi maintains a stage-string → category map, with anything unmatched falling back to the generic 1-day default.

### Per-case override

Single override per case: "remind me X hours/days before, regardless of stage." Used for the one or two hot matters where the default isn't enough.

### Trigger-based notifications

- Prep brief ready (night before hearing) — headline trigger
- New order PDF available
- Hearing date rescheduled — immediate alert
- Data fetch failed — stale data warning
- Today had a hearing — no note added yet — gentle nudge

### Delivery channels

In-app + email. The prep brief is delivered as both an in-app card and a long-form email so the lawyer can read it on the phone in bed.

---

## Litigant experience — WhatsApp only

No app for litigants. The lawyer controls exactly what gets sent and when. Simpler to build, more useful to receive.

### Two delivery modes

| Mode | Used for |
|---|---|
| Auto-send (low risk) | Date reminders, reschedule alerts |
| Draft → lawyer approves → send | Hearing summaries, new orders |
| Custom message from lawyer | Manual send, any time |

Raw court order text reaching a client unfiltered is a liability footgun — a litigant reads "petition dismissed for non-prosecution" and panics before the lawyer can explain it's procedural. Peshi drafts the message, the lawyer one-taps approve, then it sends.

### Sharing settings

Default pool set once — applies to every client. Per-client overrides for sensitive matters where the lawyer wants tighter control.

### Twilio sandbox for v1

No Meta business verification, no template pre-approval, no per-message billing during development. The demo phone joins by sending a code to Twilio's shared sandbox number. 100 free messages on a Twilio trial — enough for the entire build phase and demo.

Production path documented in README: move to a verified WABA + BSP once a real user signs on.

### Why this beats a litigant app

Litigants don't want another app. A WhatsApp message saying *"Your next hearing is 15 March 2024 — Arguments, Delhi District Court"* is more useful than an app icon they'd never open. Simpler to build, better to receive.

---

## The 90-second demo flow

Peshi v1 is built around exactly this flow. Anything not visible in 90 seconds is a v2 candidate.

```
0:00 — Paste 20 CNRs            → Dashboard populates
0:20 — Open a case              → See full timeline + AI-summarised SC order
0:45 — View tomorrow's prep brief → AI-generated, ready to read
1:05 — Record voice note        → Whisper transcribes → saves as structured hearing log
1:20 — Approve client WhatsApp  → Message lands on demo phone
1:30 — End
```

### README essentials

| Item | Status |
|---|---|
| One-line pitch in first paragraph | required |
| 90-second demo video (Loom / YouTube unlisted) | required |
| Live deployed URL with seeded demo data | required |
| Architecture diagram (one image) | required |
| "What was hard, how I solved it" | required |
| "What I chose not to build and why" (future scope) | required |

---

## Future scope

Consciously deferred. Listed here to show product thinking without carrying implementation cost. Each one is a real feature with a real reason it's not in v1.

| Feature | Status | Reason |
|---|---|---|
| High Court scraping | v2+ | 25 different scrapers — same skill repeated, no portfolio gain |
| Clerk role + daily digest | v2 | Real win for chambers, but adds RBAC complexity that doesn't pay off in the demo video |
| Junior advocate role | v2 | Same RBAC reasoning |
| Cause-list "not listed" detection | v2 | Highest-value notification, also hardest — multi-week sub-project on its own |
| Personal document vault | v2 | Storage/preview/scanning plumbing doesn't earn portfolio signal |
| Full offline mode | v2 | v1 ships PWA cache of today's hearing data only. Full offline = sync/conflict project |
| Analytics dashboard | v2 | Win/loss data is captured from v1 day one — UI deferred until there's enough data to visualise |
| Billing & payments | v2+ | Razorpay + GST = separate product |
| Conflict check | v2+ | Fuzzy name matching across cases — complex for low portfolio return |
| Time tracking | v2+ | Valid feature, doesn't showcase core engineering challenge |
| Litigant app | **dropped, never** | Replaced by WhatsApp. Litigants don't want another app. |
