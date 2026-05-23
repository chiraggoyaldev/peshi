# CLAUDE.md

Entry point for any AI assistant or future-me working on Peshi. Read this first on every new session.

This file is a **map**, not a destination. It points to where things live — it does not store decisions, code patterns, or implementation details (those go in `DECISIONS.md`, `/docs/CONVENTIONS.md`, and `/docs/FEATURES.md` respectively).

---

## Project summary

Peshi is a case tracker for Indian litigators. Lawyers add a CNR or SLP once; Peshi fetches all hearing data, generates an AI prep brief the night before each hearing, and keeps clients informed over WhatsApp. Solo-developer portfolio build over 4–6 weeks.

## Tech stack

Next.js · Supabase (Postgres + Auth + Storage) · Inngest · Playwright · GPT-4o · Whisper · Twilio · Resend · Vercel.

## Current state

**Phase:** Project setup — repo initialised, documentation skeleton in place. No application code yet.

## Last session

- 2026-05-24 — Created repo, set up SSH alias `github-personal` for personal GitHub isolation, initialised the documentation skeleton (this file + 7 others). First commit pending.

## Pointer map — where things live

| If you need… | Read this |
|---|---|
| Scope, features, persona, demo flow | [`PRD.md`](./PRD.md) |
| Why a choice was made (alternatives considered, tradeoffs) | [`DECISIONS.md`](./DECISIONS.md) |
| Logical flow, modules, external integrations | [`docs/ARCHITECTURE.md`](./docs/ARCHITECTURE.md) |
| What a specific feature does and where its code lives | [`docs/FEATURES.md`](./docs/FEATURES.md) |
| Available API endpoints | [`docs/API.md`](./docs/API.md) |
| Coding rules, patterns, anti-patterns | [`docs/CONVENTIONS.md`](./docs/CONVENTIONS.md) |
| Database schema | The schema file in the codebase (with inline comments) — no separate doc |
| Bugs, TODOs, progress | GitHub Issues |

## Cross-cutting conventions (quick reference)

Detail for each lives in `/docs/CONVENTIONS.md`. The universals:

- All background jobs run through Inngest. Never spawn workers directly.
- All eCourts fetches go through `lib/ecourts/client.ts`. Never call eCourtsIndia.com URLs from feature code.
- Court-fetched data is **read-only**. Lawyers add context via notes, never edit fetched records.
- `HearingDateChange` is append-only — every reschedule is preserved, never overwritten.
- Personal GitHub work happens only inside `~/Desktop/peshi/` — global git identity and SSH stay work-attributed.

## Domain glossary

Legal terms used throughout the codebase. Indian-jurisdiction-specific.

| Term | Meaning |
|---|---|
| **CNR** | Case Number Record — unique 16-character identifier for any district court case. Example: `DLST010234562024`. |
| **SLP** | Special Leave Petition — Supreme Court mechanism to appeal lower-court orders. |
| **Civil Appeal / CA** | A type of SC appeal, distinct from SLP. Both are SC identifiers. |
| **NJDG** | National Judicial Data Grid — government aggregation of court data. |
| **IA** | Interlocutory Application — a sub-application within a main case (e.g. for interim relief). |
| **Cause List** | Daily court schedule published the evening before, listing which matters are heard tomorrow, in what order, before which judge. |
| **Adjournment** | Postponement of a hearing to a later date. Usually no order is passed. |
| **Munshi** | Clerk / paralegal — the daily user managing case dates in a chamber. Not modelled as a user role in v1. |
| **Vakalatnama** | Power of attorney signed by a client authorising a lawyer to represent them. |
| **Limitation** | Statutory time limit within which legal action must be filed. |
| **Interlocutory Order** | A non-final order passed during the case (e.g. on bail, injunctions). |
| **Final Judgment** | The disposing order that concludes a case. |
| **Bench** | The judge(s) hearing a particular matter. |
| **Stage** | The current procedural posture of a case (Arguments, Evidence, For orders, etc.). eCourts returns this as a string, often inconsistent. |

## Documentation rules

These govern how docs are maintained. Apply on every chat touching code.

### Update silently — no asking
- Feature shipped → update its section in `docs/FEATURES.md` before closing the GitHub Issue
- Schema changed → update inline comments in the schema file
- New pattern emerged → append a rule to `docs/CONVENTIONS.md`
- New endpoint added → update `docs/API.md`
- New external integration → add to `docs/ARCHITECTURE.md` "External integrations" section
- End of session → update this file's "Last session" line

### Ask first
- Decision moment → "Store as ADR in `DECISIONS.md`? Yes / No / Maybe"
- Fundamental architecture change → confirm before editing `docs/ARCHITECTURE.md`
- Removing or rewriting an existing CONVENTIONS rule → confirm first
- Scope changes that touch `PRD.md` → confirm first

### Writing principles
- Lean. Cut every line that doesn't add information.
- Cross-reference, never duplicate.
- Examples beat explanations.
- Anti-patterns beat abstract rules.

### Never
- Put decisions in this file (they belong in `DECISIONS.md`)
- Put "how it works now" in `DECISIONS.md` (that's `docs/FEATURES.md` / `docs/ARCHITECTURE.md`)
- Edit a closed ADR — append a new one that supersedes it instead
- Skip writing an ADR because you're in a hurry — by next chat the alternatives are forgotten
- Create new doc files without asking

---

## Starting prompt — paste at every new AI chat

```
You are working on Peshi. Before anything else:

1. Read CLAUDE.md fully — including the "Documentation rules" section.
   Apply those rules to every doc change in this session.
2. For scope and features → PRD.md.
3. For why choices were made → DECISIONS.md. Treat every decision
   there as final unless I explicitly say "revisit this decision".
4. Before writing any code → read docs/CONVENTIONS.md fully.
5. For architecture, features, API → docs/ARCHITECTURE.md,
   docs/FEATURES.md, docs/API.md. For schema → the schema file directly.

6. Never assume — if unclear, check the relevant doc first.
7. If a CONVENTIONS rule conflicts with what you'd suggest, follow
   the rule exactly. Do not propose alternatives or work around it.
   If you think a convention should change, ask me first.
8. When a decision-worthy choice comes up mid-chat, ask immediately:
   "Store this as an ADR in DECISIONS.md? Yes / No / Maybe"

9. Before this chat ends:
   - Resolve every "Maybe" decision (final yes or no)
   - List every decision stored this session for my review
   - Update CLAUDE.md "Last session" line
   - Update any doc you touched

10. Keep all docs lean. Never duplicate content across files.
    Cross-reference instead.
```
