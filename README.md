# FrontDesk AI — Bright Roots Dental (Stage 3)

Patient chat + deterministic clinic engine (Stage 1) + persistent tickets and staff dashboard (Stage 2).
No AI key needed.

## Run
    npm install
    npm run serve:local   # production build + local Netlify emulation (real function, Blobs emulator) on :8888
    npm run dev           # Vite only → app runs in LOCAL DEMO MODE (labelled; this browser only)
    npm test              # engine, tickets, UI, API, real-browser E2E (needs: npx playwright-core install chromium-headless-shell)

## Routes
    /            patient chat   (#chat-input, #chat-send, .bot-msg, /?q=<question>)
    /staff       staff dashboard (#ticket-queue, [data-ticket][data-status], #staff-reply, #staff-send per ticket)
    /api/*       Netlify Function (netlify/functions/api.mts)
      GET  /api/health                     GET  /api/tickets
      POST /api/chat {conversationId?, message, clientMessageId?}
      GET  /api/conversation?id=...        GET  /api/tickets/:id
      POST /api/tickets/:id/reply  {text, staffName}
      POST /api/tickets/:id/status {status: "closed"|"open", staffName}
      POST /api/tickets/:id/assign {staffName}

## Storage (Netlify Blobs, store "frontdesk", strong consistency)
    ticket/<id>                       ticket details (status, priority, booking, contact, timestamps…)
    msg/<ticketId>/<ms>_<msgId>       each message — immutable, create-only (no lost updates, ever)
    conv/<conversationId>             patient conversation index (ticket ids, contact, greetings)
Ticket-detail updates use ETag compare-and-swap + verify-after-write with retry.

## Status rules
    answered     answered from the clinic file (or a staff member has replied)
    needs human  unknown/unverified answer, booking request
    urgent       emergency (stays urgent after replies until a person closes it)
    closed       only when staff click "Close ticket" (Reopen restores the previous status)

## Local demo mode
If /api/health is not reachable (Vite dev, the single-file preview), the SAME ticket service runs in the
browser on localStorage (or memory in sandboxed frames). It is labelled on the staff page and is NOT shared
across devices. Production on Netlify always uses the server.

## Netlify
Build `npm run build`, publish `dist`, functions `netlify/functions` (see netlify.toml). Not yet deployed.

## Stage 3 — appointments, ageing, priority
Booking requests collect name, phone, treatment, preferred dentist (only if named), day, exact time and
additional information, asking for whatever is missing. Dates/times resolve in Asia/Kolkata and are checked
against the clinic's LISTED hours, lunch break, public holidays and doctor days. FrontDesk AI never confirms.
Booking states: appointment requested → awaiting staff confirmation → confirmed by staff | rejected (reschedule).
    POST /api/tickets/:id/booking {decision: "confirm"|"reject", date?: "YYYY-MM-DD", time?: "HH:MM", note?, staffName}
Ageing (computed on every read): New <15 min, Waiting 15–60, Delayed 60–120, Overdue >2 working hours.
Working time = listed opening hours minus lunch break, excluding emergencies-only days and public holidays
(src/config/holidays.ts: Telangana G.O.Rt.No.1715 general holidays 2026; a clinic file may supply `holidays`).
Priority: Critical = emergency (always top) · High = overdue / due within 30 working min / booking for today
or tomorrow · Normal = booking requests and unanswered enquiries · Low = routine answered or closed.
    GET /api/tickets?sort=priority|oldest|newest

## Stage 4 — clinic editor, live correctness, Staff FAQ
/staff → "Clinic file" tab (ticket queue stays the default). Structured forms for details, timings, doctors,
services (+aliases), insurance, policies and FAQ; Advanced <textarea id="clinic-json"> + <button id="clinic-save">;
"Reset to original" with an inline two-step confirm. Every save is validated (required fields, HH:MM, close after
open, unique services/doctors, valid JSON) on the client AND the server; a broken file is never stored.
Storage: Blobs key clinic/current = {data, version, updatedAt, updatedBy}. Version++ on every save/FAQ/reset.
    GET /api/clinic   PUT /api/clinic {data, baseVersion, staffName} (409 if stale)   POST /api/clinic/reset {confirm:true}
    POST /api/tickets/:id/reply {text, staffName, saveFaq: true}  → appends {q, a, from} to the file's faq list
The engine builds its index from the CURRENT stored file on every message (no cache). FAQ match: normalised token
overlap |A∩B|/max(|A|,|B|) ≥ 0.6, checked only where the answer would otherwise be UNKNOWN; answer quoted verbatim,
labelled "Staff FAQ". Emergency, prompt-injection and medical rules always run first.
Acceptance check (real Chromium):  npm run test:check

## Stage 5 — Hindi/Telugu replies, quick replies, booking times
Language per message: ≥30% Devanagari letters → hi, ≥30% Telugu → te, else en (romanized Hindi/Telugu
best effort → native-script reply). Stored on the ticket; badge + data-lang on /staff.
src/engine/i18n.ts (the "i18n.js" of the brief): en/hi/te templates for every reply, glossary (weekdays +
27 dental terms), quick-reply labels. Facts stay verbatim (names, prices, times, phones, Latin digits).
Hindi/Telugu text is marked "@i18n-review" for a native speaker. "within 2 working hours" is in every language
(translated + the English phrase); the emergency number is on the first line in every language.
Quick replies: <div id="quick-replies"> below the thread, .quick-reply[data-quick=timings|doctors|prices|
insurance|location|book|human]; context chips: phone / book after UNKNOWN, tel: call after an emergency,
add-phone after a booking without a phone. "Talk to a human" → needs-human ticket, type "human".
Booking time: today/tomorrow/day after (kal, परसों, రేపు, ఎల్లుండి), [next] weekday, 21 Nov, Nov 21, 21/11,
5pm, 5:30 pm, 17:00, "at 5" (1–7 → PM, 8–11 → AM), 5 बजे / 5 గంటలకు; morning 09–12, afternoon 12–16,
evening 16–20 (window). Ticket.requestedTime = ISO +05:30 | "start/end" | null → data-requested-time.
Outside hours / closed day / holiday / past → said from the file, still a ticket, data-outside-hours="true".
Acceptance check: npm run test:check5

## Stage 6 — optional model wording (DeepSeek), non-agentic
One model call per eligible message, server-side only. No tools, no function calling, no retries, no memory.
temperature 0 · max 300 output tokens · 6 s timeout · JSON output · per-IP limit 30 / 10 min.
Environment (Netlify → Site configuration → Environment variables; never in code or chat):
    AI_API_KEY   your DeepSeek key            (missing/empty → engine only; /api/ask → 503 {"error":"no_key"})
    AI_MODEL     deepseek-chat                (default)
    AI_BASE_URL  optional, default https://api.deepseek.com (OpenAI-compatible)
Flow (src/ai/*, inside POST /api/chat so the ticket stores the final answer):
 1. engine + safety first. Emergency, injection, discount, medical, booking, greeting/thanks, human, contact,
    Staff FAQ, follow-ups, quick-reply chips and named-but-unlisted services (Invisalign…) NEVER reach the model.
 2. fact answers and generic unknowns → ONE call with the clinic file + engine facts; patient text in tags.
 3. validator (code): JSON, boolean answerable, every number/price/time/doctor in the file or the patient's
    message, the engine's exact times/prices/names kept, banned words, ≤600 chars, language, a real source.
 4. valid & answerable → shown with "Source: …" and no label · ticket.source = "ai".
    valid & not answerable → engine answer (fact) or the UNKNOWN path (needs human).
    anything else → engine answer + "Answered from the clinic file" · ticket.source = "file", ticket.aiNote = reason.
 ?nokey=1 forces the fallback for testing. POST /api/ask {message, lang, clinic, facts} is also available.
Acceptance check (test provider: scripts/mock-ai.ts):  npm run test:check6

## Final audit (contract items 1–24)
    npm run audit      # real Chromium · prints the PASS/FAIL table (43 rows: 6–19 run with a key AND ?nokey=1)
Light/dark theme: toggle in both headers, saved as localStorage "frontdesk.theme", first visit follows the device.
