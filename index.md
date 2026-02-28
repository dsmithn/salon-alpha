---
layout: default
title: Salon Alpha
---

# Salon Alpha

**An AI receptionist that answers the salon's phone when you can't.**

Caller dials in → AI greets them → handles bookings, FAQs, or transfers to a human. When you turn the agent off, calls ring straight through to the salon phone. That's the whole product.

**Cost to run: <$30/month.** One codebase. Repeatable for every salon.

---

## Start Here

### [The Playbook →](playbook)
The full technical specification. Architecture, call flows, Vagaro integration, legal compliance, cost breakdown, risk analysis. Read this to understand **what** we're building and **why** every decision was made.

### [Builder's Guide →](builders-guide)
The implementation kit. Account setup, a library of copy-paste meta-prompts for Claude Code, and a session-by-session build plan. Read this to **build it**.

---

## The 60-Second Version

```
Caller dials salon
       │
       ▼
Conditional forwarding → Twilio number
(salon phone rings 3-4x first)
       │
       ▼
Your server checks: AI agent ON or OFF?
       │
       ├─ ON  → Voice AI answers (ElevenLabs or Retell)
       │         ├─ Booking → check availability → hold slot → SMS confirm
       │         ├─ FAQ → answer from knowledge base
       │         └─ Human needed → transfer to salon phone
       │
       └─ OFF → Calls ring salon phone directly
```

**Stack:** Twilio (phone) · ElevenLabs or Retell (voice AI) · Google Calendar → Vagaro (bookings) · Supabase (logs) · Railway (hosting)

---

*Built by Salon Alpha. Internal draft — not for distribution.*
