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

## SQL Function Catalog

All analytics/statistics functions live in the `analytics` schema with public wrappers (SECURITY DEFINER) for PostgREST access. Non-analytics functions (triggers, RPC operations, auth helpers) live directly in the `public` schema with `rpc_` or `fn_` prefix.

### Core Foundation

| Function | Schema | Returns | Description |
|---|---|---|---|
| `clasificacion_base()` | analytics + public | TABLE | **Single source of truth** for M1/M2/M3 classification. Maps every (cliente, SKU) pair to M1 (botiquín-only), M2 (botiquín→ODV conversion), or M3 (ODV-only). Used by multiple downstream RPCs. |
| `get_filtros_disponibles()` | public | TABLE | Returns ALL marcas, médicos (activos), and padecimientos for filter dropdowns. Not corte-limited. |

### Corte Mensual (server-side filtered, 5 params: medicos, marcas, padecimientos, fecha_inicio, fecha_fin)

| Function | Schema | Returns | Dashboard Tab |
|---|---|---|---|
| `get_corte_actual_data(...)` | analytics + public | json | Corte → Actual. KPIs (ventas/creación/recolección with % change vs previous corte) + per-médico grid rows. |
| `get_corte_historico_data(...)` | analytics + public | json | Corte → Histórico. All-time KPIs (venta M1, creación, stock activo, recolección) + date-filtered per-visit rows. |
| `get_corte_logistica_data(...)` | analytics + public | TABLE | Corte → Logística. Detailed per-SKU movement rows with saga state, ODV links, recolección evidence. |

### Corte Mensual (legacy — no server-side filtering, loaded into Redux)

| Function | Schema | Returns | Notes |
|---|---|---|---|
| `get_corte_actual_rango()` | public | TABLE | Detects current corte date range (periods without >3 day gaps). Used internally by corte RPCs. |
| `get_corte_anterior_rango()` | public | TABLE | Previous corte range for % change calculations. |
| `get_corte_stats_generales_con_comparacion()` | public | TABLE | **Replaced by** `get_corte_actual_data()` for Corte tab. Still used by Redux `corteStatsGenerales` for header display. |
| `get_corte_stats_por_medico()` | public | TABLE | **Replaced by** `get_corte_actual_data()` medicos array. |
| `get_corte_stats_por_medico_con_comparacion()` | public | TABLE | Per-médico stats with vs-previous-corte comparison. Still loaded into Redux. |
| `get_corte_filtros_disponibles()` | public | TABLE | **Replaced by** `get_filtros_disponibles()`. Returns only corte-scoped options. |
| `get_corte_logistica_detalle()` | public | TABLE | **Replaced by** `get_corte_logistica_data()`. No filter params. |
| `get_corte_skus_valor_por_visita(...)` | public | TABLE | SKUs and value per visit. Optional id_cliente/marca filters. |
| `get_historico_skus_valor_por_visita(...)` | public | TABLE | Historical per-visit data with optional date/client filters. |
| `get_corte_anterior_stats()` | public | TABLE | Previous corte stats for comparison. |

### Conversiones & Adopción

| Function | Schema | Returns | Dashboard View |
|---|---|---|---|
| `get_conversion_metrics()` | public | TABLE | Conversiones. Total adoptions, conversions, value generated. |
| `get_conversion_details()` | public | TABLE | Conversiones. Per-client/SKU conversion details with days-to-convert. |
| `get_top_converting_skus(limit)` | analytics + public | TABLE | Conversiones. Top converting SKUs by ROI. |
| `get_top_conversion_mix()` | analytics + public | TABLE | Conversiones. Top vs non-top product conversion rates. |
| `get_sankey_conversion_flows()` | analytics + public | TABLE | Conversiones. M2/M3 flow data for Sankey diagram. |
| `get_adoption_metrics()` | analytics + public | TABLE | Adopción. Average days/periods to adoption, timeline. |
| `get_historico_conversiones_evolucion(...)` | public | TABLE | Conversiones. Time-series evolution with date range + grouping. |
| `get_crosssell_significancia()` | public | TABLE | Statistical significance test for cross-sell (chi-squared). |

### Market & Performance

| Function | Schema | Returns | Dashboard View |
|---|---|---|---|
| `get_brand_performance()` | analytics + public | TABLE | Mercado. Revenue and pieces by brand. |
| `get_padecimiento_performance()` | analytics + public | TABLE | Mercado. Revenue and pieces by condition. |
| `get_yoy_padecimiento()` | analytics + public | TABLE | Mercado. Year-over-year growth by condition. |
| `get_doctor_performance(limit)` | analytics + public | TABLE | Médicos. Top doctors by revenue, pieces, SKU diversity. |
| `get_ranking_medicos_completo()` | analytics + public | TABLE | Médicos. Full doctor ranking with rango and facturación. |
| `get_product_interest(limit)` | analytics + public | TABLE | Interés. Product movement breakdown (venta/creación/recolección/stock). |
| `get_opportunity_matrix()` | analytics + public | TABLE | Mercado. Padecimiento opportunity matrix. |

### Inventory & Financial

| Function | Schema | Returns | Dashboard View |
|---|---|---|---|
| `get_balance_metrics()` | public | TABLE | Dashboard main. Inventory balance (entradas vs salidas). |
| `get_cumulative_movements()` | analytics + public | TABLE | Tendencia. Cumulative movement values over time. |
| `get_impacto_botiquin_resumen()` | analytics + public | TABLE | Dashboard main. Botiquín impact summary (M1-M4 revenue breakdown). |
| `get_impacto_detalle(metrica)` | analytics + public | TABLE | Dashboard main. Drill-down detail by metric type (M1/M2/M3/M4). |
| `get_facturacion_composicion()` | public | TABLE | Conversiones. Per-client billing composition (baseline vs M1/M2/M3). |
| `get_recoleccion_activa()` | public | json | Dashboard main. Active collection metrics. |

### Data Sources (loaded into Redux)

| Function | Schema | Returns | Notes |
|---|---|---|---|
| `get_botiquin_data()` | public | TABLE | Raw botiquín movement data. **Warning:** Subject to PostgREST 1000-row limit — use `.limit(5000)`. |
| `get_recurring_data()` | public | TABLE | Raw recurring sales (ODV) data. **Warning:** Subject to PostgREST 1000-row limit — use `.limit(5000)`. |

### Non-Analytics Functions (DO NOT confuse with analytics)

These are operational functions for the mobile app and admin panel. They have different prefixes:

- **`rpc_*`** — Mobile app RPCs (visit lifecycle, saga operations, notifications, admin tools)
- **`fn_*`** — Trigger functions (inventory sync, notifications, state transitions)
- **`trigger_*`** — Trigger functions (movement generation, stats refresh)
- **`audit_*`** — Audit trail functions
- **Auth helpers:** `is_admin()`, `current_user_id()`, `can_access_cliente()`, `can_access_visita()`
- **Data maintenance:** `rebuild_movimientos_inventario()`, `update_rango_y_facturacion_actual()`, `refresh_all_materialized_views()`

## Flujo Operativo de la App Móvil

### Flujo de Datos: RPCs → Tablas

```
App Móvil (RPC)                    Tablas Afectadas
═══════════════                    ════════════════

rpc_submit_corte()          →  saga_transactions (BORRADOR: VENTA + RECOLECCION)
rpc_submit_levantamiento()  →  saga_transactions (BORRADOR: LEVANTAMIENTO_INICIAL)
rpc_submit_lev_post_corte() →  saga_transactions (BORRADOR: LEV_POST_CORTE)
                                   │
                                   ▼  (estado cambia a CONFIRMADO)
rpc_confirm_odv()           ─┐
rpc_set_manual_odv_id()     ─┤→ rpc_confirm_saga_pivot()
rpc_set_manual_botiquin_odv_id() ─┘     │
                                        ├→ saga_transactions.estado = 'CONFIRMADO'
                                        ├→ saga_zoho_links (zoho_id, tipo, items)
                                        ├→ movimientos_inventario (CREACION/VENTA/RECOLECCION)
                                        ├→ inventario_botiquin (upsert/delete)
                                        └→ visit_tasks (COMPLETADO)

rpc_complete_recoleccion()  →  recolecciones (ENTREGADA) + firma/evidencias
                            →  rpc_confirm_saga_pivot() [mismos efectos arriba]
```

### Secuencias de Visita

**LEVANTAMIENTO_INICIAL** (3 tasks):
1. `LEV` — Registrar productos entregados al médico
2. `ODV_BOTIQUIN` — Confirmar ODV de Zoho (pivot → CONFIRMADO → genera CREACION)
3. `INFORME` — Subir informe/evidencia de visita

**CORTE** (6 tasks):
1. `CORTE` — Registrar ventas del período actual
2. `VENTA_ODV` — Confirmar ODV de venta (pivot → CONFIRMADO → genera VENTA)
3. `RECOLECCION` — Registrar productos a recolectar
4. `LEV_POST_CORTE` — Nuevo levantamiento post-corte (reposición)
5. `ODV_BOTIQUIN` — Confirmar ODV de reposición (pivot → genera CREACION)
6. `INFORME` — Subir informe/evidencia de visita

### Saga Pattern

Cada saga_transaction sigue el patrón:
- **COMPENSABLE** — `rpc_submit_*()` crea la saga en estado BORRADOR (reversible)
- **PIVOT** — `rpc_confirm_saga_pivot()` marca como CONFIRMADO (punto de no retorno, genera movimientos)
- **RETRYABLE** — Vinculación con Zoho (`saga_zoho_links`) puede reintentarse sin duplicar movimientos

### Tablas Analíticas y Origen de Datos

| Tabla | Escrita por | Usada por analytics? |
|---|---|---|
| `movimientos_inventario` | RPCs via `rpc_confirm_saga_pivot` | SÍ — `clasificacion_base` (M1), todas las métricas |
| `inventario_botiquin` | RPCs via `rpc_confirm_saga_pivot` | SÍ — balance, stock actual |
| `ventas_odv` | CSV import (Zoho CRM) + backfill | SÍ — `clasificacion_base` (M2/M3), `get_facturacion_composicion` |
| `botiquin_odv` | CSV import (Zoho) + backfill | NO — solo display en Logística tab y auditoría |
| `saga_transactions` | RPCs `rpc_submit_*` + `rpc_confirm_saga_pivot` | Indirectamente (genera movimientos) |
| `saga_zoho_links` | `rpc_confirm_saga_pivot` | Display en Logística, auditoría |

### Anti-Duplicación en `rpc_confirm_saga_pivot`

- Si la saga ya está `CONFIRMADO` y se llama con un nuevo `p_zoho_id`: NO crea movimientos nuevos, solo actualiza el `id_saga_zoho_link` en los movimientos existentes
- `trigger_generate_movements_from_saga` verifica `EXISTS movimientos_inventario WHERE id_saga_transaction = NEW.id` antes de crear nuevos movimientos
- Esto permite re-vincular ODVs de Zoho sin generar duplicados

## Statistical Methodology & Analytics Rigor

When building analytics features, RPCs, or visualizations, follow these principles:

### Attribution vs Causality
- **Never claim causal attribution without a control group.** The dashboard uses correlational metrics (e.g., "products exposed in botiquín that later appeared in ODV"), which show temporal correlation, not causation.
- **Use precise language in UI text.** Say "% ingreso que fluye vía botiquín" (channel contribution), NOT "% de crecimiento causado por botiquín" (causal attribution).
- True causal attribution requires Difference-in-Differences (DiD), A/B testing, or propensity score matching — none of which are possible without a control group of doctors without botiquín.

### Channel Contribution Formula
The `get_facturacion_attribution()` RPC uses:
```
% ingreso vía botiquín = (ventas_directas_botiquin + odv_productos_expuestos) / (facturacion_odv + ventas_directas_botiquin) × 100
```
- **Ventas directas botiquín** = movimientos tipo VENTA × precio (100% certeza — revenue from botiquín)
- **ODV de productos expuestos** = ventas_odv de SKUs con CREACION previa en botiquín antes de la primera compra ODV (correlación temporal)
- **Facturación ODV** = `clientes.facturacion_actual` (promedio mensual calculado por `update_rango_y_facturacion_actual()`)

### Validation Checklist for New Metrics
1. Can you explain what the metric measures in one sentence without using "caused" or "attributed"?
2. If using temporal correlation (event A before event B), verify the product/SKU had NO prior purchases before the exposure event
3. Distinguish between "conversión real" (product never purchased before exposure) vs "aceleración" (already purchased, just more frequent)
4. Label UI elements with the metric type: contribution (%), correlation, or descriptive statistic

### PostgREST Schema Cache
After creating new SQL functions via `apply_migration`, PostgREST may not detect them immediately. Always run `NOTIFY pgrst, 'reload schema'` on the target database after applying migrations, or deploy via CI/CD (`staging` branch → DEV, `main` branch → PROD) which handles this automatically.

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
