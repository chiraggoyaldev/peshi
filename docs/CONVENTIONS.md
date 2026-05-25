# Conventions

Active rulebook. Code in this project follows these patterns. AI assistants reading this should apply rules exactly — do not propose alternatives unless explicitly told to revisit.

> The *why* behind each rule lives in [`../DECISIONS.md`](../DECISIONS.md). This file is the *what*.

Append a new rule the first time AI drift is caught. Never silently rewrite an existing rule — discuss first.

---

## File naming

**Rule:** kebab-case for files and directories. `PascalCase` only for React component files.

- ✓ Good: `lib/ecourts/client.ts`, `components/CaseCard.tsx`
- ✗ Bad: `lib/ecourts/Client.ts`, `components/case_card.tsx`

---

## Court data fetching

**Rule:** All eCourts fetches go through `lib/ecourts/client.ts`. Never call eCourtsIndia.com URLs directly from feature code.

**Why:** Single retry/backoff/cache layer. Easy to swap API if needed. Centralised error handling.

```ts
// Good
import { fetchByCnr } from "@/lib/ecourts/client";
const caseData = await fetchByCnr(cnr);

// Bad
const res = await fetch(`https://api.ecourtsindia.com/cnr/${cnr}`);
```

**Linked decision:** `DECISIONS.md` → "eCourtsIndia.com over kleopatra"

---

## Background jobs

**Rule:** All background work runs through Inngest. Never spawn workers, never use `setTimeout` for delayed work, never call long-running operations from API routes directly.

**Why:** Vercel API routes time out at 60 seconds. Inngest handles retries, concurrency, and long-running execution.

```ts
// Good
await inngest.send({ name: "case.added", data: { caseId } });

// Bad
setTimeout(() => fetchEcourts(caseId), 0);
```

**Linked decision:** `DECISIONS.md` → "Inngest over BullMQ"

---

## Court data is read-only

**Rule:** Never mutate fields populated from court fetches. Lawyer-added context (notes, voice notes, action items) lives in separate tables/columns.

**Why:** Court data is the official record. Editing it creates liability and confuses junior advocates / clerks.

---

## Reschedule history is append-only

**Rule:** When a hearing's next-date changes, insert a new `HearingDateChange` row. Never overwrite the previous one.

**Why:** A matter rescheduled five times tells the lawyer something about the bench. Latest-date-only loses that signal.

---

## Date handling

**Rule:** Use `date-fns` + `date-fns-tz`. Never use raw `Date` arithmetic for hearing scheduling or bucket boundaries. All bucket boundaries and cron expressions must use `Asia/Kolkata` explicitly.

**Why:** Vercel runs in UTC. Without explicit IST conversion, "today" on the server is a different day than "today" for a lawyer in Delhi at 11pm. See `DECISIONS.md` → "date-fns + date-fns-tz".

---

## Environment variables

**Rule:** Never read `process.env` from feature code. Wrap all env access in `lib/env.ts` with schema validation (e.g. via `zod`).

**Why:** Missing env vars fail loudly at boot, not silently at runtime in production.

```ts
// Good
import { env } from "@/lib/env";
const client = new TwilioClient(env.TWILIO_AUTH_TOKEN);

// Bad
const client = new TwilioClient(process.env.TWILIO_AUTH_TOKEN!);
```

---

## API response shape

**Rule:** All API routes return `{ data: ..., meta?: ... }` on success, `{ error: { code, message } }` on failure. Never throw bare strings or return mixed shapes.

**Why:** Predictable contract for the client; clean error handling pattern.

---

## Supabase access

**Rule:** Server-side queries use the service-role client from `lib/supabase/server.ts`. Browser queries use the anon client from `lib/supabase/browser.ts`. Never mix them.

**Why:** RLS bypass needs to be explicit. Mixing risks accidentally bypassing row-level security in the browser.

---

## AI calls

**Rule:** All GPT-4o and Whisper calls go through `lib/ai/`. Use JSON mode with explicit schemas for structured output. Never parse free-form LLM text in feature code.

**Why:** Centralised retry/timeout. Structured output is contract-checkable; freeform isn't.

---

## Component patterns

**Rule:** React Server Components by default. Client components only when interactivity is needed. Mark explicitly with `"use client"` at the top.

**Why:** RSC is cheaper to render and easier to reason about.

---

## Test patterns

**Rule:** Vitest for unit, Playwright for e2e. Test the hard logic (eCourts client mocking, reschedule detection, stage classification, prep brief assembly). Don't test trivial CRUD or UI happy paths.

**Why:** Aim for tests that catch the bugs you'd actually ship, not coverage percentage.

---

## Logging

**Rule:** Use a single logger (TBD: pino or console for v1). Never `console.log` from feature code in production paths.

**Why:** Searchable, level-controllable, doesn't break when stdout is captured.

---

## Naming patterns for jobs

**Rule:** Inngest events use dot-namespaced names: `case.added`, `case.refresh`, `hearing.prep_brief_due`, `whatsapp.send_requested`. Past-tense for events, imperative for commands.

**Why:** Consistency makes the event log self-documenting.
