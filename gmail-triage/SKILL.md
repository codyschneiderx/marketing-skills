---
name: gmail-triage
description: Triage the user's Gmail inbox and interview them to draft replies. Use when the user says "triage my emails", "go through my inbox", or asks for help drafting responses to unread mail. Designed for voice-driven workflows — the user speaks their answers and Claude writes the drafts.
---

# Gmail Triage & Interview-to-Draft

Triage the user's Gmail inbox via OAuth, then interview them one email at a time to write reply drafts. The user's only input is voice transcription — keep prompts short, single-question, and easy to answer out loud.

## Prerequisites

- Gmail OAuth credentials at `gmail-credentials.json` + `gmail-token.json` in the project root (or `GMAIL_CREDENTIALS` / `GMAIL_TOKEN` env vars).
- Scopes required: `gmail.readonly`, `gmail.compose` (for creating drafts).
- If credentials are missing, stop and tell the user how to set them up — do not attempt to proceed.

## Process

### Step 1: Triage the Inbox
1. Fetch unread messages from the primary inbox (`is:unread category:primary`, last 7 days, max 25).
2. For each message, extract: sender, subject, snippet, received time, thread ID, message ID.
3. Classify each into one of:
   - **Reply needed** — direct question, ask, or thread the user is in
   - **FYI** — newsletter, notification, no action
   - **Spam/promo** — ignore
4. Present a numbered list of **only the "Reply needed"** items. One line each: `N. [Sender] — [Subject] — [1-line gist]`. Nothing else. No preamble.

Ask: **"Which ones do you want to draft replies for? (numbers, 'all', or 'none')"**

### Step 2: Interview the User (one email at a time)
For each selected email:

1. Show the email in a compact block:
   ```
   ── Email N of M ──
   From: ...
   Subject: ...
   
   <body, trimmed to ~500 chars>
   ```
2. Ask **ONE short question at a time**. The user is speaking — never stack questions, never ask for structured input, never ask them to spell things. Examples:
   - "What's the gist of your reply?"
   - "Yes or no on the meeting?"
   - "Anything else to add?"
3. After 1–3 questions max, write the draft. Keep drafts in the user's natural voice — short, direct, no corporate filler, no "I hope this email finds you well."
4. Show the draft and ask: **"Send to drafts, revise, or skip?"**
5. On "drafts" → create a Gmail draft as a reply in the original thread (use `threadId` and `In-Reply-To` / `References` headers). On "revise" → ask what to change (one question), rewrite, re-show. On "skip" → move on.

### Step 3: Summary
After the last email, print a one-line summary: `Drafted N replies. M skipped. Check Gmail drafts folder.`

## Voice-Transcription Rules (CRITICAL)

The user's only input channel is speech-to-text. This shapes everything:

- **One question per turn.** Never ask "what's the tone, length, and key points?" — ask one thing.
- **Yes/no or short-phrase questions** beat open-ended ones. "Accept the meeting?" beats "How do you want to handle this?"
- **Never ask for spelling, URLs, or email addresses** — pull them from the original email.
- **Tolerate transcription errors.** If a response is ambiguous, infer from context rather than asking again. Only re-ask if truly unparseable.
- **No markdown lists in questions.** Speech UIs read them poorly. Use plain sentences.
- **Short Claude output between turns.** The user is listening, not reading — don't dump paragraphs.

## Draft Style Defaults

Unless the user says otherwise:
- Match the length of the incoming email (short → short).
- No greeting beyond "Hey [first name]," — no "Dear", no "I hope you're well."
- No sign-off boilerplate beyond the user's normal signature.
- Plain text, no formatting.
- If the user is declining something, decline cleanly in one sentence — no over-apologizing.

## Error Handling

- **OAuth token expired** → tell the user to re-auth, stop. Do not attempt silent refresh loops.
- **Rate limited** → back off and inform the user.
- **Draft create fails** → show the draft text so nothing is lost, ask whether to retry.
