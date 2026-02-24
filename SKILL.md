---
name: mern-stack
description: "Use this skill when building, reviewing, or architecting MERN stack applications (MongoDB, Express, React, Node.js). Triggers include: questions about React component patterns, state management, data fetching, TypeScript setup, Express API design, Mongoose schema design, folder structure, auth patterns, performance, or any combination of these. Covers pragmatic production standards — what to use, what to skip, and why — across the full stack."
version: "2026"
---

# MERN Engineering Playbook 2026

Pragmatic standards for scalable, maintainable production apps. What to do, what to skip, and why — with honest tradeoffs included.

---

## PART 1 — REACT

### Component Philosophy

**Do:**
- Functional components only
- Custom hooks for reusable logic (`useUser`, `useCart`, `useFilters`)
- React Hook Form for all forms
- Single responsibility per component

**Avoid:**
- Class components
- "God components" — one component fetching, filtering, sorting, validating, and rendering. Split it.
- Prop drilling past 2 levels — use query cache or context
- API calls inside component files

---

### Folder Structure

**Type-based** (works well up to ~50 features, small team):
```
src/
  components/    ← shared/reusable UI
  pages/         ← route-level pages
  hooks/         ← custom hooks
  utils/         ← helpers (cn, formatDate)
  types/         ← shared TypeScript types
  config/        ← constants, axios instance
```

**Feature-based** (switch when large team / 50+ features):
```
src/
  modules/
    user/
      components/
      hooks/
      types/
```

Pick one and be consistent. Type-based is simpler to start. Only migrate to feature-based when engineers own different modules and are stepping on each other.

---

### TypeScript in React

```ts
// ✅ Explicit prop types, no React.FC
type UserCardProps = {
  userId: string
  onDelete: (id: string) => void
}
const UserCard = ({ userId, onDelete }: UserCardProps) => { }
```

**Never use `React.FC<Props>`**. It adds implicit children, breaks generic component inference, and makes DevTools messier. No benefit.

---

### State Management

| State Type | Tool |
|---|---|
| Server data (API responses) | TanStack Query |
| Auth / current user | TanStack Query cache |
| Form state | React Hook Form |
| URL state | React Router params |
| UI globals (theme, modals) | Zustand — only if needed |

**Golden rule:** Don't duplicate server state into a global client store. TanStack Query owns server data — caching, deduplication, retries, background refetch. Treat its cache as a read layer.

Zustand is optional. If your auth and most global state live in the query cache, you may not need Zustand at all. Add it only for real client-only global state that doesn't belong in a query.

---

### Data Fetching

Always use `useQuery` and `useMutation`. Never `useEffect` + `fetch`.

```ts
// Query key constants — centralize them
export const USER_QUERY_KEY = ["current-user"] as const

const { data: user, isLoading } = useQuery<User>({
  queryKey: USER_QUERY_KEY,
  queryFn: ({ signal }) => api.get("/auth/me", { signal }).then(r => r.data),
  retry: false,
})
```

**Elaborate Query Key Factory patterns** (categoryQueryKeys.list()) are useful only when you have many related queries that need coordinated invalidation. For most apps, simple constant arrays are enough.

---

### Styling

Use TailwindCSS with a `cn()` utility for conditional classes:

```ts
// utils/cn.ts
import { clsx } from "clsx"
import { twMerge } from "tailwind-merge"
export default function cn(...inputs: Parameters<typeof clsx>) {
  return twMerge(clsx(inputs))
}
```

Vanilla CSS is fine for small projects. Tailwind wins at team scale — no naming battles, no dead CSS, utility classes are self-documenting.

---

### Forms

Always use React Hook Form + Zod resolver:

```ts
const schema = z.object({ email: z.string().email(), password: z.string().min(8) })
const { register, handleSubmit } = useForm({ resolver: zodResolver(schema) })
```

Never manage form state via `useState` per field. Never validate manually in `onSubmit`.

---

### Performance — Be Pragmatic

- Lazy load routes — always
- `React.memo` — only after profiling confirms unnecessary re-renders
- `useCallback` — only to stabilize callback references passed to memoized children
- `useMemo` — for genuinely expensive computations, not "just in case"
- Virtualization — add when lists exceed ~200 items

Premature memoization adds noise without benefit. Profile before optimizing.

---

### React Checklist

- [ ] Functional components only, no `React.FC`
- [ ] TanStack Query for all server state
- [ ] React Hook Form + Zod for all forms
- [ ] Centralized axios/fetch instance
- [ ] No `useEffect` for data fetching
- [ ] No `useState` for server data
- [ ] `cn()` utility for conditional Tailwind classes
- [ ] Zustand only when genuinely needed for client-only global state

---

## PART 2 — NODE.JS BACKEND

### Architecture: Right-Size Your Layers

The full layered architecture (Controller → Service → Repository → DB) is often recommended but adds boilerplate that only pays off at scale. Choose based on where you are:

**Thin controllers** (startup to mid-size):
```
Route → Middleware → Controller → Model → DB
```
Controllers validate input, call the ORM directly, return response. Works well when controllers are focused and ~20–50 lines.

**Service layer** (extract when needed):
```
Route → Middleware → Controller → Service → Model → DB
```
Extract a service when: business logic is shared across multiple controllers, complex multi-step operations need isolated testing, or you're running payment/financial workflows.

**Repository layer** (rarely necessary): Add only if you need to swap databases or heavily mock DB calls in unit tests. Most MERN apps don't swap databases.

> A 3-layer architecture on a 5-controller CRUD app is over-engineering. Add layers as the codebase earns them.

---

### Folder Structure

**Type-based** (simple, small-medium apps):
```
src/
  controllers/
    auth/
      Login.ts
      Logout.ts
    user/
      GetUser.ts
  routes/
    auth.routes.ts
    user.routes.ts
  models/
    User.model.ts
  middlewares/
  configs/
  shared/      ← email, s3, token utilities
  jobs/        ← background jobs
  utils/
```

**Domain-based** (large team / many features):
```
src/
  modules/
    user/
      user.route.ts
      user.controller.ts
      user.service.ts
      user.schema.ts
```
The domain-based structure is superior for cross-team ownership but adds overhead when one engineer owns the whole backend.
---

### Request Validation: Zod + Middleware

Validate in the route layer, before the controller ever runs:

```ts
// validateMiddleware.ts
function validate(schema: ZodSchema) {
  return (req: Request, res: Response, next: NextFunction) => {
    try {
      schema.parse({ body: req.body, query: req.query, params: req.params })
      next()
    } catch (err) {
      next(err) // ZodError handled by global error middleware
    }
  }
}

// auth.routes.ts
router.post("/login",
  validate(z.object({
    body: z.object({ email: z.string().email(), password: z.string().min(8) })
  })),
  loginController,
)
```

Separate DTO files are useful when the same schema is reused across multiple routes. For single-use schemas, inline Zod is cleaner.

---

### Error Handling

Global error middleware handles everything. No try/catch duplication:

```ts
// errorMiddleware.ts — must be last in Express chain
const errorMiddleware = (err: unknown, req: Request, res: Response, _: NextFunction) => {
  if (err instanceof ZodError) { res.status(400).json({ success: false, message: err.errors[0]?.message }); return }
  if (err instanceof Error) { res.status(500).json({ success: false, message: "Internal server error" }); return }
  if (typeof err === "string") { res.status(400).json({ success: false, message: err }); return }
  res.status(500).json({ success: false, message: "Something went wrong" })
}
app.use(errorMiddleware)
```

Custom `AppError` class is worth adding when you have a service layer needing domain-specific errors with status codes. Without a service layer, it's mostly ceremony.

---

### Logging

Never `console.log` in production. Use **Winston** (reliable, battle-tested) or **Pino** (fastest).

```ts
logger.info("User created", { userId, email })
logger.error("Payment failed", { error, userId, amount })
logger.http(`${req.method} ${req.url} ${res.statusCode} ${latency}ms`)
```

**Log:** request method/url/status, user ID, latency, error stacks.
**Never log:** passwords, tokens, PII in plaintext.

---

### Background Jobs

**Cron jobs** (`node-cron` or `cron`) are sufficient for: periodic reports, notifications, currency rate syncs, cleanup tasks.

```ts
new CronJob("0 9 * * *", notifyFinanceTeam).start()
```

**BullMQ + Redis** is worth adding when you need: job queuing with retries and dead letter queues, distributed workers, job priority/rate limiting, or real-time progress tracking.

Don't start with BullMQ. Upgrade to it when cron limitations become real problems.

---

### Config Management

Single config module. Never scatter `process.env` throughout the codebase:

```ts
// config.ts — validate and export once
const config = {
  MONGO_URI: process.env.MONGO_URI!,
  JWT_SECRET: process.env.JWT_SECRET!,
  PORT: process.env.PORT || 8888,
  AWS: {
    BUCKET: process.env.AWS_BUCKET_NAME!,
    REGION: process.env.AWS_REGION!,
  },
}
export default config
```

---

### Node Checklist

- [ ] Right-sized architecture (add layers only when needed)
- [ ] Zod validation middleware — no raw `req.body` in controllers
- [ ] Global error middleware (last in Express chain)
- [ ] Structured logger (Winston or Pino) — no `console.log`
- [ ] `Promise.all` for parallel async operations
- [ ] Cron for periodic jobs; BullMQ only when genuinely needed
- [ ] Centralized config module
- [ ] API versioning (`/api/v1`)

---

## PART 3 — EXPRESS API LAYER

### Request Lifecycle

Every request should follow this chain:
```
Request
  ↓ Global middleware (cors, json, cookieParser)
  ↓ Request logger (method, url, latency)
  ↓ Auth middleware
  ↓ Validation middleware (Zod)
  ↓ Controller
  ↓ Model / Service
  ↓ Response
  ↓ Error middleware (if next(err) called)
```

Break this chain → the codebase becomes unpredictable.

---

### Routes = Traffic Police

Routes only: define endpoints, attach middleware, call controller.

```ts
// ✅ Clean route
router.post("/users",
  authMiddleware,
  validate(createUserSchema),
  userController.create,
)

// ❌ Never this
router.post("/users", async (req, res) => {
  const user = await User.create(req.body) // validation? auth? error handling? 🫠
  res.json(user)
})
```

---

### Auth Middleware

```ts
const { payload } = await jwtVerify(token, secret) // use jose, not jsonwebtoken
req.user = payload
next()
```

Prefer **jose** over `jsonwebtoken` — ES module native, actively maintained, better TypeScript types.

Use **HttpOnly cookies** for JWT (not localStorage). Pair with short-lived access tokens + long-lived refresh tokens.

---

### Response Shape — Be Consistent

Pick a shape and stick to it across every endpoint:

```json
{ "success": true, "data": { } }
{ "success": false, "message": "...", "status": 400 }
```
Inconsistent response shapes are a common pain point for frontend teams. A simple response utility function eliminates this:

```ts
// responseUtil.ts
export const ok = (res: Response, data: unknown) => res.json({ success: true, data })
export const fail = (res: Response, message: string, status = 400) =>
  res.status(status).json({ success: false, message })
```

---

### Security (Non-Negotiable)

```ts
app.use(helmet())                          // HTTP security headers
app.use(cors({ origin: whitelist, credentials: true }))
app.use(mongoSanitize())                   // prevent NoSQL injection
router.post("/login", loginLimiter, controller) // rate limit auth routes
```

Bcrypt with salt rounds ≥ 12. Never store secrets in code.

---

### API Versioning

Always prefix routes: `/api/v1/resource`. Non-negotiable from day one. Without it: breaking changes crash mobile apps, third-party clients, and integrated systems.

---

### Express Checklist

- [ ] Routes only define endpoints + middleware chain
- [ ] Zod validation middleware
- [ ] Auth middleware (jose JWT, HttpOnly cookies)
- [ ] API versioning from day one
- [ ] Global error middleware
- [ ] Consistent response shape
- [ ] Helmet + CORS + mongo-sanitize
- [ ] Rate limiting on auth endpoints

---

## PART 4 — MONGODB + MONGOOSE

### Use InferSchemaType for TypeScript Types

Don't write separate interfaces for Mongoose models. Let the schema be the source of truth:

```ts
const userSchema = new mongoose.Schema({
  email: { type: String, unique: true, lowercase: true, required: true },
  permissions: { type: [String], default: [] },
}, { timestamps: true, minimize: true })

export type User = InferSchemaType<typeof userSchema>
export default mongoose.model<User>("user", userSchema)
```

This keeps types automatically in sync with schema. No drift.

---

### Schema Options (Always)

```ts
{
  timestamps: true,    // createdAt + updatedAt always
  minimize: true,      // strip empty objects (saves storage)
}
```

---

### Indexes

Index every field you query or sort by:

```ts
companyId: { type: ObjectId, index: true }    // foreign keys
email:     { type: String,   unique: true }    // unique constraint = index
status:    { type: String,   index: true }     // frequently filtered

// Compound index for pagination and dashboard queries
userSchema.index({ companyId: 1, createdAt: -1 })
```

Don't over-index. Each index increases write time and uses RAM. Only index fields that appear in `find()`, `sort()`, or aggregation `$match`.

---

### Aggregation Over `populate()`

`.populate()` is a hidden N+1 query. Avoid it for list endpoints. Use aggregation:

```ts
UserModel.aggregate([
  { $match: { companyId: id } },            // filter first — always
  { $project: { name: 1, email: 1 } },      // reduce payload early
  { $lookup: { from: "companies", localField: "companyId", foreignField: "_id", as: "company" } },
  { $addFields: { company: { $arrayElemAt: ["$company", 0] } } },
])
```

`populate()` is acceptable on detail/single-document pages where N+1 is just 1 extra query.

---

### `lean()` for Read Endpoints

```ts
// ✅ 30–50% faster, returns plain JSON (not Mongoose Document instances)
const users = await UserModel.find({ companyId }).lean()
```

Skip `lean()` only when you need to call `.save()`, use virtuals, or trigger Mongoose middleware.

---

### `Promise.all` for Parallel Operations

```ts
// ✅ Parallel — total time = slowest operation
await Promise.all([save(), sendEmail(), logActivity()])

// ❌ Sequential — total time = sum of all
await save()
await sendEmail()
await logActivity()
```

---

### Pagination

**Offset pagination** (`skip()`) — fine for small datasets:
```ts
Model.find({}).skip(page * limit).limit(limit)
```

**Cursor pagination** — use when data grows large:
```ts
Model.find({ createdAt: { $lt: lastSeen } }).limit(10).sort({ createdAt: -1 })
```

Cursor pagination scales infinitely. `skip(50000)` makes Mongo scan 50k documents first.

---

### Soft Delete

Never hard-delete user-facing data in production:

```ts
// Schema
deletedAt: { type: Date, default: null }

// All queries include:
Model.find({ deletedAt: null })
```

---

### Transactions

Use sessions for multi-document operations that must be atomic:

```ts
const session = await mongoose.startSession()
session.startTransaction()
// debit + credit + log — all or nothing
await session.commitTransaction()
```

Required for: payments, wallet transfers, inventory deductions. Don't rely on document atomicity for operations spanning multiple collections.

---

### Mongoose Checklist

- [ ] `InferSchemaType` for model types
- [ ] `timestamps: true` and `minimize: true` on every schema
- [ ] Index foreign keys and filtered/sorted fields
- [ ] Aggregation pipeline for joins (not `populate()` on lists)
- [ ] `lean()` on all read-only endpoints
- [ ] `Promise.all` for independent DB operations
- [ ] Soft delete (`deletedAt`) for user-facing data
- [ ] Transactions for multi-document atomic operations
- [ ] `$match` as early as possible in every aggregation

---

## PART 5 — TYPESCRIPT

### tsconfig Baseline

```json
{
  "compilerOptions": {
    "strict": true,
    "skipLibCheck": true,
    "verbatimModuleSyntax": true,
    "noFallthroughCasesInSwitch": true,
    "noUncheckedIndexedAccess": true,
    "moduleResolution": "bundler"
  }
}
```

- `strict: true` is non-negotiable. Without it, TypeScript gives false confidence.
- `verbatimModuleSyntax: true` enforces `import type` for type-only imports, preventing runtime bundle pollution.

---

### Inference Over Annotation

```ts
// ❌ Redundant annotation
const name: string = "John"

// ✅ Inferred
const name = "John"
const items: string[] = []  // annotation useful here — empty array needs a hint
```

Always annotate: function parameters, exported interfaces, API return types, DTO shapes.

---

### `interface` vs `type`

```ts
// interface — object shapes, extendable domain models
interface User { _id: string; email: string; permissions: string[] }

// type — unions, tuples, mapped types, utility composition
type Role = "admin" | "user" | "viewer"
type UserOrNull = User | null
```

---

### Never `any` — Use `unknown`

```ts
// ❌ Kills all type safety downstream
function process(data: any) { }

// ✅ Safe, narrow before use
function process(data: unknown) {
  if (typeof data === "string") return data.toUpperCase()
}
```

---

### `import type` is Required

With `verbatimModuleSyntax: true`, type-only imports must use `import type`:

```ts
import type { Request, Response, NextFunction } from "express"
import type { User } from "@/models/User.model"
```
This is enforced at compile time and prevents types from bleeding into the runtime bundle.
---

### Path Aliases

Configure `@/*` to map to `./src/*` in both `tsconfig.json` and your bundler:

```ts
// Instead of:
import config from "../../../configs/config"
// Use:
import config from "@/configs/config"
```

---

### Null Safety

```ts
// ❌ Runtime crash waiting to happen
user.name.length

// ✅ Safe access
user?.name?.length ?? 0
```

With `strictNullChecks` (part of `strict: true`), TypeScript catches most of these at compile time.

---

### Generic Utilities

```ts
// Reusable across users, orders, products
async function fetchData<T>(url: string): Promise<T> {
  const res = await api.get(url)
  return res.data
}
```

---

### TypeScript Checklist

- [ ] `strict: true` always
- [ ] `verbatimModuleSyntax: true` → enforce `import type`
- [ ] `noUncheckedIndexedAccess` — safe array/object access
- [ ] Path aliases configured (`@/*`)
- [ ] `interface` for object shapes, `type` for unions
- [ ] `InferSchemaType` for Mongoose model types
- [ ] No `any` — use `unknown` + narrowing
- [ ] Always annotate function parameters and public API types
- [ ] Optional chaining + nullish coalescing for null safety

---

## CROSS-STACK STANDARDS

### Dev Tooling (Non-Negotiable)

| Tool | Purpose |
|---|---|
| ESLint | Catch bugs and enforce patterns |
| Prettier | Formatting (no style debates) |
| Husky | Git hooks |
| lint-staged | Only lint/format changed files |
| commitlint | Enforce conventional commits (`feat:`, `fix:`) |
| TypeScript strict | Type safety |

Run Prettier and ESLint on every commit via Husky. No exceptions.

---

### Auth Pattern

```
Login (OAuth2/OIDC or username+password)
  → Issue Access Token (short-lived, jose JWT) + Refresh Token (long-lived)
  → Set HttpOnly cookies (__Secure- prefix in production)

Per request:
  → Auth middleware reads cookie-first, Authorization header as fallback
  → Verify with jose jwtVerify
  → Attach req.user

Token refresh:
  → POST /auth/refresh → issue new access token
```

Use **jose** (not `jsonwebtoken`) — native ES modules, better TypeScript support, actively maintained.

---

### What's Worth Adding vs. What's Often Over-Engineered

| Practice | Add When | Skip If |
|---|---|---|
| Service layer | Business logic shared across 3+ controllers | Simple CRUD, single-owner backend |
| Repository layer | Swapping databases, heavy unit testing | Mongo-only app, no test mocking needed |
| BullMQ job queues | Multi-worker, retries, rate limiting needed | Periodic jobs → use cron instead |
| DI framework | Enterprise scale, 10+ services | Most MERN startups |
| Feature-based folders | 50+ features, multi-team ownership | Small team, single codebase owner |
| Zustand | Real client-only global state needed | Query cache covers auth + server state |
| Redis caching | High-traffic read-heavy endpoints | API response times already acceptable |
| Cursor pagination | Lists grow beyond thousands of records | Small datasets with offset pagination |
| Virtualized lists | Lists exceed 200+ visible items | Normal-size data tables |
| Custom AppError class | Service layer throwing domain errors | Thin controllers with no service layer |