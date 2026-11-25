# txt→md: Text-to-Markdown Converter

A full-stack app that converts `.txt` files into structured Markdown: single files, bulk uploads, ZIP download, history, file management and live preview.

**Stack:** FastAPI · SQLAlchemy 2 · Alembic · PostgreSQL · React + TypeScript + Vite · Tailwind · TanStack Query · Docker Compose.

## Run it with Docker (recommended, works on Windows)

Requires Docker Desktop.

```powershell
copy .env.example .env      # macOS/Linux: cp .env.example .env
# edit .env: set POSTGRES_PASSWORD and SECRET_KEY (32+ random characters)
docker compose up --build
```

- App: http://localhost:3000 (create an account on first visit)
- API docs: http://localhost:8000/docs (disabled when `ENVIRONMENT=production`)
- Migrations run automatically when the backend container starts. Data persists in the `pgdata` and `filestore` volumes.

Generate a secret: `python -c "import secrets; print(secrets.token_urlsafe(48))"`

## Run without Docker

**Backend** (Python 3.11+, PostgreSQL running locally):

```bash
cd backend
python -m venv .venv && source .venv/bin/activate   # Windows: .venv\Scripts\activate
pip install -r requirements-dev.txt
cp .env.example .env                                  # fill in DATABASE_URL and SECRET_KEY
alembic upgrade head
uvicorn app.main:app --reload
```

**Frontend** (Node 20+):

```bash
cd frontend
npm install
npm run dev        # http://localhost:5173, proxies /api to localhost:8000
```

## Tests

```bash
cd backend  && pytest                     # uses SQLite by default
TEST_DATABASE_URL=postgresql+psycopg://user:pass@localhost/txt2md_test pytest   # real PostgreSQL (recommended)
cd frontend && npm test                   # Vitest + React Testing Library
```

## How conversion works

Two modes, both of which only *add* Markdown syntax and never delete text (enforced by a preservation test and a 300-case fuzz test):

| | Standard | Smart formatting |
|---|---|---|
| Existing lists, `---` separators, `====` underlined titles, existing Markdown | ✓ | ✓ |
| Pipe / tab-separated tables, fenced and indented code | ✓ | ✓ |
| Headings from ALL CAPS, numbered sections, title lines, `Chapter N` | | ✓ |
| Whitespace-aligned tables (only when columns clearly line up) | | ✓ |
| Unindented code, bare URLs/emails → autolinks, short-line breaks | | ✓ |

Options: keep original line breaks, and toggles for headings, lists and tables. The engine lives in `backend/app/converters/` with no FastAPI imports, so it can be reused from scripts or workers. Each rule is a small class in `formatting_rules.py`.

## Architecture in brief

- PostgreSQL is the source of truth for users, uploads, conversions (status, timings, JSONB options) and logs. File bytes live behind a `StorageBackend` interface (local disk today; implement the interface for S3).
- Small files convert inline; bulk and large files are queued and claimed by workers using `SELECT … FOR UPDATE SKIP LOCKED`. A reaper re-queues jobs from crashed workers. See `docs/ARCHITECTURE.md`.
- All data is owner-scoped; other users' resources return 404. Filenames are sanitised and storage names are server-generated.

More: [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) · [`docs/API.md`](docs/API.md)

## Known limitations

- The worker runs inside the API process. To scale it independently, run more API replicas or call `app.workers.tasks.run_one` from a Celery task (the claim logic is already multi-worker safe). Redis/Celery are not included.
- Rate limiting is in-memory per process; use a Redis storage URI for slowapi if you run several replicas.
- Heading/table detection is heuristic. Smart mode can misjudge unusual layouts; Standard mode is the conservative choice.
- Very large single "blocks" (over 5,000 consecutive non-blank lines) are split for memory safety.
