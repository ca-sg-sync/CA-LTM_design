# @ca-ltm/context-assembler — Module Specification

> **Version:** 0.1.0-draft
> **Phase:** 5 (the heart of the harness)

---

## 1. Module Identity

| Field | Value |
|-------|-------|
| **Package** | `@ca-ltm/context-assembler` |
| **Role** | Builds the prompt/context bundle for the model each turn |
| **Dependencies** | `@ca-ltm/domain` (types only) |
| **Dependents** | `@ca-ltm/orchestrator` |

---

## 2. Responsibilities

- Define context bucket categories and priorities
- Apply inclusion rules per bucket
- Apply pruning and eviction rules
- Enforce token/size caps per bucket and total
- Merge inputs from session state, durable memory, live retrieval, and recent turns
- Order sections for optimal model performance
- Explain why each item was included or excluded (debug/logging)

---

## 3. Design Principle

> The context assembler does **NOT** fetch data. It receives pre-fetched inputs and decides what the model sees.

The key pruning question:

> **"Does this help the model decide the next correct action?"**

If not: archive it, demote it, keep it out.

---

## 4. Context Buckets (priority order)

| Priority | Bucket | Source | Purpose |
|----------|--------|--------|---------|
| 1 | `policy` | System config | Agent role, constraints, rules |
| 2 | `objective` | SessionState | Current task objective |
| 3 | `plan` | SessionState | Current plan of attack |
| 4 | `constraints` | SessionState + durable memory | Active migration rules, compliance, conventions |
| 5 | `findings` | SessionState | Key session findings so far |
| 6 | `durable_memory` | CA-LTM retrieval | Relevant migrations, patterns, playbooks, etc. |
| 7 | `live_retrieval` | Sourcegraph / tools | Code truth, tool outputs |
| 8 | `recent_turns` | Conversation | Very short recent interaction window |

---

## 5. Default Bucket Caps (configurable)

| Bucket | Default Max Tokens |
|--------|-------------------|
| `policy` | 500 |
| `objective` | 200 |
| `plan` | 300 |
| `constraints` | 400 |
| `findings` | 600 |
| `durable_memory` | 2000 |
| `live_retrieval` | 2000 |
| `recent_turns` | 1500 |
| **Total default** | **~7500** |

When total exceeds `maxTotalTokens`, lower-priority buckets are trimmed first.

---

## 6. Public API

```typescript
interface ContextAssembler {
  assemble(inputs: ContextInputs): ContextBundle;
}

function createContextAssembler(config?: AssemblerConfig): ContextAssembler;

interface AssemblerConfig {
  bucketCaps?: Partial<Record<ContextBucket, number>>;
  maxTotalTokens?: number;       // default: 7500
  tokenEstimator?: (text: string) => number; // default: chars/4
}
```

### Input Types (from `@ca-ltm/domain`)

```typescript
interface ContextInputs {
  policy?: string;
  sessionState?: SessionState;
  durableMemory?: MemorySearchResult[];
  durableRecords?: BaseRecord[];
  liveRetrieval?: LiveRetrievalItem[];
  recentTurns?: TurnSummary[];
  maxTotalTokens?: number;       // override per-call
}
```

### Output Types (from `@ca-ltm/domain`)

```typescript
interface ContextBundle {
  sections: ContextSection[];
  totalEstimatedTokens: number;
  includedItems: IncludedItem[];
  excludedItems: ExcludedItem[];
}

interface ContextSection {
  bucket: ContextBucket;
  content: string;
  estimatedTokens: number;
  priority: number;
}

interface IncludedItem {
  source: string;
  bucket: ContextBucket;
  reason: string;
}

interface ExcludedItem {
  source: string;
  reason: string;           // e.g. "bucket cap exceeded", "below relevance threshold"
}
```

---

## 7. Assembly Algorithm

```
1. Build each bucket's content from inputs:
   - policy      ← inputs.policy (passthrough)
   - objective   ← sessionState.currentObjective
   - plan        ← synthesize from sessionState (objective + active systems + migrations)
   - constraints ← active migration rules + constraints from durable memory
   - findings    ← sessionState.findings (most recent first)
   - durable_memory ← format top-scored MemorySearchResults / BaseRecords
   - live_retrieval ← format LiveRetrievalItems by relevance
   - recent_turns   ← format TurnSummaries (most recent first)

2. Estimate tokens per bucket

3. Trim each bucket to its cap:
   - Within a bucket, keep highest-priority / most-recent items
   - Drop lowest-value items first
   - Record dropped items in excludedItems

4. If total exceeds maxTotalTokens:
   - Trim from lowest-priority bucket (recent_turns) upward
   - Never trim policy or objective below minimum

5. Assemble sections in priority order

6. Return ContextBundle with debug info
```

---

## 8. Pruning Rules

### What to keep
- Current objective
- Current plan
- Unresolved questions
- Key findings (most recent / highest impact)
- Relevant active durable records
- Relevant live evidence
- Very recent conversation (last 2-3 turns)

### What to demote
- Completed substeps
- Stale hypotheses
- Repetitive tool logs
- Prior drafts
- Verbose intermediate analysis

### What to drop
- Raw tool output beyond summary
- Exploratory dead ends
- Redundant information already covered by a durable record

### What to promote (flag for orchestrator)
- Stable reusable knowledge → flag as promotion candidate

---

## 9. Formatting Functions

Each bucket needs a formatter that converts domain objects to prompt-ready text:

```typescript
function formatDurableRecord(record: BaseRecord): string;
// → "## [MIGRATION] Authz Policy Engine\nStatus: active\nSummary: ...\nRules: do X, avoid Y\n"

function formatLiveRetrieval(item: LiveRetrievalItem): string;
// → "[sourcegraph] file.ts:42 — function doAuth() { ... }"

function formatTurnSummary(turn: TurnSummary): string;
// → "[user] How should I update the auth middleware?"

function formatSessionFindings(findings: string[]): string;
// → "- Legacy middleware still coexists with policy engine\n- New work should prefer..."

function formatConstraints(records: BaseRecord[]): string;
// → "Active migration: prefer policy engine for new routes. Avoid: new route-local branches."
```

---

## 10. Internal Structure

```
src/
  index.ts                    # barrel exports: createContextAssembler
  assembler.ts                # Main ContextAssembler implementation
  buckets/
    definitions.ts            # Bucket configs, caps, priorities
    policy.ts                 # Policy section builder
    session.ts                # Objective/plan/findings from SessionState
    memory.ts                 # Durable memory section builder
    retrieval.ts              # Live retrieval section builder
    turns.ts                  # Recent turns section builder
    constraints.ts            # Constraints from memory + session
  pruning/
    rules.ts                  # Pruning rule engine
    trimmer.ts                # Bucket and total trimming
  formatting/
    records.ts                # Record → prompt text formatters
    retrieval.ts              # LiveRetrievalItem formatter
    turns.ts                  # TurnSummary formatter
  estimation/
    tokens.ts                 # Token estimation (default: chars/4)
  debug/
    explainer.ts              # IncludedItem / ExcludedItem generation
```

---

## 11. What It Does NOT Do

- Does **not** fetch data (no storage calls, no Sourcegraph calls)
- Does **not** decide when to run (that's orchestrator)
- Does **not** persist anything
- Does **not** classify tasks
- Receives pre-fetched inputs and assembles them

---

## 12. Test Criteria

- Assembly produces valid `ContextBundle` with all sections
- Empty inputs produce minimal valid output (just policy if provided)
- Bucket caps are respected — content is trimmed when over
- Higher-priority buckets are preserved when total exceeds limit
- Lower-priority buckets trimmed first (recent_turns before findings)
- Policy and objective are never fully removed
- Pruning drops lowest-value items first within each bucket
- `includedItems` explains every inclusion with source and reason
- `excludedItems` explains every exclusion with reason
- Token estimation is reasonably accurate (within 20% of actual)
- Section order matches priority (policy first, recent_turns last)
- Formatting produces readable prompt text for all record types
- Config overrides (custom caps, custom estimator) work correctly
- `maxTotalTokens` override per-call works
