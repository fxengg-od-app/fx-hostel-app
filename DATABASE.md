# Database Specification

This document defines the database contract for the FX Hostel canteen voting application. It is intentionally documentation-only at this stage and should be treated as the source of truth for future schema and migration work.

## 1. Database Philosophy

The database is the authoritative system for vote eligibility, holiday enforcement, and access control (domain + allowlist). The frontend is a client of the database and must never be treated as the source of truth for whether a vote is allowed, whether a session is a holiday, or who is permitted to sign in.

The design favors clarity over over-engineering. The system is intended for a single hostel/canteen and should remain simple to operate and maintain.

## 2. Core Principles

- Use PostgreSQL with Supabase.
- Use UUID primary keys for user-facing business entities where practical.
- Every business table includes created_at and updated_at timestamps.
- Maintain auditability: every vote, menu post, and holiday entry records who created/changed it and when.
- Use Row Level Security to enforce access at the database layer — this is especially critical here because "who is even allowed to have an account" is a core product rule, not just a nice-to-have.

## 3. ER Diagram

```text
users ─< votes
users ─< menu_posts (canteen owner)
users ─< holidays (created_by: super admin or canteen owner)
users ─< student_allowlist (managed by super admin)

sessions (breakfast, lunch, evening_snack, dinner) ─< votes
sessions ─< menu_posts
sessions ─< holidays (nullable session — null means whole-day holiday)

calendar_days (date) ─< votes
calendar_days ─< menu_posts
calendar_days ─< holidays
```

Note: `sessions` can be implemented either as a fixed lookup table with 4 rows, or as a Postgres enum/check-constrained text column. A lookup table is recommended so session metadata (display name, sort order, typical start/end time) can be edited without a code deploy. See Section 4.

## 4. Core Tables

### users

Mirrors the Supabase auth.users record with additional application-level metadata. Applies to all three roles.

| Column | Type | Notes |
|---|---|---|
| id | uuid | Primary key, references auth.users.id |
| role | text | SUPER_ADMIN, CANTEEN_OWNER, or STUDENT |
| name | text | Display name |
| email | text | Unique, must match the configured college domain for STUDENT role |
| room_number | text | Nullable, students only |
| active | boolean | Default true. Deactivated users cannot sign in or vote, but their history is preserved |
| created_at | timestamptz | Default now() |
| updated_at | timestamptz | Default now() |

### student_allowlist

Managed exclusively by SUPER_ADMIN. A student's college email must exist here (and be active) before they may ever gain a `users` row / sign in successfully.

| Column | Type | Notes |
|---|---|---|
| allowlist_id | uuid | Primary key |
| email | text | Unique, required — must match configured college domain |
| name | text | Nullable, pre-filled by admin if known |
| room_number | text | Nullable |
| active | boolean | Default true |
| added_by | uuid | References users.id (must be a SUPER_ADMIN) |
| created_at | timestamptz | Default now() |
| updated_at | timestamptz | Default now() |

### college_settings

Single-row (or small) configuration table, editable only by SUPER_ADMIN.

| Column | Type | Notes |
|---|---|---|
| setting_id | uuid | Primary key |
| college_name | text | Required |
| allowed_email_domains | text[] | e.g. {'xyzcollege.edu'} — enforced at sign-in alongside the allowlist |
| created_at | timestamptz | Default now() |
| updated_at | timestamptz | Default now() |

### sessions (lookup table)

Fixed set of 4 rows, seeded once via migration. Not user-editable in Phase 1 beyond the seed.

| Column | Type | Notes |
|---|---|---|
| session_key | text | Primary key — breakfast, lunch, evening_snack, dinner |
| display_name | text | e.g. "Breakfast" |
| sort_order | int | 1–4 |
| typical_start_time | time | e.g. 06:00 — display/highlighting only, not a hard vote cutoff unless later configured |
| typical_end_time | time | e.g. 09:30 |

### calendar_days

One row per calendar date the app has touched (can be created lazily on first access rather than pre-generated far into the future).

| Column | Type | Notes |
|---|---|---|
| calendar_date | date | Primary key |
| is_full_day_holiday | boolean | Default false — denormalized convenience flag, derived from `holidays` where session_key is null for this date |
| created_at | timestamptz | Default now() |
| updated_at | timestamptz | Default now() |

### holidays

A holiday entry either covers a full day (`session_key` is null) or a specific session (`session_key` set).

| Column | Type | Notes |
|---|---|---|
| holiday_id | uuid | Primary key |
| calendar_date | date | References calendar_days.calendar_date |
| session_key | text | Nullable, references sessions.session_key. Null = whole day |
| reason | text | Nullable, shown in the HolidayBanner |
| created_by | uuid | References users.id — must be SUPER_ADMIN or CANTEEN_OWNER |
| created_at | timestamptz | Default now() |
| updated_at | timestamptz | Default now() |

Unique constraint: one holiday row per (calendar_date, session_key) — including a distinct handling for the "whole day" (null session_key) case, since Postgres treats NULLs as distinct by default; use a partial unique index or a coalesce-based unique expression to prevent duplicate whole-day holiday rows for the same date.

### menu_posts

One row per (calendar_date, session_key), posted/edited by CANTEEN_OWNER. If no row exists for a session, the student view shows the generic session name with no food details.

| Column | Type | Notes |
|---|---|---|
| menu_post_id | uuid | Primary key |
| calendar_date | date | References calendar_days.calendar_date |
| session_key | text | References sessions.session_key |
| items_text | text | Free-form description, e.g. "Aloo Parotta – 2 sets with Kurma". Kept as free text rather than a strict structured item list, since canteen quantities are described informally |
| posted_by | uuid | References users.id — must be CANTEEN_OWNER |
| created_at | timestamptz | Default now() |
| updated_at | timestamptz | Default now() |

Unique constraint: one row per (calendar_date, session_key) — editing re-uses the same row (updated_at bumped), it does not create a new row.

### votes

One row per (student, calendar_date, session_key). This is the core table of the whole application.

| Column | Type | Notes |
|---|---|---|
| vote_id | uuid | Primary key |
| student_id | uuid | References users.id — must be role STUDENT |
| calendar_date | date | References calendar_days.calendar_date |
| session_key | text | References sessions.session_key |
| is_interested | boolean | true = eating canteen food this session, false = not interested |
| created_at | timestamptz | Default now() |
| updated_at | timestamptz | Default now() — bumped when a student changes their vote before the session closes |

Unique constraint: (student_id, calendar_date, session_key) — enforces one vote per student per session at the database level.

## 5. Relationships

- One student has many votes, at most one per (calendar_date, session_key).
- One canteen owner posts many menu_posts, at most one per (calendar_date, session_key).
- One super admin or canteen owner may create many holiday entries.
- The student_allowlist is the gatekeeper for which emails may ever obtain a `users` row with role STUDENT — it is checked in addition to (not instead of) domain-restricted OAuth.
- calendar_days rows are the anchor that votes, menu_posts, and holidays all hang off of; they can be created on-demand (upsert) the first time any of those three tables needs that date, rather than requiring a pre-population job.

## 6. Enums and Controlled Values

Use text columns with explicit check constraints / foreign keys to the `sessions` lookup table rather than a hardcoded Postgres enum type, so session metadata can evolve without an enum migration.

Suggested values:

- role: SUPER_ADMIN, CANTEEN_OWNER, STUDENT
- session_key: breakfast, lunch, evening_snack, dinner (must match `sessions` table exactly)

## 7. Indexes

Recommended indexes:

- votes(calendar_date, session_key) — for the canteen owner's aggregate dashboard query
- votes(student_id, calendar_date) — for a student's "my day" view
- menu_posts(calendar_date, session_key) — already covered by the unique constraint, but confirm it's used as the primary lookup path
- holidays(calendar_date) — for quickly checking "is anything on this date a holiday"
- student_allowlist(email) — already unique, ensure it's the lookup path used at sign-in time

## 8. Constraints

- (student_id, calendar_date, session_key) must be unique in votes — one vote per student per session.
- (calendar_date, session_key) must be unique in menu_posts — one menu per session.
- A vote must not be insertable/updatable for a (calendar_date, session_key) that has a matching row in holidays (either a session-specific holiday or a whole-day holiday for that date) — enforce via trigger, not just RLS, since RLS alone can be awkward for cross-table existence checks.
- A vote must not be insertable for a session whose typical_end_time has already passed on a past calendar_date (i.e. the session is over) — the exact same-day cutoff behavior (see AGENTS.md Section 4a) should be confirmed with the user before being hard-enforced.
- email in student_allowlist must match one of college_settings.allowed_email_domains — enforce in a trigger or check constraint if practical, otherwise in the sign-in/edge logic, but document wherever it lives.

## 9. Triggers and Derived Data

The following should be enforced through database triggers or stored functions:

- Before insert/update on votes: reject if a holiday row exists for (calendar_date, session_key) or (calendar_date, NULL).
- Before insert/update on votes: reject if calendar_date is in the past relative to the session's end time (subject to the confirmed cutoff rule).
- After insert/update/delete on holidays where session_key is NULL: update calendar_days.is_full_day_holiday for that date.
- Upsert calendar_days row on first insert into votes, menu_posts, or holidays for a new date, so the FK is always satisfiable without a separate pre-population step.

## 10. Views

Recommended read-only views, primarily for the canteen owner dashboard and Phase 3 reporting:

- today_vote_summary — interested/not-interested counts per session for the current calendar_date
- upcoming_menu_overview — next N days of menu_posts joined with holiday status
- student_vote_history (Phase 3) — per-student vote history, readable by that student and by SUPER_ADMIN

These views are informational and should not be used as the primary write path.

## 11. Functions

Suggested server-side functions:

- is_session_votable(calendar_date, session_key) → boolean — the single source of truth used by both the vote-insert trigger and any UI pre-check, to avoid the two ever disagreeing.
- get_session_vote_counts(calendar_date, session_key) → { interested_count, not_interested_count } — backs the canteen owner dashboard.
- ensure_calendar_day(calendar_date) — upsert helper used by the triggers in Section 9.

These functions should remain simple and explicit. Avoid over-abstracting them.

## 12. RLS Philosophy

Row Level Security must protect user-facing tables according to role. The database should enforce the same rules the application expects.

The implementation should follow this pattern:

- SUPER_ADMIN: full access to all tables, including student_allowlist, college_settings, users (role management), holidays, menu_posts (read), votes (read, for oversight).
- CANTEEN_OWNER: full read/write on menu_posts and holidays; read-only aggregate access to votes (should not need to see which individual student cast which vote in the UI, even though the data is traceable at the DB level for audit purposes); no access to student_allowlist or role management.
- STUDENT: read/write on their own votes only (student_id = auth.uid()); read-only on menu_posts, holidays, and sessions; no access to other students' votes, student_allowlist, or college_settings.
- Sign-in itself must be gated: a row in student_allowlist (active = true) is required before a STUDENT-role `users` row is considered valid, in addition to the OAuth domain restriction happening at the Supabase Auth provider level.

Tables without a role's relevance should be restricted by default and not exposed to that role.

## 13. Audit Logging

The system should preserve enough information to understand who changed what and when. For the initial implementation, this can be done through:

- created_at and updated_at columns on every table,
- created_by / posted_by / added_by references where practical (already included on holidays, menu_posts, student_allowlist),
- votes are inherently audit-friendly since student_id is always recorded — never build an "anonymous vote" mode without revisiting this section,
- a later extension to a dedicated audit table if the hostel needs stronger traceability (e.g. tracking vote changes over time, not just the latest state).

## 14. Change Strategy for Votes and Menus

- A student may change their vote (interested ↔ not interested) any number of times before the session closes — this updates the existing votes row (updated_at bumped), it does not create a new row or a history table in Phase 1.
- A canteen owner may edit a menu_post for a session any number of times before/during that session — same update-in-place behavior.
- Do not hard-delete a vote or menu_post as part of normal operation; if a correction is needed, update it. Deletion should be reserved for genuine data-entry mistakes and is not a normal user-facing action in Phase 1.

## 15. Numbering Strategy

Not applicable in the same sense as an invoicing system — there is no sequential business document numbering requirement in this application. calendar_date + session_key is the natural, human-meaningful composite key for votes and menu_posts.

## 16. Session Windows and Time Handling

- Store `typical_start_time` / `typical_end_time` in the `sessions` table using the hostel's local time zone convention — confirm with the user whether the deployment needs explicit time zone handling (e.g. `timetz` or storing an explicit IANA zone in college_settings) or whether a single fixed local time zone assumption is acceptable for a single-hostel deployment. Do not silently assume UTC-naive comparisons are safe.
- "Current session" is a derived, frontend-computed value for display/highlighting (via `lib/sessions.js` / `useCurrentSession`) — it should not be persisted, and it should not be the sole gate for vote eligibility (see `is_session_votable` in Section 11, which is the actual enforcement point).

## 17. Holiday Rules

- holidays.session_key NULL = whole-day holiday; a specific session_key = that session only.
- A whole-day holiday should be treated by `is_session_votable` as covering all four sessions for that date, without requiring four separate rows.
- Both SUPER_ADMIN and CANTEEN_OWNER may create holiday entries; only SUPER_ADMIN may manage the student_allowlist and college_settings.

## 18. Vote Aggregation Rules

- The canteen owner dashboard must never require fetching every individual vote row to the client to compute a count — always use `get_session_vote_counts` or the `today_vote_summary` view.
- Counts should distinguish interested vs. not-interested vs. "not yet voted" (total allowlisted active students minus votes cast) so the canteen owner can see both "who's eating" and "who hasn't responded yet."

## 19. Migration Order

The recommended migration order is:

1. base tables: college_settings, sessions (seeded), users, student_allowlist,
2. calendar_days,
3. transactional tables: votes, menu_posts, holidays,
4. triggers and derived-value functions (is_session_votable, get_session_vote_counts, ensure_calendar_day, holiday → calendar_days.is_full_day_holiday sync),
5. RLS policies,
6. reporting views (Phase 3).

Remember: the initial SUPER_ADMIN row(s) must be seeded as part of an early migration per AGENTS.md Section 4b — document the seeded email(s) directly in that migration file.

## 20. Future Scalability

The current design is sufficient for a single hostel and a single canteen. Future growth should be handled through:

- a `hostel_id` / `canteen_id` scoping column added across the core tables if the app expands to multiple hostels,
- a proper vote-history/audit table if change-tracking (not just latest state) becomes a requirement,
- push notification support tied to menu_posts and holidays inserts,
- and richer reporting views once Phase 1 and Phase 2 are stable.
