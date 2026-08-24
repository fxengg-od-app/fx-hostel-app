# FX Hostel Canteen Voting App — Development Constitution

This document is the implementation constitution for the repository. It defines the product intent, the technical guardrails, the delivery order, and the rules that future implementation work must follow.

## API Contract Rule

Before renaming, removing, or moving any exported function:

1. Search the entire project for every import of that function.
2. Update all dependent modules.
3. Run a production build.
4. Only consider the refactor complete if the build succeeds.

Never change a public API without updating all consumers.

## Human Terminal Rule

The AI agent must NEVER wait for long-running terminal processes.

Examples:

- npm install
- npm run dev
- npm run build
- npx ...
- supabase ...

Instead:

1. Print the exact command.
2. Ask the user to run it.
3. Continue after confirmation.

Never poll timers.
Never repeatedly wait.
Never enter waiting loops.

## UI/UX Rule

Think like you are designing software a hostel student checks on their phone between classes, and a canteen owner checks once each session while cooking.

Every pixel should justify its existence.

- The student view must answer "what's on today, and did I vote?" in one glance, per session.
- The canteen owner view must answer "how many people are eating, and what do I need to cook?" in one glance.
- Prefer information density and consistency over decorative styling.
- The current/active session (based on real time) should always be visually obvious against the other three.

## Database-First Rule

Whenever a feature requires adding, removing, or modifying database fields:

1. Generate the SQL migration first.
2. Stop and present the SQL migration.
3. Wait for the user to execute it in Supabase.
4. Continue only after confirmation.
5. Then update APIs.
6. Then update React components.

Never assume the live database matches the source code.

Every database change must have a corresponding migration file.

## Handover Rule

Every implementation must end with a HANDOVER.md style summary.

The summary should contain:

- Objective
- Decisions made
- Files modified
- Database changes
- SQL migrations executed/pending
- APIs changed
- Components added
- Remaining TODOs (priority order)
- Known risks
- Exact next task for the following coding agent

Assume another AI agent with no previous context will continue development. Write the handover so they can resume work immediately without re-auditing the repository.

## 1. Project Vision

FX Hostel is a daily food-voting and planning tool for a college hostel canteen. Hostel students vote per meal session on whether they will eat the canteen's food that session, or "not interested" (e.g. ordering online instead). The canteen owner sees live vote counts per session and posts what is being cooked and in what quantity. A super admin manages who is allowed to use the system (hostel students only) and can, along with the canteen owner, mark specific sessions or full days as holidays.

The system is intentionally scoped to a single hostel/canteen. It prioritizes daily operational clarity — "how many people are eating this session" — over historical analytics or multi-branch support in the first release.

## 2. Business Rules

- The application is a lightweight daily voting and menu-posting tool, not a food ordering, billing, or payment platform.
- The primary business objects are: users, sessions (per calendar date), votes, menu posts, and holidays.
- No payments, online food ordering, or delivery tracking are handled inside this app — "not interested" simply means the student is sourcing food elsewhere (e.g. ordering online), and the app does not need to know where from.
- A calendar day is divided into four fixed sessions: Breakfast, Lunch, Evening Snack, Dinner (see Section 4a for timings). Voting and menu posting are always scoped to a (date, session) pair.
- If the canteen owner has not posted a menu for a session, the student view still shows the session (labelled with its generic name, e.g. "Breakfast") with no food details — it does not disappear.
- Backups remain outside the application through database export tooling such as pg_dump.
- Vote and menu data for a session must remain consistent even if the frontend is unavailable or misused — the database is the source of truth for who voted what.
- A session that has been marked as a holiday cannot receive new votes. Existing votes for a session that is later marked a holiday should be preserved (not deleted) but the UI should clearly flag the session as cancelled.

## 3. Product Scope and Phase Order

Phase 1: authentication (domain-restricted + admin-allowlisted), roles, the daily session calendar, student voting, canteen owner menu posting, and the canteen owner's live dashboard.

Phase 2: holiday management (full-day and per-session), super admin student management (add/remove/deactivate), and audit visibility (who voted, when).

Phase 3: reporting views — historical vote trends per session/day, no-show patterns, and simple exportable summaries for the canteen owner.

The implementation order is fixed. Do not build reporting before voting, menu posting, and holiday handling are reliable.

## 4. Roles and Access

| Role | Description | Access |
|---|---|---|
| SUPER_ADMIN | Hostel warden / hostel office admin | Full CRUD across the application: manage the student allowlist, activate/deactivate students, manage canteen owner accounts, set holidays for any session or day, view all votes and menus |
| CANTEEN_OWNER | Canteen operator | Post/edit today's (and future) menu per session with item names and quantity, view the live vote dashboard per session, mark specific sessions as holidays (cannot deactivate students or manage the allowlist) |
| STUDENT | Hostel resident | Vote "interested" or "not interested" per session for the current day (and, where the UI allows, upcoming sessions/days), view the posted menu, view their own vote history |

Implementation assumption for the initial phase: students only see and vote on their own votes — there is no shared/team voting. Canteen owner and super admin see aggregate (anonymized-in-UI, but traceable-in-DB) counts.

## 4a. Session Definition (Fixed)

| Session | Typical window | Notes |
|---|---|---|
| Breakfast | 6:00 AM – 9:30 AM | |
| Lunch | 10:00 AM – 1:00 PM | |
| Evening Snack | 2:00 PM – 5:00 PM | |
| Dinner | 5:00 PM – 7:00 PM | |

These windows are for **display and "current session" highlighting only**. A student may vote on any future session at any time (e.g. vote for all four sessions of the day at 7 AM). The database must never reject a vote purely because the wall-clock time is outside that session's typical window — it should only reject votes for sessions that are already in the past (session end time has elapsed) or marked as a holiday. The exact cutoff behavior (e.g. can a student still vote for lunch at 12:55 PM, or does it lock earlier for the canteen owner's prep time) is a Phase 1 implementation detail to be confirmed with the user before building — do not assume a prep-buffer cutoff without confirming it.

## 5. Technical Stack

The stack is fixed and must not be replaced during implementation.

| Layer | Choice |
|---|---|
| Frontend | React with Vite |
| UI library | Material UI |
| Database | PostgreSQL |
| Backend/auth | Supabase Auth and Supabase Postgres |
| Database client | @supabase/supabase-js |
| Hosting | Vercel |

No Firebase, no custom backend service, and no service-role key in the browser.

## 6. Coding Standards

- Use functional React components and hooks only.
- Prefer small, focused components and feature-level modules.
- Keep business logic close to the relevant feature. Avoid sprawling shared helpers unless the logic is genuinely reusable (session-window calculation is one legitimate shared helper — see `lib/sessions.js`).
- Place Supabase queries in feature-level api.js modules rather than inline inside components.
- Keep comments rare and only use them for session-window/date-boundary logic, vote-eligibility logic, and RLS rules — these are the easiest parts of this app to get subtly wrong.
- Favor clarity over clever abstractions.
- Do not introduce state libraries or form libraries unless the implementation absolutely requires them.

## 7. React Standards

- Use React Router for navigation and route-level composition.
- Use Material UI primitives as the default UI building blocks.
- Prefer controlled inputs and simple validation over complex form frameworks.
- Maintain clear loading, empty, and error states for every list and detail view.
- Keep dialogs and drawers lightweight and focused on a single task (e.g. "post today's lunch menu", "mark dinner as holiday").
- Use context for application-wide state only when it is truly shared, such as authentication/role and "today's date + current session" (the latter should tick over automatically without a page refresh).

## 8. Supabase Standards

- Use a single shared Supabase client module for browser access.
- Keep data-access code in feature api modules.
- Treat Row Level Security as part of the product contract, not as an afterthought.
- Use role-based claims in the JWT whenever possible rather than repeatedly querying user metadata in RLS.
- A student's ability to sign in at all is a two-part gate: (1) their email domain matches the configured college domain(s), AND (2) their email exists and is active in the super-admin-managed allowlist table. Both checks must be enforced in the database (RLS / a pre-auth check), not just in the frontend.
- Do not expose service-role credentials in frontend code, environment bundles, or deployment configuration.

## 9. Database Standards

- Use PostgreSQL and keep the schema explicit and relational.
- Add created_at and updated_at columns to every business table.
- Prefer database-enforced integrity over frontend-only checks — especially "one vote per student per (date, session)" and "no votes on holiday sessions", which must be enforced with constraints/triggers, not just UI disabling.
- Use a database function or trigger to derive whether a given (date, session) is currently votable, rather than duplicating that logic in the frontend and trusting it.

## 10. Folder Conventions

The repository should follow a feature-first structure:

```text
src/
  app/                 # theme, providers, app shell
  lib/                 # shared clients and utilities (incl. sessions.js)
  routes/              # route declarations and guards
  features/            # domain modules
  components/          # reusable UI building blocks
  hooks/               # shared hooks
```

Each feature should contain:
- api.js for Supabase access
- page components for main routes
- feature components for local UI composition
- feature-specific helpers only when necessary

## 11. Naming Conventions

- Use camelCase for JavaScript and TypeScript variables and functions.
- Use PascalCase for React component names.
- Use snake_case for database objects and PostgreSQL columns.
- Use descriptive names for business concepts such as sessionVote, menuPost, holidayEntry, studentAllowlistEntry.
- Avoid abbreviations that hide intent. Use the exact session names: breakfast, lunch, evening_snack, dinner (not "snacks", "eve", etc.) consistently across DB, API, and UI.

## 12. Security Rules

- Never commit secrets or credentials.
- Use only VITE_SUPABASE_URL and VITE_SUPABASE_ANON_KEY in the browser environment.
- Keep Row Level Security enabled for all user-facing tables.
- Deny access by default whenever a table is not explicitly meant to be accessible to a role.
- Do not allow direct client-side editing of another student's vote.
- The super admin's initial identity/identities must be seeded through a controlled mechanism (see Section 4b) — never allow the first super admin to be self-assigned through a public sign-up flow.

## 4b. Super Admin Bootstrap Rule

Because there is no super admin yet on first deploy, at least one super admin email must be embedded/seeded outside of normal in-app role assignment:

- Store the initial super admin email(s) as a seed value — either a `.env` value (`VITE_SUPER_ADMIN_EMAILS` or a server-side equivalent used only in a migration/seed script, never trusted directly from the browser for authorization decisions) or a hardcoded row inserted via a SQL migration into the `users`/`super_admins` table.
- Once seeded, that account is a normal SUPER_ADMIN row in the database and all further role management (promoting/demoting, adding more super admins) happens through the in-app super admin UI, backed by RLS — not by editing env vars again.
- Document the seeded super admin email(s) in the migration file itself so a future agent can find it without guessing.

## 13. Git Workflow

- Create a short-lived branch for each implementation task.
- Keep commits atomic and focused on one concern at a time.
- Use descriptive commit messages that reflect user-visible changes or architectural decisions.
- Prefer small pull requests with clear validation notes.

## 14. Definition of Done

A feature is considered complete when:
- the database contract and RLS rules are defined,
- the UI and API work for the requested flow are implemented,
- validation and error handling are present,
- the feature works for at least one permitted and one prohibited role path,
- session-boundary and holiday-boundary edge cases have been considered (not just the happy path),
- the documentation remains consistent with the implementation,
- and the change does not break adjacent workflows.

## 15. Development Workflow

1. Confirm the business requirement and the relevant database contract.
2. Define or update the schema and RLS policy before implementing UI.
3. Implement feature-level API modules.
4. Build the UI flow with clear loading and error handling.
5. Verify the flow against the intended role and the expected denial path.
6. Update documentation when the implementation changes the architecture or product rules.

## 16. Feature Workflow

For each feature, complete the following sequence:
- define the user story,
- define the relevant data model and validation rules,
- implement the API layer,
- implement the UI and interaction flow,
- verify role restrictions and session/holiday safety rules,
- document any new assumptions.

## 17. Performance Rules

- Keep list queries efficient and avoid unnecessary joins.
- The canteen owner dashboard's vote-count query (interested/not-interested per session) should be a single efficient aggregate query, not a per-row fetch-and-count in the client.
- Avoid loading full student vote history into memory when only today's counts are needed.
- Maintain responsive UI states and avoid blocking the first render with heavy data work.

## 18. Accessibility

- All interactive controls must be keyboard accessible.
- Provide visible focus states and meaningful labels.
- Ensure form errors and validation messages are announced clearly.
- Use semantic markup and avoid relying on color alone to communicate vote state (interested/not-interested) or holiday state.

## 19. Error Handling

- Handle network, authentication, and permission errors explicitly.
- Surface actionable messages rather than raw technical failures.
- Distinguish between "you're not allowed" (not on allowlist / wrong domain), "this session is closed" (past cutoff or holiday), and genuine system errors.
- Avoid silently swallowing failures in vote submission or menu posting.

## 20. Validation Rules

- A student may cast at most one vote per (date, session).
- A vote cannot be cast for a session that is marked as a holiday.
- A vote cannot be cast for a session that has already ended (subject to the cutoff decision noted in Section 4a).
- Menu quantity, when provided, must be a positive value or a clear descriptive string (e.g. "2 sets") — do not force it into a strict numeric-only field without confirming with the user, since canteen quantities are often described informally.
- Do not trust the frontend as the sole source of truth for whether a session is currently votable.

## 21. Data Integrity & Safety Rules

- Votes must remain traceable to a specific student, date, and session — no anonymous voting.
- Do not hard-delete a student's vote history when deactivating their account; deactivate the account and stop future access instead.
- Holiday entries must record who created them (super admin or canteen owner) and whether they apply to the whole day or a specific session.
- Menu posts should be versioned by (date, session) — editing today's menu updates that day's post; it must not silently overwrite a previous day's historical menu.

## 22. AI Agent Rules

- Preserve the existing architecture unless a change is explicitly required.
- Do not introduce speculative patterns or alternate stacks.
- When changing documentation, keep the useful parts of the current guidance and remove contradictions.
- If a requirement is ambiguous — especially around session cutoff timing or vote-editing rules — document the chosen assumption, flag it clearly to the user, and keep it consistent until told otherwise.
- Prefer the simplest implementation path that satisfies the current phase.

## 23. Never-Do Rules

- Do not replace the chosen stack.
- Do not build a custom backend when Supabase already covers the required needs.
- Do not place secrets in the frontend bundle or source control.
- Do not allow a student to self-assign the SUPER_ADMIN or CANTEEN_OWNER role.
- Do not let the UI be the only enforcement of "one vote per session" or "no votes on a holiday" — enforce both in the database.
- Do not skip Row Level Security.
- Do not build Phase 3 reporting before voting, menu posting, and holiday handling are stable.
- Do not add payment, online ordering, or delivery features without explicit confirmation — "not interested" is a signal, not a transaction.
