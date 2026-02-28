# Builder's Guide

**For: Mark (and anyone who builds the next one)**

This is the implementation kit. The Playbook tells you what we're building and why. This guide tells you how — with copy-paste prompts for Claude Code that generate working code at each step.

---

## How This Guide Works

Each **meta-prompt** below is designed to be pasted directly into a Claude Code session. They're ordered as a build sequence — each prompt builds on what the previous one created. Follow them in order for a clean weekend build.

**What you need before you start:**
- [Claude Code](https://claude.ai/claude-code) installed and working
- A code editor (VS Code recommended)
- Node.js 20+ and npm
- A Twilio account (free trial works)
- An ElevenLabs account *or* a Retell AI account (see comparison below)
- A Google Cloud account (free tier)
- A Supabase account (free tier)
- The **Salon Onboarding Form** filled out (see further below)
- The **Google Calendar Sync Verification** completed (see further below)
- The **Carrier Forwarding Verification** completed — this is a **go/no-go gate**, not optional. See [CHANGELOG.md — CH-002](changelog).

---

## Decision: ElevenLabs vs Retell AI

Pick one before you start. The meta-prompts default to ElevenLabs. If you choose Retell, see "Adapting for Retell AI" at the bottom of this guide for which prompts to swap.

| Factor | ElevenLabs | Retell AI |
|--------|-----------|-----------|
| **Voice quality** | Best in class — most natural-sounding TTS | Good — clear and professional |
| **Response latency** | Sub-second | ~300-500ms (fastest in market) |
| **Twilio integration** | Native (dashboard setup, zero code) | Managed ($2/month, Retell handles Twilio) |
| **Tool calling** | Webhook server tools + system tools (transfer, end call) | `general_tools` array with custom functions |
| **Call transfer** | Conference / Blind / SIP REFER — 3 modes | Warm transfer with context carryover |
| **Cost per minute** | ~$0.08-0.15 (Twilio + ElevenLabs) | ~$0.07-0.12 (all-in, transparent) |
| **Free tier** | Yes (limited minutes) | $10 free credits (~60-90 min) |
| **Compliance** | Standard | SOC 2 Type II, HIPAA, GDPR included |
| **Setup effort** | Low (native) to Medium (WebSocket bridge) | Medium (dashboard + API) |

**Pick ElevenLabs if:** Voice quality matters most. It's a salon — callers hear the voice first, and a natural-sounding agent builds trust. Native Twilio integration means near-zero server setup for the voice layer.

**Pick Retell if:** Conversational speed matters most. Retell's lower latency makes conversations feel more responsive. All-in pricing is simpler. Compliance certifications are included if you plan to scale to medical spas or similar.

---

## Google Calendar Sync Verification

**Do this before writing any code.** The entire booking flow depends on Google Calendar events blocking Vagaro's online booking. This takes 5 minutes to verify.

Vagaro Personal Tasks have a **"Block Online Booking" checkbox** that is enabled by default — dark brown on the calendar means blocked, light brown means not blocked. What we need to confirm is that *synced* Google Calendar events inherit this default.

**Steps:**

1. Make sure the salon's Vagaro account has [two-way Google Calendar sync enabled](https://support.vagaro.com/hc/en-us/articles/31275501148699-Sync-Vagaro-with-Google-Calendar)
2. In Vagaro settings, verify that **"Require Acceptance for Personal Tasks" is OFF** — if it's on, synced tasks stay "pending" and **do not block online booking** until a manager approves them
3. Create a test event in Google Calendar on the synced calendar — make it 1 hour, during a time that currently shows as available on Vagaro's online booking page
4. Wait for sync (1-5 minutes)
5. Check Vagaro's calendar — the event should appear as a Personal Task in **dark brown** (blocked) with a hand icon
6. Go to the salon's Vagaro online booking page as a customer and try to book that same time slot — it **should not be available**
7. Delete the test event from Google Calendar and verify it disappears from Vagaro on next sync

| Result | What It Means | Action |
|--------|--------------|--------|
| Slot is blocked (dark brown, can't book online) | Google Calendar bridge works as designed | Proceed with the build |
| Slot is NOT blocked (light brown, can still book online) | Synced events don't inherit the blocking default | You'll need to manually toggle blocking on synced tasks, or skip the Google Calendar bridge and go straight to Vagaro Enterprise API (Phase 2 architecture) |
| "Require Acceptance" is ON and can't be turned off | Salon owner's Vagaro plan or policy requires approval | Personal Tasks won't block until approved — the hold flow won't work in real-time. Use the "text me a link" path only, or go straight to Vagaro Enterprise API |

**If the test passes:** the Google Calendar bridge is safe to build on. Move on to the Onboarding Form and start building.

**If the test fails:** skip MP-06 (Google Calendar Bridge) and build only the "text me a link" booking path for Phase 1. Plan to go straight to Vagaro Enterprise API integration in Phase 2 for real-time booking.

---

## Salon Onboarding Form

Fill this out for each salon before you start building. The meta-prompts reference this data.

```
SALON NAME:           ____________________
SALON PHONE:          ____________________
SALON ADDRESS:        ____________________
OWNER NAME:           ____________________
OWNER CELL:           ____________________

BUSINESS HOURS:
  Mon: ___________  (or "Closed")
  Tue: ___________
  Wed: ___________
  Thu: ___________
  Fri: ___________
  Sat: ___________
  Sun: ___________

SCHEDULING PLATFORM:  [ ] Vagaro  [ ] Square  [ ] Other: ____
SOLO OR MULTI-STYLIST: [ ] Solo   [ ] Multi

STYLISTS (if multi):
  Name: ____________  Specialties: ____________
  Name: ____________  Specialties: ____________

SERVICES (get the full list from Vagaro):
  Service: __________  Duration: ____ min  Price: $____
  Service: __________  Duration: ____ min  Price: $____
  Service: __________  Duration: ____ min  Price: $____
  (continue for all services)

CANCELLATION POLICY:  ____________________
PARKING NOTES:        ____________________
BOOKING LINK (Vagaro):____________________

GREETING PREFERENCE:
  [ ] Record owner's voice
  [ ] Use TTS (type the greeting): ____________________

AFTER-HOURS BEHAVIOR:
  [ ] AI answers 24/7
  [ ] AI answers during business hours only (ring phone otherwise)

THINGS THE AI SHOULD NEVER DO:
  ____________________
```

Keep this form. It feeds into multiple prompts below, and you'll fill it out again for every new salon.

---

## The Meta-Prompt Library

### Phase 0: Foundation

---

#### MP-01 · Project Scaffold

Paste this into a fresh Claude Code session. Replace `[SALON_NAME_SLUG]` with a lowercase slug (e.g., `salon-alpha`).

````
I'm building a voice agent that answers phone calls for a hair salon. The agent
handles bookings, answers FAQs, and transfers to a human when needed.

Tech stack:
- Runtime: Node.js 20+ with TypeScript (strict mode)
- Framework: Fastify for webhooks and API routes
- Voice platform: Twilio (telephony) + ElevenLabs (conversational AI)
- Booking backend: Google Calendar API (bridge to Vagaro via two-way sync)
- Database: PostgreSQL via Supabase (Drizzle ORM)
- SMS: Twilio Programmable Messaging (toll-free number)
- Testing: Vitest
- Deployment: Docker on Railway

Scaffold this project with:

1. A complete directory structure:
   src/
     server.ts              — Fastify entry point with health check
     config.ts              — Environment variable loading with Zod validation
     webhooks/
       twilio-voice.ts      — Incoming call webhook handler
       twilio-status.ts     — Call status callback handler
       call-ended.ts        — Post-call logging webhook
     clients/
       twilio.ts            — Twilio client wrapper
       elevenlabs.ts        — ElevenLabs Conversational AI client
       google-calendar.ts   — Google Calendar API client
       vagaro.ts            — Vagaro API client (read-only for now)
       sms.ts               — SMS sending via Twilio
     services/
       booking.ts           — Booking logic (check availability, hold slot, confirm)
       faq.ts               — FAQ lookup from knowledge base
       call-router.ts       — Agent ON/OFF toggle logic
       confirmation.ts      — SMS confirmation flow (hold → confirm → expire)
     conversation/
       system-prompt.ts     — Voice agent system prompt template
       knowledge-base.ts    — Salon FAQ data (loaded from config)
     db/
       schema.ts            — Drizzle schema (calls, bookings, config tables)
       migrations/          — Auto-generated migrations
     admin/
       routes.ts            — Admin API routes (toggle, config, call logs)
     types/
       index.ts             — Shared TypeScript types
   tests/
     mocks/                 — Mock servers for external APIs
     fixtures/              — Test data
     webhooks/              — Webhook handler tests
     services/              — Service logic tests
     integration/           — End-to-end flow tests

2. These config files:
   - package.json with all dependencies
   - tsconfig.json (strict mode, ES2022 target)
   - .env.example with ALL required environment variables (placeholder values only)
   - Dockerfile (multi-stage build, non-root user)
   - docker-compose.yml (app + postgres for local dev)
   - vitest.config.ts
   - .gitignore (include .env*, node_modules/, dist/, .claude/settings.local.json)
   - drizzle.config.ts

3. A CLAUDE.md file that describes:
   - Project overview and architecture
   - Key file paths and their responsibilities
   - Available commands (dev, test, lint, typecheck, db:migrate)
   - Environment variable reference
   - API integration patterns
   - Convention: all salon-specific data comes from environment/config, never hardcoded

Do NOT create .env files. Only .env.example with placeholders.
All TypeScript — no `any` types, no @ts-ignore.
````

**What this gives you:** A complete, buildable project with every file in the right place. Every subsequent prompt references this structure.

---

#### MP-02 · Database Schema

````
Create the Drizzle ORM schema in src/db/schema.ts for these tables:

1. calls
   - id (uuid, primary key)
   - twilio_call_sid (text, unique)
   - caller_number (text) — store last 4 digits only for privacy
   - direction (enum: inbound)
   - started_at (timestamp)
   - ended_at (timestamp, nullable)
   - duration_seconds (integer, nullable)
   - intent (enum: booking, faq, transfer, unknown, nullable)
   - outcome (enum: booked, answered, transferred, abandoned, nullable)
   - transcript (jsonb, nullable) — full conversation transcript
   - tool_calls (jsonb, nullable) — record of API calls made during conversation
   - created_at (timestamp, default now)

2. bookings
   - id (uuid, primary key)
   - call_id (uuid, FK to calls, nullable — nullable because salon owner can book manually)
   - google_calendar_event_id (text, nullable)
   - customer_name (text)
   - customer_phone (text) — last 4 digits only
   - service_name (text)
   - stylist_name (text, nullable)
   - start_time (timestamp)
   - end_time (timestamp)
   - status (enum: held, confirmed, expired, cancelled)
   - confirmation_token (text, unique) — for the SMS confirm link
   - confirmed_at (timestamp, nullable)
   - expires_at (timestamp) — 2 hours after creation
   - created_at (timestamp, default now)

3. salon_config
   - id (uuid, primary key)
   - key (text, unique) — e.g., "agent_enabled", "greeting_message", "transfer_number"
   - value (jsonb)
   - updated_at (timestamp, default now)

Create Zod schemas that match each table for input validation.
Generate the initial migration.
Add a seed script (src/db/seed.ts) that inserts default salon_config:
  - agent_enabled: true
  - transfer_number: (from SALON_OWNER_PHONE env var)
  - greeting_message: "Thanks for calling! This call is assisted by AI and may be recorded. How can I help you today?"
  - booking_hold_minutes: 120

Run the migration against the local docker-compose postgres to verify it works.
````

---

### Phase 1: Core Agent (Saturday Morning)

---

#### MP-03 · Call Router (The ON/OFF Toggle)

````
Create the call routing logic in src/services/call-router.ts and the Twilio
voice webhook in src/webhooks/twilio-voice.ts.

The call router checks whether the AI agent is enabled:

1. Read the "agent_enabled" flag from the salon_config table
2. If enabled: return TwiML that connects to our ElevenLabs agent
   (for now, just return a <Say> greeting + <Gather> as a placeholder —
   we'll wire ElevenLabs in the next prompt)
3. If disabled: return TwiML that <Dial>s the salon owner's phone number
   (read "transfer_number" from salon_config)

The Twilio voice webhook (POST /webhooks/twilio/voice) should:
- Validate the Twilio request signature using the X-Twilio-Signature header
- Parse the incoming call data (From, To, CallSid, CallStatus)
- Log the call start to the database (create a row in the calls table)
- Call the router to get the appropriate TwiML response
- Return TwiML with Content-Type: application/xml

Also create:
- POST /webhooks/twilio/status — handles call status callbacks, updates the
  calls table with duration and final status
- A TwiML builder utility (src/webhooks/twiml.ts) that provides helper
  functions for generating valid TwiML XML strings. Keep it simple —
  functions that return XML strings, not a class hierarchy.

Create tests in tests/webhooks/twilio-voice.test.ts:
- When agent is enabled, response contains greeting TwiML
- When agent is disabled, response contains <Dial> to salon phone
- Invalid Twilio signature returns 403
- Call is logged to database on webhook receipt

Run the tests and fix any failures.
````

---

#### MP-04 · ElevenLabs Voice Agent Integration

Choose **one** of these two prompts depending on whether you want the native integration (simpler, less control) or the WebSocket bridge (more control, needed for custom tool calls).

**Option A: Native Integration (recommended for first build)**

````
Configure the ElevenLabs native Twilio integration.

This is primarily a dashboard configuration, but I need supporting code:

1. Create src/clients/elevenlabs.ts with:
   - A function to create/update an ElevenLabs agent via their API
   - Agent configuration including:
     - The system prompt (imported from src/conversation/system-prompt.ts)
     - Server tools for: check_availability, hold_booking_slot, send_sms
     - The transfer_to_number system tool (destination: salon owner's phone)
   - A function to assign a phone number to the agent

2. Create a setup script (scripts/setup-elevenlabs.ts) that:
   - Creates the agent with our system prompt and tools
   - Registers our Twilio number with ElevenLabs
   - Outputs the agent ID for reference
   - This is a one-time setup script, not a runtime service

3. Document the manual dashboard steps in a comment block at the top of the
   setup script:
   - Go to ElevenLabs dashboard → Phone Numbers → Add Phone Number
   - Select Twilio, enter Account SID and Auth Token
   - Enter the Twilio phone number
   - Assign the agent
   - Test by calling the number

4. Update src/webhooks/twilio-voice.ts:
   - When agent is enabled, the call is handled by ElevenLabs (via their
     native webhook on the Twilio number), so our webhook only fires when
     the agent is DISABLED. Update the toggle logic accordingly:
     - Agent ON: ElevenLabs manages the Twilio webhook directly
     - Agent OFF: We swap the Twilio webhook URL back to our server, which
       returns <Dial> TwiML
   - Create src/services/agent-toggle.ts that uses the Twilio API to swap
     the VoiceUrl on our phone number between ElevenLabs' URL and our own
     server URL.

Create tests for the agent toggle logic. Run them.
````

**Option B: WebSocket Bridge (more control)**

````
Create a WebSocket bridge between Twilio Media Streams and ElevenLabs
Conversational AI in src/clients/elevenlabs.ts.

Architecture:
- Twilio sends audio via <Connect><Stream> to our WebSocket endpoint
- Our server opens a second WebSocket to ElevenLabs Conversational AI
- We bridge audio bidirectionally between them

Implementation:

1. Create a WebSocket endpoint at /media-stream that:
   - Accepts the Twilio Media Stream connection
   - On "start" event: extract callSid, caller number from metadata
   - Opens a WebSocket to ElevenLabs Conversational AI API
   - Sends conversation_initiation_client_data to ElevenLabs with:
     - Our agent configuration (system prompt, tools, first message)
     - Dynamic variables: caller_number, call_time, salon_name
   - On Twilio "media" events: forward audio payload to ElevenLabs
   - On ElevenLabs "audio" events: send audio back to Twilio via the
     media stream
   - On ElevenLabs "interruption" events: send a clear message to Twilio
     to stop current playback
   - Handle connection cleanup on call end

2. Update src/webhooks/twilio-voice.ts:
   - When agent is enabled: return TwiML with <Connect><Stream> pointing
     to wss://[our-server]/media-stream
   - Pass caller metadata as <Parameter> elements in the <Stream>
   - When agent is disabled: return <Dial> TwiML (unchanged)

3. Handle ElevenLabs tool calls:
   - When ElevenLabs sends a "tool_call" event, route it to our service
     layer:
     - "check_availability" → src/services/booking.ts
     - "hold_booking_slot" → src/services/booking.ts
     - "send_sms" → src/clients/sms.ts
     - "transfer_to_number" → handled by ElevenLabs natively
   - Send the tool result back to ElevenLabs via the WebSocket

4. Create tests:
   - Mock WebSocket connections for both Twilio and ElevenLabs
   - Test audio bridging (Twilio audio → ElevenLabs, ElevenLabs → Twilio)
   - Test tool call routing
   - Test connection cleanup

Run the tests and fix any failures.
````

---

#### MP-05 · System Prompt & Knowledge Base

Replace the placeholder values with the salon's actual data from the Onboarding Form.

````
Create the voice agent's system prompt and knowledge base.

File: src/conversation/system-prompt.ts

The system prompt is a template function that accepts salon configuration and
returns the full prompt string. This keeps salon-specific data out of the code
and makes the system white-label ready.

interface SalonConfig {
  name: string;
  address: string;
  phone: string;
  ownerName: string;
  hours: Record<string, string>; // { "Monday": "Closed", "Tuesday": "9am-7pm", ... }
  services: Array<{ name: string; duration: number; priceFrom: number; description?: string }>;
  stylists: Array<{ name: string; specialties: string[] }>;
  cancellationPolicy: string;
  parkingNotes: string;
  bookingLink: string; // Vagaro booking URL
  customRules: string[]; // Things the AI should never do
}

function buildSystemPrompt(config: SalonConfig, currentDateTime: string): string

The prompt should instruct the agent to:

PERSONALITY:
- Warm, professional, efficient. Like a great receptionist — friendly but
  doesn't waste the caller's time
- Never robotic or overly scripted
- Use the salon's name naturally in conversation
- Keep responses under 2-3 sentences on the phone. This is voice, not chat.

GREETING (first message):
- "Thanks for calling [Salon Name]! This call is assisted by AI and may be
  recorded. How can I help you today?"

BOOKING INTENT:
- Offer two paths: "I can check availability and get you booked right now,
  or I can text you a link to book at your convenience. Which would you prefer?"
- "Book now" path:
  1. Ask what service they want (if not already stated). If vague ("I need my
     hair done"), ask: "What service are you looking for today?"
  2. Ask if they have a stylist preference (skip for solo stylist salons). If
     no preference: "Would you like whoever is available first?"
  3. For variable-duration services, ask a qualifying question: "Is your hair
     above or below your shoulders? That helps me find the right time slot."
     (See CHANGELOG.md — CH-008)
  4. Ask what day/time works for them
  5. Call check_availability tool with service, stylist, date
  6. Present 2-3 available slots: "We have 10am, 1pm, and 3:30pm open. Which works?"
  7. Confirm before booking: "Just to confirm — a [service] with [stylist] on
     [day] at [time]. Sound good?"
  8. Call hold_booking_slot tool
  9. "You're all set! You'll get a text when it's confirmed."
- "Text me a link" path:
  1. "What number should I text it to?" (or "Should I text it to this number?")
  2. Call send_sms tool with the booking link
  3. "Done! I just sent you the link. Anything else I can help with?"

FAQ INTENT:
- Answer from the knowledge base (services, pricing, hours, parking, policies)
- If asked about a service not on the list: "I don't have info on that specific
  service, but I can have someone call you back, or you can check our full menu
  at [booking link]."
- After answering, offer: "Would you like to book an appointment?"

CANCEL / RESCHEDULE INTENT:
- The AI CANNOT modify existing Vagaro appointments in Phase 1 (no API path).
- If caller wants to cancel or reschedule: "I can't change existing appointments
  directly, but let me connect you with [Owner Name] — or I can take a message
  and they'll get right back to you."
  → Offer transfer or take a detailed message
- See CHANGELOG.md — CH-005 for design rationale

TRANSFER INTENT:
- If caller asks for a human: "Absolutely, let me connect you now."
  → Call transfer_to_number immediately. Don't ask follow-up questions.
- If transfer fails (no answer / voicemail): take a detailed message from the
  caller, then trigger send_sms to salon owner with: "MISSED TRANSFER:
  [Name] ([number]) called about [topic]. They asked: [summary]. Please
  call back." See CHANGELOG.md — CH-003.
- If the AI fails to understand after 2 attempts: "I'm having a little trouble
  hearing — would it be easier if I texted you a booking link?" This gracefully
  degrades to the "text me a link" path. See CHANGELOG.md — CH-006.
- If caller sounds frustrated or upset: offer to transfer immediately

RULES:
- NEVER make up availability. Only offer times returned by check_availability.
- NEVER quote prices not in the service list.
- NEVER repeat a caller's full phone number back to them.
- NEVER keep the caller on a topic for more than 3 exchanges without resolving
  it or offering to transfer.
- When the caller says goodbye, respond warmly and end: "Thanks for calling
  [Salon Name]! Have a great day."

Also create: src/conversation/knowledge-base.ts

Export a function that takes a SalonConfig and returns a structured FAQ object:
  {
    hours: formatted hours string,
    services: formatted service list with prices,
    location: address + parking notes,
    cancellation: cancellation policy text,
    stylists: formatted stylist list with specialties,
    bookingLink: Vagaro URL
  }

This gets injected into the system prompt as reference material.

Create tests that verify:
- System prompt includes the salon name
- System prompt includes all services with prices
- System prompt includes hours
- Knowledge base formats hours correctly
- Knowledge base handles "Closed" days
````

---

### Phase 2: Booking Infrastructure (Saturday Afternoon)

---

#### MP-06 · Google Calendar Bridge

````
Create the Google Calendar integration in src/clients/google-calendar.ts.

This is the booking backend for Phase 1. Vagaro has native two-way Google
Calendar sync — events created in Google Calendar appear as Personal Tasks
in Vagaro within minutes.

Requirements:

1. Authentication:
   - Use a Google Cloud service account (not OAuth — no user interaction needed)
   - Service account email must be shared on the salon's Google Calendar
   - Load credentials from GOOGLE_SERVICE_ACCOUNT_KEY env var (JSON string)
   - Use the googleapis npm package

2. Functions to implement:

   checkAvailability(params: {
     calendarId: string;
     date: string;          // YYYY-MM-DD
     serviceDuration: number; // minutes
     businessHours: { open: string; close: string }; // "09:00", "19:00"
   }): Promise<TimeSlot[]>

   - Query Google Calendar for events on the given date
   - Calculate available slots based on gaps between existing events
   - Only return slots within business hours
   - Only return slots where the gap is >= serviceDuration
   - Return slots in 30-minute increments

   createHold(params: {
     calendarId: string;
     summary: string;       // "HOLD: Balayage — Sarah J."
     description: string;   // Customer name, phone (last 4), service, via AI booking
     startTime: string;     // ISO 8601
     endTime: string;       // ISO 8601
   }): Promise<{ eventId: string }>

   - Create a Google Calendar event with the given details
   - Set the event color to a distinct "hold" color (banana/yellow = colorId "5")
   - Include "AI BOOKING HOLD — pending confirmation" in the description
   - Return the event ID for later deletion if the hold expires

   deleteHold(params: {
     calendarId: string;
     eventId: string;
   }): Promise<void>

   - Delete the Google Calendar event (used when holds expire or are cancelled)

3. Create src/services/booking.ts with:

   checkAvailability(service, stylist, date):
   - Look up the service duration from config
   - Call google-calendar.checkAvailability
   - Format results for the voice agent: human-readable time strings

   holdSlot(service, stylist, startTime, customerName, customerPhone):
   - Create Google Calendar event via createHold
   - Create a booking record in the database with status "held"
   - Generate a confirmation token (nanoid, 12 chars)
   - Set expires_at to 2 hours from now
   - Return the booking ID and confirmation token

   confirmBooking(confirmationToken):
   - Find the booking by token
   - Update status to "confirmed"
   - Set confirmed_at
   - Trigger confirmation SMS to customer (via src/clients/sms.ts)

   expireStaleHolds():
   - Find all bookings where status = "held" AND expires_at < now
   - Delete each Google Calendar event
   - Update status to "expired"
   - Send fallback SMS to customer with booking link
   - This runs on a cron (setInterval every 5 minutes is fine for Phase 1)

Create tests:
- checkAvailability returns only slots within business hours
- checkAvailability excludes slots that overlap existing events
- holdSlot creates both a calendar event and a database record
- confirmBooking updates status and triggers SMS
- expireStaleHolds cleans up old holds

Mock the Google Calendar API in tests (don't make real API calls).
Run all tests and fix any failures.
````

---

#### MP-07 · Vagaro API Client (Read-Only)

````
Create a read-only Vagaro API client in src/clients/vagaro.ts.

The Vagaro Enterprise API uses JWT authentication. The API is limited — we
can READ availability and services but CANNOT create appointments. That's
why we use the Google Calendar bridge for writes.

Implementation:

1. Authentication:
   - POST to the token endpoint with clientId and clientSecretKey
   - Tokens expire after 3600 seconds
   - Auto-refresh when expired
   - Load credentials from VAGARO_CLIENT_ID and VAGARO_CLIENT_SECRET env vars
   - Store the token in memory (single-instance server)

2. Endpoints to wrap:

   getServices(businessId: string): Promise<VagaroService[]>
   - Fetch all services with name, duration, price, category
   - Cache for 1 hour (services don't change often)

   getEmployees(businessId: string): Promise<VagaroEmployee[]>
   - Fetch all stylists with name, assigned services
   - Cache for 1 hour

   searchAvailability(params: {
     businessId: string;
     serviceId: string;
     employeeId?: string;
     date: string; // yyyy-mm-dd
   }): Promise<VagaroTimeSlot[]>
   - Search available appointment slots
   - No caching (real-time data)

3. Create Zod schemas for all API responses. Parse every response through
   the schema — never trust the API to return the right shape.

4. Error handling:
   - VagaroAuthError (401) — trigger token refresh and retry once
   - VagaroRateLimitError (429) — throw with retry-after info
   - VagaroApiError (all others) — include status code and response body

5. Note in a comment: In Phase 2, when we have full Enterprise API access,
   we'll add createPersonalTask() to write directly to Vagaro instead of
   going through Google Calendar.

Create tests with mock API responses. Run them.
````

---

#### MP-08 · SMS Confirmation Flows

````
Create the SMS flows in src/clients/sms.ts and a confirmation endpoint.

Two SMS flows:

FLOW A — "Book me now" (salon-side confirmation)

When the AI holds a slot during a call, our system sends three messages:

1. To the CALLER (immediately after hold):
   "Hi [Name]! We're holding [time] on [day] with [stylist] at [Salon] for
   you. You'll get a text when it's confirmed. ✨"

2. To the SALON OWNER (immediately after hold):
   "NEW BOOKING: [Name] ([last 4 digits of phone]) — [Service] with [Stylist],
   [time] [day]. Check Vagaro for conflicts, then tap to confirm: [confirm link]"

   The "check Vagaro for conflicts" prompt mitigates sync-gap double-booking —
   see CHANGELOG.md — CH-001. The confirm link points to: GET /api/confirm/[confirmation_token]

3. To the CALLER (after salon owner confirms):
   "You're confirmed! [Service] with [Stylist], [time] [day] at [Salon].
   See you then!"

4. To the CALLER (if hold expires — 2 hours, no confirmation):
   "Sorry, we couldn't confirm your [time] [day] slot. Book anytime at:
   [Vagaro booking link]"

FLOW B — "Text me a link"

When the caller just wants a booking link:
   "Here's the booking link for [Salon]: [Vagaro booking link]. Book anytime!"

Implementation:

1. src/clients/sms.ts:
   - Use Twilio's SMS API to send messages
   - Use the toll-free number (from TWILIO_SMS_NUMBER env var) as the sender
   - Function: sendSms(to: string, body: string): Promise<string> (returns message SID)
   - Never log full phone numbers — mask to last 4 digits in logs

2. Confirmation endpoint in src/admin/routes.ts:
   - GET /api/confirm/:token
   - Look up booking by confirmation_token
   - If found and status = "held": confirm the booking (calls confirmBooking)
   - Return a simple HTML page: "✅ Booking confirmed! [details]"
   - If not found or already confirmed/expired: show appropriate message

3. SMS template functions in src/services/confirmation.ts:
   - Templates for all 4 message types above
   - Accept structured data, return formatted message strings
   - Keep messages under 160 characters when possible (single SMS segment)

Create tests:
- Template functions produce correct messages
- Confirmation endpoint confirms a held booking
- Confirmation endpoint rejects expired bookings
- Confirmation endpoint rejects invalid tokens
- Confirm flow triggers SMS to caller

Run all tests.
````

---

#### MP-09 · Call Logging Webhook

````
Create the post-call logging handler in src/webhooks/call-ended.ts.

When a call ends, ElevenLabs (or Retell) fires a webhook with the full
transcript and metadata. We store this in Supabase for analysis.

1. POST /webhooks/call-ended
   - Accept the post-call webhook payload from ElevenLabs
   - Extract: call duration, full transcript, tool calls made, outcome
   - Classify the call intent (booking, faq, transfer, unknown) based on
     which tools were called:
     - check_availability or hold_booking_slot called → "booking"
     - transfer_to_number called → "transfer"
     - Only knowledge-base answers → "faq"
     - None of the above → "unknown"
   - Classify the outcome:
     - hold_booking_slot succeeded → "booked"
     - FAQ answered, caller said goodbye → "answered"
     - transfer_to_number called → "transferred"
     - Call ended without resolution → "abandoned"
   - Update the existing call record in the database with:
     - ended_at, duration_seconds, intent, outcome, transcript, tool_calls
   - Log a structured summary to stdout (for Railway logs)

2. Webhook signature validation:
   - If using ElevenLabs: validate the webhook signature header
   - If using Retell: validate using Retell's HMAC signature

3. Handle idempotency:
   - If we receive a duplicate webhook (same call SID), update rather than
     insert. Don't create duplicate records.

Create tests:
- Valid webhook payload creates/updates a call record
- Intent classification works for each intent type
- Outcome classification works for each outcome type
- Duplicate webhooks don't create duplicate records
- Invalid signature returns 401

Run all tests.
````

---

### Phase 3: Operations

---

#### MP-10 · Admin API

````
Create admin API routes in src/admin/routes.ts for managing the voice agent.

These endpoints power the toggle and will later power a dashboard.

Authentication: Simple API key in the X-Admin-Key header.
Load the expected key from ADMIN_API_KEY env var.
Return 401 if missing or wrong.

Endpoints:

1. GET /api/admin/status
   Returns: { agentEnabled: boolean, todayCalls: number, todayBookings: number }

2. POST /api/admin/toggle
   Body: { enabled: boolean }
   - Updates "agent_enabled" in salon_config
   - If using native ElevenLabs integration: also swaps the Twilio webhook URL
   - Returns: { agentEnabled: boolean }

3. GET /api/admin/calls?page=1&limit=20&from=&to=&intent=&outcome=
   Returns paginated call logs with: id, started_at, duration, intent, outcome
   Does NOT return full transcripts (separate endpoint for that)

4. GET /api/admin/calls/:id
   Returns full call detail including transcript and tool calls

5. GET /api/admin/config
   Returns all salon_config key/value pairs

6. PUT /api/admin/config
   Body: { key: string, value: any }
   Updates a single config value. Validates known keys:
   - agent_enabled: boolean
   - transfer_number: string (phone number format)
   - greeting_message: string
   - booking_hold_minutes: number (15-480 range)

7. GET /api/admin/analytics?from=&to=
   Returns: {
     totalCalls: number,
     avgDuration: number,
     intentBreakdown: { booking: n, faq: n, transfer: n, unknown: n },
     outcomeBreakdown: { booked: n, answered: n, transferred: n, abandoned: n },
     callsByDay: Array<{ date: string, count: number }>,
     bookingConversionRate: number // booked / (booked + transferred + abandoned)
   }

Register all routes as a Fastify plugin.
Create tests for each endpoint. Run them.
````

---

#### MP-11 · Testing Infrastructure

````
Create comprehensive testing infrastructure in tests/.

1. tests/mocks/twilio-mock.ts
   - Factory functions that generate realistic Twilio webhook payloads:
     - createIncomingCallPayload(overrides?)
     - createGatherPayload(speechResult, overrides?)
     - createStatusCallbackPayload(status, overrides?)
   - A mock Twilio signature generator for testing signature validation
   - A mock Twilio API that records sent SMS messages

2. tests/mocks/google-calendar-mock.ts
   - Mock Google Calendar API responses:
     - emptyCalendar(date) — no events on the given date
     - busyCalendar(date, events) — calendar with specific events
     - createEventResponse(eventId) — successful event creation
   - Track created/deleted events for assertions

3. tests/mocks/elevenlabs-mock.ts
   - Mock post-call webhook payloads for different scenarios:
     - successfulBookingCall()
     - faqOnlyCall()
     - transferredCall()
     - abandonedCall()

4. tests/fixtures/salon-config.ts
   - A complete SalonConfig object for a test salon
   - Test services, test stylists, test hours

5. tests/integration/booking-flow.test.ts
   Full booking flow integration test:
   a. Simulate incoming call webhook → verify greeting TwiML
   b. Simulate availability check tool call → verify calendar query
   c. Simulate hold slot tool call → verify calendar event + DB record + SMS
   d. Hit confirmation endpoint → verify booking confirmed + SMS sent
   e. Verify call-ended webhook logs everything correctly

6. tests/integration/toggle-flow.test.ts
   a. Set agent_enabled = false
   b. Simulate incoming call → verify <Dial> TwiML (no AI)
   c. Set agent_enabled = true
   d. Simulate incoming call → verify AI agent TwiML

7. tests/services/hold-expiry.test.ts
   a. Create a held booking with expires_at in the past
   b. Run expireStaleHolds()
   c. Verify: booking status = expired, calendar event deleted, fallback SMS sent

Use Vitest with Fastify's .inject() for HTTP tests (no real server).
All external APIs are mocked — no real API calls in tests.
Run the full test suite. Fix any failures. Report coverage.
````

---

#### MP-12 · Monitoring & Production Hardening

````
Add production monitoring and error handling.

1. src/monitoring/logger.ts
   - Use pino for structured JSON logging
   - Create a child logger factory: createLogger(context: string)
   - Log levels: info for call events, warn for retries, error for failures
   - NEVER log full phone numbers, API keys, or tokens
   - Every log entry includes: timestamp, level, context, message, plus any
     structured data

   Standard log events (log these at specific points):
   - call.started: { callSid, callerLast4 }
   - call.routed: { callSid, destination: "agent" | "phone" }
   - call.tool_called: { callSid, tool, params (sanitized), latencyMs }
   - call.ended: { callSid, duration, intent, outcome }
   - booking.held: { bookingId, service, time }
   - booking.confirmed: { bookingId }
   - booking.expired: { bookingId }
   - sms.sent: { to: "***1234", template }
   - error: { callSid?, error, stack }

2. src/monitoring/health.ts
   - GET /health — returns 200 if server is running
   - GET /health/detailed (admin-auth required) — returns:
     - uptime
     - database: connected/disconnected (try a simple query)
     - lastCallAt: timestamp of most recent call
     - holdExpiryRunning: boolean (is the cron active)

3. Error handling in src/server.ts:
   - Global Fastify error handler that logs errors and returns clean responses
   - Unhandled rejection handler
   - Graceful shutdown on SIGTERM (close DB connections, stop cron)

4. Add request-id tracking:
   - Generate a request ID for every incoming webhook
   - Pass it through all log entries for that request
   - Include it in error responses

Integrate the logger into all existing webhook handlers and services.
Don't add any new features — just instrument what already exists.
Run all tests to make sure nothing broke. Fix any failures.
````

---

### Phase 4: Deploy

---

#### MP-13 · Railway Deployment

````
Prepare the project for deployment on Railway.

1. Verify the Dockerfile works:
   - Multi-stage build (build stage + runtime stage)
   - Runtime stage uses node:20-slim
   - Runs as non-root user
   - Exposes PORT from environment variable
   - Health check: HEALTHCHECK CMD curl -f http://localhost:$PORT/health

2. Create a railway.toml (or verify it works with auto-detection):
   [build]
   builder = "dockerfile"

   [deploy]
   healthcheckPath = "/health"
   healthcheckTimeout = 30
   restartPolicyType = "on_failure"
   restartPolicyMaxRetries = 3

3. Create scripts/setup-env.sh that prints the list of env vars needed:
   - List every variable from .env.example
   - Group by service (Twilio, ElevenLabs, Google, Supabase, Admin)
   - Include notes on where to get each value
   - DO NOT include any actual values

4. Create scripts/verify-deployment.sh that tests a deployed instance:
   - Checks /health endpoint
   - Checks /health/detailed (with admin key)
   - Reports status of each dependency

5. Update the README section of CLAUDE.md with deployment instructions.

Do not actually deploy — just make sure everything is deploy-ready.
Run the full test suite one more time. Report results.
````

---

## Session-by-Session Build Plan

Each session is one focused Claude Code chat. Start fresh (`/clear`) between sessions to keep context clean.

| Session | Duration | Prompts | What You Get |
|---------|----------|---------|--------------|
| **1** | 30 min | MP-01, MP-02 | Project scaffold, database schema, dev environment |
| **2** | 1 hour | MP-03, MP-04 | Incoming call handling, ElevenLabs wired up, agent toggle |
| **3** | 1 hour | MP-05 | System prompt, knowledge base, the AI's personality |
| **4** | 1.5 hours | MP-06, MP-07 | Google Calendar availability + booking, Vagaro client |
| **5** | 1 hour | MP-08, MP-09 | SMS confirmations, call logging |
| **6** | 1 hour | MP-10 | Admin API (toggle, config, call logs, analytics) |
| **7** | 1 hour | MP-11 | Full test suite, mocks, integration tests |
| **8** | 30 min | MP-12, MP-13 | Monitoring, health checks, deploy prep |

**Total: ~8 hours of Claude Code sessions.** The rest is account setup, configuration, and testing with real calls.

---

## After the Build: Testing Checklist

Call the number yourself and test each scenario:

**Pre-flight gate (do this first):**
- [ ] **Carrier forwarding verification** — test conditional forwarding from the salon owner's actual phone to the Twilio number. Try 3+ test calls. If it fails, switch to dedicated Twilio number with updated Google Business listing. This is a **go/no-go gate**. See [CHANGELOG.md — CH-002](changelog).

**Call scenarios:**
- [ ] "I'd like to book a haircut" → AI checks availability → offers times → books
- [ ] "How much is a balayage?" → AI answers with price from knowledge base
- [ ] "Can I speak to someone?" → AI transfers immediately
- [ ] "I need to cancel my appointment Thursday" → AI explains it can't modify existing appointments, offers to transfer or take a message (CH-005)
- [ ] "I want to reschedule" → same as cancel — transfer or message path (CH-005)
- [ ] Vague request: "I need my hair done" → AI asks what service
- [ ] No stylist preference: "whoever is available" → AI handles gracefully
- [ ] "Text me a link to book" → AI asks for number → sends SMS with Vagaro link
- [ ] Caller hangs up immediately → call logged as abandoned
- [ ] Background noise → AI handles or falls back to "want me to text you a link instead?" (CH-006)
- [ ] Multiple questions in one call → AI handles each, then offers to book
- [ ] Toggle agent OFF → call rings salon phone directly
- [ ] Toggle agent ON → AI picks up again
- [ ] Booking hold expires (wait 2 hours) → caller gets fallback SMS
- [ ] Salon owner taps confirm link → caller gets confirmation SMS
- [ ] Transfer fails (no answer) → AI takes message → salon owner gets SMS with caller details and summary (CH-003)

---

## White-Label Notes

Everything in this build is designed to be repeatable:

**What's salon-specific (configuration):**
- Salon name, address, hours → environment variables + salon_config table
- Services, pricing, stylists → loaded from config at startup
- System prompt → template function that accepts SalonConfig
- Greeting → configurable via admin API
- Vagaro booking link → environment variable
- Google Calendar → each salon gets their own calendar

**What's shared (code):**
- The entire codebase. Zero salon-specific logic in the code.
- All webhook handlers, API clients, booking logic, SMS flows, admin API.
- One Docker image. Different env vars per salon.

**To onboard a new salon:**
1. Fill out the Onboarding Form
2. Set up accounts (Twilio number, ElevenLabs agent, Google Calendar, Supabase project)
3. Deploy the same Docker image with new env vars
4. Run the seed script to populate salon_config
5. Test with 10 calls
6. Go live

**Scaling from 1 to N salons:**

| Salons | Architecture | Monthly Cost per Salon |
|--------|-------------|----------------------|
| 1-5 | Separate Railway deployments, one per salon | ~$30 |
| 5-20 | Multi-tenant: single server, route by phone number | ~$15-20 |
| 20+ | Multi-tenant + shared infrastructure, dedicated support | ~$10-15 |

The jump to multi-tenant is a Phase 2 architecture decision. Don't build it until you have 3+ salons running on the single-tenant setup and understand the real operational patterns.

---

## Adapting for Retell AI (Instead of ElevenLabs)

The playbook originally specified Retell AI. If you prefer Retell:

1. **Replace MP-04** with a Retell-specific prompt:
   - Use Retell's managed agent configuration instead of ElevenLabs
   - Retell manages the Twilio number directly ($2/month via Retell dashboard)
   - Tool calls use Retell's `general_tools` array format
   - Transfer uses Retell's `transfer_call` function

2. **Replace MP-09** call-ended webhook:
   - Retell fires a `call_ended` webhook with transcript + word-level timestamps
   - Different payload format, same data

3. **Enable Retell's denoising add-on** (+$0.005/min) — salons are noisy environments (hair dryers, music, chatter). This is on by default for salon deployments. See [CHANGELOG.md — CH-006](changelog).

4. **Everything else stays the same:** booking logic, SMS flows, Google Calendar bridge, admin API, database schema, tests.

The prompts are labeled with which voice platform they reference. Swap only the platform-specific ones.

---

## Troubleshooting

**"Twilio webhook returns 502"**
→ Your server isn't running or the URL is wrong. Check Railway logs. Verify the webhook URL in Twilio console matches your deployed server.

**"AI answers but can't check availability"**
→ The Google Calendar service account doesn't have access to the calendar. Share the calendar with the service account email. Check the GOOGLE_CALENDAR_ID env var.

**"SMS not being received"**
→ Toll-free number verification may be pending. Check Twilio console → Phone Numbers → your toll-free number → Messaging → Verification status.

**"Conditional forwarding not working"**
→ This varies across carriers, phone models, and plan types — not just T-Mobile. Test with `*67` prefix to hide caller ID and try again. If it still fails on the salon owner's specific carrier, **fall back to a dedicated Twilio number** with updated Google Business listing and voicemail greeting directing callers to the new number. Don't attempt Phase 3 number porting as a workaround — that takes 2-4 weeks. See [CHANGELOG.md — CH-002](changelog).

**"Hold expired but Google Calendar event still exists"**
→ The expiry cron may have stopped. Check `/health/detailed` for `holdExpiryRunning`. Restart the server if needed.

**"Google Calendar event synced but doesn't block Vagaro online booking"**
→ Check that the synced Personal Task is dark brown (not light brown) in Vagaro's calendar. If light brown, the "Block Online Booking" flag wasn't set. Also check that "Require Acceptance for Personal Tasks" is OFF in Vagaro settings — pending tasks don't block booking. If neither fixes it, synced events may not inherit the blocking default on your Vagaro plan. Fall back to "text me a link" only for Phase 1 and go straight to Vagaro Enterprise API in Phase 2.

**"Vagaro API returns 403"**
→ API access requires Enterprise plan with credit card processing enabled. Contact Vagaro Enterprise Sales.

**"Caller didn't get a reminder for their AI-booked appointment"**
→ Vagaro sends automated reminders for real appointments, NOT for Personal Tasks. The caller only gets reminders after the salon owner creates the real appointment in Vagaro and enters the caller's correct phone number. Make sure the confirmation SMS to the salon owner prominently displays the caller's contact info. See [CHANGELOG.md — CH-004](changelog).

---

*Last updated: February 2026*
