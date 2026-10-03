# 02 USER FLOW - FitWise

Version 1.0 | 2026-10-01 | Depends on: 01_REQUIREMENTS.md

## How the AI agent must use this file
- Build pages and redirects exactly as the flows below describe. Every screen must implement all states in section 4.
- Do not invent extra steps. If a flow is unclear, ask.

## 1. Main flow
```
Visitor
  |
Landing page ---------------------> About / Disclaimer
  |
Register / Login
  |
First time?  --yes--> Profile setup (onboarding)
  |no                        |
  v                          v
Dashboard <------------------+
  |
  +-> Nutrition analysis -> Diet plan
  +-> Fitness plan -> Exercise coach -> Session saved
  +-> Supplement guide
  +-> Wellness screening
  +-> Progress (add record, charts)
  +-> Profile / Settings (change password, export, delete account)
```

## 2. Flow details

### F1 Registration and onboarding
1. Landing -> button "Build My Wellness Plan" -> Register page.
2. Enter name, email, password -> submit.
3. Success: session created -> redirect to Profile setup.
4. Profile setup (single page, grouped sections): date of birth, sex, height, weight, activity, goal, diet, optional restrictions and experience.
5. On save: backend calculates analysis -> redirect to Dashboard with first plan.
6. If user is under 18: show notice "Some features are limited for your age" and apply minor rules (no weight-loss target, no adult BMI category, no supplements).

Errors: email already used (generic message), weak password (rules shown), invalid profile values (inline field errors).

### F2 Login
1. Login page -> email and password -> submit.
2. Success -> go to the page originally requested, else Dashboard.
3. Failure -> one generic message ("Email or password is incorrect"); after repeated failures show "Too many attempts, try again later" (429).

### F3 Daily use
Dashboard shows today's targets, score, workout and recommendations. User logs water, protein, calories, exercise minutes and weight (Progress page or quick-add card). Score and charts update.

### F4 Exercise coach
1. Choose exercise (Squat or Push-up).
2. Read the setup tips (camera at side or front, full body visible, good light).
3. Press "Start camera" -> browser asks permission.
4. Live view: reps, state (up/down/no person), form feedback.
5. Press "Finish" -> summary (reps, duration) -> "Save session".
6. Dashboard exercise progress updates.

Errors: permission denied, no camera, body not visible, low confidence.

### F5 Supplement guide
1. Open page; read safety notice.
2. Form (prefilled from profile, user can edit): conditions checklist, injury, medicines.
3. Submit -> result tier: standard, minor, or doctor-first.
4. Standard: cards with reasons, mechanism, cautions, search links. Doctor-first or minor: message only.

### F6 Wellness screening
Questions (tiredness, skipped meals, low activity, poor sleep, low diet variety, recurring symptoms) -> result (Low, Moderate, Needs Professional Attention) + disclaimer + suggestion to consult a professional when relevant.

### F7 Account management
Settings: change password, export CSV, delete account (type "DELETE" to confirm). After deletion: session ends, redirect to landing with confirmation.

## 3. Navigation map and access rules
| Page | Access | If not allowed |
|---|---|---|
| Landing, About, Login, Register, Supplement guide (anonymous view) | Public | none |
| Profile, Dashboard, Nutrition, Diet, Fitness, Exercise, Progress, Screening, Settings | Logged in | Redirect to Login, remember return page |
| Admin | Role `admin` | 403 page |
| Unknown URL | any | 404 page |

## 4. Required states for every screen
| State | Behavior |
|---|---|
| Loading | Skeleton or "Loading..." text; buttons disabled; `aria-busy` set |
| Success | Confirmation message in an `aria-live` region |
| Error | Plain explanation and next step; no technical details; show reference id for server errors |
| Empty | Friendly prompt with a button (for example "Add your first weight entry") |
| Unauthorized (401) | Redirect to Login with message "Please log in to continue" |
| Forbidden (403) | "You do not have access to this page" |
| Not found (404) | 404 page with link to Dashboard |
| Offline | Banner "You are offline. Changes will not be saved"; retry button; do not lose form input |
| Validation | Inline message next to the field and a summary at the top |

## 5. Screen x state checklist
| Screen | Loading | Empty | Error | Unauthorized | Offline |
|---|---|---|---|---|---|
| Dashboard | yes | no profile yet -> go to Profile setup | yes | yes | yes |
| Progress | yes | no records -> add first record | yes | yes | yes |
| Diet plan | yes | needs profile | yes | yes | yes |
| Exercise coach | model loading | camera not started | permission, no camera | yes | works for counting, not saving |
| Supplements | submitting | initial form | validation | public | yes |
| Screening | submitting | initial form | yes | yes | yes |

## 6. Minor (under 18) variant
| Feature | Behavior |
|---|---|
| BMI | Shown as a number only; no adult category |
| Calories | Maintenance guidance only; no deficit |
| Supplements | Not suggested; doctor or dietitian advice |
| Fitness plan | Beginner-safe, no extreme volume |
| Notices | Clear, friendly, non-shaming wording |

## 7. Edge cases
- Session expires while on a page: API returns 401 -> show message and redirect to Login, keep return URL.
- User opens two tabs and logs out in one: next API call returns 401 in the other.
- Profile changes: recalculate analysis, keep past progress records unchanged.
- Two records for the same date: second save updates the first (unique per user and date).
