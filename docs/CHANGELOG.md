# Changelog

Addenda and corrections to the Salon Alpha documentation. Each entry captures feedback, new insights, or factual corrections. When an entry modifies a claim in an existing document, the affected section is referenced.

---

## [2026-02-27] External Review — First Playbook Critique

**Source:** Independent technical review of SALON-ALPHA-PLAYBOOK.md

### Accepted — Valid Concerns to Address

**CH-001: Sync-gap double-booking is a real risk, not just a monitoring item**

The playbook acknowledges Google Calendar sync latency (1-5 minutes, footnote 4) and says "build monitoring," but doesn't describe what happens when a double-booking actually occurs during that window. Scenario: AI tells caller "you're all set for 2pm," Vagaro still shows 2pm open, someone books 2pm directly on Vagaro → true conflict (not the intentional double-booking Vagaro allows).

**Mitigation (to design before go-live):**
- When the salon owner receives the confirmation SMS, they should check Vagaro for conflicts before tapping confirm. The SMS copy should prompt this: *"Check Vagaro for conflicts before confirming."*
- If a conflict is detected post-confirm, the salon owner contacts the caller directly (phone number is in the SMS). The AI-booked caller was told "you'll get a text when it's confirmed" — they know it's not locked in yet.
- In Phase 2 (Vagaro Enterprise API), this goes away — availability is checked in real-time against Vagaro directly.

*Affects: PLAYBOOK §The Google Calendar Bridge, "How confirmation works" (step 3 SMS copy). No factual correction needed — the playbook is accurate, but the confirmation SMS template should include a conflict-check prompt.*

---

**CH-002: Carrier forwarding deserves more than a risk callout — it's a go/no-go gate**

The playbook flags T-Mobile conditional forwarding as a risk (footnote 2), but the reviewer correctly notes this varies across carriers, phone models, and plan types. Asking a non-technical salon owner to configure conditional call forwarding is a real friction point. If it doesn't work on the first try, confidence in the whole system drops before the AI even speaks.

**Decision point:** Consider making Phase 1 use a dedicated Twilio number (not forwarding) with the salon updating their Google Business listing and voicemail greeting to direct callers there. Less elegant but far more reliable for an alpha.

**Counterpoint:** Conditional forwarding preserves the salon's existing number, which matters for repeat callers and printed marketing materials. Worth testing with the specific salon owner's carrier before committing to either path.

**Action:** Add carrier forwarding verification to the pre-launch checklist as a hard gate (not just a risk to monitor). If forwarding fails testing, fall back to dedicated number approach.

*Affects: PLAYBOOK §Phase 1: The Weekend Build, "Accounts to Set Up." Also affects BUILDERS-GUIDE §Pre-Flight Checklist. No factual correction needed — needs a process addition.*

---

**CH-003: Transfer flow needs a follow-up mechanism**

The playbook says "forward to salon owner's cell" and "takes a message" if no answer, but doesn't specify how the message gets delivered or followed up on. An SMS with call details is implied but not explicit, and there's no system to track whether the salon owner actually called back.

**Design for Phase 1:**
- When transfer fails (no answer / voicemail), AI takes a detailed message and our system sends an SMS to the salon owner: *"MISSED TRANSFER: [Caller Name] (555-0123) called about [topic]. They asked: [summary]. Please call back."*
- No automated follow-up tracking in Phase 1 — the SMS is the notification. If the salon owner doesn't call back, that's the same as missing a voicemail today (no regression).
- Phase 2+: Add a simple follow-up flag in the dashboard.

*Affects: PLAYBOOK §The Full Call Flow, "TRANSFER / COMPLEX REQUEST" branch. The flow diagram shows "takes a detailed message and texts it to you" — this is correct but needs the SMS template specified in the builder's guide.*

---

**CH-004: Reminder gap for AI-booked appointments**

Vagaro sends automated reminders for real appointments but not for Personal Tasks. So between the AI booking and the salon owner creating the real appointment, there's a window where no reminder is queued. Even after the owner creates the real appointment, they need to enter the caller's correct contact info for Vagaro's reminders to work.

**Mitigation:**
- The confirmation SMS to the salon owner should include the caller's phone number and name prominently, so they can enter it accurately when creating the real appointment in Vagaro.
- This is already partially addressed — the playbook's SMS template includes caller details. Emphasis should be on making sure the owner enters the caller's phone number in Vagaro's client record.
- Phase 2 (Vagaro Enterprise API): Personal Tasks created via API could include client info in structured fields, reducing manual entry errors.

*Affects: PLAYBOOK §The Google Calendar Bridge, step 5. No factual correction — operational detail to add to the builder's guide onboarding instructions.*

---

**CH-005: Missing call types — cancel/reschedule and incomplete info**

The playbook doesn't address two very common inbound call scenarios:
1. **Cancellation / rescheduling** — caller wants to change or cancel an existing appointment
2. **Incomplete info** — caller wants to book but doesn't know which stylist, or doesn't specify a service

**Design decisions:**
- **Cancel/reschedule (Phase 1):** AI should NOT attempt to modify existing Vagaro appointments (no API path, and the risk of modifying the wrong appointment is high). Instead: *"I can't modify existing appointments directly, but let me connect you with [Salon Owner] or I can take a message and they'll get back to you right away."* → Transfer or message flow.
- **Cancel/reschedule (Phase 2):** With Vagaro Enterprise API + caller ID lookup, the AI could look up the caller's upcoming appointments and offer to cancel/reschedule. Design this later.
- **Incomplete info:** The system prompt should handle this with qualifying questions. For unknown stylist: *"Do you have a preferred stylist, or would you like whoever is available first?"* For unspecified service: *"What service are you looking for today?"* These are conversation design items for the system prompt, not architectural changes.

*Affects: PLAYBOOK §Phase 1, "Hours 2-3: The system prompt + knowledge base." Also affects BUILDERS-GUIDE system prompt meta-prompts. No factual correction — scope addition.*

---

**CH-006: STT accuracy in noisy salon environments**

No fallback strategy if speech-to-text quality is consistently poor for certain callers or when the salon-side environment is noisy (background music, hair dryers, chatter).

**Mitigation:**
- Retell offers a denoising add-on (+$0.005/min, footnote 15) — enable it by default for salon environments.
- System prompt should include a graceful degradation: if the AI doesn't understand after 2 attempts, offer to text the caller a booking link instead of continuing the voice conversation. *"I'm having a little trouble hearing — would it be easier if I texted you a booking link?"*
- This is a natural fallback to the "text me a link" path that already exists.

*Affects: PLAYBOOK §Phase 1, testing checklist (already includes "background noise" test). BUILDERS-GUIDE should include the denoising add-on in the Retell configuration meta-prompt and the graceful degradation in the system prompt.*

---

### Acknowledged But Deprioritized

**CH-007: Salon owner confirmation friction at scale**

The reviewer raised concern that the manual confirm → create appointment → delete Personal Task workflow becomes a chore at scale. After discussion, this was walked back: at alpha volume (3-5 AI bookings/day), a notification saying "someone wants to give you $150" is not something an owner ignores. The friction is real but tolerable for a single-salon alpha.

**When this becomes a problem:** 30+ salons, high daily volume per salon. At that point, either:
- Automate via `events.watch()` (detect when owner creates real appointment, auto-delete Personal Task)
- Or pursue Vagaro partnership for a proper "Create Appointment" endpoint

**No action needed for Phase 1.** Revisit at scale.

---

**CH-008: Service duration ambiguity**

The reviewer flagged that service durations can vary (e.g., "balayage on waist-length hair" vs "balayage on a bob"). After discussion, this was largely resolved: Vagaro's online booking already handles duration logic — the service-duration mapping is built into the platform. Phase 1 loads these durations into the AI's knowledge base. Phase 2's `Search Appointment Availability` endpoint handles it natively.

**Remaining edge case:** When duration genuinely varies by hair type, the AI should ask a qualifying question: *"Is your hair above or below your shoulders? That helps me find the right time slot."* Add to system prompt.

**No architectural change needed.** Conversation design item for the system prompt.

---

### Rejected / Not Applicable

**CH-009: "Use a dedicated Twilio number instead of forwarding for Phase 1"**

The reviewer suggested skipping conditional forwarding entirely and using a dedicated number with updated Google Business listing. While more reliable technically, this has significant downsides for the alpha:
- Salon's existing number is on all printed materials, signage, and repeat clients' contact lists
- Asking a salon owner to "change your phone number" is a much harder sell than "calls forward when you're busy"
- Google Business listing changes take time to propagate and affect SEO

**Decision:** Keep conditional forwarding as the primary path. Add dedicated number as the documented fallback if forwarding fails carrier testing (see CH-002). Don't lead with it.

*Note: This doesn't contradict the reviewer — they suggested it as an option, not a mandate. We're choosing the primary path based on business context.*
