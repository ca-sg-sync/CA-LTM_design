# @ca-ltm/retrieval — Module Specification

> **Version:** 0.1.0-draft
> **Phase:** 9 (after manual orchestration is working)

---

## 1. Module Identity

| Field | Value |
|-------|-------|
| **Package** | `@ca-ltm/retrieval` |
| **Role** | Search, filtering, and ranking over CA-LTM records |
| **Dependencies** | `@ca-ltm/domain`, `@ca-ltm/storage` |
| **Dependents** | `@ca-ltm/orchestrator`, `@ca-ltm/api` |

---

## 2. Responsibilities

- Exact lookup by type + id
- Metadata filtering (status, owner, system, confidence, tags)
- Keyword search across title, summary, and type-specific text fields
- Ranking by confidence, freshness, usage
- Related-record expansion (follow links)
- Retrieval planning (which types to query based on task classification)
- Bootstrap retrieval (small relevant startup set)
- Active migrations shortcut
- Hook point for future semantic/embedding retrieval

---

## 3. Retrieval Strategy (ordered)

Retrieval happens in this priority order:

1. **Exact lookup** — by id, system name, migration name, glossary term, tags
2. **Metadata filtering** — type, status=active, related system, owner, confidence ≥ threshold
3. **Keyword search** — match query terms against title, summary, symptoms, fix, term, definition
4. **Ranking** — score candidates by confidence × freshness × usage
5. **Expansion** — follow links: related_to, depends_on, supersedes, caused_by

---

## 4. Public API

```typescript
interface RetrievalService {
  search(request: MemorySearchRequest): Promise<MemorySearchResult[]>;

  get(type: RecordType, id: string): Promise<AnyRecord | null>;

  related(
    type: RecordType,
    id: string,
    relationTypes?: LinkType[]
  ): Promise<AnyRecord[]>;

  bootstrap(request: BootstrapRequest): Promise<MemorySearchResult[]>;

  activeMigrations(system?: string): Promise<MigrationRecord[]>;

  recordsBySystem(
    system: string,
    types?: RecordType[]
  ): Promise<MemorySearchResult[]>;
}

function createRetrievalService(storage: StorageService): RetrievalService;
```

### Types (from `@ca-ltm/domain`)

```typescript
interface MemorySearchRequest {
  query?: string;
  types?: RecordType[];
  status?: RecordStatus[];
  relatedSystems?: string[];
  owner?: string[];
  minConfidence?: Confidence;
  semantic?: boolean;       // reserved for future embeddings
  limit?: number;           // default: 10
}

interface MemorySearchResult {
  id: string;
  type: RecordType;
  title?: string;
  summary?: string;
  score: number;
  whyMatched: string[];
}

interface BootstrapRequest {
  query?: string;
  systems?: string[];
  limit?: number;           // default: 5
}
```

---

## 5. Ranking Algorithm

```typescript
function computeSearchScore(record: BaseRecord, query?: string): number {
  const confidenceScore = { high: 1.0, medium: 0.6, low: 0.3 }[record.confidence ?? "medium"];

  const ageMs = Date.now() - new Date(record.updatedAt).getTime();
  const ageDays = ageMs / 86_400_000;
  const freshnessScore = Math.max(0, 1.0 - ageDays / 365); // linear decay over 1 year

  const usageScore = Math.min(1.0, (record.usage?.readCount ?? 0) / 50); // cap at 50 reads

  const statusBoost = record.status === "active" ? 1.0 : record.status === "resolved" ? 0.7 : 0.3;

  const keywordScore = query ? computeKeywordMatch(record, query) : 0.5;

  return (
    confidenceScore * 0.25 +
    freshnessScore * 0.20 +
    usageScore * 0.10 +
    statusBoost * 0.10 +
    keywordScore * 0.35
  );
}
```

### Keyword Matching

- Match query terms against: `title`, `summary`
- Type-specific fields: `symptoms` (incident), `term`/`definition` (glossary), `triggerConditions` (playbook), `conditions` (alert_pattern)
- Exact match scores higher than partial
- `whyMatched` explains which fields matched

---

## 6. Related-Record Expansion

```typescript
async function expandRelated(
  storage: StorageService,
  record: AnyRecord,
  relationTypes?: LinkType[]
): Promise<AnyRecord[]> {
  const links = record.links ?? [];
  const filtered = relationTypes
    ? links.filter(l => relationTypes.includes(l.type))
    : links;

  const keys = filtered.map(l => ({
    type: l.targetType ?? guessType(l.targetId),
    id: l.targetId,
  }));

  return storage.records.batchGet(keys);
}
```

---

## 7. Bootstrap Retrieval

On session start, fetch a small, high-value set:

1. All active migrations (optionally filtered by system)
2. Top patterns by confidence for relevant systems
3. Any active detection caveats
4. Active playbooks for relevant systems

Capped at `limit` (default 5).

---

## 8. Future: Semantic / Hybrid Retrieval

- `semantic: true` flag on `MemorySearchRequest` is reserved
- When `@ca-ltm/embeddings` is added, retrieval will:
  1. Apply metadata filters first (narrow the candidate set)
  2. Run semantic similarity on candidates
  3. Combine keyword + semantic scores
- Retrieval service exposes a hook for embedding adapter injection

---

## 9. Internal Structure

```
src/
  index.ts                    # barrel exports: createRetrievalService
  service.ts                  # RetrievalService implementation
  search/
    keyword.ts                # Keyword matching across record fields
    metadata.ts               # Filter application helpers
    expansion.ts              # Related-record traversal
  ranking/
    scorer.ts                 # computeSearchScore
    signals.ts                # Individual signal extractors
  planning/
    retrieval-plan.ts         # Decide which types to query based on task
    bootstrap.ts              # Bootstrap retrieval logic
```

---

## 10. What It Does NOT Do

- Does not assemble prompt context (that's `@ca-ltm/context-assembler`)
- Does not persist anything (read-only over storage)
- Does not call Sourcegraph
- Does not decide orchestration flow
- Does not own ranking weights long-term (configurable)

---

## 11. Test Criteria

- Exact lookup by type + id returns correct record
- `get` returns `null` for missing records
- Metadata filters narrow results correctly (status, owner, system, confidence)
- Keyword search finds records matching title and summary substrings
- Keyword search finds records by type-specific fields (symptoms, term, etc.)
- Ranking respects: keyword match > confidence > freshness > usage > status
- Related expansion follows links and returns correct records
- Related expansion respects `relationTypes` filter
- Bootstrap returns small, relevant set (respects limit)
- `activeMigrations` returns only `status=active` migration records
- `recordsBySystem` returns records across specified types
- Empty query returns reasonable defaults (highest-ranked active records)
- Limit is respected on all search methods
- `whyMatched` explains the match reason
- `semantic: true` is accepted but currently no-ops (future hook)
