---
name: gmail-triage
description: Triage the user's Gmail inbox and interview them to draft replies. Use when the user says "triage my emails", "go through my inbox", or "interview me to write drafts". Voice-first — the user speaks their answers and Claude writes the drafts.
---

# Gmail Triage & Interview-to-Draft

> Comment "triage" if you want this Claude Code skill file.

Two-phase workflow:
1. **Triage** the inbox into a priority work queue (urgent / this week / fyi).
2. **Interview** the user one email at a time, voice-only, and write Gmail drafts.

## Prerequisites

- `gmail-credentials.json` + `gmail-token.json` in project root (Gmail OAuth, scopes include compose).
- `ANTHROPIC_API_KEY` in `.env`.
- Existing service code: `src/gmail-client.ts`, `src/gmail-inbox-service.ts`, `src/prompts/email-triage.ts`.
- Run `npm run verify-keys` first if anything looks missing — never read `.env` directly.

If credentials are missing, stop and tell the user. Do not attempt silent re-auth.

## Phase 1 — Triage

Run the existing triage pipeline:

```bash
npm run inbox-triage
```

Under the hood this calls `triageInbox()` from `src/gmail-inbox-service.ts`, which:
- Fetches unprocessed inbox messages via `getUnprocessedInboxMessages()`
- Classifies each with Claude using `buildClassificationPrompt()` into one of:
  `human_urgent`, `human_actionable`, `human_fyi`, `automated_important`, `automated_marketing`, `spam_or_skip`
- Applies Gmail labels (`AI-Triage/*`) and archives marketing
- Appends `human_*` items to `inbox-work-queue.json`

After it runs, read `inbox-work-queue.json` and present **only `human_urgent` and `human_actionable`** items, sorted by priority. One line each:

```
N. [from] — "[subject]" — [suggestedAction]
```

No preamble. Then ask one question:

> **"Which ones do you want to draft replies for? Numbers, 'all', or 'none'."**

## Phase 2 — Interview (voice-only)

For each selected item, fetch the full message via `getMessage(messageId)` and run this loop:

1. Show a compact block — sender, subject, ~500 char body trim. Nothing else.
2. Ask **ONE short question**. Examples:
   - "What's the gist of your reply?"
   - "Yes or no on the meeting?"
   - "Anything to add?"
3. Max 1–3 questions per email. Then write the draft in Cody's voice (short, direct, no corporate filler, no "I hope this email finds you well", no boilerplate sign-off).
4. Show the draft. Ask: **"Drafts, revise, or skip?"**
5. On "drafts" → create a Gmail draft as a reply on the original `threadId` with proper `In-Reply-To` / `References` headers (use `gmail-client.ts` helpers; add a `createDraftReply` helper there if it doesn't exist yet — do not reimplement the OAuth client).
6. After creating the draft, call `markAsResponded(messageId)` from `gmail-inbox-service.ts` to flip the work-queue flag.
7. On "revise" → one question, rewrite, re-show. On "skip" → move on without marking.

After the last email, print one line: `Drafted N. Skipped M. Check Gmail drafts.`

## Voice-Transcription Rules (CRITICAL)

The user's only input is speech-to-text. Everything follows from this:

- **One question per turn.** Never stack.
- **Yes/no or short-phrase questions** beat open-ended.
- **Never ask for spelling, URLs, email addresses, or names** — pull them from the original email.
- **Tolerate transcription errors.** Infer from context. Only re-ask if truly unparseable.
- **No markdown lists inside questions.** Plain sentences read better.
- **Short Claude output between turns.** The user is listening, not reading.

## Draft Style Defaults (Cody's voice)

- Match the length of the incoming email (short → short).
- Greeting: `Hey [first name],` — no "Dear", no "I hope you're well."
- No sign-off boilerplate beyond Cody's normal signature.
- Plain text. No formatting.
- Declines: one clean sentence, no over-apologizing.
- Never invent commitments, dates, or numbers — if the user didn't say it, don't write it.

## Error Handling

- **OAuth token expired** → tell the user to re-auth and stop.
- **Rate limited** → back off, inform the user.
- **Draft create fails** → show the draft text in chat so nothing is lost, ask whether to retry.
- **Work queue empty** → say so and stop; do not invent items.
