# CA-LTM: Long-Term Memory System — Master Specification

> **Version:** 0.1.0-draft
> **Last Updated:** 2026-04-05
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

### 6.4 SessionState Storage

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

### 6.5 Relationship Storage

**V1:** Inline in `links` array on each record.

**Later (if needed):** Separate edge items for fast neighbor expansion:
```
PK = "edge"
SK = "{sourceType}#{sourceId}#{relationType}#{targetType}#{targetId}"
```

### 6.6 TTL

- Applied to `session_note` records via DynamoDB TTL attribute.
- Durable records do not expire.

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
2. Classify task type: `code_truth` | `institutional_memory` | `mixed`
3. Query CA-LTM for relevant records
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

### 10.2 Dependency Graph

```
@ca-ltm/domain
  ↓
@ca-ltm/storage
  ↓
@ca-ltm/retrieval
  ↓
@ca-ltm/context-assembler

@ca-ltm/storage + retrieval + context-assembler + sourcegraph + promotion
  ↓
@ca-ltm/orchestrator
  ├──→ @ca-ltm/openclaw-plugin   (production integration)
  └──→ @ca-ltm/harness           (test integration — replaces openclaw-plugin)

@ca-ltm/openclaw-plugin
  ↓
@ca-ltm/api
  ↓
@ca-ltm/admin-ui
```

Note: `@ca-ltm/harness` depends on orchestrator + all core modules. It does **not** depend on `@ca-ltm/openclaw-plugin` — it calls the orchestrator directly using CA-LTM's own types.

### 10.3 Boundary Rules

- UI does **not** know DynamoDB shapes
- OpenClaw plugin contains **no** business logic
- Storage does **not** decide orchestration
- Context assembler does **not** fetch data — it decides what enters context
- Domain has **no** I/O
- Harness does **not** import OpenClaw — it calls the orchestrator directly

---

## 11. Implementation Phases

### Phase 0 — Contracts
**Module:** `@ca-ltm/domain`

Lock down all TypeScript interfaces, record schemas, SessionState, validation. Every later module codes against these types.

**Example tasks this enables:**
- An engineer creates a `MigrationRecord` object for the authz-to-policy-engine migration using the factory function, confirms Zod validation catches missing required fields, and verifies the type guard `isMigrationRecord()` correctly discriminates it from a `PatternRecord`.
- A developer writes a unit test that constructs a `PlaybookRecord` with investigation steps, patches it with `patchRecord()` to add a new step, and confirms `updatedAt` is automatically refreshed and the original object is unchanged (immutability).

### Phase 1 — Durable Storage + Test Harness
**Modules:** `@ca-ltm/storage` + `@ca-ltm/harness` (initial)

DynamoDB table, record CRUD, session state CRUD, checkpoints, TTL. Simultaneously build the test harness with `InMemoryStorage` and Vitest + DynamoDB Local integration. Prove the schemas work with real data.

The harness grows with each subsequent phase — adding mock services, scenario support, and assertion helpers as new modules come online.

**Example tasks this enables:**
- A team lead manually seeds three migration records and two pattern records into DynamoDB, then queries `queryByType("migration", { status: "active" })` and confirms only the active migrations are returned — verifying that the `PK=type, SK=id` schema and filter expressions work correctly.
- An engineer stores a `session_note` with `expiresAfterDays: 30`, verifies the TTL epoch is correctly computed, and confirms the record auto-deletes from DynamoDB after expiration — proving ephemeral records don't accumulate.
- A developer runs `createTestHarness({ storage: "dynamodb-local" })`, seeds two migration records, queries by type and status, and confirms the same results as `InMemoryStorage` — proving both storage backends behave identically.

### Phase 2 — Basic Admin UI
**Modules:** `@ca-ltm/api` (minimal) + `@ca-ltm/admin-ui` (minimal)

List page, filters, detail page, provenance display. Make memory inspectable early.

**Example tasks this enables:**
- A team lead opens the admin UI, filters to `type=migration, status=active`, and sees all three active migration records with their titles, owners, confidence levels, and last-verified dates — answering "what does the system know about our migrations?" in seconds.
- An engineer clicks into the "Authz Policy Engine" migration detail page and sees its rules (do/avoid), blockers, related systems, provenance (linked to ADR and session), and Sourcegraph query hints — giving a complete picture of what CA-LTM knows and where it learned it.

### Phase 3 — SessionState
**Extend:** `@ca-ltm/storage`

Session load/save/checkpoint/restore. Prove structured working state survives long conversations.

**Example tasks this enables:**
- During a 40-turn debugging conversation, the agent tracks `currentObjective: "diagnose cross-account SSM access failure"`, accumulates findings incrementally, and after OpenClaw compacts the thread, the restored SessionState still contains the objective, three key findings, and two open questions — no context lost.
- An engineer simulates a session interruption: the agent checkpoints at turn 15, the user comes back the next day, and `getLatestCheckpoint()` restores the full working state including active systems, candidate records, and unresolved hypotheses — proving continuity without a formal session-end.

### Phase 4 — Manual Orchestrator
**Module:** `@ca-ltm/orchestrator`

Load session, fetch tagged records, assemble simple context, update session. Hardcoded/rule-based first.

**Example tasks this enables:**
- A developer asks "how should I update the auth middleware during the policy engine migration?" The orchestrator classifies this as `mixed`, fetches the active authz migration record from CA-LTM (with its do/avoid rules) and queries Sourcegraph for current auth middleware files — combining institutional memory with live code truth in a single grounded answer.
- An engineer starts a new conversation about the onboarding service. The orchestrator's recall flow loads the bootstrap set: active migrations touching onboarding, the cross-account access pattern, and a detection caveat about noisy alerts during the current infra migration — giving the agent institutional awareness before the first question is even answered.

### Phase 5 — Context Assembler v1
**Module:** `@ca-ltm/context-assembler`

Context buckets, pruning, caps, merge logic. Deterministic and inspectable.

**Example tasks this enables:**
- After 20 turns of investigation, the agent's context includes the objective, three active findings, two relevant migration records, one playbook, and Sourcegraph search results — totaling ~6,000 tokens. Verbose intermediate tool logs and completed substeps from earlier turns are pruned automatically. The debug output shows exactly why each item was included and what was dropped.
- The agent is working on an incident involving S3 cross-account permissions. The context assembler includes the relevant incident record, the cross-account pattern record, and the active infra migration's "avoid" rules — but caps durable memory at 2,000 tokens and drops the lowest-scored glossary matches, keeping the prompt focused on what helps decide the next action.

### Phase 6 — Checkpointing
**Extend:** `@ca-ltm/orchestrator` + `@ca-ltm/storage`

Checkpoint triggers, session snapshots, resume after compaction/idle.

**Example tasks this enables:**
- During a long migration planning thread, the agent identifies a new blocker at turn 12. The content trigger fires ("blocker identified"), the orchestrator snapshots the session state, and writes a session note summarizing findings so far — even though the user hasn't stopped or the session hasn't ended. If OpenClaw compacts the thread at turn 30, the important discovery is already persisted.
- An engineer works on an incident for 45 minutes, then gets pulled away. The behavioral trigger (30 minutes elapsed) fires a checkpoint. Two days later, when the engineer returns and says "where were we?", the orchestrator resumes from the checkpoint with the full working state: root cause hypothesis, systems investigated, open questions remaining.

### Phase 7 — Manual Promotion
**Module:** `@ca-ltm/promotion`

Propose, review, approve/reject, patch existing records. Learn correct behavior before automating.

**Example tasks this enables:**
- After debugging an S3 cross-account object ownership issue, the agent proposes a new `incident` record with symptoms, root cause, fix, and prevention steps. A team lead reviews it in the admin UI, edits the prevention section to be more specific, and approves it — the incident is now durable institutional memory that will surface the next time anyone hits a similar S3 ownership problem.
- During a session about auth patterns, the agent identifies that the existing "cross-account access" pattern record is missing a recently discovered caveat about temporary credentials. The promotion pipeline proposes a `patch` action instead of creating a duplicate. The reviewer approves the patch, and the pattern record is updated with the new guidance and fresh `lastVerifiedAt`.

### Phase 8 — Sourcegraph Integration
**Module:** `@ca-ltm/sourcegraph`

Query hints on records, expand records into live code queries, launch from playbooks/migrations.

**Example tasks this enables:**
- A migration record for "authz policy engine" carries `sourcegraphQueries: ["repo:^github.com/org/main$ authz OR authorization OR policy"]`. When the agent needs to advise on an auth change, it executes this query against live Sourcegraph and returns current files and symbols — institutional intent ("migrate to policy engine") grounded against current code reality ("here are the 14 files still using legacy middleware").
- A playbook for "websocket reconnect storm" includes a `queries` section with Sourcegraph searches for `WebSocketReconnectHandler` and `connectionPool.drain`. During an incident, the agent executes these saved queries to immediately locate the relevant code paths — instead of the engineer spending 20 minutes manually searching. The playbook effectively teaches the agent *how to investigate*.

### Phase 9 — Retrieval v1
**Module:** `@ca-ltm/retrieval`

Exact lookup, keyword search, metadata filtering, hybrid scoring. Semantic/embeddings only if needed.

**Example tasks this enables:**
- An engineer asks "what do we know about the token refresh flow?" The retrieval service searches across all record types, finds: an incident record about auth token refresh errors (keyword: "token refresh"), the cross-account access pattern (related system: "sts"), and a detection caveat about noisy auth alerts during migration — ranked by relevance. The agent synthesizes these into a grounded, multi-perspective answer.
- The orchestrator calls `retrieval.bootstrap({ systems: ["api-gateway"] })` at session start. Retrieval returns the top 5 most relevant records: two active migrations touching api-gateway, the centralized auth pattern, and a recent incident — without loading every record in the system. The agent starts the conversation already aware of the critical context for that subsystem.

### Phase 10 — Auto-Promotion
**Extend:** `@ca-ltm/promotion`

Candidate extraction heuristics, type classification, create-vs-patch suggestions, confidence thresholds.

**Example tasks this enables:**
- During a conversation, the user explains "we never embed long-lived credentials for cross-account access, we always use assume-role with narrowly scoped trust policies." The extraction heuristic detects the signal phrase "we always" and proposes a `pattern` record titled "Cross-account access via assume-role" with `confidence: high`. Because confidence meets the auto-promote threshold and no conflicting record exists, it is auto-stored as a durable pattern — the team's convention is now institutional memory without anyone filling out a form.
- An engineer and the agent diagnose a deployment failure together. The agent identifies root cause ("missing SSM parameter in target account") and fix ("add parameter to CDK stack"). The extractor detects the incident structure (symptoms → root cause → fix) and proposes an `incident` record. Because it's the first time this incident is seen, it's flagged `reviewRequired: true` — but the proposal is pre-filled and ready for one-click approval.

### Phase 11 — Graph Explorer
**Extend:** `@ca-ltm/admin-ui`

Relationship navigation, migration dependency graph, playbook/incident linkage, timeline views.

**Example tasks this enables:**
- A team lead opens the graph explorer and views the "Authz Policy Engine" migration node. It shows edges to: 3 related patterns, 2 dependent decisions, 1 blocking incident, and 5 affected systems. She immediately sees that the migration is blocked by an unresolved incident in the account-service — a dependency that wasn't visible from reading individual records. She clicks through to the incident and assigns follow-up.
- An SRE views the "api-gateway" system node in the graph and sees all connected knowledge: 2 active migrations, 4 patterns, 3 playbooks, and 6 historical incidents. The cluster reveals that most incidents correlate with the auth migration period — confirming the team's suspicion that the migration is the root of recent instability. The timeline view shows incidents spiking after the migration's phase-2 milestone.

### Phase 12 — Verification & Staleness
**Extend:** multiple modules

`lastVerifiedAt`, stale queues, revalidation workflow, Sourcegraph-assisted verification.

**Example tasks this enables:**
- A pattern record for "legacy permission middleware" was last verified 6 months ago. The staleness detector flags it. The verification workflow executes the record's Sourcegraph hints and discovers that the legacy middleware was fully removed 2 months ago. The record is automatically marked `status: deprecated` and `superseded_by: pattern-policy-engine-authz` — preventing the agent from ever recommending the old approach again.
- The admin UI's verification dashboard shows 8 stale records (unverified for >90 days), 3 of which are migration records. A team lead reviews them in bulk: one migration is now complete (marked `resolved`), one has a new blocker (patched), and one is still accurate (re-verified with fresh `lastVerifiedAt`). Total time: 10 minutes. The system's knowledge is now trustworthy for another quarter.

---

## 12. What CA-LTM Does NOT Do

- **Does not store raw code.** Code truth stays in Sourcegraph.
- **Does not replace chat history.** Transcript is OpenClaw's concern.
- **Does not auto-promote everything.** Most session content stays ephemeral.
- **Does not use RAG as the architecture.** RAG is a retrieval mechanism inside CA-LTM.
- **Does not depend on session end.** Checkpoint-based, not end-of-session.
- **Does not mix record types into one blob.** Typed, separated, linked.

---

## 13. Design Principles

1. **Typed memory, not memory soup** — separate record types and collections
2. **Live code stays outside memory** — store pointers and interpretations, not code
3. **Memory must have provenance** — every record says where it came from
4. **Promotion must be selective** — most session content never becomes durable
5. **Retrieval should be hybrid** — exact + metadata + keyword + semantic
6. **Context is a managed hierarchy** — transcript → session → memory → live truth
7. **Inspectability is required** — every record must be readable by humans
