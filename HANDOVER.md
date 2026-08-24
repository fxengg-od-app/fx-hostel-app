# Handover Summary — Project Bootstrap: Docs Transformed from Printing Press App to FX Hostel Canteen Voting App

## 1. Objective

This repository was handed off from a previous, unrelated project (a printing press business management ERP). AGENTS.md, ARCHITECTURE.md, and DATABASE.md still contained that old project's content verbatim. This pass rewrites all three (plus this HANDOVER.md) to reflect the actual product: a hostel canteen food-voting app.

No application code exists in the repository yet — this was a documentation/constitution transformation only, done before any implementation begins, so the next agent (and the friend this project is being handed to) starts from an accurate spec instead of an ERP spec.

## 2. Product Summary (for quick orientation)

- Students (hostel residents, official college email only) vote per meal session — Breakfast, Lunch, Evening Snack, Dinner — on whether they'll eat canteen food or are "not interested" (e.g. ordering online).
- Canteen owner posts what's being made and roughly how much per session (free-text, e.g. "Aloo Parotta – 2 sets with Kurma"); if nothing is posted, the session still shows with no food details.
- Super admin manages which students are allowed in (domain-restricted OAuth + an explicit allowlist — both required), and, along with the canteen owner, can mark a specific session or a whole day as a holiday, which blocks voting for that scope.
- Students can vote ahead for future sessions/the whole day in one sitting if they want — sessions are just independently votable, there's no separate "whole day" vote type in the data model.

## 3. Decisions Made (confirmed with the project owner)

1. **Stack**: kept identical to the previous project — React + Vite + Material UI + Supabase (Postgres + Auth) + Vercel. No stack change.
2. **Access control for students**: two-part gate — Google OAuth restricted to the college email domain(s) AND the email must be pre-added by a super admin to a `student_allowlist` table. Neither check alone is sufficient.
3. **Voting granularity**: per-session voting only (4 independent votes/day per student). There is no combined "whole day" vote record — a student who wants to vote for the whole day in the morning just submits all 4 session votes at once through the UI; the data model stays simple (one row per student/date/session).
4. **Super admin bootstrap**: at least one super admin email must be seeded outside normal in-app role assignment (env var used only in a migration/seed script, or a hardcoded row in a SQL migration) — see AGENTS.md Section 4b and DATABASE.md Section 19.

## 4. Operational & Timing Rules (Confirmed)

1. **Advance Voting**: Students can vote for any upcoming session of the current day (or future open days) at any time (e.g. voting for Breakfast, Lunch, Snack, and Dinner all at 7:00 AM).
2. **Current Session Visual Priority**: The active session window (Breakfast: 6:00–9:30 AM, Lunch: 10:00 AM–1:00 PM, Evening Snack: 2:00–5:00 PM, Dinner: 5:00–7:00 PM) is visually emphasized on the student and canteen owner dashboards as the primary active card/tab.
3. **Session Cutoff**: A session locks for new/changed votes once its window has passed (e.g., past 9:30 AM for Breakfast), or immediately if marked as a holiday.
4. **Initial Super Admin**: Seeded directly via the initial migration / environment config so the first admin can access the portal immediately.

## 5. Files Transformed & Verified

- `AGENTS.md` — Development constitution: strict rules (API contracts, human terminal rule, database-first migration workflow, UI/UX guidelines, 4-session fixed model, RBAC and never-do rules).
- `ARCHITECTURE.md` — Technical system architecture: React + Vite + Material UI + Supabase frontend/backend design, feature-first folder organization, route hierarchy, authentication flow (Google OAuth + allowlist verification), and data flow diagrams.
- `DATABASE.md` — Complete PostgreSQL database specification: full schema definitions (`users`, `student_allowlist`, `college_settings`, `sessions`, `calendar_days`, `holidays`, `menu_posts`, `votes`), foreign keys, unique constraints, trigger specifications, and Row Level Security (RLS) policies.
- `HANDOVER.md` — This transition document outlining architecture decisions, readiness state, and immediate next implementation steps.
- `README.md` — Project orientation guide explaining repository structure and quick-start instructions.

## 6. Database Changes & Migrations

- Current state: Ready for initial Phase 1 SQL migration (`001_initial_schema.sql`).
- Schema scope: Base tables (`college_settings`, `sessions`, `users`, `student_allowlist`), operational tables (`calendar_days`, `holidays`, `menu_posts`, `votes`), functions & triggers (`is_session_votable`, `get_session_vote_counts`, `ensure_calendar_day`), and RLS policies.

## 7. Status of Codebase

- Documentation files have been thoroughly rewritten and verified against all functional requirements.
- No legacy ERP/printing press references remain.
- Ready for immediate project scaffolding (`npm create vite@latest`) and Supabase schema deployment.

## 8. Exact Next Task for the Implementation Agent

1. **Scaffold the Frontend**: Initialize Vite React application with Material UI, React Router, and `@supabase/supabase-js` matching the folder structure in `ARCHITECTURE.md`.
2. **Generate & Present Phase 1 SQL Migration**: Draft the complete `001_initial_schema.sql` file per `DATABASE.md`, present it to the user to execute in Supabase SQL Editor, and wait for confirmation.
3. **Implement Core Auth & RBAC**: Configure Supabase client, allowlist verification gate, and Google OAuth domain filter.
4. **Implement Student & Canteen Features**: Build the daily calendar/session switcher, student voting toggle, canteen owner menu manager, and live vote aggregation dashboard.
