# Roster Management System

A role-based web application for managing employees, shifts, duty rosters,
shift assignments, and shift-swap requests — built for **Vercel** (hosting)
and **Neon Postgres** (database), at zero cost on their free tiers.

---

## 1. Tech stack (and why)

| Layer | Choice | Why |
|---|---|---|
| Framework | Next.js 15 (App Router, TypeScript) | Free Vercel hosting, server actions remove the need for a separate API layer |
| Database | Neon Postgres (serverless) | Generous free tier, scales to zero, native Vercel integration |
| ORM | Drizzle ORM + `@neondatabase/serverless` | Works over HTTP (no TCP/native binaries), which is required for Vercel's serverless/edge functions and avoids the cold-start / connection-pool issues of traditional ORMs on serverless |
| Auth | Auth.js (NextAuth v5), Credentials provider | Free, self-hosted, no third-party dependency or cost; JWT sessions need no session table |
| Styling | Tailwind CSS v4 | Fast to build and free |
| Password hashing | bcryptjs | Pure JS, no native build step (safe on Vercel's build image) |

No paid services are required anywhere in this stack.

> Note: an earlier draft used Prisma, but Prisma's query engine requires
> downloading a native binary at install/build time. Drizzle was used
> instead — it's pure TypeScript, has no native binary, and is the ORM Neon
> itself recommends for serverless/edge deployments.

---

## 2. Project structure

```
src/
  app/
    login/                     # Public login page
    (app)/                     # Authenticated route group (sidebar shell)
      dashboard/                 # Role-specific dashboard
      roster/                    # Roster calendar + filters + assign/edit
      swaps/                     # Shift-swap request + approval workflow
      notifications/             # In-app notifications
      admin/
        users/                    # Admin: user management
        shifts/                   # Admin: shift catalog
        lobs/                     # Admin: LOB/Team management
        audit-logs/               # Admin: full audit trail
    api/auth/[...nextauth]/    # Auth.js route handler
  auth.ts                     # Auth.js configuration (Credentials provider)
  proxy.ts                    # Route-protection middleware (Next.js 16 naming)
  db/
    schema.ts                  # Full Drizzle schema (single source of truth)
    index.ts                   # DB client (Neon HTTP driver)
    seed/seed.ts                # Sample data seeding script
  lib/
    permissions.ts              # Central RBAC rules — the authorization core
    session.ts                  # requireUser() session helper
    audit.ts                    # writeAudit() helper
    notify.ts                   # notify() helper
    actions/                    # All server actions (mutations + queries)
      users.ts, shifts.ts, lobs.ts, roster.ts, swaps.ts, reports.ts, notifications.ts
drizzle/                      # Generated SQL migration(s)
drizzle.config.ts             # Drizzle Kit config
```

**Design principle:** every mutation goes through a server action in
`src/lib/actions/`. Every one of those actions re-checks permissions with
`src/lib/permissions.ts` before touching the database — the UI is never
trusted as the only gate. Routes are also protected by `src/proxy.ts` as a
first-line, fast redirect, but that is a convenience layer only.

---

## 3. Data model (see `src/db/schema.ts`)

| Table | Purpose |
|---|---|
| `users` | Employee/Leader/Admin accounts. Self-referencing `reportingLeaderId` FK builds the Admin → Leader → Subordinate tree. `lobId` links to their team. |
| `lobs` | Configurable LOB/Team/Group definitions. |
| `shifts` | Shift *templates* (name, code, start/end time, color) — not individual assignments. |
| `roster_assignments` | One row per employee, per date, per shift. A **unique index on `(userId, date)`** enforces the business rule that an employee cannot have two conflicting shifts on the same day. |
| `roster_history` | Immutable audit trail of every roster change: who, what, old value, new value, when. |
| `shift_swap_requests` | The full swap lifecycle — status machine described below. |
| `notifications` | In-app notifications per user. |
| `audit_logs` | System-wide activity log (user/shift/roster/swap/admin-override actions). |
| `system_settings` | Key/value store for future admin-configurable settings. |

All roster and swap mutations run inside a **Drizzle transaction**
(`db.transaction(...)`), so a crash mid-update can never leave the roster in
a half-swapped or half-written state — satisfying the "transaction-safe"
requirement.

---

## 4. The shift-swap workflow (state machine)

```
PENDING_COUNTERPARTY  ->  PENDING_LEADER  ->  APPROVED  (roster updated atomically)
        |                       |
        v                       v
REJECTED_BY_COUNTERPARTY   REJECTED_BY_LEADER

Any pending state -> CANCELLED (by requester, or Administrator override)
```

1. **Employee A** picks their own scheduled shift, an eligible Employee B,
   and B's shift, then submits a request (`createSwapRequest`).
2. **Employee B** accepts or rejects (`respondToSwapAsCounterparty`). On
   accept, the request is routed to A's reporting Leader.
3. **The Leader** (or an Administrator, who can act on any request as an
   override) approves or rejects (`decideSwapAsLeader`).
4. On approval, the two `roster_assignments` rows swap `shiftId` values
   **atomically**, two `roster_history` rows are written, both employees
   are notified, and the action is written to `audit_logs` — all inside one
   transaction.

---

## 5. Roles & permission summary

| Action | Admin | Leader | User |
|---|---|---|---|
| Manage users, shift catalog, LOBs | Yes | No | No |
| Assign/edit/cancel shifts | Yes, any employee | Own subordinates only | No |
| View roster | Everyone | Own subordinates | Own LOB |
| Approve/reject swaps | Any request | Own subordinates' requests | No |
| Request a swap | Yes | Yes | Yes |
| View audit log | Yes | No | No |

These rules live in one place — `src/lib/permissions.ts` — and are called
from every server action, so there is no route or mutation that skips the
check.

---

## 6. Local setup

### Prerequisites
- Node.js 20+
- A free [Neon](https://neon.tech) account and project
- A free [Vercel](https://vercel.com) account (for deployment)

### Steps

```bash
npm install

# 1. Copy the env template and fill in your Neon connection string
cp .env.local.example .env.local
```

Edit `.env.local`:

```env
DATABASE_URL="postgresql://<user>:<password>@<host>/<db>?sslmode=require"
AUTH_SECRET="<generate with: npx auth secret>"
NEXTAUTH_URL="http://localhost:3000"
```

Get `DATABASE_URL` from your Neon project dashboard -> **Connect** -> copy
the "pooled connection" string (works best for serverless).

```bash
# 2. Push the schema to your Neon database
npm run db:push

# 3. Seed sample data (Admin, 2 Leaders, 4 Users, shifts, 4 days of roster)
npm run db:seed

# 4. Run the dev server
npm run dev
```

Visit `http://localhost:3000` and sign in with any seeded account
(password `Password123!`):

| Role | Email |
|---|---|
| Admin | `admin@roster.local` |
| Leader (Operations) | `alice.leader@roster.local` |
| Leader (Support) | `bob.leader@roster.local` |
| User | `charlie@roster.local`, `dana@roster.local`, `evan@roster.local`, `fiona@roster.local` |

---

## 7. Deploying to Vercel + Neon (both free)

1. **Push this project to a GitHub repository.**
2. **Create a Neon project** at [neon.tech](https://neon.tech) (free tier).
   Copy its pooled connection string.
3. **Import the repo into Vercel** ([vercel.com/new](https://vercel.com/new)).
4. In Vercel's project settings -> **Environment Variables**, add:
   - `DATABASE_URL` — your Neon connection string
   - `AUTH_SECRET` — output of `npx auth secret`
   - `NEXTAUTH_URL` — your production URL, e.g. `https://your-app.vercel.app`

   Optional but recommended: install the official Neon-Vercel integration
   from the Vercel Marketplace — it wires `DATABASE_URL` automatically and
   creates preview-branch databases per PR.
5. **Deploy.** Vercel will run `npm run build` automatically.
6. **Run the schema push and seed once**, pointed at the production
   database (from your local machine, with `DATABASE_URL` in `.env.local`
   set to the *production* Neon string):
   ```bash
   npm run db:push
   npm run db:seed   # optional -- skip in a real deployment and create real users via the Admin UI instead
   ```
7. Visit your Vercel URL and sign in.

### Database schema changes going forward
This project uses **Drizzle Kit** for migrations:
- `npm run db:generate` — generate a new SQL migration from schema changes
- `npm run db:migrate` — apply generated migrations
- `npm run db:push` — push schema directly (fastest for early development)
- `npm run db:studio` — open Drizzle Studio, a free GUI for browsing your Neon data

---

## 8. Business rules enforced by the system

- A roster assignment is unique per `(employee, date)` — the database
  itself rejects a second shift for the same person on the same day.
- A shift swap cannot complete without **both** the counterparty's
  acceptance **and** the leader's approval, in that order.
- Leaders can only manage/approve for their own direct subordinates;
  Administrators can act on anyone and every such override is logged as
  `ADMIN_OVERRIDE` / `is_admin_override = true` in the swap record.
- Deleting a user or shift performs a **soft delete** (status/`isActive`
  flag) rather than a hard delete, so roster history and audit logs always
  remain traceable to a real record.
- Every create/update/delete across users, shifts, LOBs, roster
  assignments, and swaps writes a row to `audit_logs` with the actor,
  action, previous value, new value, and timestamp.

---

## 9. What's included vs. natural next steps

**Included:** authentication, full RBAC, employee/shift/LOB management,
roster calendar with filters (date range, shift, employee search) and
role-scoped visibility, duty-count reporting, the complete 4-step shift-swap
workflow, in-app notifications, and a full audit log viewer.

**Reasonable next additions** (not built, to keep this a focused v1):
- Email notifications (the `notify()` helper already exists — sending an
  email alongside each in-app notification is a small addition using e.g.
  Resend's free tier)
- CSV/Excel export of roster and duty-count reports
- A calendar/grid view of the roster (current implementation is a filterable
  table, which covers the same data but not the visual calendar layout)
- Multi-level leader hierarchies (currently one reporting leader per user,
  which matches the spec's Admin -> Leader -> Subordinate structure)
