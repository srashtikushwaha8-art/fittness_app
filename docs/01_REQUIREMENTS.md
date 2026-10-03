# 01 REQUIREMENTS - FitWise

Version 1.0 | 2026-10-01 | Depends on: 00_PROJECT_OVERVIEW.md

## How the AI agent must use this file
- Every feature must trace to a requirement ID (FR-xxx or NFR-xxx). Put the ID in the commit message and test name.
- A requirement is done only when its **acceptance criteria** pass in a test or a manual check.
- Priority: **M** = MVP, **I** = Important, **A** = Advanced.

## 1. Functional requirements

### 1.1 Accounts and authentication
| ID | Requirement | Pri | Acceptance criteria |
|---|---|---|---|
| FR-001 | User can register with email, name and password | M | Valid data creates account; duplicate email returns generic error; password is stored hashed |
| FR-002 | User can log in and log out | M | Correct credentials start a session cookie; logout ends it server-side; wrong credentials give one generic message |
| FR-003 | Protected pages and APIs require login | M | Anonymous request to protected API returns 401; page redirects to login |
| FR-004 | User can view current account (`me`) | M | Returns own name, email, role only |
| FR-005 | User can change password | I | Needs current password; all other sessions end |
| FR-006 | User can delete account and all data | M | Account, profile and all records removed; confirmation required |
| FR-007 | Roles: `user` and `admin` | A | Admin endpoints reject normal users with 403 |

### 1.2 Profile
| ID | Requirement | Pri | Acceptance criteria |
|---|---|---|---|
| FR-010 | Create and edit profile: date of birth, sex, height, weight, activity, goal, diet | M | All fields validated with ranges; saved per user |
| FR-011 | Optional food restrictions and workout experience | M | Optional, length-limited |
| FR-012 | Under-18 detection | M | Minor profile sets `is_minor` behavior: no adult BMI category, no weight-loss target, no supplements |

### 1.3 Body and nutrition analysis
| ID | Requirement | Pri | Acceptance criteria |
|---|---|---|---|
| FR-020 | Calculate BMI and adult category | M | Correct for known test values; says BMI is not a full measure |
| FR-021 | Estimate calories (Mifflin-St Jeor x activity, goal adjustment) | M | Never below BMR; no extreme deficits; documented formula |
| FR-022 | Estimate protein, carbs, fat, fiber, water | M | Match formulas in doc 04 |
| FR-023 | Show selected micronutrient guidance (iron, calcium, vitamin D, B12, zinc, potassium, iodine) | I | Worded as "estimated guidance"; no deficiency claim |
| FR-024 | Every result shows a reason ("Why?") | M | Each recommendation object has `text` and `reason` |

### 1.4 Food and diet
| ID | Requirement | Pri | Acceptance criteria |
|---|---|---|---|
| FR-030 | Indian food database (about 19 foods with calories, protein, carbs, fat, fiber, category, veg type, meal type) | M | Loaded from CSV by seed script |
| FR-031 | Search and filter foods by diet, meal type, category | I | Vegan filter never returns dairy, egg or meat |
| FR-032 | Personalized daily diet plan (breakfast, lunch, snack, dinner) | M | Matches diet type; total calories and protein within range of targets |
| FR-033 | Optional food restrictions respected | I | Restricted foods excluded |

### 1.5 Fitness
| ID | Requirement | Pri | Acceptance criteria |
|---|---|---|---|
| FR-040 | Goal-based weekly fitness plan, beginner and intermediate | M | Plan matches goal; no excessive volume; rest days included |
| FR-041 | Today's workout shown on dashboard | M | Reflects weekday and level |

### 1.6 Exercise coach (computer vision)
| ID | Requirement | Pri | Acceptance criteria |
|---|---|---|---|
| FR-050 | Webcam squat counter | M | Counts reps on recorded test landmarks with 90% or better accuracy |
| FR-051 | Webcam push-up counter | M | Same as above |
| FR-052 | Show exercise, reps, current state, basic form feedback | M | Updates live; states: up, down, no person detected |
| FR-053 | Save session result (exercise, reps, duration) | M | Only counts and time are stored; no video or images |
| FR-054 | Clear permission and error handling for camera | M | Friendly message if denied or unavailable |

### 1.7 Wellness score
| ID | Requirement | Pri | Acceptance criteria |
|---|---|---|---|
| FR-060 | Daily score out of 100: Nutrition 30, Protein 20, Activity 20, Hydration 15, Exercise 15 | M | Parts add to total; formula visible to user |
| FR-061 | Show "biggest improvement area" with reason | M | Picks the part with the largest gap |
| FR-062 | Show "not a medical measurement" notice | M | Present wherever score is shown |

### 1.8 Progress
| ID | Requirement | Pri | Acceptance criteria |
|---|---|---|---|
| FR-070 | Add daily record: weight, exercise minutes, water, protein, calories | M | One record per user per date; edit and delete allowed |
| FR-071 | Charts: weight trend, protein vs target, exercise trend, score trend | M | Chart.js; works with empty data |
| FR-072 | Export own data as CSV | I | Contains only the logged-in user's data |

### 1.9 Supplement guide
| ID | Requirement | Pri | Acceptance criteria |
|---|---|---|---|
| FR-080 | Safety gate: minors, listed conditions, medicines, fracture or surgery get no products or doses | M | Done and tested (7 tests) |
| FR-081 | Show protein, creatine, B12, vitamin D guidance with mechanism, cautions, interactions | M | Done |
| FR-082 | Marketplace **search links** only (no fake brands, prices or payments) | M | Done |
| FR-083 | Inputs not stored or logged | M | Verified in tests and log review |
| FR-084 | Can prefill from profile (age, weight, diet, goal) | I | User confirms before submit |

### 1.10 Wellness screening
| ID | Requirement | Pri | Acceptance criteria |
|---|---|---|---|
| FR-090 | Lifestyle and symptom questions | I | Output only: Low Concern, Moderate Concern, Needs Professional Attention |
| FR-091 | Always show: "This screening is not a medical diagnosis. Consult a qualified healthcare professional for medical concerns." | I | Test checks the text appears |
| FR-092 | Optional manual health metrics compared to documented reference ranges, with advice to seek review | A | Never diagnoses |

### 1.11 Dashboard, admin, chatbot
| ID | Requirement | Pri | Acceptance criteria |
|---|---|---|---|
| FR-100 | Dashboard: greeting, BMI, calories, protein target, score, recommendations, workout, progress bars | M | Loads from one API call |
| FR-110 | Admin: aggregate stats only (user count, errors, usage) | A | No private health data visible |
| FR-120 | Optional AI chatbot via backend | A | Core site works without it; cannot diagnose; cannot override safety gate |

## 2. Non-functional requirements
| ID | Area | Requirement (target) |
|---|---|---|
| NFR-001 | Performance | Calculation and plan APIs respond in under 300 ms (95th percentile) on the host; page loads in under 3 s on 4G |
| NFR-002 | Security | All items marked "must" in doc 09 complete before release |
| NFR-003 | Privacy | Health data never in logs; user can delete all data; minimum data collected |
| NFR-004 | Reliability | `/api/health` and `/api/ready` available; graceful errors; daily database backup in production |
| NFR-005 | Scalability | Stateless app server; can move to PostgreSQL and multiple instances without code changes |
| NFR-006 | Accessibility | WCAG 2.1 AA target: keyboard use, labels, contrast, focus, screen-reader friendly messages |
| NFR-007 | Maintainability | Layered code, type hints, linting, tests, docs updated with each feature |
| NFR-008 | Compatibility | Latest Chrome, Edge, Firefox and Safari; widths 360 px and up |
| NFR-009 | Safety | No diagnosis, no extreme diets, minors protected (see doc 05 and 09) |
| NFR-010 | Observability | Structured logs with request id; errors traceable by reference id |

## 3. Constraints
- Python and FastAPI backend; HTML, CSS, JavaScript frontend (no React); Chart.js for charts.
- Free-tier hosting; small team (one developer with AI assistance).
- No custom LLM training.

## 4. Out of scope
Disease diagnosis, payments, mobile apps, medical device claims, wearable sync, real-time multi-user features.
