---
name: typescript-safety
description: TypeScript type safety rules for Node.js projects. Forbids any type, requires unknown with type guards, proper interfaces/types, and generics. Covers type assertions, double assertions, and when to run type checks.
---

## TypeScript Type Safety

**FORBIDDEN**: The use of `any` type is strictly prohibited. All code must maintain full type safety.

### Type Safety Rules

1. **Never use `any`** — explicitly typing with `any` defeats TypeScript's purpose
2. **Use `unknown`** for truly unknown types, then narrow with type guards
3. **Define proper interfaces/types** for all data structures
4. **Use generic types** when the exact type varies but structure is known
5. **Leverage type inference** where TypeScript can infer correctly

### Alternatives to `any`

```typescript
// ❌ FORBIDDEN
function processData(data: any): any {
  return data.value;
}

// ✅ Use unknown with type guard
function processData(data: unknown): string {
  if (typeof data === 'object' && data !== null && 'value' in data) {
    return String((data as { value: unknown }).value);
  }
  throw new Error('Invalid data structure');
}

// ✅ Use proper interface
interface DataStructure {
  value: string;
  timestamp: number;
}
function processData(data: DataStructure): string {
  return data.value;
}

// ✅ Use generics
function processData<T extends { value: string }>(data: T): string {
  return data.value;
}

// ❌ FORBIDDEN — any in arrays
const items: any[] = [1, 'text', { id: 1 }];

// ✅ Union types or discriminated unions
type Item = number | string | { id: number };
const items: Item[] = [1, 'text', { id: 1 }];
```

### Handling Untyped External Libraries

```typescript
// ❌ FORBIDDEN
const result: any = externalLib.method();

// ✅ Define your own type declarations
interface ExternalLibResult {
  status: number;
  data: string;
}
declare module 'untyped-library' {
  export function method(): ExternalLibResult;
}
```

### Type Assertions (`as`)

Use `as` only where genuinely needed — narrowing inside a type guard, or asserting a known-safe external API response. Remove it if TypeScript can already infer or narrow the type without it.

```typescript
// ❌ Unnecessary — TypeScript already knows the type
const name = getUserName() as string;

// ✅ Let inference work
const name = getUserName();

// ✅ Acceptable — known-safe external response
const data = await response.json() as ApiResponse;

// ✅ Acceptable — inside a type guard body
function isUser(obj: unknown): obj is User {
  return typeof (obj as User).id === 'string';
}
```

**Rule:** if removing `as` still compiles without errors, the assertion is unnecessary — remove it.

### Double Assertion (`as unknown as X`) is Forbidden

```typescript
// ❌ FORBIDDEN — masking a real type mismatch
const result = someValue as unknown as ExpectedType;
```

If you need a double assertion, the types don't align — there is a real error elsewhere. Before forcing a cast:
1. Is the source type wrong? Fix it at the source.
2. Is the target type outdated? Update the interface.
3. Is there a missing conversion step? Add a proper mapper.
4. If none of the above resolves it, report to Tech Lead.

### Exception: Tests

The **only** acceptable use of `any` is in test files when stubbing complex third-party types where full type definitions would be impractical. Even then, prefer proper typing.

```typescript
// Acceptable in tests only
const mockResponse = { data: someData } as any;

// Preferred even in tests
const mockResponse: IExpectedResponse<IEntity> = { value: [] };
```

### Type Checking

**Always run TypeScript type checking before finishing implementation.**

Find the script in `package.json` — common names: `check-types`, `type-check`, `typecheck`. It runs `tsc --noEmit`.

Type checking must pass with zero errors. Fix all type errors immediately — do not suppress with `any` or `@ts-ignore`. Run after:
- Creating or modifying any TypeScript file
- Adding or changing interfaces, types, or classes
- Refactoring code that affects type signatures
- Writing or modifying test files
