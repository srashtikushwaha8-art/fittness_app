# 06 TECH STACK - FitWise

Version 1.0 | 2026-10-01 | Depends on: 04_SYSTEM_ARCHITECTURE.md

## How the AI agent must use this file
- Use only technologies listed here. To add a new library: justify it (what problem, why not an existing tool), add a row here and in `requirements.txt`, and keep versions pinned.
- Prefer the standard library and simple code over new packages.

## 1. Stack at a glance
| Area | Choice | Status |
|---|---|---|
| Frontend | HTML, CSS, vanilla JavaScript | In use |
| Charts | Chart.js | Planned |
| Backend | Python 3.12, FastAPI | In use |
| Validation | Pydantic v2 | In use |
| Configuration | pydantic-settings, python-dotenv | Planned (dotenv in use) |
| Database (dev) | SQLite | In use (plain sqlite3) |
| Database (staging, prod) | PostgreSQL | Planned |
| ORM and migrations | SQLAlchemy 2.0, Alembic | Planned (replaces plain sqlite3) |
| Authentication | Server-side sessions, HttpOnly cookie | Planned |
| Password hashing | argon2-cffi | Planned |
| Rate limiting | slowapi (in-memory now, Redis later if scaled) | Planned |
| Computer vision | MediaPipe Pose (browser, JavaScript) | Planned |
| CV (local demo, optional) | OpenCV, NumPy, MediaPipe Python | Optional |
| Optional AI | LLM API through backend | Optional |
| Testing | pytest, httpx/TestClient, Playwright (E2E) | pytest in use |
| Code quality | ruff (lint and format), mypy (types) | Planned |
| Security tooling | pip-audit, bandit, Dependabot, secret scanning | Planned |
| Version control | Git and GitHub (branches: main, development, feature/*) | Planned |
| CI/CD | GitHub Actions | Planned |
| Hosting | Render (web service + PostgreSQL), alternative Railway or Fly.io | Planned |
| Monitoring | Structured logs, `/api/health`, `/api/ready`, uptime monitor, optional Sentry with data scrubbing | Planned |
| Backups | Provider automated backups + scheduled dump script | Planned |

## 2. Why each major choice
| Technology | Reason | Alternatives considered |
|---|---|---|
| Vanilla HTML/CSS/JS | Simple, fast to build, no build tools, easy to learn and review; project goals do not need a SPA framework | React/Next.js (more complexity than needed for MVP) |
| FastAPI | Modern, fast, built-in validation and OpenAPI docs, great for Python and CV/ML work later | Flask (less built-in validation), Django (heavier) |
| Pydantic | Strong input validation with clear errors; core to security | Manual validation |
| SQLAlchemy + Alembic | Industry standard; migrations are required for professional workflows; works with SQLite and PostgreSQL | Plain sqlite3 (no migrations, not portable) |
| SQLite then PostgreSQL | Zero setup for development; PostgreSQL is reliable and persistent for production | MySQL, MongoDB (relational data fits SQL) |
| Server-side sessions | Revocable on logout and password change; cookie not readable by JavaScript | JWT in localStorage (theft risk), OAuth only (more setup) |
| argon2 | Modern password hashing recommended by security guidance | bcrypt (acceptable), plain hashes (never) |
| MediaPipe Pose in browser | Fast, free, runs on client; no video upload; works when the server has no webcam | OpenCV-only server loop (cannot work on cloud) |
| Chart.js | Simple, accessible enough, well documented | D3 (too heavy), Plotly |
| pytest + Playwright | Standard Python testing; Playwright covers real browser flows including responsive checks | Selenium |
| ruff + mypy | Fast linting and type checks that also teach good habits | flake8, pylint |
| GitHub Actions | Free CI/CD, visible to recruiters | Jenkins, GitLab CI |
| Render | Easy FastAPI deploy, free PostgreSQL option, environment groups | Railway, Fly.io, AWS (too heavy to start) |

## 3. Changes from earlier decisions
| Earlier | Now | Why |
|---|---|---|
| Plain `sqlite3` (D2) | SQLAlchemy + Alembic | Migrations and PostgreSQL support are expected in industrial projects |
| No auth in MVP | Auth in MVP | User data is sensitive; per-user access is required |
| `{"error": "..."}` | Standard error envelope (doc 08) | Consistent client handling |
Migration approach: wrap existing tables as SQLAlchemy models; create the first Alembic migration from the current schema; keep `/api/health` and `/api/supplements` behavior unchanged.

## 4. Planned dependencies (`requirements.txt`)
```
fastapi
uvicorn[standard]
pydantic
pydantic-settings
python-dotenv
sqlalchemy
alembic
psycopg[binary]        # PostgreSQL driver (staging, production)
argon2-cffi
slowapi
pytest
httpx
pytest-cov
ruff
mypy
pip-audit
bandit
# optional later: playwright, opencv-python, mediapipe, numpy (local CV demo only)
```
Pin exact versions when the first working build is tagged.

## 5. Versions and runtime
Python 3.12; Node is not required (no build step). Browser support: current Chrome, Edge, Firefox, Safari.

## 6. Dependency policy
- Max one new top-level dependency per feature, with justification.
- Run `pip-audit` in CI; update on security advisories.
- Remove unused packages monthly.
- Licenses must allow free use (MIT, BSD, Apache-2.0).
