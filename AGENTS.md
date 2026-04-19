# AGENTS.md — CA-LTM Cross-Module Standards

These rules apply to **all** `@ca-ltm/*` modules. Module-specific AGENTS.md files may add constraints but must not contradict these.

## Error Handling

**Never throw errors.** All async functions return `ResponseType<T>`.

```typescript
import { ResponseType, Status, SuccessResponse, ErrorResponse } from '@ca/response-type';
```

### Returning success

```typescript
return SuccessResponse(payload);
```

### Returning errors

```typescript
return ErrorResponse(err);
```

### Calling a function that returns ResponseType

```typescript
const resp = await someCall(creds, args);
if (resp.status === Status.ERROR) return resp;
// safe to use resp.payload
```

### Parallel calls

```typescript
const responses = await Promise.all(items.map(i => someCall(creds, i)));
for (const resp of responses) {
  if (resp.status === Status.ERROR) return resp;
}
```

### No silent errors

Never fall back to defaults on error. Never return an empty array or null when the underlying call failed. If a callee returns an error, the caller must return that error immediately.

Wrong:
```typescript
const resp = await storage.records.queryByType(creds, type);
return resp.status === Status.OK ? resp.payload : []; // swallows error
```

Right:
```typescript
const resp = await storage.records.queryByType(creds, type);
if (resp.status === Status.ERROR) return resp;
return SuccessResponse(resp.payload);
```

## Credentials

All `StorageService` methods require `CaCredentialsT` as the first argument.

```typescript
import { CaCredentialsT } from '@ca/aws-sdk/dist/ca-credentials';
```

In tests, use `{} as any`.

## Imports and Exports

### Barrel exports

All `index.ts` files use `export * from`:

```typescript
export * from './service';
export * from './search';
```

Never use named re-exports in barrel files.

### No cross-package re-exports

Each package exports only its own symbols. Never re-export types or functions from other packages. Consumers import directly from the source:

```typescript
// Consumer imports ResponseType from its source, not from @ca-ltm/retrieval
import { ResponseType, Status } from '@ca/response-type';
import { AnyRecord } from '@ca-ltm/domain';
import { createRetrievalService } from '@ca-ltm/retrieval';
```

## TypeScript

- **Target:** ES2020
- **Module:** CommonJS
- **Strict mode:** enabled (`strict: true`, `noImplicitAny: true`, `noImplicitReturns: true`)
- **Node:** >=20.0.0
- **Declarations:** enabled (`declaration: true`, `declarationMap: true`, `sourceMap: true`)
- **Output:** `dist/`
- **Source:** `src/`

## Testing

### Framework

**Vitest** with explicit imports — always import `describe`, `it`, `expect`, `vi` from `'vitest'`:

```typescript
import { describe, it, expect, vi } from 'vitest';
```

### Conventions

- Unit tests live in `test/unit/`
- Integration tests live in `test/integration/`
- Run unit tests: `npm run test:unit`
- Run all tests: `npm test` (typically runs unit tests only)
- Mock storage using a `createMockStorage()` helper returning a `StorageService` with `vi.fn()` stubs
- Build record fixtures with `createRecord` from `@ca-ltm/domain`
- All tests that call functions returning `ResponseType<T>` must assert `resp.status` before accessing `resp.payload`
- Include error-propagation tests for every method that calls storage or any other async dependency
- Include scenario tests for realistic multi-step flows described in the spec

### Test credential pattern

```typescript
const creds = {} as any;
```

## Module Boundaries

Each module has a single responsibility. These boundaries are enforced:

| Module | Owns | Does NOT do |
|--------|------|-------------|
| `@ca-ltm/domain` | Types, schemas, validation, pure logic | No I/O of any kind |
| `@ca-ltm/storage` | DynamoDB reads and writes | No business logic, no orchestration |
| `@ca-ltm/retrieval` | Search, filter, rank (read-only) | No writes, no Sourcegraph, no prompt assembly |
| `@ca-ltm/context-assembler` | Decide what enters prompt context | No data fetching |
| `@ca-ltm/orchestrator` | Turn-by-turn control plane | No direct DynamoDB access |
| `@ca-ltm/sourcegraph` | Sourcegraph MCP adapter | No CA-LTM business logic |
| `@ca-ltm/promotion` | Session → durable promotion | No orchestration |
| `@ca-ltm/openclaw-plugin` | OpenClaw wiring | No business logic |

## Code Style

- No comments unless the code is genuinely complex and requires context
- Follow existing patterns in neighboring files before introducing new ones
- Prefer small, focused functions
- Immutability — never mutate input arguments; return new objects
- ID format: lowercase alphanumeric + hyphens, 1–128 chars, starts with alphanumeric
- Timestamps: ISO 8601 strings

## Specs

- [CA-LTM Master Spec](CA-LTM-SPEC.md)
