## Implementation Phases

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

### Phase 9 — Retrieval v1 + Semantic Search
**Modules:** `@ca-ltm/retrieval` + `@ca-ltm/classifier`

Exact lookup, keyword search, metadata filtering, and **embedding-based semantic similarity** via `@ca-ltm/classifier` using AWS Titan Embeddings v2. Hybrid scoring combines metadata filters (exact), keyword hits (precise), and cosine similarity against record embeddings (semantic) into a weighted blend. Embeddings are stored as separate DynamoDB items (`PK="embedding"`, Binary float32 vectors) to avoid inflating record reads.

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

### Phase 11b — Graph Timeline Playback
**Extend:** `@ca-ltm/admin-ui` + `@ca-ltm/api`

Animated timeline playback of the knowledge graph being constructed over time. Uses the event log (`GET /events/timeline`) to identify mutation timestamps and point-in-time reconstruction (`GET /events/reconstruct?timestamp=...`) to rebuild the graph state at each point. The user scrubs a timeline slider to watch nodes and edges appear, change status, and get archived — visualizing how institutional knowledge accumulated.

**Example tasks this enables:**
- A team lead opens the graph timeline, sets the range to the last 3 months, and hits play. She watches the "Authz Policy Engine" migration node appear, then sees pattern and decision nodes link to it over subsequent weeks. An incident node appears and connects with a `caused_by` edge — she can see exactly when the team learned the migration was causing instability, and how playbooks were added in response. The replay makes the knowledge evolution visible in a way that reading individual records cannot.
- During a retrospective, the team plays back the last quarter's graph growth filtered to `type=incident`. They see incident clusters forming around the api-gateway system node, peaking in week 6, then declining after playbook and detection caveat records were added. The timeline visualization directly shows the ROI of capturing institutional knowledge — fewer repeated incidents after the knowledge was codified.

### Phase 12 — Verification & Staleness
**Extend:** multiple modules

`lastVerifiedAt`, stale queues, revalidation workflow, Sourcegraph-assisted verification.

**Example tasks this enables:**
- A pattern record for "legacy permission middleware" was last verified 6 months ago. The staleness detector flags it. The verification workflow executes the record's Sourcegraph hints and discovers that the legacy middleware was fully removed 2 months ago. The record is automatically marked `status: deprecated` and `superseded_by: pattern-policy-engine-authz` — preventing the agent from ever recommending the old approach again.
- The admin UI's verification dashboard shows 8 stale records (unverified for >90 days), 3 of which are migration records. A team lead reviews them in bulk: one migration is now complete (marked `resolved`), one has a new blocker (patched), and one is still accurate (re-verified with fresh `lastVerifiedAt`). Total time: 10 minutes. The system's knowledge is now trustworthy for another quarter.

---