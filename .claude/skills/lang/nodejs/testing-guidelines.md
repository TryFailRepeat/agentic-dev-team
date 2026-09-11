---
name: testing-guidelines
description: Node.js testing conventions. Use node:assert for assertions, randomUUID for IDs, builders for test data (not for the class under test), Sinon for stubs with proper types, no DI container in unit tests. Covers test file structure, naming, and the unit/integration/acceptance distinction.
---

## Testing Guidelines (Node.js)

### Assertions

- Use `assert` from `node:assert` — do NOT use `expect()` or other assertion libraries
- Import: `import { assert } from 'node:assert'`
- Use the **non-null assertion operator (`!`)** in test assertions when a value is possibly undefined but expected to be present:
  ```typescript
  assert.strictEqual(result[0]!.id, expectedId)  // ✅ test only
  ```
  Never use `!` in production code — only in test assertions.

### Test IDs

- Use `randomUUID()` from `node:crypto` — never hardcode UUIDs
- Import: `import { randomUUID } from 'node:crypto'`

### Test Data — Builders

Use builders for all test data setup (except the class under test):

```typescript
const lineItem = new OrderLineBuilder().withQuantity(2).build();
```

**Exception 1 — Do not use a builder for the subject under test.** Construct it directly so you test its own constructor/initialization logic:
```typescript
// Testing Order — construct it directly
const order = new Order({ id: randomUUID(), lineItems: [lineItem] });
```

**Exception 2 — Do not use builders in assertion comparisons.** Construct expected values directly:
```typescript
// ✅ Direct construction in assertion
assert.deepStrictEqual(result, new ExpectedClass({ field: value }));

// ❌ Builder in assertion — hides what is expected
assert.deepStrictEqual(result, new ExpectedClassBuilder().build());
```

**Create test data in `beforeEach`** with sensible defaults. Individual tests only override what makes them different:
```typescript
describe('OrderService', () => {
  let orderData: OrderData;

  beforeEach(() => {
    orderData = {
      id: randomUUID(),
      customerId: randomUUID(),
      status: 'pending',
      lineItems: [new OrderLineBuilder().build()],
    };
  });

  it('processes a valid order', async () => {
    const result = await orderService.process(orderData);
    assert.strictEqual(result.status, 'processed');
  });

  it('throws when order has no line items', async () => {
    orderData.lineItems = [];
    await assert.rejects(() => orderService.process(orderData), { name: 'ValidationError' });
  });
});
```

### Mocking and Stubbing (Sinon)

**For concrete classes** — `createStubInstance` with `SinonStubbedInstance<T>`:
```typescript
let service: SinonStubbedInstance<MyService>;
service = createStubInstance(MyService);
```

**For interfaces** — create a fake object manually:
```typescript
type StubbedInterface<T> = { [K in keyof T]: SinonStub };
let repository: StubbedInterface<IUserRepository>;
repository = { save: stub(), findById: stub() };
```

**Unit tests MUST NOT use `container.resolve()`** — all dependencies must be explicitly stubbed. This ensures true isolation.

Exceptions:
- Builder files in `__tests__/helpers/builders/` may use `container.resolve()` to obtain factories/validators
- Integration tests may use the container to test actual wiring

### What to Test — and What Not To

**Test:**
- Domain methods (business logic and invariant enforcement)
- Error handling and validation
- Edge cases and boundary conditions

**Do not test:**
- Simple getters that only return a private property: `get id() { return this._id; }`
- Constructors that only assign parameters to properties with no logic

Focus on behavior, not trivial assignment.

### Test Location

| Directory | Content |
|---|---|
| `__tests__/unit/` | Unit tests only — isolated, no I/O, all deps stubbed |
| `__tests__/integration/` | Integration tests — real database/API calls, multiple components |
| `__tests__/acceptance/` | End-to-end acceptance tests — full system, API through infrastructure |

Mirror the source structure in test directories:
`src/domain/User.ts` → `__tests__/unit/domain/User.spec.ts`

### Test Case Naming

- Present tense — state what the system does:
  - ✅ `'creates a user successfully'`
  - ✅ `'throws when input is invalid'`
  - ✅ `'should create a user'` (`should` is acceptable)
  - ❌ `'would create a user'` (`would` implies uncertainty — forbidden)

### File Naming

- Unit and integration tests: `*.spec.ts` or `*.test.ts`

### Structure (Arrange-Act-Assert)

```typescript
import { describe, it, beforeEach } from 'node:test';
import { assert } from 'node:assert';
import { randomUUID } from 'node:crypto';
import { createStubInstance, type SinonStubbedInstance } from 'sinon';

describe('UserService', () => {
  let userService: UserService;
  let mockRepository: SinonStubbedInstance<UserRepository>;

  beforeEach(() => {
    mockRepository = createStubInstance(UserRepository);
    userService = new UserService(mockRepository);
  });

  it('creates a user successfully', async () => {
    const userId = randomUUID();
    mockRepository.save.returns(Promise.resolve({ id: userId }));

    const result = await userService.createUser('John Doe');

    assert.strictEqual(result.id, userId);
    assert.ok(mockRepository.save.calledOnce);
  });
});
```

### Running Tests

Check `package.json` for the test script. For monorepo projects using npm workspaces:
```bash
docker compose run --rm <service> npm run test -w @scope/package-name
docker compose run --rm <service> npm run test:unit -w @scope/package-name
```

Adapt the `docker compose` command to the project's actual service name and workspace scope.
