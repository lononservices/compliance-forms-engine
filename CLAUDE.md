# CLAUDE.md — Compliance Forms Engine (CFE)

## Project Identity

**Product:** Compliance Forms Engine (CFE) v1.0 MVP
**Organization:** The Church Tech Club (TCTC)
**Tagline:** Comply with Ease
**Domain:** cfe.thechurchtechclub.com
**Type:** Multi-tenant SaaS web application

---

## What This Project Does

CFE centralizes compliance form lifecycle management for denominational headquarters and affiliated local churches. Denominations create and push forms; local churches complete and submit them. The system tracks compliance status, sends notifications, and provides reporting for both entity types.

---

## Project Root

You are operating inside the `cfe/` Next.js project root:
```
C:\Users\jblon\OneDrive\Documents\Jonathan\CCProj\CFE\CFE-Web-App\cfe\
```

The `Docs/` folder at the parent level contains reference documents. You may read from it but never write to it. All code work happens inside `cfe/`.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Framework | Next.js 16.x, App Router |
| Language | TypeScript (strict mode) |
| Styling | Tailwind CSS 4.x (utility classes only — no inline styles, no CSS modules) |
| Database | Supabase (PostgreSQL) with Row Level Security |
| Auth | Supabase Auth via `@supabase/ssr` |
| Email | Resend (transactional only) |
| Payments | Stripe |
| Icons | lucide-react |
| Validation | Zod v3.x + react-hook-form v7.x |
| AI | Claude Sonnet via `@anthropic-ai/sdk` (Phase 6 only) |

---

## Architecture Rules

### File Creation
- All new source files go under `src/`
- Pages go in `src/app/` following Next.js App Router conventions
- Reusable components go in `src/components/`
- Server utilities go in `src/lib/`
- TypeScript types go in `src/types/`
- Never put application code in the project root

### Routing Conventions
- Authenticated pages: `src/app/(dashboard)/` route group
- Auth pages: `src/app/(auth)/` route group
- API routes: `src/app/api/`
- Public page: `src/app/page.tsx` (landing page)

### Supabase Client Rules
**Critical:** Use exactly one of these three clients for every Supabase operation. Never instantiate a client directly.

- `src/lib/supabase/client.ts` — browser-side, use in Client Components
- `src/lib/supabase/server.ts` — server-side, use in Server Components, API routes, Server Actions
- `src/lib/supabase/middleware.ts` — session refresh only, use in `src/middleware.ts`

`SUPABASE_SERVICE_ROLE_KEY` may only be used in API routes and Edge Functions. It must never appear in any client-side code or component.

### Email
Resend is the only email provider for transactional messages. All email-sending code goes through `src/lib/resend.ts`. GHL is for marketing/CRM only and is never used for compliance or system notifications.

### API Route Security
Every API route must validate the user session at the top of the handler before performing any action. Exception: `/api/stripe/webhook` (validates Stripe signature) and `/api/cron/*` (validates `CRON_SECRET` header).

---

## Design System Rules

### Colors (use CSS variables or Tailwind tokens only)
```
Primary:    #12294A  (navy)      — navigation, headers, key CTAs
Secondary:  #1A7F6E  (teal)      — action buttons, active states
Accent:     #E8A825  (gold)      — highlights, warnings, gamification
Background: #F4F6FA             — page backgrounds
Surface:    #FFFFFF             — cards, panels, inputs
```

### Border Radius
**Exactly two values — no others:**
- `rounded-lg` (8px) — buttons, inputs, badges, small elements
- `rounded-xl` (12px) — modals, large cards, sidebar

### Typography
- Headings: Inter (weight 600 or 700)
- Body: Roboto (weight 400 or 500)
- Code: Roboto Mono

### What Not to Do
- No purple gradients or rainbow gradients of any kind
- No glowing hover effects
- No sparkle or emoji in headings
- No fake testimonials or placeholder names ("Sara Chen", etc.)
- No oversized hero text combined with ultra-thin body text
- No non-functional social icons
- No stock photo faces
- Button hover: 2px lift maximum (`-translate-y-0.5`), no glow
- Never use `outline: none` without a visible replacement focus ring (2px secondary color)

### Loading States
Every async action must have a visible loading state:
- Page data: skeleton screens (pulsing gray rectangles matching content shape)
- Buttons: spinner inside button, width locked, `cursor-not-allowed`
- Never leave an action with no feedback

---

## Code Quality Rules

### TypeScript
- Strict mode is enabled. Do not use `any`.
- Use the generated types from `src/types/database.ts` for all Supabase queries
- Regenerate types after every migration: `npx supabase gen types typescript --local > src/types/database.ts`

### Error Handling
- All API routes: wrap in try/catch, return structured JSON errors with appropriate HTTP status codes
- All Supabase calls: check for error before using data
- All email sends: log failures to `notification_log` with `status: 'failed'`

### Forms
- All forms use react-hook-form + Zod resolver
- Never use bare `<form>` submit without react-hook-form
- Validate on blur and on submit
- Show field-level error messages below the field (not in a toast)

### Components
- Client Components: add `'use client'` at the top when needed
- Prefer Server Components by default; only switch to Client for interactivity
- No component should fetch data and render UI in the same file if that coupling can be avoided

---

## Database Rules

- RLS is enabled on all tables. Never disable it.
- Test cross-tenant isolation after every new RLS policy
- All migrations go in `supabase/migrations/` with sequential numbering
- Never modify a migration file after it has been applied — create a new migration instead
- Run `npx supabase db push` after every new migration during development

### Required GRANT Pattern (Supabase Data API — effective May 30, 2026)

Every `CREATE TABLE` in a migration **must** be followed immediately by explicit GRANTs. Without them, `supabase-js` (PostgREST) cannot access the table and returns a `42501` error.

```sql
-- After every CREATE TABLE in public schema:
grant select on public.table_name to anon;
grant select, insert, update, delete on public.table_name to authenticated;
grant select, insert, update, delete on public.table_name to service_role;
alter table public.table_name enable row level security;
-- ...RLS policies follow
```

- `anon` — unauthenticated requests (grant select only where public access is intentional; omit if table should never be public)
- `authenticated` — all logged-in users (RLS policies then restrict to authorized rows)
- `service_role` — server-side API routes using `SUPABASE_SERVICE_ROLE_KEY`

This pattern must appear in every migration file that creates a table. No exceptions.

---

## Multi-Tenancy

All tenant-scoped data includes `organization_id`. Always scope queries to the current user's active organization. The entity switcher in the sidebar controls the active organization context for multi-entity users.

---

## Feature Flags

These features are **off by default** and must only activate when explicitly enabled:

| Feature | Guard | Default |
|---|---|---|
| AI Chatbot | `organizations.settings.ai_chatbot_enabled` | `false` |
| Gamification | `organizations.settings.gamification_enabled` | `false` |
| SMS | `notifications.sms_enabled` per user | `false` |

Phase 6 features (AI chatbot, SMS, gamification, calendar, TTS, STT) are not built until Phase 6. Do not create stubs or placeholders that could be accidentally enabled.

---

## Reference Documents

These are in the `Docs/` folder at the parent directory level:

| File | Use for |
|---|---|
| `CFE-BUILD-PROMPT-v2.2.md` | Ordered build steps — follow this sequentially |
| `CFE-BACKEND-v2.2.md` | Complete database schema and API patterns |
| `CFE-PLAN-v2.2.md` | Design system, component specs, page layouts |
| `CFE-APP-FLOW-v2.2.md` | User journey and route flow |
| `CFE-RESOURCES-v2.2.md` | Package versions and folder structure |
| `CFE-RISK-ANALYSIS-v2.2.md` | Known risks and their resolutions |

When implementing any feature, consult the relevant document before writing code.

---

## Current Build Status

**Prerequisites (Steps 0.0–0.7):** Complete
**Phase 1:** Ready to begin
**Phase 2–6:** Not yet started

Work through CFE-BUILD-PROMPT-v2.2.md sequentially. Do not skip steps. Do not start the next step until the current step's test criteria pass.

---

## Asking for Clarification

If a build step is ambiguous or a decision point has multiple valid approaches, ask before proceeding. State the two options and your recommendation. Do not guess and proceed silently.
