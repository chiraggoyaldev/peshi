# Features

Per-feature reality. What it does, where the code lives, edge cases, why this approach. Written when a feature ships — never before.

> Feature scope and intent live in [`../PRD.md`](../PRD.md). This file documents the *as-built* implementation.

Each section is filled in when the corresponding GitHub Issue is closed.

---

## Authentication

**Status:** Not built yet
**Files:** _TBD_
**Edge cases:** _TBD_
**Why this approach:** _TBD_

## Add a case (single CNR / SLP)

**Status:** Not built yet
**Files:** _TBD_
**Edge cases:** Invalid CNR format, eCourtsIndia timeout, partial data returned
**Why this approach:** _TBD_

## Bulk import (paste many CNRs)

**Status:** Not built yet
**Files:** _TBD_
**Edge cases:** Mixed valid/invalid CNRs in paste, deduplication against existing cases
**Why this approach:** _TBD_

## Client as a first-class entity

**Status:** Not built yet
**Files:** _TBD_
**Edge cases:** One client → many cases. Phone number format normalisation for WhatsApp routing.
**Why this approach:** _TBD_

## Dashboard with time-segmented buckets

**Status:** Not built yet
**Files:** _TBD_
**Edge cases:** Empty buckets collapse. Pinned cases always show regardless of bucket. Timezone handling for "today" boundary.
**Why this approach:** _TBD_

## Search (party name, client, CNR, transcribed notes)

**Status:** Not built yet
**Files:** _TBD_
**Edge cases:** Full-text search across transcribed voice notes requires Postgres `tsvector`.
**Why this approach:** _TBD_

## Morning-of view

**Status:** Not built yet
**Files:** _TBD_
**Edge cases:** Multiple courts on same day → sort by courthouse first, then time.
**Why this approach:** _TBD_

## Pin a case (max 5)

**Status:** Not built yet
**Files:** _TBD_
**Edge cases:** Attempting to pin a 6th → user must unpin one first.
**Why this approach:** _TBD_

## Case statuses (Active / Closed / Archived)

**Status:** Not built yet
**Files:** _TBD_
**Edge cases:** Archived and Closed cases excluded from nightly refresh.
**Why this approach:** _TBD_

## Win/loss marking on close

**Status:** Not built yet
**Files:** _TBD_
**Edge cases:** Prompt with four outcomes + one-line reason. Reason field optional but encouraged.
**Why this approach:** _TBD_

## Hearings timeline

**Status:** Not built yet
**Files:** _TBD_
**Edge cases:** Hearing-number auto-calculation; reschedule events shown inline.
**Why this approach:** _TBD_

## Voice notes (hearing-level + case-level)

**Status:** Not built yet
**Files:** _TBD_
**Edge cases:** Whisper transcription quality varies by language and audio quality.
**Why this approach:** _TBD_

## Reschedule detection (append-only log)

**Status:** Not built yet
**Files:** _TBD_
**Edge cases:** Diff fetched next-date against stored next-date; write `HearingDateChange` row on change.
**Why this approach:** Append-only preserves full reschedule history. See `DECISIONS.md`.

## Prep brief generation (AI headline feature)

**Status:** Not built yet
**Files:** _TBD_
**Edge cases:** Long-history cases exceed LLM context window — truncation strategy needed.
**Why this approach:** _TBD_

## Order PDF summarisation + citation extraction

**Status:** Not built yet
**Files:** _TBD_
**Edge cases:** SC order PDFs only — district court orders too thin for summarisation.
**Why this approach:** _TBD_

## Notifications (hearing-type defaults + per-case override)

**Status:** Not built yet
**Files:** _TBD_
**Edge cases:** Stage strings from eCourts are inconsistent — fallback to generic 1-day default when unknown.
**Why this approach:** _TBD_

## WhatsApp (auto-send + draft-and-approve)

**Status:** Not built yet
**Files:** _TBD_
**Edge cases:** Auto-send only for date reminders/reschedule alerts. Content messages always draft-and-approve.
**Why this approach:** _TBD_

## Calendar export (.ics feed)

**Status:** Not built yet
**Files:** _TBD_
**Edge cases:** Per-lawyer subscription URL signed with a secret to prevent enumeration.
**Why this approach:** _TBD_

## Scraper / API health panel

**Status:** Not built yet
**Files:** _TBD_
**Edge cases:** Last-successful-fetch per source, failure rate over 7 days, stale case count.
**Why this approach:** _TBD_

## Nightly refresh job

**Status:** Not built yet
**Files:** _TBD_
**Edge cases:** Only active cases. Archived/closed excluded. Failures flag the case visibly.
**Why this approach:** _TBD_
