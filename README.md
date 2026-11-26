# clinic-voice-agent

AI phone receptionist for a fictional clinic, built on LiveKit Agents. See `docs/requirements.md`.

## Layout

| Folder | What | Tooling |
|--------|------|---------|
| `agent/` | LiveKit voice agent (`clinic_agent`) | uv workspace member |
| `api/` | FastAPI tools service (`clinic_api`) | uv workspace member |
| `worker/` | Celery post-call worker (`clinic_worker`) | uv workspace member |
| `dashboard/` | Next.js dashboard | pnpm |
| `tests/` | Backend tests | pytest via uv |
| `docs/` | Requirements and design | – |

## Development setup

Backend (Python 3.12, one shared uv workspace and lockfile):

```bash
uv sync                              # all members + dev tools
uv add --package clinic-api <pkg>    # add a dependency to one member
uv run pytest
uv run ruff check .
```

Dashboard:

```bash
cd dashboard && pnpm install && pnpm dev
```
