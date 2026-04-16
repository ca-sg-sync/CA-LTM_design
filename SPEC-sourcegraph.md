# @ca-ltm/sourcegraph — Module Specification

> **Version:** 0.1.0-draft
> **Phase:** 8 (after manual orchestration and promotion)

---

## 1. Module Identity

| Field | Value |
|-------|-------|
| **Package** | `@ca-ltm/sourcegraph` |
| **Role** | Sourcegraph MCP adapter for live code truth |
| **Dependencies** | `@ca-ltm/domain` |
| **Dependents** | `@ca-ltm/orchestrator` |

---

## 2. Responsibilities

- Wrap existing Sourcegraph MCP API calls
- Build search queries from CA-LTM record hints
- Normalize Sourcegraph results into CA-LTM types
- Expand CA-LTM records into live code queries
- Execute `provenance.sourcegraphHints` and `provenance.sourcegraphQueries`
- Provide reusable search templates
- Support code truth verification for record revalidation

---

## 3. Design Principle

> **Sourcegraph = live code truth. CA-LTM = institutional memory.**

This module bridges the two. It never stores code — only queries it.

Memory records can include `provenance.sourcegraphHints` and `provenance.sourcegraphQueries`. This module executes those to connect institutional memory to live code state.

---

## 4. Public API

```typescript
interface SourcegraphService {
  search(query: string, options?: SearchOptions): Promise<CodeSearchResult[]>;

  searchFromRecord(record: BaseRecord): Promise<CodeSearchResult[]>;

  expandHints(hints: string[]): Promise<CodeSearchResult[]>;

  executeQueries(queries: string[]): Promise<CodeSearchResult[]>;

  verifyRecordAgainstCode(record: BaseRecord): Promise<VerificationResult>;

  buildQueryFromContext(text: string, systems?: string[]): string;
}

function createSourcegraphService(config: SourcegraphConfig): SourcegraphService;
```

### Types

```typescript
interface SourcegraphConfig {
  mcpEndpoint?: string;       // MCP server connection
  defaultRepos?: string[];    // default repo scope
  maxResults?: number;        // default: 10
  timeout?: number;           // ms, default: 10000
}

interface SearchOptions {
  repos?: string[];
  limit?: number;
  filePatterns?: string[];
}

interface CodeSearchResult {
  repo: string;
  file: string;
  lineRange?: { start: number; end: number };
  content: string;
  matchType: "symbol" | "text" | "file" | "commit";
  relevance?: number;
}

interface VerificationResult {
  recordId: string;
  recordType: RecordType;
  stillRelevant: boolean;
  evidence: string[];
  suggestions?: string[];
}
```

---

## 5. Query Building

### From record provenance

```typescript
function searchFromRecord(record: BaseRecord): Promise<CodeSearchResult[]> {
  const hints = record.provenance?.sourcegraphHints ?? [];
  const queries = record.provenance?.sourcegraphQueries ?? [];

  const results = await Promise.all([
    ...hints.map(h => search(h)),
    ...queries.map(q => search(q)),
  ]);

  return deduplicateAndRank(results.flat());
}
```

### From free text

```typescript
function buildQueryFromContext(text: string, systems?: string[]): string {
  // Extract likely search terms from natural language
  // Scope to relevant repos if systems are known
  // Return Sourcegraph-compatible search query string
}
```

### Reusable Templates

```typescript
function findSymbol(name: string, repos?: string[]): string;
function findReferences(symbol: string, repos?: string[]): string;
function findFilesByPattern(pattern: string, repos?: string[]): string;
function searchInRepo(query: string, repo: string): string;
```

---

## 6. Result Normalization

Sourcegraph MCP returns various shapes depending on the tool called (`search_code`, `get_file_contents`, `search_symbols`, etc.). This module normalizes all results to `CodeSearchResult`.

---

## 7. Record Verification

Used by the verification/staleness pipeline to check if a durable record's code references are still valid:

```typescript
async function verifyRecordAgainstCode(record: BaseRecord): Promise<VerificationResult> {
  // 1. Execute sourcegraphHints/Queries from provenance
  // 2. Check if expected symbols/patterns still exist
  // 3. Return { stillRelevant, evidence, suggestions }
}
```

---

## 8. MCP Integration

The team already has a Sourcegraph MCP server. This module wraps it by translating CA-LTM domain requests into MCP tool calls:

| CA-LTM Operation | MCP Tool |
|-------------------|----------|
| `search(query)` | `search_code` |
| `findSymbol(name)` | `search_symbols` |
| `getFileContents(path)` | `get_file_contents` |
| `searchInRepo(query, repo)` | `search_code` with repo filter |

---

## 9. Internal Structure

```
src/
  index.ts                    # barrel exports: createSourcegraphService
  service.ts                  # SourcegraphService implementation
  client/
    mcp-client.ts             # MCP protocol client wrapper
  queries/
    builder.ts                # Query construction from free text
    templates.ts              # Reusable query templates
    from-record.ts            # Build queries from CA-LTM record provenance
  normalization/
    results.ts                # Normalize MCP results → CodeSearchResult
  verification/
    checker.ts                # Record verification against live code
  config.ts                   # SourcegraphConfig defaults
```

---

## 10. What It Does NOT Do

- Does not store code or code summaries
- Does not make orchestration decisions
- Does not modify CA-LTM records
- Does not manage sessions
- Does not rank CA-LTM records (that's retrieval)

---

## 11. Test Criteria

- `search` returns normalized `CodeSearchResult[]`
- `searchFromRecord` expands hints and queries correctly
- `expandHints` handles empty hints gracefully
- `buildQueryFromContext` produces valid Sourcegraph queries
- Query templates produce correct formatted queries
- Result normalization handles all MCP response shapes
- `verifyRecordAgainstCode` correctly detects still-relevant vs stale
- Timeout is respected
- Error handling for MCP connection failures
- Deduplication works when multiple hints return same file
