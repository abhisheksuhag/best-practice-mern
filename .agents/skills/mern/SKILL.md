---
name: mern
description: A strict 2026 MERN stack engineering playbook with comprehensive code examples for React, Node, Express, and MongoDB.
---

# MERN ENGINEERING PLAYBOOK 

Modern Best Practices for Scalability, Maintainability & Performance.
Basically: "If someone joins my startup tomorrow — this is how we write React, Node, Express, Mongo in 2026 so that the codebase doesn't become a nightmare in 12 months." Not theory. Not tutorials. Just: what to do / what not to do in production-grade MERN apps.

---

## 🟦 PART 1 — REACT (CLIENT ARCHITECTURE)

This section answers: How should we write React apps in 2026 so they remain scalable after 50+ features?

### 1. Component Philosophy

✅ **Use:**

- Functional Components only
- Composition over inheritance
- Custom Hooks for logic reuse
- Controlled components for forms
- Presentational vs Container separation
- Headless UI approach (logic separated from UI)

```javascript
// Example:
const useCategories = () => {
  return useQuery(categoryQueryKeys.list(), fetchCategories)
}

// UI stays clean:
const CategoryList = () => {
  const { data } = useCategories()
}
```

❌ **Avoid:**

- Class Components
- Large "God Components" (Massive 400-line components)
- Prop Drilling → use Context/Zustand
- Business logic inside UI components
- API calls inside JSX files
- Components managing unrelated state

If your component fetches data, filters it, sorts it, validates it, and mutates it, that is NOT a component anymore. That's a mini-backend 😭. Move logic to: `hooks/`, `queries/`, `store/`, `utils/`.

---

### 2. TypeScript Rules

✅ **Always write like this:**

```typescript
type CategoryFormProps = {
  onSubmit: (data: CategoryInput) => void // (or data: Category)
}

const CategoryForm = ({ onSubmit }: CategoryFormProps) => {}
```

❌ **Never use:** `const CategoryForm: React.FC<Props>`

Why? React.FC adds implicit children, messes with generics (bad generic inference), breaks defaultProps typing, makes inference harder, and makes DevTools debugging messier.

---

### 3. State Management Strategy (2026 Standard)

Modern React has 4 types of state:

| State Type | Example | Tool |
|---|---|---|
| **UI State** | modal open | Zustand |
| **Server State** | users from DB | Tanstack Query |
| **Form State** | login form | React Hook Form |
| **URL State** | page=2 | React Router / nuqs |
| **Global Auth** | — | Zustand + Persist |

🔥 **GOLDEN RULE:** Server State ≠ Client State

❌ **NEVER:** Store API response in Zustand, store React Query data in Redux, or duplicate backend data locally. Let React Query own server data lifecycle: caching, retries, background refetch, deduping, and stale logic. Avoid Redux unless enterprise-level scale, and avoid `useState` for shared state.

---

### 4. Data Fetching Pattern

❌ **Old Pattern:** Fetch inside `useEffect`

```javascript
useEffect(() => {
  fetch('/api/users')
}, [])
```

This causes: race conditions, double fetching, stale UI, no caching, bad retries, and manual loading states.

✅ **New Standard:** Always use `query.ts`, `mutation.ts`, and `queryKeys.ts`. Use `useQuery()`, `useMutation()`, Query Key Factory pattern, Optimistic Updates, and Suspense Mode.

```typescript
// Example:
export const categoryQueryKeys = {
  all: ['categories'] as const,
  list: () => [...categoryQueryKeys.all, 'list'] as const,
}

export const useCategories = () =>
  useQuery({
    queryKey: categoryQueryKeys.list(),
    queryFn: fetchCategories,
    staleTime: 5 * 60 * 1000
  })
```

---

### 5. Folder Structure (Feature First)

✅ **Use:**

```
src/
  modules/
    category/
      components/
      hooks/
      api/
      types/
      category.query.ts
      category.mutation.ts
      category.store.ts
```

❌ **Avoid:** Root-level `components/`, `pages/`, `utils/`, `services/`. Type-based structure doesn't scale. Feature-based = modular.

---

### 6. Forms (Huge Source of Bugs)

✅ **Use:** React Hook Form and Zod Resolver.

❌ **Never:** Manage form state manually, use `useState` for inputs, or validate in `onSubmit`.

---

### 7. Performance Rules

✅ **Use:** Lazy loading for routes (with Suspense), `React.memo` only when needed, `useCallback` only for child prop stability (handlers passed down), Virtualized Lists (`react-virtual`) for large tables.

❌ **Avoid:** Premature memoization (memoizing everything), inline anonymous functions/heavy computations in heavy lists/JSX, derived state stored in state.

- **Bad:** `const [total, setTotal] = useState(cart.reduce(...))`
- **Good:** `const total = useMemo(() => cart.reduce(...), [cart])`

---

### 8. Global State (Zustand Pattern)

Use Zustand **ONLY** for: auth state, theme, UI toggles, and local preferences.

**Never:** backend data, form inputs, server responses.

---

### 9. Error + Loading UI

- **Never:** Handle loading manually everywhere.
- **Use:** React Query Suspense and Error Boundaries.

---

### 10. API Layer Pattern

Never call `axios` inside `component.tsx`. Always use `api/category.api.ts`.

---

### 🚨 FINAL REACT RULES CHECKLIST

✔ Hooks > Classes ✔ Tanstack Query for server state ✔ Zustand for UI state ✔ React Hook Form for forms ✔ Zod for validation ✔ Feature-based folder structure ✔ API layer abstraction ✔ No fetch in useEffect ✔ No React.FC ✔ No global server state

---

## 🟩 PART 2 — NODE.JS BACKEND ARCHITECTURE (2026 STANDARD)

### 1. First Principle: Your Backend is NOT MVC

Classic MVC (route → controller → model) looks nice in tutorials but becomes controllers with 300 lines, models called directly, business logic duplicated in 7 controllers, impossible testing, and tight coupling in real apps. Avoid MVC-only backend (doesn't scale well).

✅ **Use This Instead:** Layered Architecture OR Domain Driven Modular Architecture. `route → controller → service → repository → database`. Each layer has ONLY ONE responsibility.

| Layer | Responsibility |
|---|---|
| **Route** | define endpoint |
| **Controller** | HTTP handling |
| **Service** | business logic |
| **Repository** | DB interaction |
| **Model** | schema only |

🚨 **Rule:** Controllers should NEVER talk to database directly. If you see `User.find()` inside controller → architecture violation.

---

### 2. Project Structure (Scalable)

❌ **Avoid:** `controllers/`, `models/`, `routes/`, `services/`. This type-based structure breaks at scale.

✅ **Use Domain-Based Structure:**

```
src/
  modules/
    user/
      user.route.ts
      user.controller.ts
      user.service.ts
      user.repository.ts
      user.schema.ts
      user.dto.ts
```

Everything related to "user" lives in one module. Now: features are isolated, testing becomes easy, onboarding is faster, and less merge conflicts.

---

### 3. Controller Rules

Controllers are the translation layer between HTTP world and business world.

✅ **Controller SHOULD:** Validate input, Call service, Return response.

❌ **Controller SHOULD NOT:** Query DB, Run business logic, Transform domain logic, Call external APIs, Perform calculations.

- **Bad:** `const user = await User.findById(id); user.balance += 200; await user.save()`
- **Good:** `await userService.addBalance(id, 200)`

---

### 4. Service Layer (Your Real Backend)

Service Layer contains: business rules, workflows, calculations, multi-db operations, external integrations.

Example: `await walletService.transferMoney(sender, receiver, amount)`. Inside: validate balance, debit, credit, transaction logging, rollback logic. None of this belongs in controller.

---

### 5. Repository Layer (DB Abstraction)

Service should NOT care if you are using Mongo, Postgres, Redis, or Firestore. Repository handles DB.

Example: `userService → userRepository → mongoose`. Now: You can replace Mongo later without rewriting business logic.

---

### 6. Async Handling Pattern

❌ **Avoid:** Nested awaits or blocking operations (sync fs, crypto).

✅ **Use:** async/await everywhere.

```javascript
// Promise.all for parallel tasks
await Promise.all([
  save(),
  sendMail(),
  log()
])
```

Parallel execution = faster APIs.

---

### 7. Error Handling System

**Never:** throw raw strings, send stack traces to client, or write try/catch in every controller.

✅ **Create Custom Error Class and Global Middleware:**

```javascript
class AppError extends Error {
  constructor(message, statusCode) {
    super(message)
    this.statusCode = statusCode
  }
}

// Controller:
next(new AppError('User not found', 404))
```

Use Error Codes (not only messages).

---

### 8. DTO Pattern (MANDATORY)

Never trust `req.body`. Use DTOs (e.g., `createUser.dto.ts`) to ensure validation, shape consistency, type safety, and prevent overposting attacks.

```typescript
// Controller:
const dto = createUserSchema.parse(req.body)
await userService.create(dto)
```

Use Zod; avoid manual validation.

---

### 9. Logging Strategy

Never use `console.log()` in production. Use Pino (fastest) for structured JSON logs.

```json
{
  "level": "info",
  "service": "auth",
  "userId": "123"
}
```

Logs should track: `requestId`, `userId`, `module`, `latency`.

---

### 10. Background Jobs (Don't Block APIs)

Never `await sendEmail()` inside a signup API. The user should not wait for the email service.

Use BullMQ, Worker Threads, or Message Queue. Flow: `signup → queue email job → API returns immediately`.

---

### 11. Dependency Injection (Important)

Avoid `import userRepository` inside service directly.

Instead, inject dependency: `new UserService(userRepository)`. Now it is testable, replaceable, and loosely coupled.

---

### 12. Config Management

Never use `process.env.JWT_SECRET` everywhere inside 25 files. Use `config/env.ts` and `config/db.ts` to export validated config once.

---

### 🚨 FINAL NODE RULES CHECKLIST

✔ Domain-based modules ✔ Controller thin ✔ Service = business logic ✔ Repository = DB access ✔ DTO validation ✔ Custom Error System ✔ Global Error Middleware ✔ Structured logging ✔ Background jobs for async tasks ✔ Promise.all for parallel work ✔ Dependency Injection ✔ Central config management

---

## 🟨 PART 3 — EXPRESS API LAYER (2026 STANDARD)

Express is NOT your backend logic. Express is your **Transport Layer**.

It should only: receive request, validate, authenticate, pass to controller, send response. Nothing else.

---

### 1. Request Lifecycle (MANDATORY FLOW)

Every API request should follow:

```
Request
  ↓ Global Middleware
  ↓ Auth Middleware
  ↓ Validation Middleware
  ↓ Controller
  ↓ Service
  ↓ Repository
  ↓ DB
  ↓ Response Formatter
```

If this flow breaks → codebase becomes unpredictable.

---

### 2. Route Design Pattern

❌ **Never:** Put logic in routes, DB calls inside controller, or business logic inside route.

- **Bad:** `router.post('/user', async (req,res)=>{ const user = await User.create(req.body) })`

✅ **Routes Should Only:** Map endpoint → middleware → controller.

```javascript
router.post(
  '/',
  validate(createUserSchema),
  authMiddleware,
  userController.createUser
)
```

Routes = traffic police 🚦 They just direct the request. Controllers validate input, call service, return response. Services handle business logic and DB calls.

---

### 3. Middleware Layering

Your Express app should have 3 middleware types:

| Type | Purpose |
|---|---|
| **Global** | CORS, Helmet |
| **Route** | Auth |
| **Feature** | Validation |

- **App Level:** `app.use(cors()); app.use(helmet()); app.use(requestLogger)`
- **Route Level:** `router.use(authMiddleware)`
- **Feature Level:** `router.post('/', validate(schema), controller)`

---

### 4. Validation Flow

Never check manual `req.body` in controllers (e.g., `if(!req.body.email)`).

✅ **Use:** Zod / Joi and Celebrate middleware.

```javascript
export const validate =
  (schema) =>
  (req, res, next) => {
    schema.parse(req.body)
    next()
  }
```

Now controller only receives valid data.

---

### 5. Auth Middleware Pattern

Never decode JWT inside controller (e.g., `jwt.verify()`).

✅ **Create:** `auth.middleware.ts`. Attach user to request: `req.user = decodedUser`. Now downstream layers use: `req.user.id`. Clean separation.

---

### 6. Response Formatter (VERY IMPORTANT)

Avoid: `res.json(user)`, `res.send(data)`, `res.status(200).json()` — Different devs = different response shapes 😭

✅ **Create:** `response.util.ts`. Example: `res.success(data)`, `res.error(message)`.

Now every API response becomes predictable frontend handling:

```json
{
  "success": true,
  "data": {}
}
```

---

### 7. API Versioning

Never use `/api/users`. Use `/api/v1/users`. Later: `/api/v2/users`. Avoids breaking mobile apps, old clients, and frontend crashes.

---

### 8. Rate Limiting & Security (Production Mandatory)

Use Helmet, CORS config, JWT Rotation, Bcrypt with salt rounds ≥ 10, and `express-rate-limit`. Avoid storing secrets in code or sending stack traces to client.

Protect: login, OTP, public endpoints. Example: Attach `loginLimiter` → `router.post('/login', loginLimiter, controller)`.

---

### 9. Centralized Error Handling

Never use `res.status(500)` inside controller.

- **Controller:** `next(new AppError('Not found', 404))`
- **Global Middleware:** `error.middleware.ts` handles formatting, status code, logging.

---

### 10. Async Wrapper

Avoid try/catch everywhere. Use `asyncHandler(controller)` to wrap controller once for a cleaner codebase.

---

### 11. Request Logging

Track: `requestId`, `userId`, `route`, `latency`. Attach requestId: `req.id = uuid()`. Now logs become traceable in production.

---

### 12. Route Protection Strategy

| Route Type | Middleware |
|---|---|
| **Public** | none |
| **Private** | auth |
| **Admin** | auth + role |

Example: `authorize('admin')`.

---

### 🚨 FINAL EXPRESS RULES CHECKLIST

✔ No logic in routes ✔ Validation middleware ✔ Auth middleware ✔ Versioned APIs ✔ Global error handler ✔ Async wrapper ✔ Response formatter ✔ Request logging ✔ Rate limiting ✔ Role-based route protection

---

## 🟥 PART 4 — MONGODB + MONGOOSE (2026 STANDARD)

### 1. Golden Rule of MongoDB

MongoDB is a **Query-first** database. Not relation-first, normalization-first, or table-first. You design based on: **How the data will be read most frequently.**

---

### 2. Embed vs Reference (MOST IMPORTANT DECISION)

✅ **EMBED WHEN:**
- One-to-Few relationship, frequently read together, data size is small, data rarely updated separately.
- Examples: user → addresses, product → tags, order → shipping status.

✅ **REFERENCE WHEN:**
- One-to-Many relationship, data grows over time, frequently updated independently, used across collections.
- Examples: user → orders, product → reviews, course → students.

🚨 **Avoid:** Unbounded arrays (e.g., `comments: []`). This can break Mongo's 16MB document limit later.

---

### 3. Hybrid Pattern (Production Standard)

Use Hybrid when needed. Embed frequently read fields, reference heavy dynamic data.

Example `order`: `userId`, `total`, `itemsSummary` (embedded), `itemsRef` (referenced). Fast reads + scalable writes.

---

### 4. Must Use Schema Options

Always use `timestamps: true` and `versionKey: false`. Also use `toJSON: { virtuals: true }` and `toObject: { virtuals: true }` — needed for computed fields.

---

### 5. Index Strategy (NON-NEGOTIABLE)

If your query filters → it needs an index. Avoid `find()` without index.

- **Index:** email, status, createdAt, foreign keys, sort fields.
- **Compound Indexes:** `userId + createdAt` needed for pagination and dashboard queries.

🚨 **Avoid:** Indexing everything. Each index increases write time and consumes memory.

---

### 6. Query Optimization: Use lean() for Read APIs

Normal Mongoose returns heavy document instances. You don't need that for APIs.

Always use `.lean()` for reads (e.g., `User.find().lean()`). Lean is 30–50% faster, uses lower memory, and returns plain JSON.

Avoid lean when: using virtuals, using middleware, or modifying document.

---

### 7. Pagination Pattern

Never use `skip(50000)` — Mongo will scan 50k docs first 😭

Use **Cursor Pagination** based on `createdAt` or `_id`.

```javascript
// Pattern: Scales infinitely.
find({ createdAt: { $lt: lastSeen } })
  .limit(10)
  .sort({ createdAt: -1 })
```

---

### 8. Aggregation Pipeline Rules

Aggregation is Mongo's SQL JOIN + GROUP BY equivalent.

Always:
1. Use `$match` first / early.
2. Use `$project` early to reduce payload.
3. Use indexed fields.
4. Use `$facet` for dashboards / analytics.
5. Use `$lookup` only when necessary / sparingly.

🚨 **Avoid:** Aggregation without index support or pipeline before filtering (`$lookup → $match` explodes memory usage).

---

### 9. populate() Usage

Never use `.populate('orders')` for lists (Populate = hidden join). Use aggregation or manual join for large datasets. Populate okay only for detail pages.

---

### 10. Soft Delete Pattern

Never use `User.deleteOne()`. Avoid hard deletes in production apps.

Use `deletedAt: Date | null`. Query: `{ deletedAt: null }`.

---

### 11. Virtual Fields

Use for `fullName`, computed totals, derived stats. Example: `userSchema.virtual('fullName')`. No DB storage needed.

---

### 12. Projection (Huge Performance Gain)

Never use `User.find()` blindly. Use `.select('name email')` for less payload → faster APIs.

---

### 13. Pre/Post Middleware

Use for hashing password, audit logs, timestamps, cascading soft delete. Avoid heavy logic inside middleware.

---

### 14. Transaction Usage

Use for payments, wallet updates, inventory deduction with `session.startTransaction()`. Never rely on Mongo atomicity for multi-document ops.

---

### 🚨 FINAL MONGOOSE RULES CHECKLIST

✔ Embed small related data ✔ Reference growing data ✔ Hybrid for balance ✔ Always index query fields ✔ Use lean() for reads ✔ Cursor pagination ✔ Match early in aggregation ✔ Avoid populate for lists ✔ Use soft delete ✔ Use projections ✔ Use transactions for multi-doc ops ✔ Virtuals for computed data

---

## 🟪 PART 5 — TYPESCRIPT ENGINEERING GUIDELINES (2026)

### 1. tsconfig.json (MANDATORY BASELINE)

If this is not strict → your whole type safety is fake.

✅ **Must Enable:**

```json
{
  "compilerOptions": {
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "strictFunctionTypes": true,
    "strictBindCallApply": true,
    "strictPropertyInitialization": true,
    "noImplicitThis": true,
    "alwaysStrict": true,
    "isolatedModules": true,
    "forceConsistentCasingInFileNames": true,
    "skipLibCheck": true
  }
}
```

---

### 2. Type Inference Philosophy

❌ **Avoid:** `const name: string = "Abhishek"` (Redundant).

✅ **Use:** `const name = "Abhishek"` (Let TS infer internally).

🚨 **Exception:** Always type function parameters, public API responses, DTOs, service returns.

---

### 3. Interfaces vs Types (Production Rule)

- **Use `interface` when:** defining object shapes, extendable domain models, class contracts.

```typescript
interface User { id: string; email: string }
```

- **Use `type` when:** unions, tuples, mapped types, utility composition.

```typescript
type Role = 'admin' | 'user'
```

---

### 4. Never Use `any`

❌ **Bad:** `function process(data: any)` — Removes autocomplete, compile checks, runtime safety.

✅ **Use:** `unknown` and narrow later.

```typescript
function parse(data: unknown) {
  if (typeof data === "string") {
    return data.toUpperCase()
  }
}
```

---

### 5. DTO Pattern (Full Stack Safety)

Backend should NEVER accept `req.body` directly. Use DTO validation via Zod.

```typescript
// createUser.dto.ts
export interface CreateUserDTO {
  email: string
  password: string
}
```

Now: controller safe, service safe, DB safe. Prevents overposting attack (User sending `admin: true` manually).

---

### 6. Async API Pattern

Always type API responses.

❌ **Bad:** `async function getUser()`

✅ **Good:** `async function getUser(): Promise<User>`

Frontend: `useQuery<User>()`. Full stack type sync achieved.

---

### 7. Generics for Reusability

Use in repositories, API clients, services.

```typescript
async function fetchData<T>(url: string): Promise<T>
```

Now reusable across users, orders, products.

---

### 8. Type-Only Imports (Performance)

- **Avoid:** `import { User } from './types'`
- **Use:** `import type { User } from './types'` — Prevents runtime bundle pollution.

---

### 9. const Assertions (Union Safety)

```typescript
const ROLES = ['admin', 'user'] as const
type Role = typeof ROLES[number]
```

Strongest possible union.

---

### 10. Type Guards

Needed for external APIs, unknown input, JSON parsing.

```typescript
function isUser(value: unknown): value is User {
  return typeof value === 'object'
}
```

---

### 11. Null Handling (Huge Runtime Crash Source)

- **Never:** `user.name.length`
- **Use:** `user?.name ?? "Anonymous"`

---

### 12. Avoid Deep Type Magic

❌ **Avoid:** Recursive mapped types everywhere. Slows IDE, build time, compile time.

✅ **Prefer:** `Partial<User>`, `Pick<User, 'id'>`, `Readonly<User>`.

---

### 13. Dependency Injection Friendly Types

```typescript
interface PaymentGateway {
  charge(amount: number): Promise<boolean>
}
```

Service depends on interface — not implementation. Testable architecture achieved.

---

### 14. Folder Structure for Types

Never use a generic `types.ts`. Use domain based typing:

```
user/user.types.ts
user/user.dto.ts
```

---

### 15. Testing Types

Use `@ts-expect-error` to test invalid cases.

---

### 🚨 FINAL TYPESCRIPT RULES CHECKLIST

✔ Strict mode enabled ✔ Inference > explicit types ✔ Interface for models ✔ Type for unions ✔ No any ✔ unknown for external data ✔ DTO validation ✔ Typed async returns ✔ Generic APIs ✔ type-only imports ✔ const assertions ✔ Type guards ✔ Safe null handling ✔ Domain based typing ✔ Avoid deep mapped types

---

## ⚙️ CROSS-STACK BEST PRACTICES

- **API:** Use REST naming consistency, Pagination, DTO Pattern.
- **Caching:** Use React Query cache, Redis for backend caching.
- **Auth Flow:** Use Access Token (short-lived), Refresh Token (long-lived), HTTPOnly cookies.
- **Testing:** Use Vitest (frontend), Jest (backend), Supertest (API).
- **Dev Tooling:** Use ESLint, Prettier, Husky, Lint-staged.
