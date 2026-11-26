# api/

FastAPI + SQLAlchemy tools service. The agent stays thin and calls this over HTTP.

- `clinic_api/routers/` – endpoints: patients, slots, appointments, messages, call lifecycle (start / turn / end), dashboard read endpoints (V1).
- `clinic_api/models/` – SQLAlchemy models (see `docs/requirements.md` §6).
- `clinic_api/schemas/` – Pydantic request/response models.
- `clinic_api/services/` – booking logic: idempotency, atomic reschedule, slot conflicts.
- `migrations/` – Alembic migrations.
- `seed/` – fake patients, 2 doctors, 30-minute slots. Fake data only.
