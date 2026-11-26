# Requirements Analysis – End Product

Source: Project Spec – Clinic Voice Agent (LiveKit), Nov 26, 2026.

Target: the **end product** = MVP + V1 + Stretch. Build order still follows the spec: finish each tier (code, tests, README, demo) before starting the next. The point of analysing the whole thing now is to make early choices (schema, providers, hosting, API shape) that later tiers won't force you to redo.

---

## 1. Definition of done (end product)

- A stranger can call the number and book, cancel or reschedule end to end.
- README: architecture diagram, latency table (median + p95), cost per call, ≥3 failure scenarios demonstrated, eval pass rate.
- Tests + LLM-judge evals run in GitHub Actions on every push.
- Dashboard of calls, transcripts, latency and eval scores is deployed.
- 60-second demo video + article on sarmadkhalique.com + LinkedIn post.
- Each Stretch item you pick (spec says 1–2) has a README section with numbers.

---

## 2. Scope by tier

| Tier | Features | Done when |
|------|----------|-----------|
| **MVP** (~2 wks) | Twilio → SIP → LiveKit; STT → LLM → TTS; greet, identify (name + DOB), list slots, book; Postgres with fake seed data; per-turn latency; README + diagram | Phone the number, book, see the row in Postgres; README shows median latency |
| **V1** (+3 wks) | Cancel, reschedule; idempotent booking; medical-advice refusal; human handoff / callback; post-call summary via Redis + Celery with retries; text-mode + LLM-judge tests in CI; React dashboard | CI green; 3 failure scenarios in README; dashboard deployed |
| **Stretch** (+2–4 wks, pick 1–2) | Outbound no-show follow-up; answering-machine detection; Urdu or Arabic; simulated multi-turn callers; concurrent-call load test | Each has a README section with numbers |

---

## 3. Call flows

| # | Flow | Tier | Key behaviour |
|---|------|------|---------------|
| 1 | Greeting | MVP | Answer in a few seconds, name clinic, ask how to help; interruptible |
| 2 | Identify caller | MVP | Name + DOB → `find_patient`; after 2 misses offer message or transfer |
| 3 | Book | MVP | Day/time window + optional doctor → ≤3 slots → read back → book only after explicit "yes" |
| 4 | Cancel | V1 | List upcoming appointments → confirm which → cancel after explicit "yes" |
| 5 | Reschedule | V1 | Atomic: old slot freed only if new booking succeeds |
| 6 | Medical questions | V1 | Never advise; offer handoff; emergencies → tell caller to contact emergency services |
| 7 | Human handoff | V1 | Transfer or take callback message when asked or when stuck |
| 8 | Wrap-up | MVP | Summarise, "anything else?", end call cleanly |
| 9 | Outbound no-show | Stretch | Call patients who missed an appointment, offer rebooking, record outcome (rebooked / call back later / no answer) |

---

## 4. Architecture

```
Inbound:   Caller → Twilio number → Elastic SIP trunk → LiveKit inbound trunk + dispatch rule
                 → LiveKit room → agent (STT → LLM → TTS) ──HTTP──► FastAPI tools API → Postgres
Post-call: agent signals call end → API enqueues job → Redis → Celery worker
                 → summary + eval results → Postgres → Next.js dashboard (reads via API)
Outbound:  Celery beat finds no-shows → LiveKit outbound SIP trunk → Twilio → patient   (Stretch)
```

| Component | Folder | Tech | Tier |
|-----------|--------|------|------|
| Voice agent | `agent/` | LiveKit Agents (Python), turn detector + VAD | MVP |
| Tools API | `api/` | FastAPI + SQLAlchemy + Alembic | MVP |
| Database | – | Postgres | MVP |
| Async worker | `worker/` | Celery + Redis (+ Celery beat for Stretch outbound) | V1 |
| Dashboard | `dashboard/` | Next.js / React, simple password | V1 |
| CI | `.github/workflows/` | GitHub Actions | V1 |
| Local infra | `docker-compose.yml` | Postgres, Redis, API, worker | MVP (Postgres) → V1 (all) |

---

## 5. Agent tools

| Tool | Tier | Notes |
|------|------|-------|
| `find_patient(full_name, dob) -> Patient \| None` | MVP | Exact DOB + loose name match (see gap G4) |
| `list_available_slots(start, end, provider_id?) -> list[Slot]` | MVP | Max 3 |
| `book_appointment(patient_id, slot_id, idempotency_key) -> Appointment` | MVP (key used from V1) | 409 on taken slot → agent offers alternatives |
| `list_patient_appointments(patient_id) -> list[Appointment]` | V1 | **Not in spec**, but flow 4 needs it ("finds the caller's upcoming appointments") |
| `cancel_appointment(appointment_id, idempotency_key) -> Appointment` | V1 | |
| `reschedule_appointment(appointment_id, new_slot_id, idempotency_key) -> Appointment` | V1 | One DB transaction |
| `transfer_to_human(reason) -> None` | V1 | Transfer or callback message |
| `take_message(name, phone, note)` | MVP | **Not in spec**, but flow 2 needs a fallback before V1 handoff exists |
| `end_call()` | MVP | **Not in spec**, but flow 8 needs to hang up |

---

## 6. Data model (end product)

| Table | Columns | Tier |
|-------|---------|------|
| `patients` | id, name, dob, phone | MVP |
| `providers` | id, name, specialty | MVP |
| `slots` | id, provider_id, start_at, end_at | MVP |
| `appointments` | id, patient_id, slot_id, status (`booked`, `cancelled`, `completed`, `no_show`), idempotency_key, created_at, cancelled_at | MVP; statuses used from V1/Stretch |
| `idempotency_keys` | key (PK), operation, request_hash, response_json, created_at | V1 (see G2) |
| `calls` | id, direction (`inbound`/`outbound`), from_number, to_number, started_at, ended_at, outcome, summary, language, stt_seconds, llm_input_tokens, llm_output_tokens, tts_chars, est_cost_usd | MVP (core) → V1 (summary, cost) |
| `call_turns` | id, call_id, role, text, e2e_latency_ms, llm_ttft_ms, tts_ttfb_ms, created_at | MVP |
| `messages` | id, call_id, patient_id?, name, phone, note, created_at | MVP (take-message) / V1 (callbacks) |
| `eval_results` | id, call_id or test_case, judge, verdict, reason, created_at | V1 |
| `follow_ups` | id, appointment_id, call_id, outcome (`rebooked`, `call_back_later`, `no_answer`, `voicemail`), attempted_at | Stretch |

Constraint that matters most: **partial unique index on `appointments(slot_id) WHERE status = 'booked'`**. Add it in MVP.

---

## 7. Non-functional requirements

| Requirement | What it means concretely |
|-------------|--------------------------|
| Latency | Store per-turn metrics; report median + p95; set a target after first measurements; document what you changed to improve it |
| Correctness | No double-booking under retries or concurrent callers (unique index + idempotency keys) |
| Safety | No medical advice; explicit confirmation before book / cancel / reschedule |
| Privacy | Fake data only; transcripts never printed to console in production mode (env flag) |
| Cost | Per-call STT seconds, LLM tokens, TTS characters, estimated USD stored on `calls` |

---

## 8. Failure scenarios (V1 needs ≥3; aim for all)

| Scenario | Expected behaviour | Mechanism |
|----------|--------------------|-----------|
| Caller interrupts | Agent stops, listens | LiveKit interruptions (on by default); verify on the greeting |
| Long silence (15 s) | Prompt once, then end politely | LiveKit user-away state / timer in agent → `end_call` |
| Tools API times out | Apologise, retry once, offer callback | HTTP client timeout (~3 s) + 1 retry; safe because of idempotency key |
| Duplicate booking request | Same key → same appointment | `idempotency_keys` table lookup before insert |
| Two callers, one slot | One wins, other offered alternatives | Unique index → 409 → agent calls `list_available_slots` again |
| Hang-up mid-booking | No half booking; call marked incomplete | Booking is one atomic request; on participant-disconnect set `calls.outcome = 'incomplete'` |
| Medical question | Refuse, offer handoff | System prompt + LLM-judge eval |
| Worker crash | Retry with backoff, summary not lost | Celery `acks_late=True`, `task_reject_on_worker_lost=True`, `autoretry_for` + `retry_backoff` |

---

## 9. Evals and tests

| Suite | Folder | Runs | Tier |
|-------|--------|------|------|
| API unit/integration (booking, idempotency, concurrency) | `tests/api/` | CI, Postgres service container | MVP → V1 |
| Agent text-mode (correct tool, args match request, confirmation before change, refuses medical advice, caller changes mind) | `tests/agent/` | CI, real LLM, fake tool backend | V1 |
| LLM-judge evals | `tests/evals/` | CI | V1 |
| Worker (retry / crash recovery) | `tests/worker/` | CI | V1 |
| Simulated multi-turn callers | `tests/simulations/` | CI or nightly | Stretch |
| Load test (concurrent calls) | `tests/load/` | Manual | Stretch |

---

## 10. Gaps and ambiguities in the spec

Things the spec leaves open or that tiers quietly depend on. Decide these early; several affect MVP code.

**Affect MVP code**

- **G1 – Missing tools.** Spec has no `take_message`, `end_call` or `list_patient_appointments`, but flows 2, 8 and 4 need them (table in §5).
- **G2 – Idempotency for cancel/reschedule doesn't fit a column on `appointments`.** A cancel key doesn't create a new appointment row. Use a separate `idempotency_keys` table storing the response per key; keep `appointments.idempotency_key` optional or drop it.
- **G3 – Who generates the idempotency key.** If the LLM invents it, a retry may use a different key and defeat the purpose. Generate it in agent code (e.g. deterministic from `call_id + operation + slot_id`) and keep it out of the LLM-visible tool signature.
- **G4 – Name matching from speech.** STT misspells names. Exact DOB + fuzzy name (`pg_trgm`) or ask the caller to spell their surname.
- **G5 – Dates and time zones.** Put today's date and clinic time zone in the system prompt; store UTC, speak local time.
- **G6 – Call lifecycle API.** Agent needs endpoints to start a call, log turns, and end a call (which in V1 enqueues the summary job). Design these in MVP so V1 is additive.
- **G7 – Language choice affects provider choice now.** If Urdu/Arabic Stretch is likely, pick STT/TTS providers (and the multilingual turn-detector model) that support it, or accept switching providers later.

**Affect V1 / Stretch**

- **G8 – Human handoff target.** A real transfer needs a real number to ring (yours?) and costs per minute. Callback message only is cheaper and safer for a public demo; decide which.
- **G9 – Emergency number depends on caller country.** 911 / 112 / 1122 (Pakistan). Tie this to the phone-number country decision.
- **G10 – Slot not held between offer and confirm.** Another caller can take it in between. Spec's "two callers" scenario implies this is acceptable (409 → alternatives); confirm you don't want a short hold.
- **G11 – Hosting beyond the agent.** LiveKit Cloud hosts the agent, but API, Postgres, Redis, worker and dashboard need somewhere too (small VM, Fly, Railway, Vercel for dashboard). Factor into budget.
- **G12 – API auth once deployed.** A public tools API that can book/cancel needs at least a shared secret between agent and API; dashboard needs its password.
- **G13 – Agent tests need a fake tool backend.** To run agent evals in CI without a live API, make the HTTP tool client injectable. CI also needs an LLM API key as a secret and will spend tokens on every push — consider running LLM-judge evals only on `main` or with a cap.
- **G14 – Answering-machine detection on SIP.** Verify what Twilio Elastic SIP Trunking and LiveKit offer for AMD; you may need to detect voicemail in the agent itself (e.g. long uninterrupted greeting / beep).
- **G15 – Outbound needs its own trunk.** Outbound calls require a LiveKit outbound SIP trunk + Twilio termination config, and appointments need a way to become `no_show` (seed script or scheduled job).

---

## 11. Decisions to record in `docs/design.md`

- [ ] STT / LLM / TTS providers (consider G7)
- [ ] LiveKit Cloud vs self-hosted; where API/DB/Redis/worker/dashboard run (G11)
- [ ] Phone number: country, new or existing (affects G9)
- [ ] Clinic: 2 doctors, 30-min slots, opening hours, time zone
- [ ] Language(s)
- [ ] Monthly budget cap (phone + APIs + hosting + CI evals)
- [ ] Identification fallback and handoff target (G1, G8)
- [ ] Which 1–2 Stretch items

---

## 12. Build order

1. **MVP** – `api/` schema (with V1-ready columns, unique index, idempotency table), seed, tools + call lifecycle endpoints → `agent/` in console mode → latency logging → Twilio/SIP → measure → README.
2. **V1** – cancel/reschedule + idempotency → failure-scenario handling → `worker/` post-call summary → `tests/` + CI → `dashboard/` → README failure section + eval pass rate.
3. **Stretch** – chosen 1–2 items, each with its own README section and numbers.
4. **Deliverables** – demo video, article, LinkedIn post.
