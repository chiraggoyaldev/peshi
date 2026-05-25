# Architecture Decision Records

Append-only thinking archive. Each entry captures the context, alternatives considered, the decision, and consequences accepted. Treat every entry here as final unless explicitly revisited.

Never edit a closed ADR. To change direction, append a new ADR that supersedes the earlier one.

---

## 2026-05-24 — eCourtsIndia.com over kleopatra for eCourts data

**Status:** Accepted

**Context:**
Need a way to fetch eCourts district-court case data by CNR. Three options considered:

- `court-api.kleopatra.io` — free third-party API, scrapes eCourts. Tested and appears inactive/unresponsive at decision time.
- `eCourtsIndia.com` — paid third-party API. ₹200 free credits on signup, pay-per-lookup after.
- In-house Playwright + Tesseract CAPTCHA scraper against `services.ecourts.gov.in` — fully self-hosted but multi-week build.

Constraints: 4–6 week portfolio build, ₹500 cost cap, demo must work reliably for recruiters.

**Decision:** Use `eCourtsIndia.com`.

**Reason:**
Demo reliability outweighs cost at portfolio scale. Free credits cover the full build phase. An in-house scraper is weeks of engineering effort that wouldn't appear in the demo video. A paid third-party dependency is acceptable risk for a portfolio project.

**Consequences:**
- Hard dependency on a third party (acceptable for portfolio, not production)
- README documents the production path: in-house scraper or NJDG partnership
- If eCourtsIndia.com also goes inactive, fallback work is unscoped — would require an emergency in-house scraper sprint

**Alternatives explicitly rejected:**
- kleopatra: tested, unresponsive
- In-house scraper: out of time budget for v1

---

## 2026-05-24 — Inngest over BullMQ + Upstash for jobs and cron

**Status:** Accepted

**Context:**
Peshi needs nightly cron (refresh all active cases), event-driven jobs (run prep brief generator the evening before each hearing), and a queue for AI calls. Two paths:

- **BullMQ + Upstash Redis** — Node-native job queue, requires a long-running worker process. Workers can't run on Vercel serverless — need a separate worker host (Railway, Fly.io, or Render).
- **Inngest** — serverless-native job platform with cron + event-driven jobs, designed for Next.js + Vercel. Generous free tier. No separate worker host.

**Decision:** Use Inngest.

**Reason:**
Eliminates the "where does my worker run" problem entirely. Vercel hosts the web layer, Inngest hosts the job layer, no fourth service to manage. Free tier is more than sufficient for portfolio scale. Cleaner integration with Next.js API routes.

**Consequences:**
- Vendor lock-in to Inngest's job-definition syntax — acceptable for portfolio scope
- Lose the "I wired up a Redis-backed queue" portfolio signal of BullMQ — judged not worth the operational overhead
- Playwright scraping for SC portal runs inside Inngest functions (long-running serverless execution model handles this)

**Alternatives explicitly rejected:**
- BullMQ + Upstash + Railway worker — works, but four services to manage and a worker host that may go to sleep on free tier
- Trigger.dev — viable alternative, Inngest picked for slightly stronger Vercel-native ergonomics

---

## 2026-05-24 — GPT-4o for AI features

**Status:** Accepted

**Context:**
Peshi has four AI features: hearing prep brief generation, voice note → structured log, order PDF summarisation, citation extraction. Choice between providers:

- **GPT-4o** (OpenAI) — strong on structured output, free API access available to this project.
- **Claude (Sonnet/Opus)** (Anthropic) — strong on legal-text reasoning and long context.
- **Whisper** (OpenAI) — for voice transcription, no real alternative at portfolio cost.

**Decision:** GPT-4o for text features. Whisper for voice transcription. No model abstraction layer in v1.

**Reason:**
Free API access available for GPT-4o seals the choice on cost. At portfolio scope with no real users, model differences are not material — both providers produce demo-quality output for the features described. Whisper is the standard for transcription and is essentially free at this scale.

**Consequences:**
- If real users came on, the choice would be re-evaluated — Claude is a reasonable contender for legal-text reasoning
- No model abstraction = small swap cost if we ever switch providers; acceptable for portfolio scale
- All AI calls route through Inngest jobs for retry/rate-limit handling

**Alternatives explicitly rejected:**
- Claude — viable but not chosen due to GPT-4o free access
- Model abstraction layer (provider-agnostic interface) — over-engineering for v1 portfolio scope

---

## 2026-05-25 — date-fns + date-fns-tz for date handling

**Status:** Accepted

**Context:**
Peshi is India-specific — all lawyers and hearings are in IST. The question was whether a timezone library is needed at all, or whether plain `Date` arithmetic is sufficient.

**Decision:** Use `date-fns` + `date-fns-tz`.

**Reason:**
Vercel servers run in UTC (US/EU data centres). Without explicit IST conversion, `new Date()` on the server produces a UTC timestamp. Dashboard bucket boundaries ("today", "tomorrow", "this week") and Inngest cron schedules expressed in IST will silently miscalculate — e.g. a 20:00 IST prep brief cron fires at 14:30 UTC, and "tomorrow's hearings" resolves to the wrong day for a lawyer checking at 11pm IST. `date-fns-tz` adds IST-aware `zonedTimeToUtc` / `utcToZonedTime` with no mutable global state. `date-fns` is tree-shakeable and has no side-effects. Together they cover all cases.

**Consequences:**
- All bucket-boundary calculations and cron expressions must use IST explicitly — `Asia/Kolkata`
- Raw `Date` arithmetic is banned in feature code (rule already in `CONVENTIONS.md`)

**Alternatives explicitly rejected:**
- `dayjs` + `dayjs/plugin/timezone` — viable, terser API, but mutable plugin model and less tree-shaking
- Plain `Date` + UTC offsets hardcoded — brittle, breaks on DST edge cases (India has no DST but UTC offset arithmetic is still error-prone)
