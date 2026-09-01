# Build Plan — Akura

The specs define **what** to build. This defines **the order**, and how you know each
phase is actually finished.

Repos: `akura-api` (NestJS) and `akura-app` (Next.js). Separate repos, separate
deploys, HTTP between them.

## How this is sliced

Each phase covers both repos for one thin slice of functionality and ends in something
you can run and see.

Building vertically — one feature through the whole stack — rather than horizontally
(all backend, then all frontend) means integration problems surface in Phase 1 when
almost nothing exists, instead of in week three when everything is tangled. CORS,
cookie flags, and route mismatches are the usual culprits: cheap to fix early,
miserable later.

Do not start a phase until the previous phase's exit check passes.

---

## Phase 0 — Foundations

**Goal:** two apps that start, talk to each other, and log everything. No features.
This phase feels like nothing is happening and is the highest-leverage phase in the
plan.

### akura-api
1. `nest new`, install dependencies (spec §2).
2. `src/shared/config/` — zod env schema validated at boot. The app must refuse to
   start without `JWT_SECRET`, `DATABASE_URL`, `FRONTEND_ORIGIN`.
3. `npx prisma init`. Write the **whole schema now** — all four tables plus the
   indexes (spec §4) — and run `npx prisma migrate dev --name init`. One migration
   beats four.
4. `PrismaService` with connect/disconnect, registered `@Global()`.
5. `src/shared/logging/` — `FileLoggerService`, three winston transports, the
   timestamp formatter, the recursive redaction helper (spec §11).
6. `src/shared/correlation/` — `nestjs-cls` middleware generating a request UUID.
7. Global `HttpLoggingInterceptor` and `AllExceptionsFilter`.
8. `main.ts` — helmet, cookieParser, CORS with credentials, validation pipe with
   `forbidNonWhitelisted`, throttler, `enableShutdownHooks()`.
9. A throwaway `GET /health` returning `{ ok: true }`.

### akura-app
10. `create-next-app --typescript --app --tailwind`.
11. Rename `next.config.js` → `next.config.ts`. Confirm no `tailwind.config.ts` exists
    (correct for v4) and `globals.css` uses `@import "tailwindcss";`.
12. `tsconfig` strict flags and the ESLint `no-explicit-any` rule with test overrides.
13. `lib/types.ts` — the full type set from spec §4.
14. `lib/api.ts` — generic `request<T>()` with `credentials: 'include'`.
15. A temporary button on `/` calling `/health`.

### Exit check
- [ ] The button shows `{ ok: true }` — CORS is correct.
- [ ] `logs/http-1.txt` has one NDJSON line, stamped `+0800`.
- [ ] `logs/app-1.txt` has a matching readable line.
- [ ] `logs/system-1.txt` shows bootstrap, config validation, Prisma connect.
- [ ] Ctrl+C → shutdown line appears in `system-1.txt`.
- [ ] A route that throws returns a **generic** message; the stack trace is in the log
      only.
- [ ] Unsetting `JWT_SECRET` → the app refuses to start.
- [ ] `npm run build` passes in both repos.

> If timestamps or config validation are wrong, fix them here. Every later phase
> depends on being able to read what happened.

---

## Phase 1 — Auth end to end

**Goal:** register, log in, stay logged in, log out. The hardest plumbing, done while
there is nothing else to break.

### akura-api — `identity` context
1. Domain: `User` entity, `Email` value object, `UserRepository` interface.
2. Infrastructure: `PrismaUserRepository`, bcrypt cost 12.
3. Application: `RegisterUser`, `LoginUser` commands; `GetUserByEmail` query.
4. Interface: `/auth/register`, `/auth/login`, `/auth/logout`, `/auth/me`.
5. Cookie set/clear with matching options, `passport-jwt` reading
   `req.cookies.access_token`, `JwtAuthGuard`.
6. Tighter throttle on `/auth/login` and `/auth/register`.

### akura-app
7. `app/login/page.tsx` — mode toggle, role select on register.
8. `AuthContext` — `me()` on mount, exposing `user`, `loading`, `refresh()`, `logout()`.
9. `app/dashboard/page.tsx` placeholder showing name, role, logout button.
10. `middleware.ts` redirecting `/dashboard` when the cookie is absent.

### Exit check
- [ ] Register as contractor → dashboard shows name and role.
- [ ] Login response body has **no token**.
- [ ] Cookie is present with **HttpOnly**; `document.cookie` does not show it.
- [ ] Hard refresh → still logged in.
- [ ] Logout → `/auth/me` 401s and `/dashboard` redirects.
- [ ] Duplicate email → clear 409 in the UI.
- [ ] Wrong password and unknown email give **identical** messages.
- [ ] Six rapid failed logins → **429**.
- [ ] **`grep -ri "password" logs/` shows no plaintext password.**
- [ ] **`grep -ri "access_token=" logs/` returns nothing.**

> Run those two greps now. If redaction is wrong, every later phase writes more
> credentials to disk.

---

## Phase 2 — Projects

**Goal:** a contractor creates a project naming a client; both parties see it.

### akura-api — `projects` context
1. Domain: `Project` entity with `hasAccess()`, repository interface.
2. Infrastructure: `PrismaProjectRepository` plus the `project_participants` read
   model.
3. Application: `CreateProject`; `ListProjectsForUser` (paginated) and
   `GetProjectDetail` (403 for non-participants).
4. Interface: `ProjectsController`, guarded.

### akura-app
5. Real `/dashboard` — project list, empty state, create form (contractor only).
6. `app/project/[id]/page.tsx` stub showing the project name.

### Exit check
- [ ] Register a second account as **client** in incognito.
- [ ] Contractor creates a project with the client's email → appears in both lists.
- [ ] Unregistered email → clear 404 in the UI.
- [ ] A third unrelated user opening the project URL → **403**.
- [ ] The client does not see the create form.
- [ ] `GET /projects` returns a paginated shape.
- [ ] Every request appears in `http-1.txt` with a correlation id.

> Test the 403 by pasting the URL into a third account's browser. Authorization lives
> in the query handler — a hidden link is not protection.

---

## Phase 3 — Stages

**Goal:** the contractor's checklist.

### akura-api — `progress` context
1. Domain: `Stage` entity, `updateProgress()` raising `StageProgressLogged`.
2. Infrastructure: `PrismaStageRepository`, read port for `project_participants`.
3. Application: `CreateStage`, `LogStageProgress` (contractor only), plus an event
   handler writing a business line to `app-1.txt`.
4. Interface: routes under `/projects/:projectId/stages`.

**On the PATCH:** authorize against the stage's own `projectId`, never the URL's.

### akura-app
5. Stage list, status buttons, add-stage form. Contractor-only controls.
6. Re-fetch after each mutation.

### Exit check
- [ ] Contractor adds a stage; marking `COMPLETED` persists across refresh.
- [ ] Client sees the same stage and status.
- [ ] Client attempting the PATCH via curl → **403**.
- [ ] A stage id from another project → **403**, not a silent update.
- [ ] `app-1.txt` has a readable business line for the update.

---

## Phase 4 — Approvals

**Goal:** the core loop closes. This is why the product exists.

### akura-api — `approvals` context
1. Domain: `Approval` entity, `Money` value object,
   **`decide()` throwing unless `PENDING`**, raising `ApprovalDecided`.
2. Infrastructure: `PrismaApprovalRepository`.
3. Application: `RaiseApproval` (contractor only), `DecideApproval` (**client only**),
   event handler logging the decision.
4. Interface: `POST /projects/:projectId/approvals`, `PATCH /approvals/:approvalId`.

### akura-app
5. Approvals list; approve/decline while `PENDING`, hidden once decided.
6. Raise-VO form (contractor only) with money conversion both directions.

### Exit check
- [ ] RM 3,200 stores as `320000`, displays as `3200.00`.
- [ ] Client approves → `APPROVED` with `decidedAt` set; contractor sees it on refresh.
- [ ] Buttons disappear once decided.
- [ ] **Contractor attempting `PATCH /approvals/:id` → 403.**
- [ ] **Re-deciding a decided approval → rejected, `decidedAt` unchanged.**
- [ ] `app-1.txt` records who decided what and when.

> When this passes you have a working product: a decision made by one person, recorded
> once, visible to both, with a timestamp that settles disputes.

---

## Phase 5 — Public site and SEO

**Goal:** something a contractor can find and understand before signing up.

### akura-app
1. `/` as a Server Component, statically generated — the problem, the product, a call
   to action. Written for a contractor tired of arguing over WhatsApp, not for
   "project management" generally.
2. `metadata` exports with OG and Twitter tags; `metadataBase` set.
3. `app/robots.ts` disallowing `/dashboard`, `/project/`, `/login`.
4. `app/sitemap.ts` listing public pages only.
5. JSON-LD `SoftwareApplication` on the landing page.
6. `next/font` and `next/image` where applicable.

### Exit check
- [ ] Build output shows `/` as static.
- [ ] `view-source:` shows real copy, not an empty div.
- [ ] `/robots.txt` and `/sitemap.xml` are correct.
- [ ] Lighthouse on `/`: Performance and SEO both 90+.

---

## Phase 6 — Hardening

1. Unit test `Approval.decide()` — happy path and the already-decided rejection. No
   Nest, no database. If it needs the app booted, the layering is wrong.
2. Unit test `Money` — rejects negatives and non-integers, converts correctly.
3. Re-run every 403 case from Phases 2–4 in one sitting.
4. Re-run both redaction greps.
5. Verify rotation: lower `LOG_MAX_SIZE_BYTES` temporarily, generate traffic, confirm
   the shift to `-2` and the 6-file cap, then restore.
6. Confirm the retention job runs (move the cron a minute ahead, watch it, restore).
7. Confirm no context imports another context's `domain/` or `infrastructure/`.
8. Confirm no `any` outside tests in either repo.
9. `npm audit` in both repos.
10. Frontend: loading states everywhere, errors shown to the user rather than swallowed.

### Exit check
- [ ] All 30 items in `BACKEND_SPEC.md` §12 pass.
- [ ] All 19 items in `FRONTEND_SPEC.md` §11 pass.

---

## Phase 7 — Deploy

1. **Buy `akura.my` first** (check MyIPO for trademark conflicts too). Both apps must
   sit on one registrable domain or the cookie has to become `sameSite: 'none'`, which
   removes the browser's CSRF protection.
2. Railway: deploy `akura-api`, add Railway Postgres in the same project. Set
   `JWT_SECRET` (fresh, not the dev one), `FRONTEND_ORIGIN=https://app.akura.my`,
   `COOKIE_DOMAIN=.akura.my`, `TZ`, `NODE_ENV=production`.
3. `prisma migrate deploy` against the production database.
4. Point `api.akura.my` at Railway.
5. Vercel: deploy `akura-app` with `NEXT_PUBLIC_API_URL=https://api.akura.my`. Point
   `app.akura.my` at Vercel.
6. **Re-run the Phase 1 and Phase 4 exit checks against production.** Cookie flags and
   CORS behave differently under HTTPS and are the most common thing to break.
7. Confirm `Secure` is set on the production cookie.
8. Set up a `logs/` volume on Railway — the container filesystem is ephemeral, so
   without a mounted volume your logs vanish on every redeploy.

---

## Rough effort

| Phase | Feel |
|---|---|
| 0 | Slowest relative to visible output. Logging and config are fiddly. |
| 1 | Second slowest — first pass through the CQRS pattern. |
| 2 | Faster; the pattern starts repeating. |
| 3 | Faster still. |
| 4 | Mechanical, apart from the `decide()` invariant. |
| 5 | Content-writing more than coding. |
| 6 | Short if you tested as you went. |
| 7 | Half a day plus DNS propagation. |

The CQRS overhead is front-loaded — six files for what would be one service method
feels absurd in Phase 1 and unremarkable by Phase 4.

---

## What comes after

Not part of the MVP. Revisit once a real contractor has used it.

| Next | Where it attaches |
|---|---|
| Email notifications | Subscribe to `ApprovalDecided` |
| Photo uploads | Presigned URLs; `photoUrl` already exists |
| Progress claim PDFs | Read completed stages + approved VOs |
| WhatsApp notifications | Same event seam, different transport |
| Queues (BullMQ) | Needs the persistent host you already chose |
| Microservice split | Contexts already isolated; swap in-process events for a broker |
