# FX Hostel App (fx-hostel-app)

A daily food-voting and planning application tailored for college hostel students and canteen operators.

## 🎯 Overview

- **Hostel Students**: Vote availability ("Interested" in canteen food or "Not Interested" e.g., buying outside) for each meal session (Breakfast, Lunch, Evening Snack, Dinner). Can vote in advance for upcoming sessions or the whole day.
- **Canteen Owners**: Post today's/upcoming session menus (e.g. *"Aloo Parotta – 2 sets with Kurma"*) and view live vote aggregations on their dashboard to plan food preparation quantities accurately.
- **Super Admins**: Manage student allowlists, college domain restrictions, canteen owner accounts, and declare full-day or session-specific holidays.

---

## 🕒 Meal Sessions & Highlighting

| Session | Typical Window | Active Highlighting | Advance Voting |
|---|---|---|---|
| **Breakfast** | 6:00 AM – 9:30 AM | Highlighted during window | Permitted anytime before session ends |
| **Lunch** | 10:00 AM – 1:00 PM | Highlighted during window | Permitted anytime before session ends |
| **Evening Snack** | 2:00 PM – 5:00 PM | Highlighted during window | Permitted anytime before session ends |
| **Dinner** | 5:00 PM – 7:00 PM | Highlighted during window | Permitted anytime before session ends |

---

## 🛠️ Technology Stack

- **Frontend**: React (Vite) + Material UI (MUI) + React Router
- **Backend / Database**: Supabase PostgreSQL + Supabase Auth (Google OAuth)
- **Security**: Row Level Security (RLS) + database triggers for vote eligibility & holiday validation
- **Hosting**: Vercel

---

## 📚 Documentation & Constitution

Before contributing or modifying this repository, review the core documentation:

- [AGENTS.md](file:///d:/Git/fx-hostel-app/AGENTS.md) — The development constitution, coding guardrails, Human Terminal Rule, and RBAC rules.
- [ARCHITECTURE.md](file:///d:/Git/fx-hostel-app/ARCHITECTURE.md) — Application layout, feature organization, routing, and data flow.
- [DATABASE.md](file:///d:/Git/fx-hostel-app/DATABASE.md) — Full PostgreSQL schema, foreign keys, triggers, functions, and RLS policies.
- [HANDOVER.md](file:///d:/Git/fx-hostel-app/HANDOVER.md) — Current state of the repository and step-by-step next tasks for the coding agent.
