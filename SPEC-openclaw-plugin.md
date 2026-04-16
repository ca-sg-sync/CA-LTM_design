# @ca-ltm/openclaw-plugin — Module Specification

> **Version:** 0.1.0-draft
> **Phase:** 4+ (alongside or after orchestrator skeleton)

---

## 1. Module Identity

| Field | Value |
|-------|-------|
| **Package** | `@ca-ltm/openclaw-plugin` |
| **Role** | OpenClaw integration wiring — no business logic |
| **Dependencies** | `@ca-ltm/domain`, `@ca-ltm/orchestrator` |
| **Dependents** | None (top-level integration point) |

---

## 2. Responsibilities

- Plugin registration with OpenClaw runtime
- Memory slot integration (replaces default `memory-core`)
- Lifecycle hook bindings (session start, pre-compaction, housekeeping)
- Turn event translation (OpenClaw events → CA-LTM `TurnEvent`)
- Session ID mapping (OpenClaw sessions → CA-LTM session IDs)
- Tool registration (expose CA-LTM tools to the agent)
- Forwarding all lifecycle events to the orchestrator

---

## 3. Design Principle

> This module is **pure wiring**. It contains no business logic, no storage access, no record creation. Everything is delegated to the orchestrator.

---

## 4. OpenClaw Integration Points

### Memory Slot

CA-LTM registers via `plugins.slots.memory`, replacing `memory-core`. This provides:

- Memory read/write lifecycle access
- Pre-compaction hooks
- Session context integration

### Lifecycle Hooks

| OpenClaw Event | CA-LTM Action |
|----------------|---------------|
| Session attach / first turn | → `orchestrator.onTurn()` (triggers recall flow) |
| Each agent turn | → `orchestrator.onTurn()` |
| Pre-compaction | → `orchestrator.onCompaction()` |
| Silent housekeeping | → `orchestrator.onCheckpoint()` |
| `/new` | → `orchestrator.onCompaction()` + new session |
| `/reset` | → `orchestrator.onCompaction()` + clear session |
| `/stop` | → `orchestrator.onCompaction()` |

### Registered Tools

Tools exposed to the OpenClaw agent for direct invocation:

| Tool Name | Description | Maps To |
|-----------|-------------|---------|
| `ltm_search` | Search CA-LTM records | `retrieval.search()` via orchestrator |
| `ltm_get` | Get a specific record by type + id | `retrieval.get()` |
| `ltm_remember` | Propose a new durable record | `promotion.extractCandidates()` → `storage.put()` |
| `ltm_update` | Patch an existing record | `storage.update()` |
| `ltm_active_migrations` | List active migrations | `retrieval.activeMigrations()` |
| `ltm_related` | Find related records | `retrieval.related()` |
| `ltm_playbook` | Find relevant playbook | `retrieval.search({ types: ["playbook"] })` |

---

## 5. Public API

```typescript
interface CaLtmPlugin {
  name: string;               // "ca-ltm"
  version: string;

  register(runtime: OpenClawRuntime): void;

  onSessionStart(session: OpenClawSession): Promise<void>;

  onTurn(turn: OpenClawTurnEvent): Promise<OrchestratorResult>;

  onPreCompaction(session: OpenClawSession): Promise<void>;

  onHousekeeping(session: OpenClawSession): Promise<void>;

  onLifecycleEvent(event: OpenClawLifecycleEvent): Promise<void>;
}

function createCaLtmPlugin(orchestrator: Orchestrator, config?: PluginConfig): CaLtmPlugin;
```

### Configuration

```typescript
interface PluginConfig {
  enabledTools?: string[];        // which tools to register (default: all)
  autoRecallOnSessionStart?: boolean; // default: true
  autoCheckpointOnCompaction?: boolean; // default: true
}
```

---

## 6. Event Translation

```typescript
function toTurnEvent(openclawTurn: OpenClawTurnEvent): TurnEvent {
  return {
    sessionId: mapSessionId(openclawTurn.session),
    text: openclawTurn.content,
    role: mapRole(openclawTurn.role),
    timestamp: new Date().toISOString(),
    metadata: openclawTurn.metadata,
  };
}

function mapSessionId(session: OpenClawSession): string {
  // Map OpenClaw session identifiers to CA-LTM session IDs
  // DM sessions: use user identifier
  // Channel sessions: use channel identifier
  // Webhook/cron: use source identifier
  return `openclaw-${session.source}-${session.id}`;
}

function mapRole(openclawRole: string): "user" | "assistant" | "tool" {
  // Map OpenClaw role strings to CA-LTM role enum
}
```

---

## 7. Tool Handlers

```typescript
// Example tool handler for ltm_search
async function handleLtmSearch(args: {
  query?: string;
  type?: string;
  system?: string;
  status?: string;
  limit?: number;
}): Promise<MemorySearchResult[]> {
  return orchestrator.retrieval.search({
    query: args.query,
    types: args.type ? [args.type as RecordType] : undefined,
    relatedSystems: args.system ? [args.system] : undefined,
    status: args.status ? [args.status as RecordStatus] : undefined,
    limit: args.limit,
  });
}
```

---

## 8. Internal Structure

```
src/
  index.ts                    # barrel exports: createCaLtmPlugin
  plugin.ts                   # CaLtmPlugin implementation + registration
  hooks/
    session.ts                # Session start hook
    compaction.ts             # Pre-compaction hook
    housekeeping.ts           # Silent housekeeping hook
    lifecycle.ts              # /new, /reset, /stop handling
  tools/
    registration.ts           # Tool registration with OpenClaw
    handlers.ts               # Tool call handlers (ltm_search, ltm_get, etc.)
  mapping/
    sessions.ts               # OpenClaw → CA-LTM session ID mapping
    events.ts                 # Event translation
  config.ts                   # PluginConfig defaults
```

---

## 9. What It Does NOT Do

- **No business logic** — pure wiring
- **No storage access** — delegates to orchestrator
- **No record creation/patching** — delegates to orchestrator → promotion
- **No context assembly** — delegates to orchestrator → context-assembler
- **No retrieval logic** — delegates to orchestrator → retrieval

---

## 10. OpenClaw-Specific Types (expected from OpenClaw SDK)

These are types this module expects from OpenClaw. Actual shapes depend on OpenClaw's plugin API:

```typescript
// Expected from OpenClaw — not defined by CA-LTM
interface OpenClawRuntime {
  registerPlugin(plugin: unknown): void;
  registerTools(tools: unknown[]): void;
}

interface OpenClawSession {
  id: string;
  source: string;           // "dm" | "channel" | "webhook" | "cron"
  metadata?: Record<string, unknown>;
}

interface OpenClawTurnEvent {
  session: OpenClawSession;
  content: string;
  role: string;
  metadata?: Record<string, unknown>;
}

interface OpenClawLifecycleEvent {
  type: "new" | "reset" | "stop";
  session: OpenClawSession;
}
```

---

## 11. Test Criteria

- Plugin registers correctly with OpenClaw runtime
- All lifecycle hooks forward to orchestrator correctly
- Turn events are correctly translated (session ID, role, text)
- Session ID mapping is deterministic and consistent
- Registered tools are callable and return correct result shapes
- Pre-compaction triggers `orchestrator.onCompaction()`
- `/new` / `/reset` / `/stop` events trigger compaction
- Plugin handles missing/undefined session metadata gracefully
- Tool handlers validate arguments before forwarding
- Plugin respects `enabledTools` configuration
