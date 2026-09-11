---
name: import-path-conventions
description: TypeScript import path conventions using package.json path aliases instead of relative paths. Only applies to new projects or existing projects that already have an imports section in package.json. Covers alias setup, .ts extension requirement, and type imports.
---

## Import Path Conventions

**This skill applies only when:**
- Working on a **new project** (set up the `imports` field as part of project scaffolding), OR
- The existing project's `package.json` already has an **`imports` field** defined

For existing projects without an `imports` field, do not introduce path aliases — it would require touching every file in the project. Use relative paths consistently instead, and do not mix conventions.

---

**REQUIRED when applicable**: All imports MUST use TypeScript path aliases defined in the `package.json` `imports` field. Never use relative paths for cross-layer or cross-directory imports.

### Runtime Model

Projects use Node.js `--experimental-transform-types` to run TypeScript natively without a build step:
- Applications run directly from `src/` — no compile step in development
- Import extension must be **`.ts`** (set `allowImportingTsExtensions: true` in tsconfig)

### package.json `imports` Configuration

```json
{
  "imports": {
    "#tests/*": ["./src/__tests__/*"],
    "#config/*": ["./src/config/*"],
    "#application/*": ["./src/application/*"],
    "#domain/*": ["./src/domain/*"],
    "#infrastructure/*": ["./src/infrastructure/*"],
    "#interface/*": ["./src/interface/*"]
  }
}
```

**tsconfig.json:**
```json
{
  "compilerOptions": {
    "noEmit": true,
    "allowImportingTsExtensions": true
  }
}
```

### How to Check Available Aliases

```bash
grep -A 15 '"imports"' package.json
```

Always check before writing imports — do not assume aliases match the template above.

### Import Rules

1. **Use path aliases** (`#domain/*`, `#application/*`, etc.) for all cross-directory imports
2. **File extension**: `.ts` (required by `allowImportingTsExtensions`)
3. **Use `type` imports** for type-only imports:
   ```typescript
   import type { IContactRepository } from '#domain/interfaces/IContactRepository.ts';
   ```
4. **npm package imports** never get file extensions — only local/alias imports do
5. Relative imports (`./`, `../`) are acceptable only within the same immediate subdirectory

### Examples

```typescript
// ✅ CORRECT
import { ContactRepository } from '#infrastructure/repositories/Contact/v2/ContactRepository.ts';
import type { IContactRepository } from '#domain/interfaces/IContactRepository.ts';

// ❌ INCORRECT — .js extension
import { ContactRepository } from '#infrastructure/repositories/Contact/v2/ContactRepository.js';

// ❌ INCORRECT — relative path across layers
import { ContactRepository } from '../../../infrastructure/repositories/Contact/v2/ContactRepository.ts';

// ✅ npm packages — no extension
import { injectable } from 'inversify';
```

### New Project Setup

When creating a new project, add the `imports` field to `package.json` before writing any source files. Define aliases for all layers defined in the architecture spec. Update tsconfig to enable `allowImportingTsExtensions`.
