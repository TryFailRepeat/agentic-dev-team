---
name: interface-naming
description: TypeScript interface naming conventions for Node.js projects. Use the I prefix only for polymorphic contracts implemented by classes (repositories, service adapters). No prefix for data structures, DTOs, domain models, and query/filter types.
---

## Interface Naming Conventions

### Core Rule

**Use the `I` prefix ONLY for interfaces that define polymorphic contracts implemented by classes.**

### Use `I` Prefix For

**Repository contracts** — multiple implementations exist or are expected:
```typescript
interface IContactRepository {
  get(id: string): Promise<Contact>;
  save(contact: Contact): Promise<void>;
}
class ContactRepositoryV1 implements IContactRepository { ... }
class ContactRepositoryV2 implements IContactRepository { ... }
```

**Service abstractions** (dependency inversion / ports):
```typescript
interface INotificationService {
  send(message: string): Promise<void>;
}
class EmailNotificationService implements INotificationService { ... }
class SmsNotificationService implements INotificationService { ... }
```

**Infrastructure adapters:**
```typescript
interface IHttpClient {
  get<T>(url: string): Promise<T>;
}
class AxiosHttpClient implements IHttpClient { ... }
```

### No `I` Prefix For

**Data structures and DTOs:**
```typescript
type ContactDTO = { id: string; name: string; };
type ContactFilter = { name?: string; email?: string; };
```

**Domain models:**
```typescript
interface Contact { id: string; name: string; }
class Contact { constructor(private readonly id: string) {} }
```

**Query/Filter/Include/Request/Response types:**
```typescript
type ContactQuery = { filter?: ContactFilter; include?: ContactInclude; };
type CreateContactRequest = { name: string; email: string; };
type CreateContactResponse = { id: string; };
```

**Configuration objects:**
```typescript
type DatabaseConfig = { host: string; port: number; };
```

### Decision Tree

1. Will classes implement this interface with multiple implementations? → `I` prefix
2. Is it a data structure, DTO, or domain model? → No prefix
3. Does it define behavior/methods with polymorphism for dependency inversion? → `I` prefix
4. Otherwise → No prefix

### File Naming

- Interface with `I`: `IContactRepository.ts`
- Type without `I`: `ContactFilter.ts`, `Contact.ts`, `ContactQuery.ts`

### Rationale

TypeScript is structurally typed — consumers shouldn't need to care whether something is an interface or type. The `I` prefix adds noise for data structures. For ports/adapters (Clean Architecture dependency inversion), it explicitly marks the contract and makes boundaries discoverable.

### Common Mistakes

- Adding `I` to all interfaces by habit
- Using `I` for DTOs, query objects, or domain models
- Inconsistent naming within the same module (some with `I`, some without for similar types)
