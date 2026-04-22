# CA-LTM: Long-Term Memory System — Master Specification

### Lucio Flores — [CleverAlpha Technologies](https://www.cleveralphatechnologies.com)

> **Version:** 0.1.0-draft
> **Last Updated:** 2026-04-21
> **Status:** Design — Pre-Implementation

---

## 1. What CA-LTM Is

CA-LTM is a **typed long-term memory system** for OpenClaw-based AI agents. It stores durable organizational knowledge — migrations, patterns, decisions, incidents, playbooks, glossary terms, and surveillance-adjacent records — so agents accumulate institutional expertise over time.

CA-LTM does **not** store raw code or replace live code intelligence. The codebase remains queryable through Sourcegraph MCP. CA-LTM stores the *interpretations, decisions, patterns, and context* that Sourcegraph cannot provide.

### Core Principle

> **Sourcegraph MCP = "what the code says now"**
> **CA-LTM = "what the organization has learned over time"**

---

## 2. Motivation

### 2.1 The Problem with Existing OpenClaw Memory

OpenClaw's memory ecosystem has evolved rapidly, but every existing solution solves only a slice of the real problem. None addresses the full context lifecycle that a serious engineering agent requires.

| System | What It Does | What It Cannot Do |
|--------|-------------|-------------------|
| **memory-core** (built-in) | File-based persistence (`MEMORY.md` + dated notes). Transparent, human-editable. | No structure, no retrieval logic, no lifecycle management. Becomes a landfill at scale. The agent decides when to read/write — and frequently forgets. |
| **memory-lancedb** (official) | Vector database with auto-capture and auto-recall. Injects relevant memory every turn. | Still just a retrieval layer. No working state, no task/objective awareness, no separation of memory types. Similar paragraphs surface; precise operational facts often do not. |
| **memory-lancedb-pro** (community) | Hybrid retrieval (vector + BM25), reranking, recency weighting, memory decay, multi-scope isolation. | The most advanced "drop-in brain" available — but still fundamentally "better search over stored text." No session state layer, no explicit objective/plan tracking, no typed records. |
| **Mem0 / @mem0/openclaw-mem0** | Enforced memory at the system layer. Automatically captured and injected every turn. Removes "agent forgot to store memory." | Still mostly "facts about user/system." No working-state layer, no structured context hierarchy, no operational context awareness. |
| **Triple-memory / ShadowDB hybrids** | Combine vector memory + structured notes + file-based workspace. Reduce token duplication. | Closer to the goal, but still no full context model. No promotion pipeline, no verification, no typed institutional records. |
| **Obsidian-based memory** | Local Markdown files with `[[wiki-links]]`, graph visualization of note relationships, rich plugin ecosystem. Human-readable, no lock-in. | No typed schemas, no structured retrieval, no session state, no promotion pipeline, no verification or staleness management. The graph is powerful for visualizing connections but edges are untyped and undirected. Becomes a landfill at scale — the agent reads/writes files but has no lifecycle governance over what accumulates. |

**The common gap across all of these:**

Every existing system treats memory as **a smarter database the model can search**. None of them manage **context as a state problem** — deciding what the model should think about *right now*, tracking working objectives, separating provisional findings from durable knowledge, or governing how session-level discoveries become long-term institutional memory.

The ecosystem today solves:

- ✅ Storage
- ✅ Retrieval
- ✅ Persistence
- ✅ Sometimes lifecycle (decay, auto-capture)

It does **not** solve:

- ❌ Working context / session state
- ❌ Task and objective tracking
- ❌ Plan management
- ❌ Context layering (transcript vs working state vs durable memory vs live truth)
- ❌ Typed institutional knowledge
- ❌ Promotion pipeline (session → durable, with review)
- ❌ Verification and staleness management
- ❌ "What should the model see right now?" as a first-class concern

### 2.2 What CA-LTM Solves

CA-LTM fills the layer **above** memory. It is not a better database — it is a **context management system** with a durable knowledge store underneath.

| Problem | CA-LTM Solution |
|---------|----------------|
| **Session amnesia** — agents lose context across compaction, restarts, and long threads | Structured `SessionState` with checkpoint-based persistence. Resumable without relying on session-end events. |
| **Memory as landfill** — append-only stores accumulate noise until retrieval degrades | Typed records with promotion rules. Most session content stays ephemeral. Only high-signal findings become durable memory. |
| **No separation of knowledge types** — migrations, patterns, incidents, and chat notes all mixed into one blob | 10 distinct record types, each with purpose-built schemas and retrieval characteristics. |
| **Loss of institutional knowledge** — the "why" behind decisions, the traps to avoid, the migration context that matters | First-class record types for decisions, patterns, migrations, playbooks, and caveats — the knowledge that makes an agent more expert over time. |
| **Memory rot** — durable records go stale without anyone noticing | Verification status, `lastVerifiedAt`, staleness detection, and Sourcegraph-assisted revalidation for code-adjacent records. |
| **Context window mismanagement** — either too much is stuffed in (noise) or too little (amnesia) | Context assembler with priority-ordered buckets, per-bucket caps, pruning rules, and the explicit question: "Does this help the model decide the next correct action?" |
| **No provenance or trust** — the agent "knows" something but cannot say where it learned it | Every durable record carries provenance (source sessions, source docs, evidence refs, Sourcegraph query hints). Records are auditable. |
| **Code truth mixed with interpretation** — stale code summaries drift from reality | Hard separation: Sourcegraph MCP owns live code truth. CA-LTM stores only interpretations, decisions, and organizational meaning. Records carry Sourcegraph query hints to bridge back to live code. |

### 2.3 The Execution Environment

CA-LTM is not a generic memory plugin. It is designed for a specific and demanding operational context.

#### Engineering at Scale

The team maintains a large, multi-repo codebase with:

- Long-running migrations that span months (legacy auth → policy engine, infrastructure moves, runtime upgrades)
- Proprietary internal patterns that are not documented anywhere except in the heads of senior engineers
- Complex cross-service dependencies where subsystem boundaries are not obvious from code alone
- Accumulated tech debt and transitional states where "the code" and "the intent" diverge

A generic "remember things" system fails here because the *type* of knowledge matters. A migration blocker is not the same as a coding pattern is not the same as a glossary term. They have different schemas, different lifetimes, different retrieval needs, and different verification requirements.

#### Financial Services Domain

The environment operates under constraints that most memory systems ignore:

- **Regulatory and compliance rules** that must be durably understood and never contradicted
- **Domain-specific vocabulary** (account types, portfolio states, regulatory classifications) where a wrong definition can propagate into wrong code
- **Sensitive data boundaries** that the memory system must respect — CA-LTM stores institutional knowledge, never customer data or credentials
- **Audit expectations** — the ability to explain *why* the agent recommended a particular approach, traced back to a decision record with provenance

#### Real-Time Stack Surveillance

Beyond code assistance, the team manages real-time event streams and operational monitoring:

- Recurring incident patterns that compound in value when captured as durable knowledge
- Alert interpretation that changes during migrations ("this alert is noisy right now because of migration X")
- Playbooks and investigation sequences that should improve over time instead of being rediscovered each incident
- Cross-system correlation knowledge ("when A spikes and B lags, it usually means C is degraded")

This is why CA-LTM includes surveillance-adjacent record types — `playbook`, `alert_pattern`, `incident_cluster`, `detection_caveat` — that most memory systems would never consider. These are not for real-time event processing (that belongs in a separate surveillance pipeline). They are for the **durable investigative knowledge** that makes future triage faster and more accurate.

#### Sourcegraph as Live Code Truth

The team already operates a Sourcegraph MCP that exposes the full Sourcegraph API to agents. This fundamentally changes the memory architecture:

- The agent does **not** need to "learn the codebase" by memorizing code
- Code truth is always available live — symbols, references, files, history, search
- Memory should store what Sourcegraph **cannot** know: the *why*, the *intent*, the *context*, the *traps*
- Memory records can carry Sourcegraph query hints, making memory *operationally useful* — not just descriptive, but a launch point into live investigation

This separation is the architectural foundation of CA-LTM: **mechanical understanding from Sourcegraph, institutional understanding from memory.**

### 2.4 Why This Solution Is Necessary

The honest framing is:

> Existing memory systems answer: **"What should I remember?"**
>
> CA-LTM answers: **"What should I think about right now?"**

That is a fundamentally harder problem, and it is the one that actually determines whether an agent becomes more useful over months or just accumulates stale text.

For a team operating across a complex codebase, financial domain constraints, and real-time operational concerns, the gap between "memory plugin" and "context management system" is the difference between an agent that helps and one that hallucinates from its own outdated notes.

CA-LTM closes that gap.

---

## 3. High-Level Architecture

```
                    ┌──────────────────────┐
                    │      OpenClaw        │
                    │   agent/orchestrator │
                    └──────────┬───────────┘
                               │
                ┌──────────────┼──────────────┐
                │                             │
                ▼                             ▼
      ┌──────────────────┐         ┌──────────────────┐
      │  Sourcegraph MCP │         │      CA-LTM      │
      │  live code truth │         │  long-term memory │
      └──────────────────┘         └──────────────────┘
                                              │
                         ┌────────────────────┼────────────────────┐
                         │                    │                    │
                         ▼                    ▼                    ▼
                ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
                │  DynamoDB    │     │  retrieval   │     │  admin UI    │
                │  typed recs  │     │  hybrid RAG  │     │  human-read  │
                └──────────────┘     └──────────────┘     └──────────────┘
```

---

## 4. Context Hierarchy

CA-LTM operates within a four-layer context model. These are **not** the same thing:

| Layer | What It Holds | Lifetime | Owner |
|-------|--------------|----------|-------|
| **Transcript** | Raw conversation + tool history | Archive | OpenClaw |
| **SessionState** | Structured working ledger (objective, findings, plan, open questions) | Per-session, checkpoint-resumed | `@ca-ltm/storage` |
| **Durable Memory** | Typed institutional knowledge (migrations, patterns, etc.) | Months/years | `@ca-ltm/storage` |
| **Live Truth** | Code, symbols, references, repo state | Real-time | Sourcegraph MCP |

### Context Flow

```
Transcript → SessionState → CA-LTM durable records
                                    ↕
                            Sourcegraph (live truth)
```

**Key rule:** Memory stores interpretations and decisions. Live truth is queried, not memorized.

---

## 5. Record Types

CA-LTM uses **10 typed record categories**. All share a common `BaseRecord` shape with type-specific payload fields.

### 4.1 RecordType Enum

```typescript
type RecordType =
  | "migration"
  | "pattern"
  | "decision"
  | "incident"
  | "glossary"
  | "session_note"
  | "playbook"
  | "alert_pattern"
  | "incident_cluster"
  | "detection_caveat";
```

### 4.2 BaseRecord

Every record shares this shape. Note: `pk`/`sk` are **not** part of the domain model — they are derived by the storage layer.

```typescript
interface BaseRecord {
  id: string;
  type: RecordType;
  title: string;
  summary?: string;

  status?: "draft" | "active" | "resolved" | "deprecated" | "archived";
  owner?: string;
  confidence?: "low" | "medium" | "high";

  tags?: string[];
  relatedSystems?: string[];
  relatedRepos?: string[];

  createdAt: string;         // ISO timestamp
  updatedAt: string;         // ISO timestamp
  createdBy?: string;        // "human" | "agent" | "importer"
  updatedBy?: string;

  lastVerifiedAt?: string;
  verificationStatus?: "unverified" | "verified" | "stale" | "conflicted";

  provenance?: {
    sourceSessions?: string[];
    sourceDocs?: string[];
    sourcegraphHints?: string[];
    sourcegraphQueries?: string[];
    evidenceRefs?: string[];
  };

  links?: Array<{
    type:
      | "related_to"
      | "depends_on"
      | "supersedes"
      | "superseded_by"
      | "caused_by"
      | "uses"
      | "touches"
      | "documents"
      | "explains";
    targetId: string;
    targetType?: RecordType;
  }>;

  usage?: {
    readCount?: number;
    lastReadAt?: string;
    lastUsedByAgentAt?: string;
  };

  review?: {
    promotionSource?: "manual" | "auto" | "imported";
    reviewStatus?: "pending" | "approved" | "rejected" | "not_required";
    reviewedBy?: string;
    reviewedAt?: string;
  };

  ttl?: number;              // epoch seconds, mainly for session_note

  embeddingModel?: string;   // model that generated the embedding (e.g. "amazon.titan-embed-text-v2:0")
}
```

### 4.3 Type-Specific Payloads

#### migration
```typescript
interface MigrationRecord extends BaseRecord {
  type: "migration";
  goal?: string[];
  currentState?: string;
  targetState?: string;
  rules?: { do?: string[]; avoid?: string[] };
  blockers?: string[];
  milestones?: Array<{
    label: string;
    status: "pending" | "in_progress" | "done" | "blocked";
    dueAt?: string;
    notes?: string;
  }>;
  affectedAreas?: string[];
  rollbackNotes?: string[];
}
```

#### pattern
```typescript
interface PatternRecord extends BaseRecord {
  type: "pattern";
  whenToUse?: string[];
  whenNotToUse?: string[];
  canonicalShape?: string[];
  antiPatterns?: string[];
  examples?: Array<{
    label: string;
    repo?: string;
    path?: string;
    symbol?: string;
    note?: string;
  }>;
}
```

#### decision
```typescript
interface DecisionRecord extends BaseRecord {
  type: "decision";
  context?: string;
  decision?: string;
  rationale?: string[];
  consequences?: string[];
  alternativesConsidered?: string[];
  supersedesIds?: string[];
}
```

#### incident
```typescript
interface IncidentRecord extends BaseRecord {
  type: "incident";
  symptoms?: string[];
  rootCause?: string;
  impact?: string[];
  fix?: string[];
  prevention?: string[];
  severity?: "sev0" | "sev1" | "sev2" | "sev3" | "sev4";
  firstSeenAt?: string;
  lastSeenAt?: string;
  affectedTenants?: string[];
}
```

#### glossary
```typescript
interface GlossaryRecord extends BaseRecord {
  type: "glossary";
  term: string;
  definition: string;
  aliases?: string[];
  relatedTerms?: string[];
  domain?: string;
}
```

#### session_note
```typescript
interface SessionNoteRecord extends BaseRecord {
  type: "session_note";
  sessionId: string;
  topic?: string;
  findings?: string[];
  openQuestions?: string[];
  nextActions?: string[];
  candidatePromotionIds?: string[];
  expiresAfterDays?: number;
}
```

#### playbook
```typescript
interface PlaybookRecord extends BaseRecord {
  type: "playbook";
  triggerConditions?: string[];
  prerequisites?: string[];
  firstChecks?: string[];
  investigationSteps?: Array<{
    step: number;
    instruction: string;
    expectedOutcome?: string;
    ifNotTrueThen?: string;
  }>;
  dashboards?: Array<{ name: string; url?: string; note?: string }>;
  queries?: Array<{
    system: "sourcegraph" | "logs" | "metrics" | "sql" | "other";
    label: string;
    query: string;
    note?: string;
  }>;
  escalation?: { owners?: string[]; conditions?: string[] };
  mitigations?: string[];
  rollbackGuidance?: string[];
}
```

#### alert_pattern
```typescript
interface AlertPatternRecord extends BaseRecord {
  type: "alert_pattern";
  signalSummary?: string;
  conditions?: string[];
  likelyInterpretations?: Array<{
    interpretation: string;
    confidence: "low" | "medium" | "high";
  }>;
  knownFalsePositives?: string[];
  relatedPlaybookIds?: string[];
  relatedIncidentIds?: string[];
}
```

#### incident_cluster
```typescript
interface IncidentClusterRecord extends BaseRecord {
  type: "incident_cluster";
  signature?: string;
  commonSymptoms?: string[];
  likelyRootCauses?: string[];
  commonMitigations?: string[];
  memberIncidentIds?: string[];
  historicalSeverity?: {
    highest?: "sev0" | "sev1" | "sev2" | "sev3" | "sev4";
    typical?: "sev0" | "sev1" | "sev2" | "sev3" | "sev4";
  };
}
```

#### detection_caveat
```typescript
interface DetectionCaveatRecord extends BaseRecord {
  type: "detection_caveat";
  appliesWhen?: string[];
  caveat?: string;
  doNotAssume?: string[];
  requireAlsoChecking?: string[];
  expiresAt?: string;
  relatedPatternIds?: string[];
  relatedPlaybookIds?: string[];
}
```

---

## 6. SessionState

The structured working ledger between transcript and durable memory.

```typescript
interface SessionState {
  sessionId: string;
  topic?: string;
  currentObjective?: string;
  activeSystems?: string[];
  activeMigrations?: string[];
  openQuestions?: string[];
  findings?: string[];
  hypotheses?: string[];
  decisionsThisSession?: string[];
  candidateRecordIds?: string[];
  lastCheckpointAt?: string;
  lastCompactedAt?: string;
}
```

---

## 7. DynamoDB Design

### 6.1 Table Structure

**Single-table design.** All record types coexist in one table.

| Attribute | Value | Purpose |
|-----------|-------|---------|
| **PK** (Partition Key) | `RecordType` (e.g. `"migration"`, `"playbook"`) | Partitions by record type |
| **SK** (Sort Key) | Record `id` (e.g. `"authz-policy-engine"`) | Unique within type |

### 6.2 Key Derivation

The storage layer derives keys from domain fields:

```typescript
// Storage-layer concern — not in domain model
function toDynamoKeys(record: BaseRecord) {
  return {
    PK: record.type,
    SK: record.id,
  };
}
```

### 6.3 Access Patterns (No GSIs in v1)

| Pattern | How |
|---------|-----|
| Get one record | `GetItem(PK=type, SK=id)` |
| All records of a type | `Query(PK=type)` |
| Active migrations | `Query(PK="migration")` + `FilterExpression: status = "active"` |
| Records by owner | `Query(PK=type)` + `FilterExpression: owner = X` |
| Records by system | `Query(PK=type)` + `FilterExpression: contains(relatedSystems, X)` |
| Stale records | `Query(PK=type)` + `FilterExpression: verificationStatus = "stale"` |
| Cross-type by system | Parallel queries across relevant type partitions |

At the expected scale (hundreds to low-thousands per type), `FilterExpression` is efficient. GSIs can be added later if access patterns demand it.

### 6.4 Embedding Storage

Embeddings are stored as **separate items** in the same table, not inline on record items. This avoids inflating record reads with ~4 KB of vector data per record (~20x RCU increase if inline).

| Attribute | Value | Type |
|-----------|-------|------|
| **PK** | `"embedding"` | String |
| **SK** | `{recordType}#{recordId}` | String |
| **vector** | Float32 array (1,024 dimensions, AWS Titan v2) | Binary |
| **model** | Embedding model identifier | String |
| **sourceText** | Concatenated text used to generate the embedding | String |
| **computedAt** | ISO timestamp of last computation | String |

**Access patterns:**

| Pattern | How |
|---------|-----|
| Get one embedding | `GetItem(PK="embedding", SK="{type}#{id}")` |
| All embeddings (for similarity search) | `Query(PK="embedding")` |
| Embeddings by record type | `Query(PK="embedding", SK begins_with "{type}#")` |

Embeddings are computed on record create and update. A `reindex` operation recomputes embeddings for all records (used when switching embedding models).

**Model:** AWS Titan Embeddings v2 (`amazon.titan-embed-text-v2:0`), 1,024 dimensions. Stored as Binary (float32 array) — 4 KB per vector. At 1,000 records, the full embedding set is ~4 MB.

### 6.5 SessionState Storage

SessionState uses the same table:

| Attribute | Value |
|-----------|-------|
| **PK** | `"session"` |
| **SK** | `sessionId` |

Checkpoints:

| Attribute | Value |
|-----------|-------|
| **PK** | `"checkpoint"` |
| **SK** | `{sessionId}#{timestamp}` |

### 6.6 Relationship Storage

**V1:** Inline in `links` array on each record.

**Later (if needed):** Separate edge items for fast neighbor expansion:
```
PK = "edge"
SK = "{sourceType}#{sourceId}#{relationType}#{targetType}#{targetId}"
```

### 6.7 TTL

- Applied to `session_note` records via DynamoDB TTL attribute.
- Durable records do not expire.

### 6.8 Event Log — Event-Sourced History

CA-LTM maintains an **append-only event log** that captures every mutation to the system. The event log is the source of truth — the records table (`ca-ltm-records`) is a materialized projection that can be rebuilt from scratch by replaying events, similar to git.

#### Why Event Sourcing

1. **Auditability** — every change is traceable: who changed what, when, why, from which session
2. **Replay & reconstruction** — rebuild the entire records table from the event log at any point in time
3. **Debugging & tuning** — replay session state evolution to analyze context assembly effectiveness
4. **Visualization** — animate knowledge graph growth, show record lifecycles, track session behavior over time
5. **Compliance** — financial services audit requirements demand explainable, traceable knowledge evolution

#### Infrastructure

```
ca-ltm-records      ← current state (records, sessions, checkpoints)
ca-ltm-events       ← append-only event log (separate table)
ca-ltm-snapshots    ← S3 bucket for monthly full-state snapshots
```

The event log lives in a **separate DynamoDB table** (`ca-ltm-events`). This separation is required because:
- Events are append-only and never updated; records are mutable
- The rebuild-from-scratch requirement means events must be independent of the table they reconstruct
- Different throughput, retention, and backup characteristics

#### MemoryEvent

Every mutation produces a `MemoryEvent` containing a **full snapshot** of the record state after the mutation. Snapshots are chosen over deltas because they are tolerant of gaps, require no sequential replay ordering for individual record reconstruction, and DynamoDB storage is cheap.

```typescript
interface MemoryEvent {
  eventId: string;                // ULID or UUID
  timestamp: string;              // ISO timestamp

  action:
    | "created"
    | "patched"
    | "promoted"
    | "deprecated"
    | "archived"
    | "verified"
    | "marked_stale"
    | "linked"
    | "unlinked"
    | "ttl_expired"
    | "restored"
    | "superseded"
    | "review_approved"
    | "review_rejected"
    | "session_checkpoint"
    | "session_resumed"
    | "session_closed";

  actor: "agent" | "human" | "system";
  actorId?: string;               // user ID or agent ID
  sessionId?: string;             // originating session

  recordId: string;
  recordType: RecordType | "session";

  snapshot: Record<string, unknown>;  // full record state after mutation

  reason?: string;                // human/agent-provided context for the change
  triggerSource?: string;         // what caused this event (e.g., "checkpoint_trigger", "admin_ui", "promotion_pipeline")
}
```

#### Event Table Schema — Dual-Write

Every event is written as **two items** in a single `TransactWriteItems` call:

| Write | PK | SK | Purpose |
|---|---|---|---|
| Global timeline | `"event"` | `{timestamp}#{eventId}` | Chronological replay, global queries |
| Per-record history | `"event#{recordType}#{recordId}"` | `{timestamp}#{eventId}` | Full lifecycle of a single record |

Both items contain the same `MemoryEvent` payload. No GSI required — dual-write gives both access patterns natively.

**Access patterns:**

| Query | How |
|---|---|
| Global timeline (all events) | `Query(PK="event", SK between start and end)` |
| Events for a specific record | `Query(PK="event#migration#authz-policy-engine")` |
| Events since last snapshot | `Query(PK="event", SK > lastSnapshotTimestamp)` |
| Recent events by type | `Query(PK="event")` + `FilterExpression: action = "promoted"` |

#### Event Sources

All mutations generate events, regardless of origin:

| Source | Actor | Examples |
|---|---|---|
| **Agent** | `"agent"` | Record created during session, auto-promotion, link inferred |
| **Human** | `"human"` | Admin UI edits, review approval/rejection, manual verification |
| **System** | `"system"` | TTL expiry, staleness detection, auto-archive, monthly snapshot |
| **Session** | `"agent"` or `"human"` | Checkpoint saved, session resumed, session closed |

#### SessionState Events

SessionState mutations are **in-scope** for the event log. This is critical for:

- **Debugging context assembly** — replay how context was built turn-by-turn to identify what was surfaced vs. what was missed
- **Tuning retrieval** — analyze which records were pulled into session context and whether they were useful
- **Measuring efficacy** — track session state size, token utilization, and bucket behavior over time

SessionState events use `recordType: "session"` and `recordId: sessionId`. The snapshot contains the full `SessionState` after the mutation.

At the expected user scale (small team), session event volume is manageable. If volume grows, session events can be moved to a separate partition prefix or given a shorter retention window without affecting durable record events.

#### S3 Snapshots

Monthly consolidated snapshots are stored in S3 as compressed JSON. A pointer record in `ca-ltm-events` references each snapshot.

```
ca-ltm-events table (pointer):
  PK: "snapshot"
  SK: "2026-04-01T00:00:00Z"
  s3Key: "snapshots/2026-04-01.json.gz"
  recordCount: 847
  sizeBytes: 2_340_000
  createdAt: "2026-04-01T00:05:00Z"

S3:
  s3://ca-ltm-snapshots/2026-04-01.json.gz
```

Full-state snapshots are necessary because a single DynamoDB item is limited to 400KB — far too small for a full memory dump at scale.

**Snapshot contents:** Every record in `ca-ltm-records` at snapshot time, serialized as a JSON array, gzipped.

#### Replay

Rebuilding the records table from the event log:

1. **Find nearest snapshot** — `Query(PK="snapshot", ScanIndexForward=false, Limit=1)`
2. **Load snapshot from S3** — decompress, deserialize to record array
3. **Apply subsequent events** — `Query(PK="event", SK > snapshotTimestamp)`, apply each event's snapshot as the current state for its `recordType#recordId`
4. **Write to records table** — batch-write the reconstructed state

**Point-in-time reconstruction:**
- Same process, but stop applying events at the target timestamp
- Answers: "What did CA-LTM know on March 15th?"

**Per-record reconstruction:**
- `Query(PK="event#migration#authz-policy-engine", SK <= targetTimestamp, ScanIndexForward=false, Limit=1)`
- The most recent event snapshot before the target time *is* the record state — no replay needed

#### Visualization Use Cases

The event log enables the following visualization capabilities in the admin UI:

| Visualization | Data Source | Value |
|---|---|---|
| **Global timeline** | `PK="event"` | "What happened to memory this week?" |
| **Record life history** | `PK="event#{type}#{id}"` | "How did this migration record evolve from draft to resolved?" |
| **Knowledge graph animation** | Global timeline filtered to `created`, `linked`, `deprecated` | "Watch the knowledge graph grow over Q1" |
| **Point-in-time diff** | Two PITR reconstructions | "What changed between these two dates?" |
| **Session state evolution** | Session events for a conversation | "Replay how context was built turn-by-turn" |
| **Session efficiency metrics** | Aggregated session events | "Context size, token usage, bucket utilization over time" |
| **Session misses & errors** | Session events + agent feedback signals | "What was needed but not surfaced? What was surfaced but irrelevant?" |

---

## 7. CA-LTM API Surface

### Read APIs
- `ltm.search(query, filters)` — hybrid search across records
- `ltm.get(type, id)` — direct lookup
- `ltm.related(type, id, relationTypes?)` — expand linked records
- `ltm.bootstrap(taskContext)` — load small relevant startup set
- `ltm.activeMigrations(system?)` — shortcut for active migration records

### Write APIs
- `ltm.writeSessionNote(note)` — store ephemeral session finding
- `ltm.proposeRecord(record)` — create candidate durable record
- `ltm.promote(candidateId, targetType)` — promote session finding to durable
- `ltm.updateRecord(type, id, patch)` — patch existing record
- `ltm.markStale(type, id)` — flag record for re-verification
- `ltm.linkRecords(a, relation, b)` — create relationship

### Embedding APIs
- `ltm.semanticSearch(query, filters?)` — embed query, cosine similarity against record embeddings, return ranked matches
- `ltm.reindex(type?, id?)` — recompute embeddings for all records or a specific record/type

### Verification APIs
- `ltm.verify(type, id, evidence)` — mark verified with evidence
- `ltm.revalidate(filters)` — batch re-check records
- `ltm.logUsage(type, id, context)` — track agent usage

---

## 8. Orchestrator Flow

### 8.1 Design Principle

The orchestrator is **checkpoint-based**, not end-of-session based. OpenClaw sessions are long-lived and may never formally end.

### 8.2 Four Flows

#### A. Recall Flow (before/near turn start)
1. Identify session → load `SessionState`
2. Classify task type (see §8.5)
3. Query CA-LTM for relevant records (scoped by task type)
4. Query Sourcegraph when code truth is needed
5. Assemble context bundle

#### B. Working-State Flow (during session)
1. Update `SessionState` incrementally: findings, hypotheses, open questions
2. Track candidate records (not yet promoted)
3. Avoid writing durable memory on every turn

#### C. Promotion Flow (at checkpoints)
1. Extract durable candidates from session state + recent response
2. Decide: no-op / session note / create durable / patch existing
3. Queue for review or auto-promote based on confidence

#### D. Compaction/Resume Flow
1. Snapshot `SessionState` on compaction
2. Write session summary
3. Persist queued candidates
4. On resume: rehydrate from checkpoint + CA-LTM context

### 8.3 Checkpoint Triggers

**Content triggers:**
- Root cause identified
- Migration rule stated
- User says "this is the standard way"
- Repeatable investigation sequence discovered
- Important correction made
- Glossary-worthy term defined
- Incident pattern recognized

**Behavioral triggers:**
- Every N assistant turns
- Every N tool calls
- Every N minutes of activity
- Before compaction
- After major answer synthesis
- On `/new`, `/reset`, `/stop`

**Confidence triggers:**
- Only promote when confidence passes threshold

### 8.4 Context Assembly Order

For each meaningful turn:

1. Policy / role instructions
2. Current objective
3. Current plan
4. Critical constraints
5. Top session findings
6. Relevant durable memory
7. Relevant live retrieval (Sourcegraph)
8. Very recent conversational window

**Pruning rule:** Include only what helps the model decide the *next correct action*.

### 8.5 Task Classification

The orchestrator classifies each turn into a **task type** that determines which data sources to query and which record types to prioritize during retrieval.

```typescript
type TaskType =
  | "code_truth"
  | "institutional_memory"
  | "mixed"
  | "operational"
  | "compliance"
  | "investigation"
  | "domain_lookup";
```

#### Classification Mechanism

Task classification uses **embedding-based semantic similarity** via the `@ca-ltm/classifier` module. Each task type has pre-computed anchor embeddings derived from its description and example phrases. The user's input is embedded and compared against all anchors via cosine similarity — the highest-scoring type wins.

**Fallback chain:** If the embedding provider is unavailable, classification falls back to keyword/regex pattern matching (the original v1 mechanism), then to `"mixed"` as the safe default.

This replaces the original keyword-only classification, which missed semantic matches — e.g., "help me understand why we chose DynamoDB" is clearly `institutional_memory` but contains no keyword signals for that type.

#### Classification Profiles

| TaskType | Purpose | Queries CA-LTM? | Queries Sourcegraph? | Priority Record Types |
|---|---|---|---|---|
| `code_truth` | Live code navigation: "where is", "what calls", "find references" | No | Yes | — |
| `institutional_memory` | Organizational knowledge: "why do we", "what pattern", "what was decided" | Yes | No | migration, pattern, decision, incident, session_note |
| `mixed` | Cross-cutting or ambiguous queries (default) | Yes | Yes | All |
| `operational` | Real-time ops: alerts, degradation, service health | Yes | No | playbook, alert_pattern, incident_cluster, detection_caveat |
| `compliance` | Regulatory and policy constraints: PCI, GDPR, SOX, data classification | Yes | No | decision, pattern, glossary |
| `investigation` | Active triage and diagnosis: "diagnose", "troubleshoot", "root cause" | Yes | Yes | playbook, incident, alert_pattern, detection_caveat |
| `domain_lookup` | Business terminology: "define", "what does X mean", glossary lookups | Yes | No | glossary, decision |

#### Why Seven Types, Not Three

The original three types (`code_truth`, `institutional_memory`, `mixed`) only controlled **which system** to query. The expanded set also controls **which record types** to prioritize during retrieval. This matters because:

- **Operational** questions should surface playbooks and alert patterns, not migration records or glossary terms.
- **Compliance** questions should surface decisions and patterns tagged with regulatory context, not incident history.
- **Investigation** needs both CA-LTM (historical incidents, playbooks) and Sourcegraph (code paths), but scoped to operational record types.
- **Domain lookups** should hit glossary records directly, not return noise from every record type.

Without scoped retrieval, the system returns the same ranked list regardless of intent — acceptable at small scale, but increasingly noisy as the memory store grows.

**Default:** `mixed` — the safest fallback, queries everything.

### 8.6 Hybrid Retrieval Flow

When the orchestrator's recall flow needs to find relevant records, it delegates to the retrieval module, which executes a **four-stage hybrid retrieval pipeline**. This pipeline combines exact metadata filtering, keyword matching, and embedding-based vector search into a single ranked result set.

#### Why Hybrid, Not Just Vector Search

Pure vector search (embed query → cosine similarity → top-K) works well for open-ended similarity but has weaknesses that matter for institutional memory:

- **Metadata must be exact.** When the task type is `operational`, the retrieval plan says "only surface playbooks, alert patterns, incident clusters, and detection caveats." Vector search cannot enforce this — a semantically similar migration record would score high even though the task type excludes it. Metadata filtering must come first.
- **Keywords catch what embeddings miss.** A search for "SSM" should find every record mentioning SSM, regardless of semantic similarity. Keyword matching is precise and fast.
- **Embeddings catch what keywords miss.** A search for "cross-account access" should find a record titled "IAM AssumeRole for multi-account operations" even though the words don't overlap. Semantic similarity bridges vocabulary gaps.
- **Freshness and trust matter.** A verified record from last week is more valuable than an unverified record from last year, even if the older record scores higher on similarity.

Hybrid retrieval combines all four signals. No single signal is sufficient alone.

#### Stage 1 — Scope (exact, fast)

The task classification result (§8.5) determines which record types to query and narrows the candidate set before any scoring:

```
TaskType → Retrieval Plan → Priority Record Types
  "operational" → [playbook, alert_pattern, incident_cluster, detection_caveat]
  "mixed"       → [all 10 types]
```

The retrieval module queries storage for records matching the priority types, optionally filtered by metadata:

- `status` — typically `"active"` or `"draft"` (exclude archived/deprecated)
- `relatedSystems` — if the session has `activeSystems`, scope to those
- `relatedRepos` — if the query references specific repos

This stage reduces the candidate set from potentially thousands of records to tens or low hundreds — the set that will be scored.

**Cost:** DynamoDB queries with FilterExpression. Fast and cheap. No embeddings involved.

#### Stage 2 — Score (three parallel signals)

Every candidate record from Stage 1 is scored across three independent signals, computed in parallel:

##### Signal A: Keyword Match

Substring matching of the query against the record's searchable fields (title, summary, type-specific key fields). Returns a score based on:

- Number of distinct query terms found
- Whether matches are in the title (weighted higher) or body fields
- Exact matches weighted higher than partial matches

**Characteristics:** Fast, precise, zero false positives. Misses semantic matches ("cross-account" won't match "multi-account").

##### Signal B: Semantic Similarity (Vector Search)

The query is embedded using AWS Titan Embeddings v2 and compared via cosine similarity against pre-computed record embeddings stored in DynamoDB (`PK="embedding"`).

```
1. Embed the query text → 1,024-dimension vector
2. Load record embeddings for candidate set (from Stage 1)
3. Cosine similarity: query vector · record vector / (|query| × |record|)
4. Score range: 0.0 (unrelated) to 1.0 (identical meaning)
```

**Characteristics:** Catches semantic matches that keywords miss. "Cross-account access patterns" matches "IAM AssumeRole for multi-account operations" because the concepts are semantically close. May produce false positives on superficially similar but contextually different content.

**Embedding loading:** Record embeddings for the candidate set are loaded from DynamoDB in a single `Query(PK="embedding", SK begins_with "{type}#")` per record type. At typical scale (~50-200 candidates), this is 1-3 DynamoDB queries returning ~200-800 KB of Binary data — fast and cheap.

##### Signal C: Freshness, Trust & Usage

Metadata-derived signals that capture recency, verification status, and usage patterns:

| Signal | Computation | Weight Rationale |
|--------|------------|-----------------|
| Recency | Decay function on `updatedAt` — newer records score higher | Recent knowledge is more likely to be relevant and accurate |
| Verification | `verified` > `unverified` > `stale` > `conflicted` | Verified records are more trustworthy |
| Usage | Boost based on `usage.readCount` and `usage.lastUsedByAgentAt` | Frequently used records are proven useful |
| Confidence | `high` > `medium` > `low` | Higher confidence records are more reliable |

**Characteristics:** No text analysis. Pure metadata. Differentiates between two records that score equally on content relevance.

#### Stage 3 — Blend (weighted combination)

The three signal scores are normalized to [0, 1] and combined with configurable weights:

```typescript
interface ScoringWeights {
  keyword: number;     // default: 0.25
  semantic: number;    // default: 0.45
  freshness: number;   // default: 0.30
}

finalScore = (keyword × w.keyword) + (semantic × w.semantic) + (freshness × w.freshness)
```

**Why these defaults:**
- **Semantic (0.45)** gets the highest weight because it catches the most important matches that other signals miss — the whole reason for adding embeddings.
- **Freshness/trust (0.30)** is second because institutional memory must be trustworthy. A stale, unverified record scoring high on similarity should rank below a verified, recent record.
- **Keyword (0.25)** is lowest because it is the most brittle signal (vocabulary-dependent) but provides precision that embeddings cannot — exact term matches should still be rewarded.

Weights are configurable per deployment. The orchestrator can also override weights per task type — e.g., `domain_lookup` may increase keyword weight since glossary lookups benefit from exact term matching.

#### Stage 4 — Top-K and Handoff

Records are sorted by `finalScore` descending. The top K results (configurable, default: 10) are returned to the orchestrator, which passes them to the context assembler.

```
Top-K records → Context Assembler
  → Priority bucket: "Relevant durable memory" (bucket 6 of 8)
  → Token budget applied per bucket
  → Records formatted as readable prompt text (not JSON)
  → Pruned if total context exceeds cap
```

The context assembler does not re-rank. It trusts the retrieval ranking and applies only token budget constraints. If the durable memory bucket's token cap is reached, lower-ranked records are dropped.

#### Full Flow Diagram

```
User input: "What do we know about cross-account access?"
  │
  ▼
┌──────────────────────────────────────────────────────┐
│ Stage 1: SCOPE                                       │
│                                                      │
│ Task classification → "institutional_memory"          │
│ Priority types → migration, pattern, decision,        │
│                   incident, session_note              │
│ Metadata filter → status in (active, draft)           │
│ Result: 47 candidate records                          │
└──────────────────────┬───────────────────────────────┘
                       ▼
┌──────────────────────────────────────────────────────┐
│ Stage 2: SCORE (parallel)                            │
│                                                      │
│ ┌─────────────┐ ┌──────────────┐ ┌────────────────┐ │
│ │ A: Keyword   │ │ B: Semantic  │ │ C: Freshness   │ │
│ │ "cross"      │ │ embed query  │ │ updatedAt      │ │
│ │ "account"    │ │ cosine sim   │ │ verified?      │ │
│ │ "access"     │ │ vs 47 record │ │ usage count    │ │
│ │              │ │ embeddings   │ │ confidence     │ │
│ │ hits: 12/47  │ │ all 47 scored│ │ all 47 scored  │ │
│ └─────────────┘ └──────────────┘ └────────────────┘ │
└──────────────────────┬───────────────────────────────┘
                       ▼
┌──────────────────────────────────────────────────────┐
│ Stage 3: BLEND                                       │
│                                                      │
│ finalScore = (keyword × 0.25) +                      │
│              (semantic × 0.45) +                     │
│              (freshness × 0.30)                      │
│                                                      │
│ Sort by finalScore descending                        │
└──────────────────────┬───────────────────────────────┘
                       ▼
┌──────────────────────────────────────────────────────┐
│ Stage 4: TOP-K → Context Assembler                   │
│                                                      │
│ Top 10 records:                                      │
│  1. "IAM AssumeRole multi-account" (pattern)   0.91  │
│  2. "Cross-account S3 ownership" (incident)    0.87  │
│  3. "STS temporary credentials" (pattern)      0.83  │
│  4. "Authz policy engine migration" (migration)0.79  │
│  5. ...                                              │
│                                                      │
│ → Formatted as prompt text                           │
│ → Token budget: 2,000 tokens for durable memory      │
│ → Records 8-10 pruned (over budget)                  │
└──────────────────────────────────────────────────────┘
```

#### Fallback Behavior

If the embedding provider is unavailable (network error, service outage, configuration disabled):

1. **Signal B (semantic) is skipped.** Scoring uses only keyword + freshness.
2. **Weights are rebalanced automatically:** keyword = 0.45, freshness = 0.55.
3. **No error is returned.** The retrieval pipeline continues with degraded quality, not failure.
4. **A warning is logged** so the issue is visible but not blocking.

This ensures the system never fails to retrieve records. It may miss semantic matches during an outage, but metadata and keyword retrieval continue working.

---

## 9. Promotion Rules

### 9.1 Most things stay ephemeral

Normal chat transcripts should **not** become durable memory.

### 9.2 Promotion Categories

Only promote when content fits:
- Migration update
- Stable internal pattern
- Reusable debugging lesson
- Architecture decision
- Domain term
- Durable system caveat
- Playbook / investigation recipe
- Alert pattern / detection caveat

### 9.3 Promotion Thresholds

Require at least some of:
- Likely useful again
- Not just task-local
- Stable enough to survive weeks/months
- Tied to evidence
- Understandable out of context

### 9.4 Promotion Modes

| Mode | Risk | When |
|------|------|------|
| Auto-store as session note | Low | Default |
| Auto-promote durable | Medium | High-confidence structured items only |
| Review-required durable | Low | Default for early versions |

---

## 10. NPM Modules

### 10.1 Module List

| # | Module | Role |
|---|--------|------|
| 1 | `@ca-ltm/domain` | Domain types, schemas, validation, pure business logic |
| 2 | `@ca-ltm/storage` | DynamoDB persistence (durable records + session state) |
| 3 | `@ca-ltm/retrieval` | Search, filtering, ranking over CA-LTM records |
| 4 | `@ca-ltm/context-assembler` | Builds prompt/context bundle for model |
| 5 | `@ca-ltm/orchestrator` | Turn-by-turn control plane |
| 6 | `@ca-ltm/sourcegraph` | Sourcegraph MCP adapter |
| 7 | `@ca-ltm/promotion` | Session → durable promotion logic |
| 8 | `@ca-ltm/openclaw-plugin` | OpenClaw integration wiring |
| 9 | `@ca-ltm/api` | Backend service layer for UI/external access |
| 10 | `@ca-ltm/admin-ui` | Human-facing inspection and management UI |
| 11 | `@ca-ltm/harness` | Test runtime — simulated agent environment for E2E testing without OpenClaw |
| 12 | `@ca-ltm/classifier` | Embedding-based classification and semantic search (AWS Titan v2) |
| 13 | `@ca-ltm/importer` | Batch ingestion pipeline for historical/external knowledge sources |

### 10.2 Dependency Graph

```
@ca-ltm/domain
  ↓
@ca-ltm/storage
  ↓
@ca-ltm/classifier (embedding provider + anchor management)
  ↓
@ca-ltm/retrieval (uses classifier for semantic search)
  ↓
@ca-ltm/context-assembler

@ca-ltm/storage + retrieval + context-assembler + sourcegraph + promotion + classifier
  ↓
@ca-ltm/orchestrator (uses classifier for task classification)
  ├──→ @ca-ltm/openclaw-plugin   (production integration)
  └──→ @ca-ltm/harness           (test integration — replaces openclaw-plugin)

@ca-ltm/domain + storage + promotion + classifier
  ↓
@ca-ltm/importer (batch ingestion — peer of orchestrator, not a dependency)

@ca-ltm/openclaw-plugin
  ↓
@ca-ltm/api
  ↓
@ca-ltm/admin-ui
```

Note: `@ca-ltm/harness` depends on orchestrator + all core modules. It does **not** depend on `@ca-ltm/openclaw-plugin` — it calls the orchestrator directly using CA-LTM's own types.

Note: `@ca-ltm/importer` is a peer of the orchestrator — both depend on domain + storage, but they do not depend on each other. The orchestrator handles live sessions; the importer handles historical ingestion.

### 10.3 Boundary Rules

- UI does **not** know DynamoDB shapes
- OpenClaw plugin contains **no** business logic
- Storage does **not** decide orchestration
- Context assembler does **not** fetch data — it decides what enters context
- Domain has **no** I/O
- Harness does **not** import OpenClaw — it calls the orchestrator directly
- Classifier has **no** business logic — it embeds text and computes similarity
- Importer does **not** run continuously — it is a batch tool for historical ingestion

---



## 11. What CA-LTM Does NOT Do

- **Does not store raw code.** Code truth stays in Sourcegraph.
- **Does not replace chat history.** Transcript is OpenClaw's concern.
- **Does not auto-promote everything.** Most session content stays ephemeral.
- **Does not use RAG as the architecture.** RAG is a retrieval mechanism inside CA-LTM.
- **Does not depend on session end.** Checkpoint-based, not end-of-session.
- **Does not mix record types into one blob.** Typed, separated, linked.

---

## 12. Design Principles

1. **Typed memory, not memory soup** — separate record types and collections
2. **Live code stays outside memory** — store pointers and interpretations, not code
3. **Memory must have provenance** — every record says where it came from
4. **Promotion must be selective** — most session content never becomes durable
5. **Retrieval should be hybrid** — exact + metadata + keyword + semantic
6. **Context is a managed hierarchy** — transcript → session → memory → live truth
7. **Inspectability is required** — every record must be readable by humans
