# 07 DATABASE DESIGN - FitWise

Version 1.0 | 2026-10-01 | Depends on: 04, 05, 06

## How the AI agent must use this file
- Never change tables by hand or by `CREATE TABLE` in app code. Every change is an Alembic migration (`database/migrations/`).
- Every user-owned table has `user_id` with a foreign key and `ON DELETE CASCADE`. Every repository query filters by `user_id`.
- Use the exact column names, types and constraints below.

## 1. Current state
Phase 1 created 8 tables with plain `sqlite3`: users, profiles, food_items, nutrition_records, exercise_records, progress_records, wellness_screenings, recommendations. Authentication was not included. Migration 001 below formalizes and extends them.

## 2. Conventions
- Primary key: `id` (integer, auto increment; use UUID later if needed).
- Timestamps: `created_at`, `updated_at` in UTC.
- Dates: `date` type (`YYYY-MM-DD`).
- Units: kg, cm, g, ml or liters (named in the column).
- Text lengths limited; enumerations enforced with CHECK constraints.

## 3. ER diagram
```
users 1---1 profiles
  |
  +--- 1..* sessions
  +--- 1..* progress_records     (unique user_id + date)
  +--- 1..* nutrition_records    (unique user_id + date)
  +--- 1..* exercise_records
  +--- 1..* wellness_screenings
  +--- 1..* recommendations
  +--- 1..* audit_log (user_id nullable, SET NULL on delete)

food_items  (shared catalog, no user_id)
```

## 4. Tables

### users
| Column | Type | Constraints |
|---|---|---|
| id | integer | PK |
| email | varchar(254) | NOT NULL, UNIQUE (stored lowercase) |
| name | varchar(80) | NOT NULL |
| password_hash | varchar(255) | NOT NULL (argon2) |
| role | varchar(10) | NOT NULL, default `user`, CHECK in (`user`, `admin`) |
| is_active | boolean | NOT NULL, default true |
| created_at | timestamp | NOT NULL |
| last_login_at | timestamp | NULL |

### sessions
| Column | Type | Constraints |
|---|---|---|
| id | integer | PK |
| user_id | integer | FK users.id ON DELETE CASCADE, NOT NULL |
| token_hash | char(64) | NOT NULL, UNIQUE (SHA-256 of cookie token) |
| created_at | timestamp | NOT NULL |
| expires_at | timestamp | NOT NULL |
Index: `(user_id)`, `(expires_at)` for cleanup.

### profiles
| Column | Type | Constraints |
|---|---|---|
| id | integer | PK |
| user_id | integer | FK users.id CASCADE, UNIQUE, NOT NULL |
| date_of_birth | date | NOT NULL (age is calculated) |
| sex | varchar(10) | CHECK in (`male`, `female`) |
| height_cm | numeric(5,1) | CHECK 100 to 250 |
| weight_kg | numeric(5,1) | CHECK 20 to 300 |
| activity | varchar(12) | CHECK in (`sedentary`, `light`, `moderate`, `very_active`) |
| goal | varchar(16) | CHECK in (`fat_loss`, `muscle_gain`, `maintenance`, `general_fitness`) |
| diet | varchar(16) | CHECK in (`vegetarian`, `non_vegetarian`, `vegan`, `eggetarian`) |
| restrictions | varchar(200) | NULL |
| experience | varchar(12) | NULL, CHECK in (`beginner`, `intermediate`) |
| updated_at | timestamp | NOT NULL |

### food_items (shared catalog)
| Column | Type | Constraints |
|---|---|---|
| id | integer | PK |
| name | varchar(80) | NOT NULL, UNIQUE |
| serving_desc | varchar(60) | NOT NULL (for example "1 roti, 40 g") |
| calories, protein_g, carbs_g, fat_g, fiber_g | numeric | NOT NULL, CHECK >= 0 |
| category | varchar(30) | NOT NULL |
| veg_type | varchar(20) | CHECK in (`vegan`, `vegetarian`, `eggetarian`, `non_vegetarian`) |
| meal_type | varchar(20) | breakfast, lunch, snack, dinner, any |
Index: `(veg_type)`, `(meal_type)`.

### nutrition_records
| Column | Type | Constraints |
|---|---|---|
| id | integer | PK |
| user_id | integer | FK CASCADE, NOT NULL |
| date | date | NOT NULL |
| calories, protein_g, carbs_g, fat_g | numeric | NULL, CHECK >= 0 |
| water_l | numeric(4,1) | NULL, CHECK 0 to 15 |
Unique: `(user_id, date)`.

### exercise_records
| Column | Type | Constraints |
|---|---|---|
| id | integer | PK |
| user_id | integer | FK CASCADE, NOT NULL |
| date | date | NOT NULL |
| exercise | varchar(20) | CHECK in (`squat`, `pushup`, `other`) |
| reps | integer | CHECK 0 to 1000 |
| duration_min | numeric(5,1) | CHECK 0 to 300 |
Index: `(user_id, date)`. No video or image data is stored.

### progress_records
| Column | Type | Constraints |
|---|---|---|
| id | integer | PK |
| user_id | integer | FK CASCADE, NOT NULL |
| date | date | NOT NULL |
| weight_kg | numeric(5,1) | NULL, CHECK 20 to 300 |
| exercise_min | numeric(5,1) | NULL |
| water_l | numeric(4,1) | NULL |
| protein_g, calories | numeric | NULL |
| wellness_score | integer | NULL, CHECK 0 to 100 |
Unique: `(user_id, date)`.

### wellness_screenings
| Column | Type | Constraints |
|---|---|---|
| id | integer | PK |
| user_id | integer | FK CASCADE, NOT NULL |
| date | date | NOT NULL |
| answers | json | NOT NULL (only if the user chooses to save) |
| result | varchar(30) | CHECK in (`low_concern`, `moderate_concern`, `needs_professional_attention`) |

### recommendations
| Column | Type | Constraints |
|---|---|---|
| id | integer | PK |
| user_id | integer | FK CASCADE, NOT NULL |
| date | date | NOT NULL |
| category | varchar(20) | nutrition, fitness, hydration, protein, other |
| text | varchar(300) | NOT NULL |
| reason | varchar(300) | NOT NULL |

### audit_log (security events only)
| Column | Type | Constraints |
|---|---|---|
| id | integer | PK |
| user_id | integer | FK users.id ON DELETE SET NULL |
| event | varchar(40) | login_success, login_failed, logout, password_changed, account_deleted, admin_action |
| created_at | timestamp | NOT NULL |
| meta | json | NULL, never health data |

## 5. Not stored
Supplement guide inputs (conditions, injury, medicines), chatbot messages, webcam video or frames, plain passwords, raw session tokens.

## 6. Indexes summary
`users(email)`, `sessions(token_hash)`, `sessions(user_id)`, `profiles(user_id)`, `progress_records(user_id, date)`, `nutrition_records(user_id, date)`, `exercise_records(user_id, date)`, `food_items(veg_type)`.

## 7. Migrations (Alembic)
| Migration | Content |
|---|---|
| 001 | Baseline: users, profiles, food_items, nutrition_records, exercise_records, progress_records, wellness_screenings, recommendations |
| 002 | Add auth: users.email, password_hash, role, is_active, last_login_at; sessions table |
| 003 | Profile: replace `age` with `date_of_birth`; add CHECK constraints |
| 004 | Unique `(user_id, date)` on progress and nutrition records; indexes |
| 005 | audit_log |
Rules: one purpose per migration; each has upgrade and downgrade; test upgrade on a copy first; never edit a migration that was already applied in staging or production.

## 8. Seed data
- `database/seeds/foods.csv` (about 19 Indian foods): roti, rice, oats, dal, rajma, chana, soybean, paneer, tofu, curd, milk, egg, chicken, fish, peanuts, fruits, vegetables, nuts, seeds.
- `database/seeds/seed_foods.py` is idempotent (safe to run twice).
- `data/exercises.json` for workout templates; `data/supplements.json` already exists.
- Demo user for development only, created by a script, never in production.

## 9. Retention and deletion
| Data | Retention |
|---|---|
| Account data and records | Until the user deletes the account |
| Expired sessions | Deleted daily |
| Audit log | 90 days |
| Backups | 7 to 30 days |
Account deletion removes the user and cascades to all records in one transaction.

## 10. PostgreSQL notes
Use `numeric`, `timestamptz`, `jsonb`. Connect with TLS. Use a least-privilege application user (no superuser, no schema changes at runtime; migrations run with a separate admin role). Enable automated backups and test a restore.

## 11. Queries to support
- Dashboard: latest profile, today's progress record, last 7 days of progress, today's exercise sum.
- Charts: last 30 or 90 days of `progress_records` for the user ordered by date.
- Food filter: by `veg_type` (diet-compatible) and `meal_type`.
