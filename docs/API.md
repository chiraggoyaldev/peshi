# API

REST endpoints exposed by the Next.js app. All endpoints require authenticated session via Supabase Auth unless marked Public.

> If/when OpenAPI/Swagger is generated from code, this file is replaced by the generated reference. Until then, this is the source of truth for the API surface.

Each endpoint is filled in when its handler ships.

---

## Conventions

- All endpoints return JSON
- Successful responses: `{ data: ..., meta?: ... }`
- Errors: `{ error: { code: string, message: string } }` with appropriate HTTP status
- Auth via `Authorization: Bearer <token>` header (Supabase JWT) or session cookie
- Versioned under `/api/v1/`

---

## Auth

_Handled by Supabase Auth — see Supabase docs. No custom auth endpoints._

## Cases

### `POST /api/v1/cases`
Add a single case by CNR or SLP. Triggers immediate fetch.

### `POST /api/v1/cases/bulk`
Add many cases from a newline-separated list of CNRs/SLPs.

### `GET /api/v1/cases`
List cases. Supports `?bucket=today|tomorrow|this_week|later|pinned`, `?status=active|closed|archived`, `?q=search-term`.

### `GET /api/v1/cases/:id`
Single case detail including hearings, orders, parties, notes.

### `PATCH /api/v1/cases/:id`
Update case-level fields: status, pinned, private metadata.

### `POST /api/v1/cases/:id/close`
Mark a case as closed. Body includes outcome (Won/Lost/Settled/Withdrawn) and one-line reason.

## Hearings

### `GET /api/v1/cases/:id/hearings`
All hearings for a case, chronological. Includes reschedule history.

### `POST /api/v1/hearings/:id/note`
Add a private note (text) to a hearing.

### `POST /api/v1/hearings/:id/voice-note`
Upload a voice note. Triggers Whisper transcription job.

## Clients

### `POST /api/v1/clients`
Create a client (name, phone, optional email).

### `GET /api/v1/clients`
List clients.

### `GET /api/v1/clients/:id`
Client detail including all linked cases.

## AI

### `POST /api/v1/ai/prep-brief/:case_id`
Manually trigger prep brief generation. Normally auto-triggered by nightly cron.

### `POST /api/v1/ai/transcribe`
Transcribe a voice note. Usually triggered by the voice-note upload handler, not called directly.

## Notifications

### `GET /api/v1/notifications`
List in-app notifications for the current user.

### `POST /api/v1/notifications/preferences`
Update notification preferences (hearing-type defaults, per-case override).

## WhatsApp

### `POST /api/v1/whatsapp/draft/:hearing_id`
Generate a draft client message for a hearing. Returns the proposed message for lawyer approval.

### `POST /api/v1/whatsapp/send`
Send an approved or custom message to a client.

### `POST /api/v1/whatsapp/settings`
Per-client sharing settings.

## Calendar

### `GET /api/v1/calendar/:user_secret.ics`
Public. Returns iCalendar feed of upcoming hearings. Signed URL — `user_secret` is per-lawyer.

## Health

### `GET /api/v1/health/scrapers`
Last-successful-fetch per source, failure counts, stale case count.

---

## Webhooks (incoming)

### `POST /api/webhooks/twilio`
Twilio delivery status callbacks.

### `POST /api/webhooks/inngest`
Inngest job event ingress.
