# Dental Clinic AI Voice Receptionist

An AI-powered voice receptionist that answers real phone calls (and a browser widget) for a dental clinic, checks live calendar availability, books and cancels appointments, and handles FAQs — end to end, with no human in the loop.

**Live demo:** https://vocal-buttercream-e1a4a7.netlify.app/ · **Test line:** +1 (716) 670 2389

---

## The problem

Small clinics lose bookings to hold times and missed calls. A receptionist can't be on the phone 24/7, and most "AI answering services" just take a message instead of actually completing the booking. The goal here was a voice agent that behaves like a real front-desk employee — checks the calendar, books the slot, confirms it out loud, and can also cancel or answer general questions, all inside a single phone call.

## Architecture

```
Caller (phone or browser mic)
        │
        ▼
   Vapi (STT → LLM → TTS voice pipeline)
        │  tool calls (JSON)
        ▼
   n8n webhook (Header Auth secured)
        │
        ▼
   Route by Function ──┬── check_availability ──► Google Calendar (freebusy)
                        ├── book_appointment ────► Google Calendar (create event)
                        ├── cancel_appointment ──► Google Calendar (search + delete)
                        └── faq_lookup ──────────► static clinic knowledge
        │
        ▼
   Respond to Webhook → back to Vapi → spoken to caller
```

**Voice layer:** Vapi (GPT-4o-mini, Deepgram transcription, ElevenLabs voice)
**Backend:** n8n (webhook-driven, function-routed)
**Data store:** Google Calendar (single source of truth for bookings)
**Frontend:** Static HTML/CSS/JS landing page embedding the Vapi Web SDK, with a live-captioned call widget

## Why these decisions

- **Single webhook, function-routed** rather than one webhook per tool — keeps the Vapi tool configuration simple (one URL to maintain) and centralizes auth/logging in one place.
- **Google Calendar over a custom database** — the clinic already lives in Calendar; freebusy queries give real-time accuracy without syncing a second data store.
- **System-prompt-enforced booking sequence** — the model was initially skipping the caller's name before booking. Rather than trying to catch this downstream, the fix was upstream: an explicit numbered sequence in the system prompt ("you MUST ask for the caller's name before calling book_appointment") — because a required field in the tool schema alone doesn't force the model to *ask* for it, only to include it if it has it.
- **Case-insensitive, date-scoped matching for cancellations** — matching a spoken name against a calendar event title by exact string comparison fails constantly (case, partial names, extra words). Matching is scoped by name substring **and** date together to avoid cancelling the wrong appointment.

## Problems solved during build (worth knowing for the interview)

1. **Silent 403s on every tool call.** The n8n webhook had Header Auth enabled but Vapi's tools had no matching header configured — requests were reaching n8n and being rejected before they ever ran. Fixed by adding a shared secret header to all four Vapi tools.
2. **Web SDK never loaded in the browser.** `@vapi-ai/web` is a bundler-only package — including it via a plain `<script src>` tag never actually defines `window.Vapi`, regardless of connection speed. Switched to Vapi's dedicated `html-script-tag` SDK, which is built for exactly this integration path.
3. **Type-mismatch in cancellation flow.** An `If` node compared a boolean (`eventFound: true`) against a string (`"true"`) with strict type validation on — silently failing even when the calendar match succeeded. Fixed by enabling type coercion on the condition.
4. **Free Vapi numbers are inbound-only.** New Vapi accounts can't place outbound test calls on their free number — this is a platform limitation, not a config bug. Testing was done by calling the number directly instead of using Vapi's outbound test tool.

## Stack

`Vapi` · `n8n` · `Google Calendar API` · `HTML/CSS/JS` · `Deepgram` · `ElevenLabs` · `GPT-4o-mini`

## Status

Functional end-to-end: live calendar checks, bookings, cancellations, and FAQ handling all confirmed working via real phone calls. Built as a portfolio case study using publicly available clinic information — not an official commissioned deployment.
