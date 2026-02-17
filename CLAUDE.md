# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Structure

This is a monorepo using git submodules for a pharmaceutical sales management platform (DermiTrack). Three submodules, each with its own repo and CLAUDE.md:

| Submodule | Purpose | Tech |
|---|---|---|
| `dermitrack` | Mobile app for sales reps | React Native (Expo 54), TypeScript, Redux Toolkit, NativeWind |
| `dermitrack-analytics` | Management analytics dashboard | Next.js 16, React 19, TypeScript, Tailwind CSS 4, Redux Toolkit |
| `dermitrack-supabase` | Backend infrastructure | PostgreSQL 17, Supabase migrations (52+), Deno Edge Functions (11) |

All three share a single Supabase backend. **Always read the submodule's own CLAUDE.md** before working within it.

## Supabase Environments

| Environment | Project Ref | Usage |
|---|---|---|
| **DEV** | `ysxmbijpskjpwuvuklag` | All development, testing, migrations |
| **PROD** | `lrjlucjraeuihfzzgdvm` | Production only — never modify without explicit user authorization |

**Critical rules:**
- Default to DEV for all operations
- Never run DDL (CREATE, ALTER, DROP) via `apply_migration` or `execute_sql` — create migration files in `dermitrack-supabase/supabase/migrations/` instead
- `execute_sql` is only for SELECT queries and authorized DML
- Push to `main` in `dermitrack-supabase` triggers CI/CD that deploys migrations and Edge Functions to PROD

## Commands

### Mobile App (`dermitrack/`)
```bash
npm start              # Expo dev server
npm run ios            # iOS simulator
npm run android        # Android emulator
npm run web            # Web version
npm run lint           # ESLint
```

### Analytics Dashboard (`dermitrack-analytics/`)
```bash
npm run dev            # Dev server (http://localhost:3000)
npm run build          # Production build
npm run lint           # ESLint
npm run lint -- --fix  # Auto-fix lint issues
```

### Supabase Backend (`dermitrack-supabase/`)
```bash
supabase start         # Local dev environment (API: 54321, DB: 54322)
supabase migration list
supabase migration new <name>
supabase db push       # Apply migrations
supabase functions serve <name>  # Test Edge Function locally
```

## Architecture Overview

### Data Flow
Mobile app and analytics dashboard both connect to Supabase via client SDKs. The mobile app uses `@supabase/supabase-js`, the dashboard uses `@supabase/ssr` with separate browser/server clients. Auth is handled by Supabase Auth with RLS policies enforcing access control. Edge Functions handle admin operations (user management, Zoho CRM integration, push notifications).

### Mobile App (`dermitrack/`)
- **Routing:** Expo Router (file-based in `app/`)
- **State:** Redux Toolkit with Redux Persist + MMKV for offline support
- **Services:** `src/services/` — data access layer per domain (clientes, visitas, cortes, recolecciones)
- **Offline-first:** Drafts stored locally via `src/utils/offlineDrafts.ts`, synced on reconnection
- **Saga pattern:** Visits tracked through state transitions with compensation support
- **Auth gate:** `app/_layout.tsx` listens to Supabase session changes and routes accordingly

### Analytics Dashboard (`dermitrack-analytics/`)
- **Clean Architecture:** `core/types/` (domain interfaces) → `infrastructure/adapters/` (Supabase implementations) → `infrastructure/hooks/` (React wrappers)
- **Route groups:** `(auth)/` for login, `(protected)/` for dashboard/admin (requires OWNER or ADMINISTRADOR role), `(public)/` for landing
- **Middleware:** `middleware.ts` validates sessions and roles before allowing access to protected routes
- **CSV analytics:** Two CSV types — "Botiquín" (initial placements) and "Ventas Recurrentes" (recurring sales). Parsing handles DD/MM/YYYY and DD-mmm-YY date formats
- **Path alias:** `@/*` → `./src/*`
- **Theme:** Emerald primary (#10b981), centralized in `src/lib/constants/theme.ts`

### Backend (`dermitrack-supabase/`)
- **Migrations:** Versioned SQL files in `supabase/migrations/` — **single source of truth for all schema changes**
- **Edge Functions:** Deno/TypeScript in `supabase/functions/` — includes admin user management, push notifications, Zoho CRM integration, CSV import
- **Key tables:** usuarios, clientes, zonas, medicamentos, visitas, recolecciones, cortes, padecimientos

## Key Domain Concepts

- **Visitas** — Sales rep visits to doctors, tracked via saga pattern with state transitions
- **Cortes** — Monthly cut-off periods for reporting
- **Recolecciones** — Product collection/return tracking
- **Levantamiento** — Initial product placement orders
- **Zonas** — Geographic sales territories

## Working Across Submodules

```bash
# Clone with all submodules
git clone --recurse-submodules https://github.com/Aleksei115/dermitrack-project.git

# Each submodule is an independent git repo — commit/push within each
cd dermitrack && git add . && git commit -m "message"

# Update root repo's submodule references
cd .. && git add dermitrack && git commit -m "Update dermitrack submodule"
```

Schema changes always go through `dermitrack-supabase` migration files, never through direct SQL execution.

## CI/CD Workflows

Branch strategy: `main` (PROD) / `staging` (DEV).

| Repo | Workflow | Trigger | What it does |
|---|---|---|---|
| `dermitrack-supabase` | `production.yaml` | push to `main` | Runs migrations + deploys Edge Functions to PROD |
| `dermitrack-supabase` | `staging.yaml` | push to `staging` | Runs migrations + deploys Edge Functions to DEV |
| `dermitrack` | `production.yaml` | push to `main` | Lint → EAS Update (OTA). Native build with `[build]` in commit or `workflow_dispatch` |
| `dermitrack-analytics` | `production.yaml` | push to `main`/`staging`, PRs to `main` | Lint (CI only — Vercel handles deploys via Git Integration) |

### GitHub Secrets Required

| Repo | Environment | Secrets |
|---|---|---|
| `dermitrack-supabase` | `production` | `SUPABASE_ACCESS_TOKEN`, `PRODUCTION_DB_PASSWORD`, `PRODUCTION_PROJECT_ID` |
| `dermitrack-supabase` | `staging` | `SUPABASE_ACCESS_TOKEN`, `DEV_DB_PASSWORD`, `DEV_PROJECT_ID` |
| `dermitrack` | `production` | `EXPO_TOKEN` |
| `dermitrack-analytics` | — | No secrets needed (Vercel Git Integration handles deploys) |

### Migration Centralization

All Supabase migrations live exclusively in `dermitrack-supabase/supabase/migrations/`. The `dermitrack` and `dermitrack-analytics` repos do **not** contain a `supabase/` directory.
