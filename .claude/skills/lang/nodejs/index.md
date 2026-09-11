---
name: nodejs
description: Node.js and TypeScript language skill. Load this when the project uses JavaScript or TypeScript with Node.js. Instructs which sub-skills to load and covers runtime conventions not handled by the sub-skills (async patterns, error handling, module system).
---

# Node.js / TypeScript Language Skill

Load this skill when the project uses **Node.js** (with or without TypeScript). It tells you which additional skills to load and covers Node.js-specific conventions not addressed by the mandatory engineering skills.

---

## Skills to Load

After loading this skill, load the following immediately:

### Always load (TypeScript projects)
1. **`typescript-safety`** — type system rules, forbidden patterns (`any`, double assertions), type checking workflow
2. **`interface-naming`** — `I` prefix rules for TypeScript interfaces

### Load for testing (QA Engineer)
3. **`testing-guidelines`** — Node.js test conventions (node:assert, Sinon, builders, file structure)

### Load conditionally
4. **`import-path-conventions`** — path alias conventions using `package.json` `imports` field

**Load `import-path-conventions` only when:**
- This is a **new project** (you are scaffolding the project structure), OR
- The existing project's `package.json` already has an **`imports` field**

To check: `grep -A 5 '"imports"' package.json`

If the project uses relative imports throughout and has no `imports` field, do not load this skill — do not introduce path aliases mid-project.

---

## Node.js Runtime Conventions

### Module System

Use **ESM** (`import`/`export`) — not CommonJS (`require`/`module.exports`). The `package.json` should have `"type": "module"`.

```typescript
// ✅ ESM
import { something } from '#domain/Something.ts';
export class MyService { ... }

// ❌ CommonJS
const something = require('./something');
module.exports = { MyService };
```

### Async / Await

Always use `async`/`await` — not raw Promise chains. Handle errors at the boundary, not deep in the call stack.

```typescript
// ✅ Await and handle at the right level
const result = await repository.findById(id);
if (!result) throw new NotFoundError(id);

// ❌ Unhandled floating promises
repository.save(entity);  // no await
```

**Never** let a rejected Promise go unhandled. Every `async` function that is called but not awaited must attach `.catch()` explicitly, or be awaited.

### Error Handling

Throw typed errors — not raw `Error` objects or strings. Define domain error classes that extend `Error`:

```typescript
class OrderNotFoundError extends Error {
  constructor(orderId: string) {
    super(`Order not found: ${orderId}`);
    this.name = 'OrderNotFoundError';
  }
}
```

Catch at boundaries (application layer, controllers). Do not catch-and-rethrow inside domain logic — let errors propagate.

### Environment Variables

- Read all config from environment variables at startup — never hardcode in source files
- Validate required env vars at startup and fail fast with a clear message if any are missing
- Group config into a typed config object rather than reading `process.env` scattered throughout code

```typescript
// ✅ Centralized, typed, validated at startup
const config = {
  dbUrl: requireEnv('DATABASE_URL'),
  port: parseInt(requireEnv('PORT'), 10),
};

function requireEnv(key: string): string {
  const value = process.env[key];
  if (!value) throw new Error(`Missing required environment variable: ${key}`);
  return value;
}
```

### Container Readiness

Node.js services must be container-ready. The Backend Engineer must ensure:
- Configurable port via `PORT` env var
- `GET /health` endpoint that returns `200 OK`
- Logging to stdout (not files)
- Clean SIGTERM handling: close connections and exit gracefully
- Stateless — no local file system state between requests

These are requirements for the DevOps Engineer to containerize correctly. If the service doesn't meet them, the Backend Engineer must address them before the DevOps Engineer starts.

### Dependency Injection

Use a DI container (e.g., `inversify`, `tsyringe`) to wire dependencies at the Composition Root. Never instantiate infrastructure classes (repositories, external clients) directly in domain or application code — use interfaces injected via the container.

### Logging

Log to stdout in structured format (JSON preferred for production). Use a logger interface injected as a dependency — not `console.log` scattered in business logic. The logger is an infrastructure concern.

---

## Type Checking Before Finishing

After all implementation is done, run the project's type check script before reporting as complete:

```bash
# Find the script
grep '"check-types\|type-check\|typecheck"' package.json

# Run it
npm run check-types
```

Zero type errors required. Do not suppress errors with `any` or `@ts-ignore`.
