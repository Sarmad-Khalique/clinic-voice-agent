# agent/

LiveKit Agents (Python) voice agent: STT → LLM → TTS pipeline with turn detector + VAD.

- `clinic_agent/prompts/` – system prompt and instruction templates (include today's date + clinic time zone).
- `clinic_agent/tools/` – function tools the LLM can call; thin HTTP wrappers over the tools API. Keep the client injectable so tests can use a fake backend.
- `clinic_agent/outbound/` – (Stretch) no-show follow-up call logic and answering-machine handling.

Responsibilities:
- MVP: greet, identify (name + DOB), offer ≤3 slots, confirm, book, take a message, wrap up, end call; per-turn latency metrics.
- V1: cancel, reschedule, medical-advice refusal, human handoff, API timeout/retry, silence handling, cost metrics.
- Idempotency keys are generated in agent code, not by the LLM.
