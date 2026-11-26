# worker/  (V1)

Celery worker on Redis for post-call work.

- `clinic_worker/tasks/` – post-call summary, call evals, (Stretch) scheduled no-show follow-ups via Celery beat.

Must survive a killed worker: `acks_late=True`, `task_reject_on_worker_lost=True`, retries with backoff.
