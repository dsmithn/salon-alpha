# Salon Alpha Playbook

**For: [Salon Name]**
**Classification: Internal Draft**

---

## What We're Building

An AI receptionist that answers the salon's phone when you can't pick up. A caller dials in, the AI greets them warmly, and handles the conversation. It does three things:

1. **Bookings** — two paths: the AI can check real availability and hold a slot (salon owner confirms, caller gets a text), or simply text the caller a link to book on Vagaro at their convenience
2. **FAQs** — answers questions about pricing, hours, cancellation policy, parking, etc. from a knowledge base you define
3. **Transfers** — if the caller wants a human, the AI forwards them to your cell or takes a message

The salon owner's experience: you open Vagaro in the morning and see your appointments. Nothing changes about your workflow.

**Cost to run: <$30/month.** [^1]

---

## The Full Call Flow

```
Caller dials salon
       │
       ▼
Call forwards to our Twilio number
(conditional forwarding — rings your phone 3-4x first)
       │
       ▼
Retell AI agent picks up
       │
       ▼
AI greets the caller
"Thanks for calling [Salon]! This call is assisted by AI
 and may be recorded. How can I help you today?"
       │
       ▼
Caller speaks
       │
       ▼
AI responds (one of three paths):
       │
       ├─► BOOKING INTENT
       │   AI: "I can check availability and get you booked right now,
       │         or text you a link to book at your convenience."
       │   │
       │   ├─► "Book me now"
       │   │   AI checks Google Calendar in real-time (mid-call function call)
       │   │   → "Jessica has 2pm and 4pm Thursday. Which works?"
       │   │   → Caller picks a slot
       │   │   → AI creates Google Calendar event (syncs to Vagaro as Personal Task)
       │   │   → AI: "You're all set — you'll get a text when it's confirmed!"
       │   │   → Salon owner gets SMS notification with booking details
       │   │   → Salon owner confirms → caller gets confirmation text
       │   │
       │   └─► "Text me a link"
       │       AI: "What number should I send it to?"
       │       → AI texts the salon's Vagaro booking page link
       │       → Done — caller books at their convenience
       │
       ├─► FAQ INTENT
       │   AI answers from knowledge base
       │   → "A balayage starts at $150 and takes about 2.5 hours."
       │   → Offers to book if caller is interested
       │
       ├─► CANCEL / RESCHEDULE INTENT
       │   AI does NOT modify existing appointments (no API path in Phase 1)
       │   → "I can't change existing appointments directly — let me
       │      connect you with [Salon Owner], or I can take a message
       │      and they'll get right back to you."
       │   → Transfer or message flow (see below)
       │
       └─► TRANSFER / COMPLEX REQUEST
           AI forwards call to salon owner's cell
           → "Let me connect you with someone at [Salon] directly."
           → If no answer, takes a detailed message
           → System texts salon owner: "MISSED TRANSFER: [Name] ([number])
             called about [topic]. They asked: [summary]. Please call back."
```

> **Risk: Carrier Conditional Forwarding (go/no-go gate)**
> Conditional call forwarding behavior varies across carriers, phone models, and plan types — not just T-Mobile. [^2] **Carrier forwarding verification is a hard go/no-go gate, not just a risk to monitor.** If forwarding fails testing with the salon owner's specific carrier, fall back to a dedicated Twilio number with updated Google Business listing and voicemail greeting. See [CHANGELOG.md — CH-002](changelog) for the full decision framework.

---

## The Google Calendar Bridge

Vagaro's API has no "Create Appointment" endpoint. [^3] The only write path is creating a Personal Task, which blocks time on the calendar but isn't a real appointment. To get around this, we use Google Calendar as a bridge:

1. Vagaro has native two-way Google Calendar sync (launched December 2024) [^4]
2. Events created in Google Calendar appear as Personal Tasks in Vagaro
3. We write to Google Calendar via its well-documented, free API [^5]
4. The sync propagates to Vagaro within minutes

**How confirmation works:**

The **salon owner** confirms — not the caller. The caller already asked to book; making them click a link to "confirm" what they just asked for is confusing and adds friction.

When the AI books a slot via the "book me now" path:

1. AI creates Google Calendar event with caller details in the description → syncs to Vagaro as Personal Task
2. AI tells the caller: *"You're all set for 2pm Thursday with Jessica. You'll get a text when it's confirmed!"*
3. Our system texts the **salon owner**: *"NEW BOOKING: Sarah Johnson (555-0123) — Balayage with Jessica, 2pm Thursday. Check Vagaro for conflicts, then tap to confirm: [link]"*
4. Salon owner taps the confirm link → our server texts the **caller**: *"You're confirmed! Balayage with Jessica, 2pm Thursday at [Salon]. See you then!"*
5. Salon owner creates the real appointment in Vagaro (takes ~30-60 seconds — Vagaro allows unlimited double-booking [^7]) and deletes the Personal Task

**If the salon owner doesn't confirm within 2 hours:**
1. The hold expires — we delete the Google Calendar event (which removes the Vagaro Personal Task on next sync)
2. We text the caller: *"Sorry, we couldn't confirm your 2pm Thursday slot. Book anytime at: [Vagaro booking link]"*

This gives the caller a graceful fallback to the "text me a link" path if the hold falls through.

> **Important: There is no one-click "convert" button.** Vagaro's "Convert to Appointment" feature only exists for Google Calendar-imported events viewed in certain contexts, not for Personal Tasks created via API sync. The actual workflow is: create the real appointment first (double-booking is allowed), then delete the Personal Task. This works reliably but is two steps, not one. [^8]

> **Risk: Google Calendar Sync Latency + Double-Booking Gap**
> Vagaro's docs describe the sync as taking "a few moments" with no SLA. [^4] In practice this appears to be 1-5 minutes, but Vagaro launched two-way sync only in December 2024 — the feature is roughly a year old. During this sync window, the AI-held slot is not yet visible in Vagaro — someone could book the same slot directly on Vagaro, creating a true conflict. The salon owner's confirmation SMS prompts them to check Vagaro for conflicts before confirming. If sync reliability becomes an issue, the fallback is the Vagaro Enterprise API (webhook add-on $10/month) to create Personal Tasks directly. [^9] Build monitoring from day one. See [CHANGELOG.md — CH-001](changelog) for the full mitigation plan.

> **Note: `events.watch()` is optional for Phase 1.**
> The confirmation flow is driven by the salon owner tapping a link, not by monitoring Google Calendar changes. `events.watch()` [^6] becomes useful in Phase 2 if you want to automatically detect when the salon owner creates the real appointment in Vagaro (eliminating the manual confirm step). If you do use it, note that watch channels expire after a maximum of 7 days and must be renewed programmatically.

---

## What the AI Discloses (and Why)

The greeting includes two disclosures:

1. **"This call is assisted by AI"** — best practice for building caller trust and proactively complying with emerging state-level AI disclosure laws (e.g., Colorado AI Act, effective June 2026 [^10]). The FCC's February 2024 ruling classified AI-generated voices as "artificial" under TCPA, but that ruling applies to **outbound robocalls**, not inbound business calls. [^11] [^12] There is no current federal requirement to disclose AI on inbound calls. We disclose anyway because it's the right thing to do, builds trust, and future-proofs against state laws that are actively being written.

2. **"This call may be recorded"** — legally required. Florida is an all-party consent state (Fla. Stat. § 934.03) — recording without consent is a **felony**. [^13] Twelve other states have similar laws. [^14] This disclosure is non-negotiable.

---

## Phase 1: The Weekend Build

**Goal:** A working phone number the salon owner can forward calls to. AI answers, handles bookings via Google Calendar, texts confirmations, and logs every interaction.

### Accounts to Set Up (Friday Night, ~1 hour)

| Account | What For | Cost | Link |
|---------|----------|------|------|
| Retell AI | Voice AI platform (hosts the agent, handles STT/TTS/LLM) | $10 free credits (~60-90 min depending on config) [^15] | retellai.com |
| Google Cloud | Calendar API access for availability checks + booking | Free tier (1M queries/day) [^5] | console.cloud.google.com |
| Supabase | PostgreSQL database for call logs + webhook data | Free tier | supabase.com |
| Twilio (via Retell) | Phone number — Retell manages Twilio, no separate account needed | $2/month [^16] | Provisioned through Retell |
| Toll-free SMS number | Confirmation texts to callers | ~$2/month + $0.01/SMS | Via Twilio or Telnyx |

> **Risk: Supabase Auto-Pause**
> Supabase's free tier pauses the database after 7 days of inactivity. This is fine during the weekend build, but if the salon goes a slow week without calls, the database will pause and webhook ingestion will break silently. Monitor this. If it becomes an issue, Supabase Pro is $25/month, or use Railway's PostgreSQL addon. [^17]

### Saturday Morning (4 hours) — Core Agent

**Hour 1: The greeting.**

Configure the Retell agent's opening behavior. Two approaches:

- *Option A (recommended):* Record the salon owner saying a greeting in their own voice. Configure it as the agent's initial audio. Feels personal and familiar to regulars.
- *Option B (fallback):* Have the LLM's system prompt instruct it to speak the greeting via TTS. Less personal but guaranteed to work and easy to iterate on.

Example greeting: *"Thanks for calling [Salon]! This call is assisted by AI and may be recorded. How can I help you today?"*

**Hours 2-3: The system prompt + knowledge base.**

Build the AI agent's personality and knowledge:

- **Booking flow:** Offer the caller two options — "I can check availability and book you now, or text you a link to book at your convenience." For the "book now" path: trigger a mid-call function call to Google Calendar API [^18] to check availability, offer 2-3 open slots, create the event when the caller picks one, and notify the salon owner. For the "text a link" path: collect the caller's number and send the Vagaro booking URL via SMS.
- **Incomplete info handling:** Callers often don't specify everything upfront. For unknown stylist: *"Do you have a preferred stylist, or would you like whoever is available first?"* For unspecified service: *"What service are you looking for today?"* For variable-duration services (e.g., balayage on different hair lengths): *"Is your hair above or below your shoulders? That helps me find the right time slot."* See [CHANGELOG.md — CH-005, CH-008](changelog).
- **FAQ handling:** Load 10-15 Q&As specific to the salon (pricing for each service, hours, cancellation policy, parking, stylists and their specialties)
- **Cancel/reschedule handling:** The AI should not attempt to modify existing Vagaro appointments in Phase 1. Offer to transfer or take a message. See [CHANGELOG.md — CH-005](changelog).
- **Transfer behavior:** If caller asks for a human or the AI can't handle the request, execute a call transfer [^19] to the salon owner's cell. If transfer fails, take a detailed message and text it to the salon owner with caller's name, number, and request summary. See [CHANGELOG.md — CH-003](changelog).
- **Noise fallback:** If the AI can't understand after 2 attempts, offer to text a booking link instead: *"I'm having a little trouble hearing — would it be easier if I texted you a booking link?"* This gracefully degrades to the "text me a link" path. Enable Retell's denoising add-on (+$0.005/min) by default for salon environments. See [CHANGELOG.md — CH-006](changelog).
- **Conversation limits:** Keep it concise. Handle the request, confirm, wrap up. Don't let the AI ramble.

Key parameters to tune:
- **Response tone:** Natural and warm. Friendly but efficient — like a good receptionist.
- **Confirmation style:** Always confirm understanding before booking: *"Just to confirm — you'd like a balayage with Jessica at 2pm Thursday?"*
- **Handoff triggers:** If the caller sounds frustrated or the AI isn't understanding, offer to transfer immediately. Don't make people fight the bot.

**Hour 4: End-to-end testing.**

Call the number yourself 10+ times. Test:
- Normal booking request
- FAQ question ("How much is a balayage?")
- "Can I speak to someone?"
- Vague request ("I need my hair done")
- Multiple services in one request
- Immediately hanging up
- Background noise

### Saturday Afternoon (3 hours) — Observation Infrastructure

**Hour 5: Call logging webhook.**

Retell fires a `call_ended` webhook with the full transcript and metadata. [^20] Set up a webhook handler that writes to Supabase:

```
call_id | timestamp | duration | transcript | intent (book/faq/transfer) |
booking_completed | caller_satisfied | outcome | notes
```

Host the webhook handler on **Railway ($5/month) or Render ($7/month)**. [^21] [^22]

**Hour 6: SMS flows.**

Set up the toll-free SMS number and two SMS flows:

**Flow A: "Book me now" — salon-side confirmation**

When the AI holds a slot during a call:
1. Create Google Calendar event with caller details in the description
2. Text the **salon owner**: *"NEW BOOKING: Sarah Johnson (555-0123) — Balayage with Jessica, 2pm Thursday. Tap to confirm: [link]"*
3. Text the **caller**: *"Hi Sarah! We're holding 2pm Thursday with Jessica at [Salon] for you. You'll get a text when it's confirmed."*
4. If salon owner taps confirm → text caller: *"You're confirmed! Balayage with Jessica, 2pm Thursday at [Salon]. See you then!"*
5. If no confirmation in 2 hours → delete the Google Calendar event, text caller: *"Sorry, we couldn't confirm your 2pm Thursday slot. Book anytime at: [Vagaro link]"*

The confirm link points to a simple page on your webhook server (one endpoint, returns a success message, triggers the confirmation SMS). No `events.watch()` needed.

**Flow B: "Text me a link" — booking link only**

When the caller just wants a link:
1. Text the **caller**: *"Here's the booking link for [Salon]: [Vagaro booking URL]. Book anytime!"*

This is triggered by a Retell mid-call function call [^18] — the AI collects the phone number and your server sends the SMS. ~30 lines of code.

**Why toll-free instead of a local number?** Since February 2025, all carriers block unregistered local number (10DLC) business texting. [^23] Registering for 10DLC takes 15+ days and requires brand registration ($4-15) plus campaign registration ($15 one-time + $10/month) per salon. [^24] A toll-free number has a separate, simpler verification process, adequate throughput (~3 messages/second — more than enough for one salon), and no registration wait. 10DLC is a scaling concern for when you have dozens of salons — not an alpha blocker.

**Hour 7: Self-testing round 2.**

Test both booking flows end-to-end:
- **"Book me now" path:** booking → caller gets "holding" SMS → salon owner gets notification SMS → salon owner taps confirm → caller gets "confirmed" SMS → Google Calendar event syncs to Vagaro. Also test: salon owner ignores → 2-hour timeout → caller gets fallback SMS with Vagaro link.
- **"Text me a link" path:** caller asks for a link → receives SMS with Vagaro booking URL. Fix what breaks.

### Sunday — Real Callers + Analysis

**Morning (4 hours):** Get 15-25 people to call the number cold. Tell them to pretend they're calling a salon to book an appointment or ask about pricing. Don't tell them it's AI.

**Afternoon (4 hours):** Score every call. Calculate:
- **Completion rate** — % of calls that reached a resolution (booking, FAQ answered, transfer completed)
- **Booking conversion rate** — % of booking-intent calls that resulted in a confirmed booking
- **Transfer rate** — % of calls the AI couldn't handle and had to transfer (target: <20%)
- **Caller satisfaction** — did they get what they needed? Would they call again?

Listen to the 5 best and 5 worst calls. Write up findings.

---

## Phase 2: Vagaro Direct Integration (Weeks 3-8)

Once the Google Calendar bridge proves the concept, upgrade to direct Vagaro API integration for better reliability and real-time data.

**What changes:**
- **Vagaro Enterprise API** (webhook add-on is $10/month for 5,000 webhook calls; full API access may require contacting Vagaro Enterprise Sales) replaces Google Calendar as the primary integration [^9]
- Real-time availability checks via `Search Appointment Availability` endpoint (no sync delay)
- Personal Tasks created directly via API (no Google Calendar intermediary)
- Customer data synced via Vagaro webhooks → local PostgreSQL for caller ID lookup
- Returning callers greeted by name: *"Hi Sarah, welcome back to [Salon]!"*

**What stays the same:**
- Retell AI agent (same prompts, same voice)
- Toll-free SMS flows (salon-side confirmation + booking link)
- The manual appointment conversion workflow (salon owner creates real appointment → deletes Personal Task)
- Call logging to Supabase

**Requirements for Vagaro Enterprise API:**
- Salon must be on Vagaro's plan that includes Enterprise API access
- Credit card processing must be enabled through Vagaro [^25]
- Token-based auth — salon generates an API token during onboarding

> **Risk: Vagaro Personal Task API Usage**
> Using Personal Tasks as booking holds is a creative use of Vagaro's API. The endpoint is documented and intended for blocking time on a provider's calendar (breaks, admin tasks, etc.). Salons already use Personal Tasks manually for exactly this purpose — holding a phone booking while entering it. However, Vagaro hasn't explicitly blessed automated usage at scale. If Vagaro restricts this use case, the fallback is the Google Calendar bridge (Phase 1 architecture). Mitigate by pursuing a Vagaro partnership early. [^3]

---

## Phase 3: Number Porting (Month 2-3)

Port the salon's existing phone number to Twilio for full programmatic control.

**What this enables:**
- **Scheduled routing:** AI answers during business hours overflow and after-hours. During quiet periods, calls ring through to the salon phone first.
- **Caller ID passthrough:** Twilio can preserve the original caller's number on forwarded calls (requires passing `CallToken` — not automatic) [^26]
- **Full call analytics:** Every inbound call flows through your infrastructure, not just forwarded ones
- **Simultaneous ring:** Ring the salon phone AND the AI agent at the same time — whoever picks up first wins

**Porting timeline:** Twilio number porting typically takes 2-4 weeks for US local numbers. [^27] Start the process early.

---

## Decisions to Make With the Salon Owner

Before building, sit down and get answers to these:

| Decision | Why It Matters | Default If No Preference |
|----------|---------------|------------------------|
| Which scheduling platform? | Determines integration path. Vagaro = Google Calendar bridge → Enterprise API. Square = direct API (free). Others = SMS booking link only. | Vagaro (what you use) |
| Solo stylist or multi-stylist? | Affects availability logic and stylist matching | Solo (simpler) |
| Service list with durations and prices | AI needs this to offer accurate availability and answer pricing questions | Get the full list from the Vagaro account |
| Preferred booking method | AI offers both "book now" (AI holds slot, salon confirms) and "text me a link" (Vagaro booking URL). Disable either? | Both enabled |
| What should the AI NOT do? | Boundaries matter — e.g., never quote prices for services not on the list, never promise same-day availability without checking | AI offers to transfer to human for anything outside its knowledge base |
| Greeting voice | Record the owner's voice for the greeting, or use TTS? | Record the owner (more authentic) |
| After-hours behavior | Answer 24/7 or only during business hours? | 24/7 — missed calls happen most after hours |

---

## Cost Breakdown

| Item | Monthly Cost | Notes |
|------|-------------|-------|
| Retell AI voice agent | ~$10-20 | At 100-200 min/month ($0.101/min all-in with GPT-4.1 mini) [^15] |
| Phone number (via Retell) | $2 | Retell-managed Twilio number [^16] |
| Toll-free SMS number | ~$2 | Plus $0.01/SMS [^28] |
| Webhook hosting | $5-7 | Railway Hobby plan ($5) [^21] or Render ($7) [^22] |
| Supabase (call logs) | $0 | Free tier (monitor for auto-pause) [^17] |
| Google Calendar API | $0 | Free tier, 1M queries/day [^5] |
| **Total** | **<$30/month** | At typical single-salon call volume (100-200 min/month) |

**Phase 2 addition:** Vagaro webhook add-on adds $10/month (full Enterprise API access may cost more — contact Vagaro sales). [^9]

---

## What We're NOT Building for the Alpha

- No client-facing dashboard
- No billing system
- No multi-salon management
- No outbound calls or reminders (TCPA minefield — later). Note: Vagaro sends automated reminders for real appointments but NOT for Personal Tasks. Callers booked via the AI only get Vagaro reminders after the salon owner creates the real appointment with correct contact info. See [CHANGELOG.md — CH-004](changelog).
- No Spanish language support
- No integration with platforms other than Vagaro
- No automated Personal Task → appointment conversion (manual for now)

---

## Success Criteria

After 2 weeks of live calls:

| Metric | Target | What It Tells Us |
|--------|--------|-----------------|
| Call completion rate | >70% | Do callers stay on the line and get their question answered? |
| Booking conversion rate | >40% | Of callers who want to book, how many complete the flow? |
| Transfer rate | <20% | How often does the AI have to hand off to a human? |
| Caller satisfaction | No complaints to salon owner | Is the AI good enough to not damage the brand? |
| Salon owner effort | <5 min/day | Is the manual conversion workflow tolerable? |

**Decision framework:**
- All targets met → **Go.** Start onboarding more salons.
- Completion >70% but booking conversion <40% → **Iterate.** Callers engage but the booking flow needs work.
- Completion <50% → **Rethink.** Callers aren't staying on the line. Voice quality, greeting, or tone needs a rework.

---

## Footnotes

[^1]: Running cost at 100-200 min/month: Retell AI $10-20 (at $0.101/min all-in with GPT-4.1 mini) + $2 phone number + $5-7 hosting + $0-2 SMS = $17-31. Source: [Retell AI Pricing](https://www.retellai.com/pricing), [Railway Pricing](https://railway.com/pricing), [Render Pricing](https://render.com/pricing).

[^2]: T-Mobile conditional call forwarding to VoIP DIDs has known routing propagation issues. Source: [VoIP-Info Forum](https://www.voip-info.org/forum/threads/t-mobile-conditional-call-forwarding-to-voip-did-fails-seeking-workarounds.28345/).

[^3]: Vagaro Enterprise API V2 has no "Create Appointment" endpoint. The only calendar write path is Personal Tasks. Source: [Vagaro Developer Docs](https://docs.vagaro.com/).

[^4]: Vagaro launched two-way Google Calendar sync in December 2024. Google Calendar events appear as Personal Tasks in Vagaro. Source: [Vagaro Google Sync Announcement](https://www.vagaro.com/pro/updates/google-sync), [Vagaro Google Calendar Sync Support](https://support.vagaro.com/hc/en-us/articles/31275501148699-Sync-Vagaro-with-Google-Calendar).

[^5]: Google Calendar API is free with a quota of 1,000,000 queries/day. Source: [Google Calendar API Docs](https://developers.google.com/workspace/calendar/api/guides/quota).

[^6]: Google Calendar push notifications via `events.watch()` provide near-real-time change notifications. Watch channels expire after a maximum of 7 days and must be renewed programmatically. Source: [Google Calendar Push Notifications](https://developers.google.com/workspace/calendar/api/guides/push), [events.watch() Reference](https://developers.google.com/workspace/calendar/api/v3/reference/events/watch).

[^7]: Vagaro supports unlimited double-booking: "No limit to how many bookings per time slot." Source: [Vagaro Support: Double-Book Time Slots](https://support.vagaro.com/hc/en-us/articles/360010603213).

[^8]: Vagaro's "Convert to Appointment" exists only for Google Calendar-imported events in certain views, not for native API-created Personal Tasks. Source: [Vagaro Personal Tasks Support](https://support.vagaro.com/hc/en-us/articles/360010603213-Schedule-Personal-Tasks-and-Closures-on-the-Calendar).

[^9]: Vagaro webhook add-on: $10/month including 5,000 webhook calls ($0.002 per additional call). Full Enterprise API access may require contacting Vagaro's Enterprise Sales team. Requires credit card processing enabled. Source: [Vagaro Webhooks Setup](https://support.vagaro.com/hc/en-us/articles/29521637950875-Set-Up-Webhooks-From-Vagaro), [Vagaro Plans & Pricing](https://support.vagaro.com/hc/en-us/articles/22781768988187-Vagaro-Plans-Pricing-and-Premium-Features), [Vagaro APIs & Webhooks](https://www.vagaro.com/pro/updates/webhooks).

[^10]: Colorado SB24-205 (AI Act) was originally set for February 1, 2026, but SB 25B-004 (signed August 28, 2025) postponed the effective date to **June 30, 2026**. Other states are actively legislating. Source: [Colorado AI Act](https://leg.colorado.gov/bills/sb24-205), [SB 25B-004](https://leg.colorado.gov/bills/sb25b-004), [State AI Laws Overview](https://www.venable.com/insights/publications/2025/11/state-ai-laws-continue-evolving-in-2025).

[^11]: FCC Declaratory Ruling (FCC 24-17), adopted February 2, 2024: AI-generated voices classified as "artificial" under TCPA. Applies to **outbound robocalls**. Source: [FCC Ruling](https://www.fcc.gov/document/fcc-makes-ai-generated-voices-robocalls-illegal), [Full Text](https://docs.fcc.gov/public/attachments/FCC-24-17A1.pdf).

[^12]: The FCC's August 2024 NPRM (FCC 24-84) on AI robocalls proposes to **exempt** "technologies used to answer inbound calls, such as virtual customer service agents." **This is a proposed rule, not final law — it has not been adopted as of February 2026.** Under new FCC Chairman Brendan Carr (January 2025), TCPA regulatory direction is shifting broadly. Do not rely on this proposed exemption as current legal protection. Source: [FCC NPRM](https://www.fcc.gov/document/fcc-proposes-first-ai-generated-robocall-robotext-rules-0), [Wiley Legal Analysis](https://www.wiley.law/alert-FCC-Proposes-New-Rules-for-AI-Generated-Calls-and-Texts).

[^13]: Florida Statute 934.03: all-party consent required for call recording. Violation is a third-degree felony. Source: [Florida Recording Laws](https://recordinglaw.com/party-two-party-consent-states/florida-recording-laws/).

[^14]: Twelve states require all-party consent for phone call recording: CA, CT, DE, FL, IL, MD, MA, MT, NV, NH, PA, WA. Note: Nevada (NRS 200.620) requires all-party consent specifically for telephone calls. Michigan is excluded — courts have consistently held it is a one-party consent state. Source: [Two-Party Consent States](https://recordinglaw.com/party-two-party-consent-states/), [Nevada Recording Laws](https://www.shouselaw.com/nv/blog/laws/is-it-illegal-to-record-someone-without-consent-in-nevada/).

[^15]: Retell AI all-in cost with GPT-4.1 mini: $0.101/min base (voice infra $0.055/min + TTS $0.015/min + GPT-4.1 mini $0.016/min + managed Twilio $0.015/min). Optional add-ons (knowledge base +$0.005/min, denoising +$0.005/min) increase cost. $10 free credits = ~60-90 minutes in practice. Source: [Retell AI Pricing](https://www.retellai.com/pricing), [Retell GPT-4.1 Blog](https://www.retellai.com/blog/chatgpt-4-1-ai-phone-calling-retell).

[^16]: Retell-managed phone numbers: $2/month. No separate Twilio account needed. Source: [Retell Phone Numbers](https://docs.retellai.com/deploy/purchase-number).

[^17]: Supabase free tier auto-pauses after 7 days of inactivity. Source: [Supabase Pricing](https://supabase.com/pricing).

[^18]: Retell AI supports mid-call function calling via webhooks to external APIs. Source: [Retell Function Calling Docs](https://docs.retellai.com/integrate-llm/integrate-function-calling).

[^19]: Retell AI supports cold and warm call transfers. Source: [Retell Call Transfer Docs](https://docs.retellai.com/build/single-multi-prompt/transfer-call).

[^20]: Retell fires `call_ended` webhooks with full transcript and word-level timestamps. Source: [Retell Webhook Docs](https://docs.retellai.com/features/webhook).

[^21]: Railway Hobby plan: $5/month. Source: [Railway Pricing](https://railway.com/pricing).

[^22]: Render minimum paid web service: $7/month. Source: [Render Pricing](https://render.com/pricing).

[^23]: As of February 1, 2025, all US carriers block unregistered 10DLC A2P SMS traffic from local numbers. Source: [10DLC Deadline](https://www.txtimpact.com/blog/10dlc-announcement).

[^24]: 10DLC registration: brand registration ($4-15 one-time) + campaign registration ($15 one-time + ~$10/month). Takes 15+ days. Source: [10DLC Registration Guide](https://callhub.io/blog/compliance/10dlc-2025-registration-callhub/).

[^25]: Vagaro Enterprise API requires credit card processing enabled through Vagaro. Source: [Vagaro Support](https://support.vagaro.com/hc/en-us/articles/22781768988187-Vagaro-Plans-Pricing-and-Premium-Features).

[^26]: Twilio supports preserving original caller ID on forwarded calls, but this requires explicitly passing the `CallToken` parameter — it does not happen automatically. Source: [Twilio Caller ID Forwarding](https://www.twilio.com/en-us/changelog/maintain-caller-id-when-call-forwarding-via-programmable-voice-a0), [Call Forwarding with Caller ID](https://www.twilio.com/code-exchange/call-forwarding-caller-id).

[^27]: Twilio number porting typically takes 2-4 weeks for US local numbers. Source: [Twilio Porting](https://help.twilio.com/articles/223179468).

[^28]: Twilio SMS pricing: $0.0083/segment base plus carrier surcharges (~$0.003-$0.006/msg), total ~$0.011-$0.015/message. Toll-free numbers have separate, simpler verification (no 10DLC registration), though requirements are tightening in 2026. Source: [Twilio SMS Pricing](https://www.twilio.com/en-us/sms/pricing/us), [Twilio Carrier Fees](https://help.twilio.com/articles/360016571913-What-are-SMS-and-MMS-Carrier-Fees-).
