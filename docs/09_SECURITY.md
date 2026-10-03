# 09 SECURITY - FitWise

Version 1.0 | 2026-10-01 | Depends on: 05_THREAT_MODEL.md, 08_API_SPECIFICATION.md

## How the AI agent must use this file
- This is a **binding rulebook**. Implement the "must" items. If a task would break a rule, stop and explain.
- After each feature run the per-feature security check (section 17) before committing.
- Never put secrets, API keys, passwords or tokens in source code, frontend files, logs or documentation.
- Health data is sensitive: never log it, never put it in URLs.

## 1. Security goals
Confidentiality of user data, integrity of calculations and safety rules, availability, accountability (audit log), privacy by design.

## 2. Authentication
| Rule | Detail |
|---|---|
| Password hashing | **Must** use argon2id (`argon2-cffi`, library defaults or stronger). Never MD5/SHA for passwords. Never log or return hashes. |
| Password policy | 10 to 128 characters; reject very common passwords; no composition tricks; allow paste and password managers |
| Login errors | One generic message for unknown email and wrong password; run a dummy hash check when user is missing |
| Throttling | Rate limits from doc 08; delay or temporary block after repeated failures |
| Registration | Do not reveal whether an email is already registered |
| Password change | Needs current password; invalidates all other sessions |
| Reset by email | Not in MVP. If added: single-use, short-lived, hashed tokens |
| Multi-factor | Future (advanced) |

## 3. Session security
- Random token (at least 32 bytes from `secrets.token_urlsafe`), stored as SHA-256 hash in `sessions`.
- Cookie `fitwise_session`: `HttpOnly`, `Secure` (production), `SameSite=Lax`, `Path=/`, no `Domain`.
- Lifetime: 7 days absolute; renew on activity up to the limit; delete expired sessions daily.
- New token on every login (prevents fixation); logout and password change delete sessions.
- Never store tokens in `localStorage`, URLs or logs.

## 4. Authorization (access control)
| Resource | Anonymous | User | Admin |
|---|---|---|---|
| Landing, About, Login, Register | yes | yes | yes |
| Supplement guide API | yes (rate limited) | yes | yes |
| Own profile, analysis, plans, progress, sessions, screening | no | yes (own only) | no direct access to others' health data |
| Other users' data | no | no | no |
| Admin stats | no | no | yes (aggregates only) |
Rules:
- Default deny. Every protected route uses `get_current_user`; admin routes use `require_admin`.
- Use the id from the session, never from the request, for own data. For `/{id}` routes, check ownership in the repository query and return **404** when not owned.
- Enforce permissions in the backend; frontend hiding is only cosmetic.
- Role cannot be set through any public endpoint.

## 5. Input validation
- Pydantic model for every body, query and path parameter: types, ranges, enums, max lengths, trimmed strings.
- Reject unknown enum values; ignore or forbid extra fields (`extra="forbid"` for sensitive schemas).
- Dates validated; weights, heights and amounts within realistic ranges.
- Free text (name, restrictions) limited in length; no HTML allowed; control characters removed.
- Database CHECK constraints as a second layer.
- Validate on the server even if the form validates.

## 6. Injection and output safety
- **SQL:** use SQLAlchemy ORM or bound parameters only. Never build SQL with f-strings.
- **XSS:** render server data with `textContent` or DOM methods; never `innerHTML` with data; sanitize if rich text is ever needed.
- **Command injection:** never pass user input to a shell or `eval`.
- **Open redirects:** the post-login return URL must be a relative path on this site.
- **Log injection:** strip line breaks from anything logged.

## 7. CSRF
- `SameSite=Lax` cookies.
- State-changing endpoints accept only JSON and require `X-Requested-With: fitwise`; reject missing header with 403.
- Validate `Origin` (or `Referer`) against the allowed origin for state-changing requests.
- No state changes with GET.

## 8. Transport and HTTP headers
HTTPS only in staging and production; redirect HTTP to HTTPS; HSTS (`max-age=31536000; includeSubDomains`) once stable.

Required response headers:
```
Content-Security-Policy: default-src 'self'; script-src 'self' https://cdn.jsdelivr.net; style-src 'self' https://fonts.googleapis.com; font-src https://fonts.gstatic.com; img-src 'self' data:; connect-src 'self'; frame-ancestors 'none'; base-uri 'self'; form-action 'self'
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
Referrer-Policy: same-origin
Permissions-Policy: camera=(self), microphone=(), geolocation=()
Cache-Control: no-store        (on authenticated API responses)
```
Notes:
- Inline `<style>` and inline scripts are not allowed under this CSP; move them to files first.
- If MediaPipe is loaded from a CDN, its script, WebAssembly and model hosts must be added deliberately; self-hosting those files is preferred (add `'wasm-unsafe-eval'` only if required).
- Camera permission is allowed only for this site.

## 9. CORS
Frontend and API share one origin, so CORS is disabled by default. If a separate frontend origin is ever needed, allow only that exact origin with credentials; never `*` with cookies.

## 10. API security and rate limiting
- Rate limits per doc 08; return 429 with `Retry-After`.
- Maximum body 100 KB; pagination caps; request timeouts.
- Do not expose internal ids in error messages; same 404 for "missing" and "not yours".
- Disable or protect `/docs` and `/openapi.json` in production.
- Return only the fields the client needs (no password hash, no internal flags).

## 11. Secrets management
- All secrets in environment variables (`.env` locally, host secret store in staging and production).
- `.env` is in `.gitignore`; `.env.example` has names only.
- Required secrets: `SECRET_KEY`, `DATABASE_URL`, `LLM_API_KEY` (optional).
- Run secret scanning in CI (for example gitleaks or GitHub secret scanning). If a secret is ever exposed: **rotate it immediately**, then clean history.
- The frontend never contains keys; the chatbot key is backend only.

## 12. Database security
- ORM with parameterized queries; least-privilege DB user (no superuser, no DDL at runtime).
- TLS to PostgreSQL; database not publicly reachable; strong generated password.
- Migrations only through Alembic; test on staging first.
- Backups encrypted and access-controlled; restore tested (doc 04 and deployment).
- Foreign keys with cascade so account deletion removes all data.

## 13. Encryption
- In transit: TLS everywhere.
- At rest: provider disk and backup encryption; passwords hashed (argon2); session tokens stored as SHA-256 hashes.
- Application-level encryption of health fields is not required for MVP; revisit if requirements change.

## 14. Privacy and sensitive data
- Collect the minimum (see doc 07, "Not stored").
- Supplement inputs, chatbot messages and webcam frames are not stored.
- Users can export and delete their data.
- Minors get restricted features enforced by the server.
- No advertising or third-party tracking scripts.
- LLM calls send minimal context: no name, email or full profile.

## 15. Logging and error handling (security view)
- Global exception handler returns the error envelope with a `reference`; no stack trace to users.
- Log events (login failure, 401/403/429 spikes, validation failures by field name) with request id; **never** passwords, tokens, cookies, health data, request bodies, or chatbot text.
- Audit log table for login, logout, password change, account deletion and admin actions.
- Keep logs 30 days at most; restrict access.
- Frontend: no `console.log` of user data; remove debug output before release.

## 16. Computer vision and uploads
- Camera frames are processed in the browser only; nothing is uploaded or stored; the camera stops when leaving the page or pressing Finish.
- No file uploads in MVP. If added: allow-list types, size limits, random server-side names, store outside the web root, scan, serve with `Content-Disposition` and correct type.

## 17. Per-feature security check (run before every commit)
- [ ] Input validated by a Pydantic schema; limits set
- [ ] Route requires login; ownership enforced in queries
- [ ] No secrets, tokens or health data in code, logs or URLs
- [ ] Frontend renders data with `textContent`
- [ ] Rate limit and body limit considered
- [ ] Error responses use the envelope and leak nothing
- [ ] Tests added: valid, invalid, unauthorized, other-user access
- [ ] Threat model rows reviewed (doc 05)

## 18. Dependency and supply-chain security
- Pin versions; Dependabot enabled; `pip-audit` and `bandit` run in CI and fail the build on high severity.
- Review any new package (maintenance, license, downloads) before adding.
- Self-host or use subresource integrity (SRI) for third-party scripts.

## 19. Security testing
| Test | Tool or method |
|---|---|
| Unauthorized access to every protected endpoint | pytest API tests |
| IDOR (user A reads user B) | pytest with two users |
| Rate limiting and brute-force lockout | pytest |
| SQL injection and XSS payloads | pytest + Playwright |
| Security headers present | pytest checks response headers |
| Session invalid after logout and password change | pytest |
| Static analysis | bandit |
| Dependency vulnerabilities | pip-audit |
| Secret scanning | gitleaks / GitHub |
| Webcam: no network upload | Browser network tab check |

## 20. Incident response (basic)
1. Detect (alerts, logs, report) -> 2. Contain (disable feature, rotate secrets, revoke sessions) -> 3. Fix and test -> 4. Notify affected users if personal data was exposed -> 5. Write a short post-incident note and update the threat model.

## 21. Implementation status
| Control | Status |
|---|---|
| Pydantic validation on `/api/supplements` | Done |
| Supplement safety gate in code | Done |
| Supplement inputs not stored or logged | Done |
| DOM-safe rendering (supplements page) | Done |
| `.env.example`, `.gitignore` for `.env` and `*.db` | Done |
| Authentication, sessions, argon2 | Planned (slice S1) |
| Per-user authorization and IDOR tests | Planned (S1 onward) |
| CSRF protections | Planned (S1) |
| Security headers and CSP (move inline styles first) | Planned |
| Rate limiting, body limits | Planned |
| Global error handler and logging config | Planned |
| CI with pip-audit, bandit, secret scan | Planned |
| Backups and restore test | Planned (before production) |

## 22. Pre-release checklist ("must")
- [ ] HTTPS, HSTS, all headers set; CSP verified
- [ ] `APP_ENV=production`; debug off; `/docs` protected
- [ ] No secrets in repository or history; secrets rotated if exposed
- [ ] Auth, sessions, authorization and IDOR tests pass
- [ ] Rate limits active; error envelope everywhere; no stack traces
- [ ] Logs reviewed: no health data
- [ ] `pip-audit` and `bandit` clean
- [ ] Backup and restore tested; account deletion tested
- [ ] Medical disclaimer visible; safety gate tests pass
