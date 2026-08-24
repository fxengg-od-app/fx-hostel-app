# Architecture Guide

This document explains the software architecture for the FX Hostel canteen voting application. It is intended to guide implementation and future maintenance without introducing a different stack or a custom backend.

## 1. Application Architecture

The application is a React single-page application built with Vite and Material UI, hosted on Vercel. It uses Supabase for authentication, PostgreSQL storage, and Row Level Security. All business data is stored in PostgreSQL, and the browser interacts with Supabase using the public anonymous key.

The architecture is intentionally simple:

- the browser renders the UI,
- React routes define the feature entry points,
- feature-level API modules call Supabase,
- database triggers and RLS enforce vote-eligibility, holiday, and access-control safety.

No custom backend layer is required for this phase.

## 2. Architectural Principles

- Keep the database authoritative for "who can vote, when, and once."
- Keep feature logic close to the feature module.
- Use shared components only for truly reusable UI patterns (e.g. the session tab strip is used by student, canteen owner, and admin views alike).
- Keep the routing layer thin and explicit, with role-based redirects at the top level.
- Treat authentication, domain-restriction, and the student allowlist as first-class concerns — they are the primary access-control mechanism.

## 3. Folder Structure

```text
src/
  app/
    App.jsx
    providers/
    theme/
  lib/
    supabaseClient.js
    constants.js
    sessions.js          # session window definitions + "current session" logic
  routes/
    index.jsx
    guards/
  features/
    auth/
    students/            # super admin: allowlist management
    canteenOwners/        # super admin: canteen owner account management
    voting/               # student: cast/view votes for today & upcoming sessions
    menu/                 # canteen owner: post/edit today's & upcoming menus
    dashboard/            # canteen owner: live vote counts per session
    holidays/             # super admin + canteen owner: mark day/session holidays
    calendar/             # shared: the day/session calendar view
  components/
    layout/
    tables/
    forms/
    feedback/
  hooks/
    useAuth.js
    usePermissions.js
    useCurrentSession.js  # ticks over automatically as time passes
```

## 4. Feature Architecture

Each feature should follow the same internal structure:

```text
features/<feature>/
  api.js
  page.jsx
  components/
  hooks/ (only if needed)
  utils/ (only if needed)
```

This keeps data-access logic, presentation, and feature-specific helpers in one place.

## 5. Component Architecture

The UI should be composed from small, purpose-driven components.

### Layout hierarchy

```text
AppShell
  ├─ AppHeader
  ├─ SidebarNavigation (role-aware)
  └─ MainContent
       ├─ PageHeader
       ├─ SessionTabStrip (Breakfast | Lunch | Evening Snack | Dinner — current one highlighted)
       ├─ FeaturePage
       │    ├─ VotingCard / MenuPostForm / VoteDashboard (role-specific)
       │    ├─ HolidayBanner (shown when the selected session/day is a holiday)
       │    └─ DialogForm
       └─ Feedback / Empty / Error states
```

Shared UI should live in the top-level components folder. Feature-specific presentation should remain inside the feature module. `SessionTabStrip` and `HolidayBanner` are shared because all three roles view the same day/session structure.

## 6. Authentication Flow

1. The user signs in through Supabase Auth using Google OAuth restricted to the configured college email domain(s).
2. On first sign-in, the app checks the `student_allowlist` (or the seeded super admin / canteen owner tables) for that email. If the email is not present and active, access is denied with a clear "contact your hostel office" message — a valid college email alone is not sufficient.
3. The application loads the authenticated session from the browser client and resolves the user's role (STUDENT, CANTEEN_OWNER, SUPER_ADMIN) from application data (not just the JWT, since role changes should take effect without requiring a token refresh workaround).
4. Route guards enforce whether the user may view or edit a feature based on role.
5. Database RLS enforces the final authorization boundary, including the domain + allowlist check for any direct data access.

The frontend should never assume that UI visibility alone is sufficient protection. RLS remains mandatory, especially since "who is allowed to sign up at all" is a core product requirement here, not a nice-to-have.

## 7. Routing

Routes should be organized by role and feature.

Suggested routing structure:

- /login
- /today                     (student: current day, all four sessions)
- /vote/:date/:session        (student: vote detail for a specific session)
- /history                    (student: past votes)
- /menu                       (canteen owner: post/edit menu per session)
- /dashboard                  (canteen owner: live vote counts)
- /holidays                   (super admin + canteen owner: manage holidays)
- /students                   (super admin: allowlist management)
- /canteen-owners              (super admin: canteen owner account management)
- /settings                    (super admin: college domain config, etc.)

Protected routes should use route-level guards that verify the session and role before rendering the feature page. `/today` should be the default landing route for students; `/dashboard` for canteen owners.

## 8. API Layer

Feature API modules should own Supabase queries and mutations.

Each API module should provide a small, explicit surface:

- list operations (e.g. list today's sessions with vote state),
- retrieve by id,
- create/update operations (e.g. cast vote, post menu, set holiday),
- and any role-specific or business-specific logic (e.g. `getVoteCounts(date, session)` for the canteen owner dashboard should return pre-aggregated counts, not raw rows).

The API layer should not contain large UI concerns such as dialog state or form validation logic. It should focus on data access and the domain rules that are safe to express in the client (e.g. "don't even attempt to submit a vote for a past session" — the database is still the final check).

## 9. React Hooks

Reusable hooks should stay small and focused. Suggested hooks:

- useAuth for session and role state,
- usePermissions for role-based UI gates,
- useCurrentSession for "what session are we in right now, and does that need to change" — should re-derive on an interval or on focus, not just on mount,
- useAsyncList for loading and error states.

Do not introduce a global state library for this project.

## 10. Reusable Components

Reusable components should be generic enough to be used by multiple features. Examples:

- DataTable
- SessionTabStrip
- VoteToggle (Interested / Not Interested)
- StatusChip (e.g. Holiday, Menu Posted, No Menu Yet)
- HolidayBanner
- ConfirmDialog
- EmptyState
- ErrorAlert
- PageHeader
- FormField

Feature-specific UI should remain within the feature module.

## 11. State Management

The application should use a combination of:

- React local state for form and dialog state,
- context for shared user/session/role state and "current date + current session,"
- and server state from Supabase for votes, menus, and holidays.

No Redux or other external state library is required.

## 12. Supabase Interaction

Supabase access should be centralized in a shared client module:

```text
src/lib/supabaseClient.js
```

This module should expose the initialized client and any reusable helper functions. Feature modules call it rather than constructing their own clients.

## 13. Error Handling

The UI should distinguish between:

- network errors,
- authentication errors (bad domain, not on allowlist),
- authorization errors (wrong role for this action),
- validation errors (e.g. already voted, session is a holiday, session already ended),
- and business-rule violations.

Each feature should provide clear empty, loading, and error states. Voting and menu-posting should never silently fail — a failed vote must be visibly distinguishable from a successful one.

## 14. Loading Strategy

The initial experience should remain responsive. The application should render shell UI quickly and load data progressively.

Recommended approach:

- show skeletons or simple placeholders on initial data queries,
- display empty states when no menu has been posted yet ("Breakfast — menu not posted yet"),
- keep the day/session view lightweight — it is fetching at most four sessions' worth of data at a time,
- avoid loading a student's full vote history when only today's status is needed.

## 15. Caching

Caching should stay minimal. The current scale does not require a complex caching layer.

Recommended defaults:

- rely on Supabase query results for current data,
- use React state for transient UI state,
- re-fetch the canteen owner dashboard on a short interval (e.g. every 30–60 seconds) or via Supabase Realtime subscription if live updates matter more than polling simplicity — decide based on actual student headcount, don't over-engineer for a small hostel.

## 16. Pagination and Large Lists

For larger lists (e.g. super admin's student allowlist, or a canteen owner's historical menu log), the UI should use server-side filtering and pagination rather than loading all records at once.

The day/session voting and dashboard views are small by nature (four sessions, one hostel's worth of students) and do not need pagination in Phase 1.

## 17. Dialogs and Forms

Dialogs should be lightweight and focused on a single action, such as posting a session's menu, adding a student to the allowlist, or marking a session as a holiday.

Forms should use controlled inputs and simple validation rules. Avoid introducing a large form framework unless the UI later becomes much more complex.

## 18. Validation

Validation should be implemented in two layers:

- UI validation for usability (e.g. disable the vote button once already voted, disable it on a holiday session),
- database constraints and triggers for correctness (one vote per student per session, no votes on holidays, no votes on elapsed sessions).

The database is the final authority for vote eligibility and access control.

## 19. Data Flow Diagrams

### Cast vote flow

```text
Student -> SessionTabStrip / VoteToggle -> Voting API -> Supabase -> PostgreSQL
                                                        -> Constraint blocks duplicate/holiday/expired votes
                                                        -> UI reflects confirmed vote state
```

### Post menu flow

```text
Canteen Owner -> MenuPostForm -> Menu API -> Supabase -> PostgreSQL
                                            -> UI refreshes student-facing menu view
```

### Holiday flow

```text
Super Admin / Canteen Owner -> Holiday Form -> Holiday API -> Supabase -> PostgreSQL
                                                              -> Trigger/RLS blocks new votes for that session
                                                              -> UI shows HolidayBanner across all role views
```

### Dashboard aggregation flow

```text
Canteen Owner opens Dashboard -> Dashboard API (aggregate query) -> Supabase -> PostgreSQL
                                                                    -> Returns interested/not-interested counts per session
                                                                    -> UI renders counts, refreshed periodically
```

## 20. Module Dependency Diagram

```text
AppShell
  -> AuthProvider (role + allowlist resolution)
  -> Router
  -> Feature Pages
       -> Feature API modules
       -> Shared UI components (SessionTabStrip, VoteToggle, HolidayBanner)
       -> Shared hooks (useCurrentSession, usePermissions)
       -> Supabase client
```

## 21. Future Scalability

The current architecture is appropriate for a single hostel and a small relational data model. As the system grows (e.g. multiple hostels/canteens), the project can evolve by adding:

- a `hostel_id` / `canteen_id` scoping column across the core tables,
- richer reporting views over historical vote and menu data,
- push notifications when a menu is posted or a holiday is declared,
- and stronger separation between domain services and UI modules if the number of canteens grows beyond one.

That growth should be handled incrementally rather than by introducing a different architecture upfront.
