# 04 SYSTEM ARCHITECTURE - FitWise

Version 1.0 | 2026-10-01 | Depends on: 01, 02, 03

## How the AI agent must use this file
- Respect the layers: Route -> Service -> Repository -> Database. Business logic never goes in routes or in the frontend.
- Calculations are pure functions (no database, no network) so they can be unit tested.
- If you must change the architecture, update this file first and add a row to the decision log (section 13).

## 1. Context diagram
```
        USER (browser)
             |
      FRONTEND (HTML, CSS, JS, Chart.js, MediaPipe in browser)
             | HTTPS, JSON, session cookie
             v
          FastAPI  (serves API at /api and static frontend)
             |
   +---------+-----------------+----------------------+
   |                           |                      |
 DATABASE                 Rule engines            External services
 (SQLite dev,             (calculations,          (optional LLM API,
  PostgreSQL prod)         safety gate)            CDN for Chart.js)
```

## 2. Backend layers
```
Request
  v
Middleware      (request id, security headers, rate limit, logging, CORS)
  v
Route           (FastAPI router, Pydantic validation, auth dependency)
  v
Service         (business logic, orchestrates calculations and repositories)
  v
Repository      (all database queries; always filtered by user_id)
  v
Database
```
| Layer | Folder | Responsibility | Must not |
|---|---|---|---|
| Middleware | `backend/core/` | Request id, headers, rate limits, error handler | Contain business rules |
| Routes | `backend/routes/` | Parse and validate input, call service, return response | Run SQL or formulas |
| Schemas | `backend/schemas.py` | Pydantic request and response models | Hold logic |
| Services | `backend/services/` | Business flows (register, build diet plan, compute score) | Import FastAPI objects |
| Calculations | `backend/calculations/` | BMI, BMR, macros, score, supplement rules | Touch database or network |
| Repositories | `backend/repositories/` | SQLAlchemy queries | Make decisions |
| Models | `backend/models.py` | SQLAlchemy table classes | Hold logic |
| CV | `backend/cv/` | Angle math and rep-counter state machine (pure, testable) | Read webcam directly |

## 3. Frontend structure
```
frontend/
  *.html                  one page each, shared header and footer from nav.js
  css/style.css           tokens and components
  js/nav.js               shared header/footer
  js/services/api.js      the only place that calls fetch (handles 401, errors)
  js/pages/<page>.js      page logic
  js/components/          reusable pieces (score ring, bars, toast)
  js/cv/                  pose + angle + rep counter modules
```
Rules: one `api.js` wrapper; DOM built with `textContent`; no secrets; no `innerHTML` with data.

## 4. Technology summary
See doc 06 for reasons. FastAPI, Pydantic, SQLAlchemy 2.0, Alembic, SQLite (dev) and PostgreSQL (staging, production), argon2, Chart.js, MediaPipe, pytest, Playwright, GitHub Actions.

## 5. Authentication and session flow
```
Browser                 FastAPI                    Database
  |  POST /api/auth/login   |                          |
  |------------------------>| validate input           |
  |                         | find user by email ----->|
  |                         | verify argon2 hash       |
  |                         | create random token      |
  |                         | store SHA-256(token) --->| sessions
  |  Set-Cookie: session=token (HttpOnly, Secure, SameSite=Lax)
  |<------------------------|                          |
  |  GET /api/dashboard (cookie)                       |
  |------------------------>| hash cookie token, look up session
  |                         | check not expired, user active
  |                         | run query WHERE user_id = session.user_id
  |  200 JSON               |                          |
  |<------------------------|                          |
```
Authorization: a `get_current_user` dependency on every protected route; admin routes also require role `admin`. The user id always comes from the session, never from the request body or URL for own data.

## 6. Core data flows

### 6.1 Profile -> analysis
```
PUT /api/profile -> validate -> save profile -> service.build_analysis(profile)
   -> calculations: BMI, BMR, TDEE, goal calories, macros, fiber, water
   -> return analysis with reasons
```

### 6.2 Calculation rules (documented, testable)
| Item | Rule |
|---|---|
| BMR | `10*kg + 6.25*cm - 5*age + 5` (male) or `-161` (female) |
| TDEE | BMR x multiplier: sedentary 1.2, light 1.375, moderate 1.55, very active 1.725 |
| Goal calories | Fat loss about -15% (never below BMR); muscle gain about +10%; maintenance and general fitness 0% |
| Protein | g per kg: muscle gain 1.6, fat loss 1.4, general fitness 1.2, maintenance 1.0 |
| Fat | About 25% of calories (divide by 9) |
| Carbs | Remaining calories (divide by 4) |
| Fiber | About 14 g per 1000 kcal |
| Water | About 35 ml per kg |
| BMI | kg / m^2; adult categories only for age 18+ |
| Minor | No deficit, no adult BMI category, no supplements |

### 6.3 Diet plan
```
Targets (calories, protein) + diet type + restrictions
   -> filter foods (diet, restrictions)
   -> choose items per meal type to approach targets
   -> each meal item gets a "reason"
```
Rule-based, deterministic, no ML.

### 6.4 Wellness score
`score = nutrition(30) + protein(20) + activity(20) + hydration(15) + exercise(15)`. Each part = max points x min(actual / target, 1). Biggest improvement area = part with the largest lost points. Stored per day in `progress_records`.

### 6.5 Supplement guide
```
Request -> Pydantic validation -> SAFETY GATE (minor? condition? medicine? injury?)
   -> blocked: message only  |  allowed: catalog rules -> cards + search links
```
Implemented in `backend/calculations/supplements.py` with `data/supplements.json`. No database writes.

### 6.6 Computer vision (decision D9)
```
Camera -> video element -> MediaPipe Pose (browser) -> 33 landmarks
 -> joint angles (squat: hip-knee-ankle; push-up: shoulder-elbow-wrist)
 -> smoothing -> state machine (up/down) -> rep counter -> form feedback
 -> on finish: POST /api/exercise-sessions (reps, duration only)
```
- Detection runs **in the browser**, so the deployed server never receives video.
- The angle and rep-counter logic is written as a small pure module with the same rules in Python tests (recorded landmark sequences) so it can be verified without a camera.
- A Python OpenCV version may exist for local demos only.

### 6.7 Optional chatbot
`Browser -> POST /api/chat -> safety pre-check -> LLM API (key on server) -> output filter -> response`. The user's profile is sent only as minimal context (no name or email). The safety gate decides what can be advised; the LLM only explains.

## 7. External dependencies
| Service | Purpose | If it fails |
|---|---|---|
| Chart.js CDN (or self-hosted) | Charts | Show table fallback |
| Google Fonts | Fonts | System font fallback |
| MediaPipe assets | Pose model | Hide exercise coach, show message |
| LLM API (optional) | Chatbot | Chat disabled message; site unaffected |
| Marketplaces | Search links only | No dependency (plain links) |
Self-host Chart.js and MediaPipe files in production to tighten the Content Security Policy.

## 8. Storage
| Data | Where |
|---|---|
| Users, profiles, records | Relational database |
| Foods, exercises, supplements catalog | CSV/JSON in `data/`, seeded into database (foods) or loaded at start |
| Sessions | `sessions` table (token hash only) |
| Logs | Standard output collected by host; no health data |
| Files / media | None in MVP (no uploads, no video storage) |

## 9. Environments
| Environment | Database | Purpose | Config |
|---|---|---|---|
| Development | SQLite file | Local work | `APP_ENV=development`, debug logs |
| Staging | PostgreSQL | Test before release | Same as prod with test data |
| Production | PostgreSQL with backups | Real users | `APP_ENV=production`, HTTPS, restricted CORS |

## 10. Project structure (target)
```
fitwise/
  docs/               00 to 09 documents
  frontend/
  backend/
    main.py  core/  routes/  services/  repositories/
    calculations/  cv/  models.py  schemas.py  database.py
  database/
    migrations/       Alembic versions
    seeds/            foods.csv, exercises.json loader
  data/               supplements.json (and catalogs)
  tests/              unit/ api/ e2e/ security/
  scripts/            seed, backup, check scripts
  .github/workflows/  ci.yml
  .env.example  .gitignore  README.md  LICENSE  CHANGELOG.md  requirements.txt
```

## 11. Cross-cutting concerns
| Concern | Approach |
|---|---|
| Configuration | `pydantic-settings` reads environment variables; no secrets in code |
| Logging | Structured logs with request id; never health data (doc 09) |
| Errors | Global handler; one error envelope (doc 08) |
| Validation | Pydantic on every endpoint; database CHECK constraints as backup |
| Time | Store UTC; show local time in the browser |
| Idempotency | Progress records unique per user and date (upsert) |

## 12. Failure modes
| Failure | Behavior |
|---|---|
| Database down | `/api/ready` fails; API returns 503 with friendly message |
| Session expired | 401 -> frontend redirects to login |
| Camera blocked | Message with steps; rest of site works |
| LLM down | Chat disabled message |
| Slow request | Timeout and friendly message; no data loss in forms |

## 13. Decision log (architecture)
| # | Decision | Reason |
|---|---|---|
| A1 | Single FastAPI app serves API and frontend | Simple deployment, no CORS |
| A2 | Server-side sessions with HttpOnly cookie | Revocable, safer than tokens in JavaScript storage |
| A3 | Layered backend | Testable and maintainable |
| A4 | Pure calculation modules | Easy unit tests and explainable results |
| A5 | CV in browser | Cloud server has no webcam; no video leaves the device |
| A6 | Rule-based recommendations | Transparent, no ML in version 1 |
| A7 | SQLAlchemy and Alembic | Migrations and PostgreSQL portability |
