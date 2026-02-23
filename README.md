# 🧠 MERN ENGINEERING PLAYBOOK (2026)

Modern Best Practices for Scalability, Maintainability & Performance.
Basically: "If someone joins my startup tomorrow — this is how we write React, Node, Express, Mongo in 2026 so that the codebase doesn't become a nightmare in 12 months." Not theory. Not tutorials. Just: what to do / what not to do in production-grade MERN apps.

---

## 🟦 PART 1 — REACT (CLIENT ARCHITECTURE)

This section answers: How should we write React apps in 2026 so they remain scalable after 50+ features?

### 1. Component Philosophy

✅ Use:

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

❌ Avoid:

- Class Components
- Large "God Components" (Massive 400-line components)
- Prop Drilling → use Context/Zustand
- Business logic inside UI components
- API calls inside JSX files
- Components managing unrelated state

> If your component fetches data, filters it, sorts it, validates it, and mutates it, that is NOT a component anymore. That's a mini-backend 😭. Move logic to: `hooks/`, `queries/`, `store/`, `utils/`.

---

### 2. TypeScript Rules

✅ Always write like this:

```typescript
type CategoryFormProps = {
  onSubmit: (data: CategoryInput) => void
}

const CategoryForm = ({ onSubmit }: CategoryFormProps) => {}
```

❌ Never use: `const CategoryForm: React.FC<Props>`

**Why?** React.FC adds implicit children, messes with generics (bad generic inference), breaks defaultProps typing, makes inference harder, and makes DevTools debugging messier.

---

### 3. State Management Strategy (2026 Standard)

Modern React has 4 types of state:

| State Type | Example | Tool |
|---|---|---|
| UI State | modal open | Zustand |
| Server State | users from DB | Tanstack Query |
| Form State | login form | React Hook Form |
| URL State | page=2 | React Router / nuqs |
| Global Auth | — | Zustand + Persist |

🔥 **GOLDEN RULE: Server State ≠ Client State**

❌ NEVER:
- Store API response in Zustand
- Store React Query data in Redux
- Duplicate backend data locally

Let React Query own the server data lifecycle: caching, retries, background refetch, deduping, and stale logic. Avoid Redux unless enterprise-level scale, and avoid `useState` for shared state.

---

### 4. Data Fetching Pattern

❌ Old Pattern: Fetch inside `useEffect`

```javascript
useEffect(() => {
  fetch('/api/users')
}, [])
```

This causes: race conditions, double fetching, stale UI, no caching, bad retries, and manual loading states.

✅ New Standard: Always use `query.ts`, `mutation.ts`, and `queryKeys.ts`. Use `useQuery()`, `useMutation()`, Query Key Factory pattern, Optimistic Updates, and Suspense Mode.

```typescript
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

✅ Use:

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

❌ Avoid: Root-level `components/`, `pages/`, `utils/`, `services/`. Type-based structure doesn't scale. Feature-based = modular.

---

### 6. Forms (Huge Source of Bugs)

✅ Use: React Hook Form and Zod Resolver.

❌ Never:
- Manage form state manually
- Use `useState` for inputs
- Validate in `onSubmit`

---

### 7. Performance Rules

✅ Use:
- Lazy loading for routes (with Suspense)
- `React.memo` only when needed
- `useCallback` only for child prop stability (handlers passed down)
- Virtualized Lists (`react-virtual`) for large tables

❌ Avoid:
- Premature memoization (memoizing everything)
- Inline anonymous functions/heavy computations in heavy lists/JSX
- Derived state stored in state

```javascript
// Bad:
const [total, setTotal] = useState(cart.reduce(...))

// Good:
const total = useMemo(() => cart.reduce(...), [cart])
```

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

- ✔ Hooks > Classes
- ✔ Tanstack Query for server state
- ✔ Zustand for UI state
- ✔ React Hook Form for forms
- ✔ Zod for validation
- ✔ Feature-based folder structure
- ✔ API layer abstraction
- ✔ No fetch in useEffect
- ✔ No React.FC
- ✔ No global server state

---

## 🟩 PART 2 — NODE.JS BACKEND ARCHITECTURE (2026 STANDARD)

### 1. First Principle: Your Backend is NOT MVC

Classic MVC (route → controller → model) looks nice in tutorials but becomes controllers with 300 lines, models called directly, business logic duplicated in 7 controllers, impossible testing, and tight coupling in real apps.

✅ Use This Instead: **Layered Architecture OR Domain Driven Modular Architecture**

```
route → controller → service → repository → database
```

Each layer has ONLY ONE responsibility:

| Layer | Responsibility |
|---|---|
| Route | Define endpoint |
| Controller | HTTP handling |
| Service | Business logic |
| Repository | DB interaction |
| Model | Schema only |

🚨 **Rule:** Controllers should NEVER talk to the database directly. If you see `User.find()` inside a controller → architecture violation.

---

### 2. Project Structure (Scalable)

❌ Avoid: `controllers/`, `models/`, `routes/`, `services/`. Type-based structure breaks at scale.

✅ Use Domain-Based Structure:

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

Everything related to "user" lives in one module. Features are isolated, testing becomes easy, onboarding is faster, and there are fewer merge conflicts.

---

### 3. Controller Rules

Controllers are the translation layer between the HTTP world and the business world.

✅ Controller SHOULD:
- Validate input
- Call service
- Return response

❌ Controller SHOULD NOT:
- Query DB
- Run business logic
- Transform domain logic
- Call external APIs
- Perform calculations

```javascript
// Bad:
const user = await User.findById(id)
user.balance += 200
await user.save()

// Good:
await userService.addBalance(id, 200)
```

---

### 4. Service Layer (Your Real Backend)

Service Layer contains: business rules, workflows, calculations, multi-db operations, external integrations.

```javascript
// Example:
await walletService.transferMoney(sender, receiver, amount)
// Inside: validate balance, debit, credit, transaction logging, rollback logic
// None of this belongs in controller.
```

---

### 5. Repository Layer (DB Abstraction)

Service should NOT care if you are using Mongo, Postgres, Redis, or Firestore. Repository handles DB.

```
userService → userRepository → mongoose
```

Now: You can replace Mongo later without rewriting business logic.

---

### 6. Async Handling Pattern

❌ Avoid: Nested awaits or blocking operations (sync fs, crypto).

✅ Use: `async/await` everywhere.

```javascript
await Promise.all([
  save(),
  sendMail(),
  log()
])
```

Parallel execution = faster APIs.

---

### 7. Error Handling System

- **Never:** throw raw strings, send stack traces to client, or write try/catch in every controller.

✅ Create Custom Error Class and Global Middleware:

```javascript
class AppError extends Error {
  constructor(message, statusCode) {
    super(message)
    this.statusCode = statusCode
  }
}
```

---

### 8. DTO Pattern (MANDATORY)

Never trust `req.body`. Use DTOs to ensure validation, shape consistency, type safety, and prevent overposting attacks.

```typescript
const dto = createUserSchema.parse(req.body)
await userService.create(dto)
```

---

### 9. Logging Strategy

Never use `console.log()` in production. Use **Pino** for structured JSON logs.

```json
{
  "level": "info",
  "service": "auth",
  "userId": "123"
}
```

---

### 10. Background Jobs (Don't Block APIs)

Never `await sendEmail()` inside a signup API. Use **BullMQ** or queues.

---

### 11. Dependency Injection (Important)

Avoid importing repositories inside service directly. Inject dependency instead.

---

### 12. Config Management

Never use `process.env` everywhere. Use `config/env.ts` and `config/db.ts`.

---

### 🚨 FINAL NODE RULES CHECKLIST

- ✔ Domain-based modules
- ✔ Controller thin
- ✔ Service = business logic
- ✔ Repository = DB access
- ✔ DTO validation
- ✔ Custom Error System
- ✔ Global Error Middleware
- ✔ Structured logging
- ✔ Background jobs
- ✔ Promise.all
- ✔ Dependency Injection
- ✔ Central config

---

## 🟨 PART 3 — EXPRESS API LAYER (2026 STANDARD)

Express is NOT your backend logic. Express is your **Transport Layer**.

### 1. Request Lifecycle (MANDATORY FLOW)

```
Request → Global Middleware → Auth Middleware → Validation Middleware
→ Controller → Service → Repository → DB → Response Formatter
```

---

### 2. Route Design Pattern

Routes should only map `endpoint → middleware → controller`.

```javascript
router.post(
  '/',
  validate(createUserSchema),
  authMiddleware,
  userController.createUser
)
```

---

### 3. Middleware Layering

- **Global** → CORS, Helmet
- **Route** → Auth
- **Feature** → Validation

---

### 4. Validation Flow

Use Zod/Joi.

```javascript
export const validate =
  (schema) =>
  (req, res, next) => {
    schema.parse(req.body)
    next()
  }
```

---

### 5. Auth Middleware Pattern

Create `auth.middleware.ts` and attach user to request.

---

### 6. Response Formatter

Create `response.util.ts`:

```json
{
  "success": true,
  "data": {}
}
```

---

### 7. API Versioning

Use `/api/v1/users`

---

### 8. Rate Limiting & Security

Use: Helmet, CORS, JWT Rotation, Bcrypt ≥ 10, express-rate-limit.

---

### 9. Centralized Error Handling

```javascript
// Controller:
next(new AppError('Not found', 404))
```

---

### 10. Async Wrapper

Use `asyncHandler(controller)`

---

### 11. Request Logging

Track `requestId`, `userId`, `route`, `latency`.

---

### 12. Route Protection Strategy

| Route Type | Protection |
|---|---|
| Public | None |
| Private | Auth |
| Admin | Auth + Role |

---

### 🚨 FINAL EXPRESS RULES CHECKLIST

- ✔ No logic in routes
- ✔ Validation
- ✔ Auth
- ✔ Versioned APIs
- ✔ Global error
- ✔ Async wrapper
- ✔ Response formatter
- ✔ Logging
- ✔ Rate limiting
- ✔ Role-based protection

---

## 🟥 PART 4 — MONGODB + MONGOOSE (2026 STANDARD)

### 1. Golden Rule of MongoDB

MongoDB is a **Query-first** database.

---

### 2. Embed vs Reference

- **Embed** for small one-to-few.
- **Reference** for growing one-to-many.
- Avoid unbounded arrays.

---

### 3. Hybrid Pattern

Embed summary, reference heavy dynamic data.

---

### 4. Must Use Schema Options

```javascript
{
  timestamps: true,
  versionKey: false
}
```

---

### 5. Index Strategy

Index: `email`, `status`, `createdAt`, foreign keys.

---

### 6. Query Optimization

Use `.lean()` for reads.

---

### 7. Pagination Pattern

```javascript
find({ createdAt: { $lt: lastSeen } })
  .limit(10)
  .sort({ createdAt: -1 })
```

---

### 8. Aggregation Pipeline Rules

Match early. Project early.

---

### 9. populate() Usage

Avoid for lists.

---

### 10. Soft Delete Pattern

Use `deletedAt`.

---

### 11. Virtual Fields

Use for computed fields.

---

### 12. Projection

Use `.select('name email')`

---

### 13. Pre/Post Middleware

For hashing, audit logs.

---

### 14. Transaction Usage

Use for multi-doc ops.

---

### 🚨 FINAL MONGOOSE RULES CHECKLIST

- ✔ Embed
- ✔ Reference
- ✔ Hybrid
- ✔ Index
- ✔ lean
- ✔ Cursor
- ✔ Match early
- ✔ Avoid populate
- ✔ Soft delete
- ✔ Projections
- ✔ Transactions
- ✔ Virtuals

---

## 🟪 PART 5 — TYPESCRIPT ENGINEERING GUIDELINES (2026)

### 1. tsconfig.json (MANDATORY BASELINE)

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

Avoid explicit types when redundant. Type params and API responses.

---

### 3. Interfaces vs Types

```typescript
interface User { id: string; email: string }
type Role = 'admin' | 'user'
```

---

### 4. Never Use `any`

Use `unknown` and narrow later.

```typescript
function parse(data: unknown) {
  if (typeof data === "string") {
    return data.toUpperCase()
  }
}
```

---

### 5. DTO Pattern

Use Zod.

---

### 6. Async API Pattern

```typescript
async function getUser(): Promise<User>
```

---

### 7. Generics

```typescript
fetchData<T>()
```

---

### 8. Type-Only Imports

```typescript
import type { User } from './user.types'
```

---

### 9. const Assertions

```typescript
as const
```

---

### 10. Type Guards

```typescript
isUser(value: unknown): value is User
```

---

### 11. Null Handling

```typescript
user?.name ?? "Anonymous"
```

---

### 12. Avoid Deep Type Magic

Prefer `Partial`, `Pick`, `Readonly`

---

### 13. Dependency Injection Friendly Types

```typescript
interface PaymentGateway {
  charge(amount: number): Promise<boolean>
}
```

---

### 14. Folder Structure for Types

```
user/user.types.ts
```

---

### 15. Testing Types

```typescript
// @ts-expect-error
```

---

### 🚨 FINAL TYPESCRIPT RULES CHECKLIST

- ✔ Strict
- ✔ Inference
- ✔ Interface
- ✔ Type
- ✔ No any
- ✔ unknown
- ✔ DTO
- ✔ Typed async
- ✔ Generics
- ✔ type-only imports
- ✔ const assertions
- ✔ Guards
- ✔ Null handling
- ✔ Domain types
- ✔ Avoid deep mapped types

---

## ⚙️ CROSS-STACK BEST PRACTICES

- **API:** REST naming consistency, Pagination, DTO Pattern.
- **Caching:** React Query cache, Redis for backend caching.
- **Auth Flow:** Access Token (short-lived), Refresh Token (long-lived), HTTPOnly cookies.
- **Testing:** Vitest (frontend), Jest (backend), Supertest (API).
- **Dev Tooling:** ESLint, Prettier, Husky, Lint-staged.
