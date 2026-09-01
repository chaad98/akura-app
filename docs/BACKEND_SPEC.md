# Backend Spec — akura-core

NestJS + Prisma + PostgreSQL. TypeScript only.
Architecture: **CQRS + DDD, deployed as a modular monolith.**

Bounded contexts are isolated in code from day one so a later split into microservices
is a move plus a transport swap, not a rewrite. It deploys as **one app** until there
is a real scaling reason to split.

---

## 1. What this service is responsible for

One shared, timestamped record of a renovation project that both the contractor and
the client can see:

- **Stages** — the work checklist. Only the contractor updates these.
- **Approvals** — variation orders and design decisions. Only the client decides these.

The product exists to settle disputes. When a client approves a variation order, that
decision is recorded once, immutably, with a timestamp both parties can see. Treat
anything touching approvals as the highest-stakes code in the repo.

---

## 2. Architectural rules

These are what make the later microservice split cheap. Break them and there is no
point doing CQRS + DDD now.

1. **Four bounded contexts:** `identity`, `projects`, `progress`, `approvals`.
2. **No context imports another context's internals.** No importing a domain entity,
   repository, or handler across contexts. Cross-context communication goes through
   domain events or a published read model.
3. **The domain layer imports nothing from NestJS or Prisma.** No decorators, no
   `PrismaClient`. Pure TypeScript. If a domain entity can't be unit tested without
   booting Nest, the layering is wrong.
4. **Dependencies point inward.** Interface → Application → Domain. Infrastructure
   implements interfaces the domain declares. Never the reverse.
5. **Commands change state and return nothing meaningful** (an id at most).
   **Queries read and never write.**

### Layout per context

```
src/contexts/<context>/
├── domain/
│   ├── entities/          # aggregate roots, pure TS
│   ├── value-objects/     # Money, Email
│   ├── events/            # domain events
│   └── repositories/      # INTERFACES only
├── application/
│   ├── commands/          # command + handler pairs
│   ├── queries/           # query + handler pairs
│   └── events/            # domain event handlers
├── infrastructure/
│   └── persistence/       # Prisma implementations
└── interface/
    └── http/              # controllers, DTOs
```

Shared, cross-cutting:

```
src/shared/
├── logging/               # FileLoggerService, interceptor, filter (§9)
├── correlation/           # request id via AsyncLocalStorage
├── config/                # env schema + boot-time validation
└── domain/                # base AggregateRoot, DomainEvent
```

### Install

```
npm i @nestjs/cqrs @nestjs/jwt @nestjs/passport @nestjs/config @nestjs/throttler \
      @nestjs/schedule passport passport-jwt bcrypt cookie-parser helmet \
      class-validator class-transformer @prisma/client winston nestjs-cls zod
npm i -D prisma @types/passport-jwt @types/bcrypt @types/cookie-parser tsx
```

Use the latest Prisma. Prisma exists **only in this repo** — never in `akura-app`.

---

## 3. Domain model

### Value objects

**`Money`** — wraps an integer number of cents. Rejects negatives and non-integers in
the constructor. Exposes `fromRinggit(n)` and `toCents()`.

> Floats can't represent 0.1 exactly, so money in floats drifts. RM 3,200.00 is
> `320000` cents. The value object exists so the rule can't be forgotten at a call site.

**`Email`** — validates format on construction.

### Aggregates

**`User`** (identity) — id, email, passwordHash, name, role (`CONTRACTOR` | `CLIENT`).
`User.register(...)` is a static factory taking an already-hashed password. Hashing is
infrastructure; the domain never imports bcrypt.

**`Project`** (projects) — id, name, contractorId, clientId. `hasAccess(userId)`,
`isContractor(userId)`, `isClient(userId)`.

**`Stage`** (progress) — id, projectId, title, status, note?, photoUrl?.
Status: `PENDING` | `IN_PROGRESS` | `COMPLETED` | `BLOCKED`.
`updateProgress(status, note?, photoUrl?)` raises `StageProgressLogged`.

**`Approval`** (approvals) — id, projectId, title, description, cost (Money), status,
createdAt, decidedAt?. Status: `PENDING` | `APPROVED` | `DECLINED`.

`decide(decision)` — **must throw if status is not `PENDING`.** A decided approval is
settled; re-deciding destroys the audit trail. This invariant _is_ the product. On
decide, set `decidedAt` and raise `ApprovalDecided`.

### Cross-context authorization

`progress` and `approvals` need to know who the contractor and client are on a project
but must not import `Project`. `projects` publishes a read model
`project_participants` (projectId, contractorId, clientId) that other contexts query
through their own thin read port. A table read, not a class import — so the boundary
survives a later split into separate databases.

---

## 4. Persistence

Tables mirror the aggregates: `User`, `Project`, `Stage`, `Approval`.

- `Project` has two FKs to `User`, so Prisma needs **named relations**
  (`@relation("ContractorProjects")` / `@relation("ClientProjects")`) — without names
  the schema won't validate.
- Money is `Int` cents. Never `Float`, never `Decimal`.
- IDs are **UUIDs**, never sequential integers — sequential ids advertise row counts
  and make enumeration trivial.
- `decidedAt` is nullable, set only on decision.

**Indexes** — add these in the initial migration, not later:

```prisma
@@index([contractorId])   // Project
@@index([clientId])       // Project
@@index([projectId])      // Stage
@@index([projectId])      // Approval
```

Every list query filters on one of these. Without them you table-scan from the first
real project.

**Repositories:** interface in `domain/repositories/`, Prisma implementation in
`infrastructure/persistence/`, bound by token:

```ts
{ provide: PROJECT_REPOSITORY, useClass: PrismaProjectRepository }
```

Handlers inject the token, never the Prisma class.

**PrismaService** extends `PrismaClient`, implements `OnModuleInit` (`$connect()`) and
`OnModuleDestroy` (`$disconnect()`). Log both to the system log.

**Never use `$queryRawUnsafe`.** Prisma parameterizes everything by default, which is
why SQL injection is a non-issue here — `$queryRawUnsafe` is the one way to reintroduce
it. If raw SQL is ever genuinely needed, use `$queryRaw` with tagged template
parameters.

---

## 5. Commands and queries

### identity

| Type    | Name             | Notes                                  |
| ------- | ---------------- | -------------------------------------- |
| Command | `RegisterUser`   | 409 on duplicate email; bcrypt cost 12 |
| Command | `LoginUser`      | issues JWT                             |
| Query   | `GetUserByEmail` | used when creating a project           |

### projects

| Type    | Name                  | Notes                                                    |
| ------- | --------------------- | -------------------------------------------------------- |
| Command | `CreateProject`       | resolves client by email; 404 if missing or not `CLIENT` |
| Query   | `ListProjectsForUser` | filters by role; paginated                               |
| Query   | `GetProjectDetail`    | includes stages + approvals; 403 unless participant      |

### progress

| Type    | Name               | Notes                                         |
| ------- | ------------------ | --------------------------------------------- |
| Command | `CreateStage`      | contractor only                               |
| Command | `LogStageProgress` | contractor only; raises `StageProgressLogged` |

### approvals

| Type    | Name             | Notes                                     |
| ------- | ---------------- | ----------------------------------------- |
| Command | `RaiseApproval`  | contractor only                           |
| Command | `DecideApproval` | **client only**; raises `ApprovalDecided` |

> `DecideApproval` carries the single most important rule in the app: a contractor must
> never be able to approve their own cost request. Enforce it in the handler.

### Domain events

`StageProgressLogged` and `ApprovalDecided` have no subscribers in v1 beyond a handler
writing a business line to the plain-English log.

Raise them anyway. They are the seams where notifications and claim generation attach
later, and where a broker plugs in when contexts split. Raising them now costs
nothing; retrofitting means touching every handler.

---

## 6. HTTP interface

Controllers validate the DTO, dispatch a command or query, return the result. **No
business logic in controllers.**

| Method | Path                                   | Dispatches            | Who           |
| ------ | -------------------------------------- | --------------------- | ------------- |
| POST   | `/auth/register`                       | `RegisterUser`        | public        |
| POST   | `/auth/login`                          | `LoginUser`           | public        |
| POST   | `/auth/logout`                         | —                     | authenticated |
| GET    | `/auth/me`                             | —                     | authenticated |
| POST   | `/projects`                            | `CreateProject`       | contractor    |
| GET    | `/projects`                            | `ListProjectsForUser` | both          |
| GET    | `/projects/:id`                        | `GetProjectDetail`    | participants  |
| POST   | `/projects/:projectId/stages`          | `CreateStage`         | contractor    |
| PATCH  | `/projects/:projectId/stages/:stageId` | `LogStageProgress`    | contractor    |
| POST   | `/projects/:projectId/approvals`       | `RaiseApproval`       | contractor    |
| PATCH  | `/approvals/:approvalId`               | `DecideApproval`      | client        |

`PATCH /approvals/:id` is **not** nested under a project — approval ids are globally
unique and the client reaches it from a notification link.

**On the stage PATCH:** load the stage, then authorize against **the stage's own
`projectId`** — never the one in the URL. Otherwise a caller pairs a project they own
with a stage they don't.

**Pagination:** `GET /projects` takes `?page` and `?limit` (default 20, max 100).
Validate both. An unbounded list endpoint is a denial-of-service vector once real data
exists.

### Bootstrap (`main.ts`)

```ts
app.use(helmet());
app.use(cookieParser());
app.enableCors({ origin: config.FRONTEND_ORIGIN, credentials: true });
app.useGlobalPipes(
  new ValidationPipe({
    whitelist: true,
    forbidNonWhitelisted: true,
    transform: true,
  }),
);
app.useGlobalInterceptors(app.get(HttpLoggingInterceptor));
app.useGlobalFilters(app.get(AllExceptionsFilter));
app.enableShutdownHooks();
```

`whitelist` strips unknown body properties; `forbidNonWhitelisted` rejects them
outright with a 400, which surfaces client bugs instead of silently ignoring fields.

---

## 7. Auth — httpOnly cookies

The token is never in a response body and never touched by JavaScript.

### Setting the cookie

`/auth/register` and `/auth/login` return only `{ id, email, name, role }` and set the
cookie as a side effect:

```ts
res.cookie("access_token", token, {
  httpOnly: true,
  secure: config.NODE_ENV === "production",
  sameSite: "lax",
  domain: config.COOKIE_DOMAIN || undefined, // .akura.my in production
  maxAge: 7 * 24 * 60 * 60 * 1000,
  path: "/",
});
```

Controllers need `@Res({ passthrough: true })`.

### Reading it

```ts
jwtFromRequest: ExtractJwt.fromExtractors([
  (req: Request) => req?.cookies?.access_token ?? null,
]),
```

`JwtAuthGuard extends AuthGuard('jwt')` on every controller except `/auth/register`
and `/auth/login`.

### Logout

`POST /auth/logout` → `res.clearCookie('access_token', { ...same options })` → 204.
The clear options must **match the set options exactly** (domain, path, sameSite,
secure) or the browser keeps the cookie and logout silently fails.

### Domain strategy

Frontend at `app.akura.my`, backend at `api.akura.my`, `COOKIE_DOMAIN=.akura.my`.
Because both are on one registrable domain, `sameSite: 'lax'` works and the browser
provides CSRF protection for free. This is why the domain matters — deploying on
`*.vercel.app` + `*.railway.app` would be cross-site, forcing `sameSite: 'none'` and
requiring explicit CSRF defence.

### Other rules

- **Login failures return an identical message** whether the email is unknown or the
  password is wrong. Different messages let someone enumerate registered accounts.
- `GET /auth/me` returns the current user. The frontend cannot decode the token, so
  this is how it learns the role.

---

## 8. Security

The real risks for this application, in order.

### 8.1 Broken object-level authorization (the main one)

Someone with a valid login obtaining another project's id and reading it. UUIDs raise
the bar but are not the defence — **every query and command handler must verify the
caller is a participant.** This is why the 403 tests in §12 matter more than anything
else in this document.

Never trust an id from the URL as proof of ownership. Load the entity, check the
relationship, then act.

### 8.2 Information leakage

What people mean by "reverse engineering" a backend. The code never ships to clients;
what leaks is detail in responses.

- The global exception filter returns **generic messages**. `500` is
  `{ statusCode: 500, message: 'Internal server error' }` — no stack trace, no ORM
  error text, no SQL, no file paths.
- Full detail goes **into the logs** at ERROR level with the correlation id.
- Disable the `x-powered-by` header (Helmet does this).
- Validation errors may name the invalid field but must not echo internal structure.

### 8.3 Credential attacks

Rate limit with `@nestjs/throttler`. Global default ~100 req/min per IP, and a much
tighter limit on the auth endpoints specifically — roughly 5 attempts per minute on
`/auth/login` and `/auth/register`. Unlimited login attempts is the realistic attack
on an app like this, not injection.

bcrypt cost **12**.

### 8.4 Configuration

`@nestjs/config` with a **zod schema validated at boot**. The app must refuse to start
if `JWT_SECRET`, `DATABASE_URL`, `FRONTEND_ORIGIN`, or `COOKIE_DOMAIN` are missing or
malformed. A `JWT_SECRET` silently defaulting to a dev value in production is a total
compromise, and it fails silently — boot-time validation is what makes it loud.

`JWT_SECRET` must be a long random string, different per environment, never committed.

### 8.5 Transport and headers

Helmet for security headers. HTTPS only in production (Railway terminates TLS). CORS
with an explicit origin — never a wildcard, which cannot be combined with credentials
anyway.

### 8.6 Dependencies

`npm audit` before each deploy. Keep Prisma, NestJS, and `jsonwebtoken` current.

---

## 9. Performance

- **Indexes** on every filtered foreign key (§4).
- **No N+1 queries.** `GetProjectDetail` fetches stages and approvals in a single
  Prisma `include`, not per-row lookups.
- **Pagination** on list endpoints from day one (§6).
- **Connection pooling** — the default Prisma pool is fine for one instance; if you
  later move to a serverless host, the pool must be revisited.
- **Select only needed fields** in query handlers. Never return `passwordHash`, not
  even accidentally — map explicitly to a response shape rather than spreading the
  entity.

---

## 10. Environment

```
NODE_ENV=development
PORT=3001
DATABASE_URL=postgresql://...
JWT_SECRET=<long random string, unique per environment>
FRONTEND_ORIGIN=http://localhost:3000     # https://app.akura.my in production
COOKIE_DOMAIN=                            # empty locally; .akura.my in production
TZ=Asia/Kuala_Lumpur
LOG_DIR=./logs
LOG_MAX_SIZE_BYTES=104857600              # 100 MB
LOG_MAX_FILES=6
LOG_RETENTION_DAYS=90
LOG_TRUNCATE_BODY_CHARS=                  # empty = no truncation
THROTTLE_TTL=60
THROTTLE_LIMIT=100
THROTTLE_AUTH_LIMIT=5
```

`TZ` must be set in every environment — local, Docker, production. Never hardcode the
`+08:00` offset; changing timezone later must be a config change.

---

## 11. Logging

Implements `docs/MY-LOGGING-REQUIREMENTS.md`, which is the authority. This section maps
it onto this codebase.

### 11.1 One central class

A single `@Injectable() FileLoggerService`. No `console.log` anywhere, no per-file
logger instances.

Enforcement is structural, not disciplinary: a **global interceptor** logs every
successful request/response, a **global exception filter** logs every error. Both
registered in `main.ts` so no endpoint can bypass logging. Never a base controller
someone must remember to extend.

### 11.2 Three files

All in `LOG_DIR`, plain text `.txt`.

| File           | Content                                                                              | Format                      |
| -------------- | ------------------------------------------------------------------------------------ | --------------------------- |
| `http-1.txt`   | Every HTTP request/response, inbound and outbound                                    | NDJSON, one object per line |
| `app-1.txt`    | Human-readable summaries plus business events                                        | One sentence per line       |
| `system-1.txt` | Bootstrap, shutdown, module init, DB connect, config validation, uncaught exceptions | One line per event          |

Each is a winston `File` transport with:

```ts
{ maxsize: LOG_MAX_SIZE_BYTES, maxFiles: 6, tailable: true }
```

`tailable: true` produces the required numbering — current is always `-1`, older shift
down to `-6`, the seventh is deleted. **Never hand-write rotation.**

### 11.3 Line format

```
2026-08-19 14:32:07.481 +0800 [INFO] <content>
```

`YYYY-MM-DD HH:mm:ss.SSS ZZ`, then INFO / WARN / ERROR (DEBUG in dev only). Implement
as a custom winston formatter; because `TZ=Asia/Kuala_Lumpur` is set, local formatting
yields `+0800` without hardcoding.

> KL time is for **log display only**. The database and every API payload stay UTC /
> ISO 8601.

### 11.4 HTTP line contents

```
timestamp, level, direction (inbound|outbound), method, url, httpStatus,
durationMs, correlationId, requestBody, responseBody, retryAttempt?
```

If a response envelope with its own code is added later, log **both** the HTTP status
and the envelope code — a 200 carrying an internal error code is a failure the status
alone would hide.

### 11.5 Correlation id

Generate a UUID per request in middleware, store it in `AsyncLocalStorage` via
`nestjs-cls`, and include it on **every** line across all three files. Also include the
project id where one exists.

Goal: `grep <correlationId> logs/*` tells the complete story of one request.

### 11.6 Redaction — three leak paths

Redact before writing, inbound and outbound:

- Request headers: `authorization`, **`cookie`**, `x-api-key`
- **Response headers: `set-cookie`**
- **Body keys (recursive): `password`, `passwordHash`, `token`, `accessToken`,
  `secret`**

> **Path 1 — passwords in bodies.** Full bodies are logged by default and
> register/login carry plaintext passwords. Header-only redaction writes real user
> passwords to disk on every signup. Redaction must recurse over body keys.
>
> **Path 2 — the `Cookie` request header** carries the JWT on every authenticated
> request.
>
> **Path 3 — the `Set-Cookie` response header** carries the freshly issued JWT on
> login. Most redaction examples cover only request headers, so this is the one that
> slips through.

The redaction function handles values of unknown shape — type them `unknown` and walk
with type guards, never `any`. A loosely typed redactor fails silently, which is the
worst possible failure mode here.

Stack traces go into the logs at ERROR level, never into responses.

### 11.7 Retention job

`@nestjs/schedule` cron at `0 3 * * *`: scan `LOG_DIR`, delete files with mtime older
than `LOG_RETENTION_DAYS`, log the deletion to the system log. Rotation caps by size
and count; this caps by age. Both run.

### 11.8 Lifecycle events

Bootstrap start/complete with port, config validation result, Prisma connect/
disconnect, module init, shutdown signal, uncaught exception, unhandled rejection.
`enableShutdownHooks()` is required or shutdown lines never get written.

---

## 12. Acceptance checklist

### Functional

1. Register a contractor and a client (different emails).
2. Contractor creates a project using the client's email.
3. Contractor adds a stage, marks it `COMPLETED`.
4. Contractor raises an approval at `costCents: 320000`.
5. Client approves → `status: APPROVED`, `decidedAt` set.
6. Client fetches project detail → sees both.

### Auth

7. Login response body contains **no token** — only user fields.
8. `Set-Cookie` has `HttpOnly`, and `Secure` in production.
9. `POST /auth/logout` clears it; `GET /auth/me` then returns 401.
10. A request from an origin other than `FRONTEND_ORIGIN` is rejected by CORS.
11. Unknown email and wrong password produce **identical** error messages.

### Authorization — the cases that matter most

12. Contractor tries `PATCH /approvals/:id` → **403**.
13. Client tries to PATCH a stage → **403**.
14. An unrelated third user tries `GET /projects/:id` → **403**.
15. A stage id from another project passed to the stage PATCH → **403**.
16. Deciding an already-decided approval → **rejected** (domain invariant).

### Security

17. A forced 500 returns a generic message with **no stack trace**; the stack is in
    `system-1.txt`.
18. Six rapid failed logins → **429**.
19. Booting with `JWT_SECRET` unset → the app **refuses to start**.
20. A request body with an unknown extra field → **400**.
21. No response anywhere contains `passwordHash`.

### Architecture

22. A domain entity unit tests with no Nest, no Prisma, no database.
23. No file in one context imports another context's `domain/` or `infrastructure/`.
24. `grep -r "any" src/` finds no type annotations outside `*.spec.ts`.

### Logging

25. A successful request writes one NDJSON line to `http-1.txt` **and** one summary
    line to `app-1.txt`, both stamped `+0800`.
26. `grep -ri "password" logs/` after a registration returns **no plaintext password**.
27. `grep -ri "access_token=" logs/` returns nothing.
28. `grep <correlationId> logs/*` returns every line for that request.
29. Forcing a file past `LOG_MAX_SIZE_BYTES` shifts content to `-2` and never exceeds
    6 files per type.
30. A file with mtime older than 90 days is removed by the 03:00 job.

---

## 13. Out of scope for v1

| Feature                          | Why it waits                                         |
| -------------------------------- | ---------------------------------------------------- |
| Separate microservice deploys    | Contexts already isolated; split on a scaling reason |
| Event sourcing                   | CQRS does not require it; don't conflate them        |
| Separate read database           | One Postgres is fine                                 |
| Progress claim PDF generation    | Validate the core loop first                         |
| Notifications (email / WhatsApp) | `ApprovalDecided` is the seam                        |
| File upload handling             | Store a `photoUrl` string for now                    |
| Refresh tokens, password reset   | Before real users, not before a pilot                |
