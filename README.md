# telegram-ai-appointment-booking

An AI-powered Telegram bot that books appointments from a plain-language chat message. Built in n8n. You message the bot something like *"Book a 30 min call next Tuesday 2pm"*, and it extracts the intent, checks your Google Calendar for a conflict, books the slot if it's free, and replies in chat — all automatically.

This is the **Telegram sibling** of [gmail-ai-appointment-booking](https://github.com/tazahein/gmail-ai-appointment-booking). Same calendar "brain," different front door: the entire AI + calendar core was reused, and only the trigger and reply nodes were swapped from Gmail to Telegram.

## What it does

1. A user sends the bot a free-text booking request on Telegram.
2. An **AI Agent** (Claude Sonnet 4.6) reads the message and extracts structured details: start datetime, duration, and purpose.
3. The workflow checks **Google Calendar** to see whether that slot is free.
4. If free → it **creates the calendar event** and replies with a confirmation.
5. If busy → it replies asking for a different time. No event is created.

## Architecture

```
Telegram Trigger (On message)
        │
        ▼
   AI Agent  ──◄ Anthropic Chat Model (Claude Sonnet 4.6)
        │     ──◄ Structured Output Parser  →  { startDateTime, durationMinutes, purpose }
        ▼
Get availability in a calendar  (Google Calendar – FreeBusy → { available: true/false })
        │
        ▼
       If  ({{ $json.available }} is true?)
        │
   true ├────────────► Create an event ───► Send a text message  (✅ confirmation)
        │
  false └────────────► Send a text message1 (😕 slot taken)
```

| Node | Role |
|------|------|
| **Telegram Trigger** | Fires on every incoming message. Message text at `message.text`, sender chat ID at `message.chat.id`. |
| **AI Agent + Structured Output Parser** | Resolves relative dates ("next Tuesday") against today and returns clean JSON with a `+07:00` offset. |
| **Get availability in a calendar** | FreeBusy check over the requested window; returns `{ available: true/false }`. |
| **If** | Boolean branch on `available`. |
| **Create an event** | Books the slot; end time computed as `startDateTime + durationMinutes`. |
| **Send a text message / …1** | Telegram replies for the confirmed and conflict paths. |

## Key concepts demonstrated

- **Reusable workflow design** — the AI Agent, output parser, availability check, IF logic, and event creation were lifted wholesale from the Gmail version. Only the channel-specific endpoints changed.
- **Telegram Trigger setup** — listening on `On message`, and reading the trigger output paths (`message.text`, `message.chat.id`, `message.from.username`).
- **Telegram Send Message replies** — dynamic `chatId` so the bot always answers whoever messaged it.
- **Relative-date resolution** — today's date is injected into the prompt via `{{ $now.toFormat('yyyy-MM-dd') }}` so the model can resolve phrases like "next Tuesday" correctly.
- **Timezone-correct datetimes** — the model outputs full ISO 8601 with the `+07:00` (Asia/Bangkok) offset; the end time is derived in n8n with `DateTime.fromISO(...).plus({ minutes: ... }).toISO()`.

## Setup

You'll need an n8n instance (this was built on n8n Cloud) and three credentials:

1. **Telegram** — a bot token from [@BotFather](https://t.me/BotFather), stored as a Telegram credential.
2. **Anthropic** — an API key for the Claude model.
3. **Google Calendar** — OAuth2 (on n8n Cloud this is one-click managed OAuth; no Google Cloud Console needed).

Then:

1. Import `Appointment Booking — Telegram.json` into n8n.
2. Open each credential-bearing node and select your own credentials (the file ships with references only — **no secrets are included**).
3. In both Google Calendar nodes, pick your target calendar.
4. Activate the workflow (or run it in test mode), then message your bot.

> **Note on the calendar email:** the JSON references a specific calendar address as the selected calendar. Replace it with your own when importing.

## Design notes & gotchas

- **Single-user by design.** This bot is intended for one operator (me). For a multi-user / client-facing version, add **Restrict to Chat IDs** on the Telegram Trigger to whitelist allowed chats, or remove the restriction entirely to let anyone book.
- **One trigger per bot.** Telegram allows only one active trigger workflow per bot token at a time. If you reuse a bot that other workflows already *send* through, that's fine — but don't run two *trigger* workflows on the same bot.
- **Timezone inheritance.** Because this workflow was duplicated from the Gmail version, it inherited its calendar/timezone configuration. The original had a bug where an event booked 30 minutes early because the calendar's timezone was set to Yangon (UTC+06:30) instead of Bangkok (UTC+07:00). For maximum portability, pin the **Timezone** field on the Google Calendar nodes rather than relying on the account default. Always verify a test booking lands at the exact requested time.
- **Credentials are never committed.** n8n exports store credentials as references (`id` + `name`), not as secret values. Keep it that way — never paste a raw token into a node field or commit a `.env`.

## Tech

n8n · Claude Sonnet 4.6 (Anthropic) · Telegram Bot API · Google Calendar API

---

*Part of the Zero to AI Automation Agency Bootcamp — Phase 4 (n8n Mastery). Project 7b.*
