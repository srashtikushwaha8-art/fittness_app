# 00 PROJECT OVERVIEW - FitWise

| | |
|---|---|
| Version | 1.0 (draft for implementation) |
| Date | 2026-10-01 |
| Status | Phase 1 and the supplement guide are built; everything else is planned |
| Read order | This file first, then 01 to 09 |

## How the AI agent must use this file
- This is the source of truth for **what** we build and **why**. If a request conflicts with this file, stop and ask.
- Do not add features that are not listed here. Do not remove safety limits.
- After finishing a feature, update the status table in section 13 and `CHANGELOG.md`.

## 1. Project name and positioning
**FitWise**: an AI-assisted personalized fitness, nutrition and wellness platform with computer-vision-based exercise analysis.
It is a wellness tool. It is **not** a medical or diagnostic system and must never be presented as one.

## 2. Problem statement
Generic fitness advice (YouTube videos, social media plans, one-size-fits-all apps) ignores a person's age, height, weight, activity level, goal and food preference. Beginners, especially students eating Indian diets, either follow plans that do not suit them or give up because they cannot tell whether they are doing exercises correctly or eating enough protein.

## 3. Who experiences this problem
- Students and young adults starting fitness for the first time
- People on vegetarian, eggetarian or vegan diets who struggle to hit protein targets
- Home exercisers with no trainer to check squat or push-up form
- People confused by supplement marketing

## 4. How it is solved today
| Current option | Gap |
|---|---|
| Generic YouTube / Instagram plans | Not personalized, no accountability, often Western foods |
| Calorie apps (international) | Weak Indian food coverage, no form feedback, no explanation of "why" |
| Personal trainer / dietitian | Costly, not available daily |
| Supplement shops and influencers | Sales-driven, little safety screening |

## 5. Proposed solution
A personalized wellness loop:
```
Profile -> Analysis -> Nutrition -> Fitness -> Exercise Analysis
        -> Wellness Score -> Progress -> (Optional) AI Assistant
```
Unique value:
1. Personalized Wellness Loop
2. Daily Wellness Score out of 100 with transparent parts
3. Explainable recommendations ("Why am I getting this?")
4. Computer-vision exercise analysis (squat and push-up)
5. Safety-first supplement guide (rule-based safety gate)

## 6. Main objective
Build one complete, secure, tested, deployed and documented full-stack product that follows a real software engineering lifecycle and can be demonstrated to recruiters.

## 7. Target users (personas)
| Persona | Description | Needs |
|---|---|---|
| Aarav, 20, college student | Vegetarian, sedentary, wants to lose fat | Simple plan, Indian foods, motivation |
| Meera, 24, working professional | Eggetarian, light activity, wants muscle tone | Protein guidance, home workouts, progress charts |
| Rohan, 17, school student | Wants to get fitter | Safe, age-appropriate guidance only (no weight-loss targets, no supplements) |
| Admin (project owner) | Maintains the system | Health of the service, aggregate usage, no access to private health details |

## 8. Scope
**In scope:** website (responsive), accounts, profile, body and nutrition estimates, Indian food database, diet and fitness plans, wellness score, progress charts, webcam exercise coach (squat and push-up), supplement guide, wellness screening, optional chatbot.
**Out of scope:** disease diagnosis, medical advice, training a custom LLM, mobile apps, payments or real e-commerce, wearable integration (future), machine learning in version 1.

## 9. Limitations (state these in the UI and README)
- All numbers are estimates from general formulas (for example Mifflin-St Jeor); BMI is not a full health measure.
- Form feedback is basic and depends on camera angle and light.
- The wellness score is not a medical measurement.
- Micronutrient output is "estimated guidance", never a claim of deficiency.
- Food database is small (about 19 common items) in version 1.

## 10. Expected outcome
- A logged-in user reaches a personalized plan in under 3 minutes.
- Every recommendation shows a plain-language reason.
- Squat and push-up reps are counted correctly in normal lighting.
- Safety gate blocks unsafe supplement advice for minors, medical conditions and medicines.
- Public GitHub repository with docs, tests, CI, a live demo and screenshots.

## 11. Feature tiers
```
MVP
 |- Authentication (register, login, logout), account deletion
 |- Profile setup and validation
 |- BMI, calories, protein, carbs, fat, fiber, water
 |- Indian food database + rule-based diet plan with reasons
 |- Goal-based fitness plan
 |- Daily wellness score and improvement area
 |- Progress tracking and charts
 |- Squat and push-up counter with basic form feedback
 |- Supplement guide with safety gate (built)
 |- Responsive UI, security basics, tests, deployment

Important
 |- Wellness screening (not a diagnosis)
 |- Explainable recommendation history
 |- Food search and filters
 |- CSV export of own data

Advanced
 |- Admin page (aggregate stats only)
 |- Reminders and notifications
 |- Optional AI chatbot (backend, safety-gated)
 |- Dark mode

Future
 |- More exercises, wearables, smartwatch data
 |- Machine learning (adherence, activity patterns)
 |- Regional Indian diets, multilingual, voice assistant
```

## 12. Success metrics
| Metric | Target |
|---|---|
| MVP features complete | 100% of MVP list |
| Unit test coverage of calculations | 90% or more |
| Critical API endpoints with tests | 100% |
| Security checklist (doc 09) | All "must" items done |
| Lighthouse accessibility score | 90 or more |
| Page load on mobile 4G | under 3 seconds |

## 13. Current status
| Item | Status |
|---|---|
| Project structure, FastAPI, SQLite schema, frontend shell | Done |
| Supplement guide (safety gate, API, page, tests) | Done |
| Documentation set | In progress (this file) |
| Authentication and accounts | Next |
| Everything else in MVP | Planned |

## 14. Assumptions and risks
| Risk | Plan |
|---|---|
| Webcam CV is harder than expected | Start with angles on recorded landmark data, then live camera |
| Health data is sensitive | Collect minimum, never log it, delete on request (docs 05 and 09) |
| Cloud host resets SQLite | Use PostgreSQL in staging and production |
| AI generates insecure code | Follow vertical slices with review and security check per feature |
| Scope creep | Anything not in MVP waits until MVP is finished |

## 15. Build strategy (vertical slices)
Each slice = database + backend + API + frontend + tests + security check, then commit.
| Slice | Content |
|---|---|
| S1 | Auth and account (register, login, logout, delete) |
| S2 | Profile and body analysis (BMI, calories, macros) |
| S3 | Food database and diet plan with reasons |
| S4 | Fitness plan |
| S5 | Wellness score and dashboard |
| S6 | Progress tracking and charts |
| S7 | Supplement guide linked to profile (base already built) |
| S8 | Exercise coach (squat, then push-up) |
| S9 | Wellness screening |
| S10 | Admin stats, monitoring, backups |
| S11 | Optional chatbot |

## 16. Changes from earlier documents
The earlier documentation PDFs described a simpler MVP. This file and docs 01 to 09 **replace** them where they differ:
- Authentication and per-user access control are now part of the MVP (earlier: "to do later").
- Database access moves to SQLAlchemy with Alembic migrations (earlier: plain `sqlite3`).
- Error responses use one standard envelope (see doc 08).

## 17. Glossary
| Term | Meaning |
|---|---|
| BMR | Basal metabolic rate: calories the body uses at rest |
| TDEE | Total daily energy expenditure (BMR x activity) |
| Safety gate | Code rules that block or limit advice for minors, conditions, medicines |
| Vertical slice | One feature built end to end |
| IDOR | Accessing another user's data by changing an id |
