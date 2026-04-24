# CA-LTM Code Assist — Design Specification

### Lucio Flores — [CleverAlpha Technologies](https://www.cleveralphatechnologies.com)

> **Version:** 0.1.0-draft
> **Last Updated:** 2026-04-24
> **Status:** Design — Pre-Implementation

---

## 1. What Code Assist Is

CA-LTM Code Assist is a **local IDE coding assistant** powered by a remote OpenClaw agent running on NemoClaw (AWS). It provides an Amp/Cursor-like experience — chat, selection-aware actions, code edits, tool use — on CleverAlpha's own infrastructure, enriched by CA-LTM institutional memory.

The user works from a **single local git repo** on their laptop. The local filesystem is the source of truth for code. The remote agent reasons, retrieves institutional memory, and issues tool calls that execute locally.

### Core Principle

> **Local bridge = code truth (filesystem, git, terminal)**
> **Remote OpenClaw = reasoning and orchestration**
> **CA-LTM = institutional memory (patterns, decisions, migrations, playbooks…)**
> **Sourcegraph = cross-repo search and org-wide references (enrichment, not primary)**

---

## 2. Motivation

### What This Adds Beyond Amp/Cursor

General-purpose coding assistants are powerful but lack:

| Gap | Code Assist Solution |
|-----|---------------------|
| **No institutional memory** — every session starts from zero organizational knowledge | CA-LTM surfaces relevant migrations, patterns, decisions, playbooks, and caveats automatically |
| **No audit trail** — the agent's reasoning is ephemeral | CA-LTM event log + SessionState checkpoints provide full traceability |
| **No org-owned infra** — model, data, and prompts controlled by vendor | NemoClaw runs on CleverAlpha's AWS — model choice, prompt control, data boundaries enforced |
| **No domain awareness** — agents don't know financial services constraints, internal vocabulary, or compliance rules | CA-LTM glossary, decision, and compliance records provide domain grounding |
| **No durable learning** — agents don't get better over months | CA-LTM promotion pipeline captures reusable knowledge from sessions |

### What This Does NOT Replace

- **Amp/Cursor for open-source or greenfield work** — Code Assist is optimized for CleverAlpha's codebase and institutional context. It is not a general-purpose coding assistant replacement.
- **Sourcegraph** — Sourcegraph remains the live code intelligence layer. Code Assist does not index or store code.
- **OpenClaw** — Code Assist is a client of OpenClaw, not a replacement. The agent runtime is OpenClaw.

---

## 3. Architecture

### 3.1 Topology

```
┌─────────────────────────────────────┐
│           Developer Laptop          │
│                                     │
│  ┌──────────────┐  ┌─────────────┐  │
│  │  IDE Plugin   │  │ Local Bridge│  │
│  │  (JetBrains)  │←→│   Daemon    │  │
│  │              │  │             │  │
│  │  • Chat UI    │  │ • File I/O  │  │
│  │  • Selection  │  │ • Git ops   │  │
│  │  • Patch UX   │  │ • Terminal  │  │
│  │  • Diagnostics│  │ • Grep/Glob │  │
│  └──────────────┘  └──────┬──────┘  │
│                           │         │
└───────────────────────────┼─────────┘
                            │ outbound WSS
                            ▼
              ┌─────────────────────────┐
              │     IDE Gateway         │
              │   (@ca-ltm/api ext)     │
              │                         │
              │  • WebSocket broker     │
              │  • Session management   │
              │  • Tool-call routing    │
              │  • Auth / audit         │
              │  • Streaming            │
              └────────────┬────────────┘
                           │
                           ▼
              ┌─────────────────────────┐
              │   OpenClaw (NemoClaw)   │
              │                         │
              │  • Agent reasoning      │
              │  • Prompt construction  │
              │  • Tool orchestration   │
              │  • vLLM inference       │
              └────────────┬────────────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
              ▼            ▼            ▼
        ┌──────────┐ ┌──────────┐ ┌──────────┐
        │ CA-LTM   │ │Sourcegraph│ │ CA-LTM  │
        │Orchestr. │ │   MCP    │ │ Storage  │
        │Context   │ │          │ │ DynamoDB │
        │Assembly  │ │          │ │ Events   │
        └──────────┘ └──────────┘ └──────────┘
```

### 3.2 What Runs Where

| Location | Component | Responsibility |
|----------|-----------|---------------|
| **Laptop** | IDE Plugin | UI shell — chat, selection actions, patch preview/apply, editor context capture, user approvals |
| **Laptop** | Local Bridge Daemon | Workspace tools — file read/write, grep, glob, git, terminal. Outbound WSS to AWS. IDE-agnostic. |
| **AWS** | IDE Gateway | WebSocket session broker, tool-call routing (local ↔ remote), streaming, auth, audit |
| **AWS** | OpenClaw | Agent brain — reasoning, prompt construction, tool orchestration, model inference |
| **AWS** | CA-LTM | Institutional memory — orchestrator, retrieval, context assembly, storage, promotion, events |
| **AWS** | Sourcegraph MCP | Cross-repo search, org-wide symbol lookup, references |

### 3.3 Key Design Decisions

1. **Agent brain stays remote.** No local OpenClaw runtime. Centralizes reasoning, model config, CA-LTM access, and audit in one place.

2. **Local bridge is IDE-agnostic.** Speaks a protocol (WebSocket + JSON-RPC), not an IDE API. JetBrains and VS Code plugins are thin UI shells connecting to the same bridge.

3. **Local filesystem is the code truth.** The user works from a single local git clone. No "baseline vs delta" reconciliation needed. Sourcegraph is for cross-repo enrichment only.

4. **CA-LTM is an optional enrichment layer.** The code assistant is functional without CA-LTM (OpenClaw + local tools + Sourcegraph). CA-LTM is additive, not a hard dependency.

5. **Edits require user approval.** The agent proposes edits (structured diffs or patches). The IDE plugin shows a preview. The user applies.

---

## 4. Components

### 4.1 IDE Plugin (`@ca-ltm/jetbrains-plugin`)

The thin UI layer inside JetBrains (IntelliJ, WebStorm).

**Capabilities:**

- Chat panel with streaming responses
- Selection-aware actions: "Ask", "Edit", "Explain", "Fix"
- Patch preview and apply via IntelliJ APIs
- Editor context capture:
  - Active file path and content
  - Selection / caret position
  - Open tabs
  - Unsaved buffer contents
  - Diagnostics / problems from IDE inspections
- User approval UX for write operations
- Session history (list of past chat sessions)
- Citations linking back to CA-LTM records and Sourcegraph results

**Does NOT do:**
- Reasoning, retrieval, or context assembly
- Direct DynamoDB or Sourcegraph access
- File I/O outside the IDE's own APIs

**Future:** A VS Code extension (`@ca-ltm/vscode-extension`) would implement the same capabilities against the same bridge protocol.

### 4.2 Local Bridge Daemon (`@ca-ltm/local-bridge`)

A lightweight Node.js daemon running on localhost. The reusable "desktop sidecar" that any IDE plugin connects to.

**Capabilities:**

| Tool | Description |
|------|-------------|
| `readFile` | Read file contents by path (workspace-scoped) |
| `writeFile` | Write/patch file (requires user approval via plugin) |
| `listDir` | List directory contents |
| `grep` | Regex search across files |
| `glob` | Find files by pattern |
| `gitStatus` | Current branch, dirty files, staged changes |
| `gitDiff` | Diff of working tree or between refs |
| `gitLog` | Commit history |
| `exec` | Terminal command execution (guarded, approval-required) |
| `getDiagnostics` | IDE diagnostics forwarded from plugin |

**Protocol:** JSON-RPC over WebSocket.

**Security constraints:**
- Listens on **localhost only** (127.0.0.1)
- Workspace allowlist — can only access the approved project root
- Read-only tools auto-allowed; write/exec require user approval routed through the IDE plugin
- Never sends secrets, credentials, or env vars to the remote agent
- Outbound-only connection to AWS (no inbound ports)

**Lifecycle:** Starts when the IDE plugin activates. Stops when the IDE closes or the user explicitly stops it.

### 4.3 IDE Gateway (extension of `@ca-ltm/api`)

The server-side WebSocket endpoint that bridges the local client to OpenClaw and CA-LTM.

**Capabilities:**

- WebSocket session management (create, resume, close)
- Tool-call routing:
  - Local tools → forward to bridge over WebSocket, return result to OpenClaw
  - Remote tools (CA-LTM, Sourcegraph) → execute server-side, return to OpenClaw
- Streaming token/event relay from OpenClaw → plugin
- Authentication and authorization
- Audit logging (who asked what, which tools were called)
- Session metadata tracking (userId, repo, branch, projectName)

**Implementation:** New route group within `@ca-ltm/api` (`/api/ide/`). Separate service only if load demands it later.

---

## 5. Data Flow

### 5.1 Session Start

```
IDE Plugin                    Bridge              Gateway              OpenClaw + CA-LTM
    │                           │                    │                       │
    │── user opens chat ──────→ │                    │                       │
    │   sends:                  │                    │                       │
    │   • userId                │                    │                       │
    │   • repo / branch / HEAD  │                    │                       │
    │   • active file + content │                    │                       │
    │   • selection / caret     │                    │                       │
    │   • open tabs             │                    │                       │
    │   • dirty file list       │                    │                       │
    │   • user message          │                    │                       │
    │                           │── WSS connect ───→ │                       │
    │                           │   + session init   │                       │
    │                           │                    │── create/resume ────→ │
    │                           │                    │   session              │
    │                           │                    │                       │
    │                           │                    │   CA-LTM orchestrator: │
    │                           │                    │   • load SessionState  │
    │                           │                    │   • classify task      │
    │                           │                    │   • retrieve memory    │
    │                           │                    │   • assemble context   │
    │                           │                    │                       │
    │                           │                    │   OpenClaw:            │
    │                           │                    │   • build prompt       │
    │                           │                    │   • call vLLM          │
    │                           │                    │← streaming response ──│
    │                           │← stream tokens ───│                       │
    │← render in chat ─────────│                    │                       │
```

### 5.2 Tool-Call Routing

When the remote agent needs local information:

```
OpenClaw                    Gateway              Bridge              Laptop FS
    │                          │                    │                    │
    │── tool_call:             │                    │                    │
    │   readFile("src/auth.ts")│                    │                    │
    │                          │── route to bridge →│                    │
    │                          │   (local tool)     │── fs.readFile ───→│
    │                          │                    │← file contents ───│
    │                          │← tool result ──────│                    │
    │← tool result ────────────│                    │                    │
    │                          │                    │                    │
    │── tool_call:             │                    │                    │
    │   ltm.search("auth")     │                    │                    │
    │                          │── execute locally  │                    │
    │                          │   (remote tool)    │                    │
    │← tool result ────────────│                    │                    │
```

**Routing rule:** The gateway inspects the tool name prefix. `local.*` tools are forwarded to the bridge. All others execute server-side.

### 5.3 Edit Flow

```
OpenClaw                    Gateway              Bridge              IDE Plugin
    │                          │                    │                    │
    │── response with          │                    │                    │
    │   structured edits       │                    │                    │
    │   (unified diff or       │                    │                    │
    │    file patches)         │                    │                    │
    │                          │── stream to plugin │                    │
    │                          │                    │                    │← render diff
    │                          │                    │                    │   preview
    │                          │                    │                    │
    │                          │                    │                    │── user clicks
    │                          │                    │                    │   "Apply"
    │                          │                    │← apply patch ──────│
    │                          │                    │── write files ────→│ (filesystem)
    │                          │                    │← success ─────────│
    │                          │← patch applied ────│                    │
    │← confirmation ───────────│                    │                    │
```

### 5.4 Checkpoint and Promotion

After N turns, elapsed time, or content triggers:

1. CA-LTM orchestrator updates `SessionState` (findings, hypotheses, open questions)
2. Writes checkpoint to DynamoDB
3. Extracts promotion candidates (if any)
4. Queues candidates for review or auto-promotes based on confidence
5. Emits `MemoryEvent` to event log

This happens entirely server-side. The user is not interrupted.

### 5.5 Session Resume

When the user reopens a previous chat session:

1. Plugin reconnects with `sessionId`
2. Gateway resumes the OpenClaw session
3. CA-LTM rehydrates `SessionState` from latest checkpoint
4. Plugin re-sends current editor context (active file, dirty buffers)
5. Agent resumes with prior objective/findings + current workspace state

---

## 6. Session Model

### 6.1 Mapping

**1 chat tab = 1 OpenClaw session = 1 CA-LTM SessionState**

### 6.2 Session Metadata

```typescript
interface IdeSession {
  sessionId: string;
  userId: string;
  projectName: string;
  repo: string;
  branch: string;
  gitHead: string;
  ideType: "jetbrains" | "vscode" | "cli";
  workspaceRoot: string;        // local path (for display only, not used server-side)
  createdAt: string;
  lastActiveAt: string;
}
```

### 6.3 What Is NOT in Scope

- Shared workspace-wide memory across multiple chat tabs
- Multi-user collaborative sessions
- Multi-repo workspace sessions (future, different client)

---

## 7. Tool Integration

### 7.1 Local Tools (via Bridge)

| Tool | Auto-Allow | Description |
|------|-----------|-------------|
| `local.readFile` | ✅ | Read file contents |
| `local.listDir` | ✅ | List directory |
| `local.grep` | ✅ | Regex search |
| `local.glob` | ✅ | File pattern match |
| `local.gitStatus` | ✅ | Branch, dirty files |
| `local.gitDiff` | ✅ | Working tree diff |
| `local.gitLog` | ✅ | Commit history |
| `local.writeFile` | ❌ | Write file (approval required) |
| `local.exec` | ❌ | Run terminal command (approval required) |

### 7.2 Remote Tools (server-side)

| Tool | Description |
|------|-------------|
| `ltm.search` | Hybrid search across CA-LTM records |
| `ltm.get` | Direct record lookup |
| `ltm.related` | Expand linked records |
| `ltm.bootstrap` | Load relevant startup set |
| `ltm.activeMigrations` | Active migration records |
| `ltm.writeSessionNote` | Store ephemeral finding |
| `ltm.proposeRecord` | Propose durable record |
| `sourcegraph.search` | Cross-repo code search |
| `sourcegraph.symbols` | Symbol lookup |
| `sourcegraph.references` | Find references |

### 7.3 Tool Routing Rule

The gateway inspects the tool name prefix:

- `local.*` → forward to bridge WebSocket → execute on laptop → return result
- `ltm.*` → execute via CA-LTM modules server-side
- `sourcegraph.*` → execute via Sourcegraph MCP server-side

---

## 8. Context Management

All context management happens **server-side**. The local client is a context provider, not a context manager.

### 8.1 Context Sources

| Source | Provider | What It Contributes |
|--------|----------|-------------------|
| Editor state | IDE Plugin (via bridge) | Active file, selection, open tabs, diagnostics, dirty buffers |
| Workspace files | Local Bridge | File contents, directory structure, search results |
| Git state | Local Bridge | Branch, diff, log, uncommitted changes |
| Institutional memory | CA-LTM | Relevant records (migrations, patterns, decisions, playbooks…) |
| Session state | CA-LTM | Current objective, findings, hypotheses, open questions |
| Cross-repo code | Sourcegraph MCP | Symbols, references, search results from indexed repos |
| Conversation | OpenClaw | Message history, tool call/response history |

### 8.2 Assembly

The CA-LTM context assembler builds the context bundle using the 8-bucket priority order from the master spec:

1. Policy / role instructions
2. Current objective
3. Current plan
4. Critical constraints
5. Top session findings
6. Relevant durable memory
7. Relevant live retrieval (Sourcegraph)
8. Very recent conversational window

**Pruning rule:** Include only what helps the model decide the *next correct action*.

### 8.3 When CA-LTM Is Unavailable

If CA-LTM modules are not yet operational or encounter errors:

- Buckets 2–6 are empty (no session state, no institutional memory)
- OpenClaw still functions with: system prompt + editor context + conversation history + Sourcegraph
- The assistant is still useful — it just lacks institutional grounding

---

## 9. Protocol

### 9.1 Plugin ↔ Bridge

**Transport:** WebSocket on `ws://127.0.0.1:{port}`

**Messages (Plugin → Bridge):**

```typescript
// Session lifecycle
{ type: "session.start", sessionId: string, workspace: WorkspaceContext }
{ type: "session.resume", sessionId: string, workspace: WorkspaceContext }
{ type: "session.close", sessionId: string }

// User input
{ type: "user.message", sessionId: string, message: string, editorContext: EditorContext }

// Approval responses
{ type: "approval.response", requestId: string, approved: boolean }
```

**Messages (Bridge → Plugin):**

```typescript
// Streaming response
{ type: "assistant.token", sessionId: string, token: string }
{ type: "assistant.done", sessionId: string }

// Tool activity
{ type: "tool.calling", sessionId: string, tool: string, args: unknown }
{ type: "tool.result", sessionId: string, tool: string, result: unknown }

// Approval requests
{ type: "approval.request", requestId: string, tool: string, args: unknown, description: string }

// Edits
{ type: "edit.proposed", sessionId: string, edits: StructuredEdit[] }

// Errors
{ type: "error", sessionId: string, message: string }
```

### 9.2 Bridge ↔ Gateway

**Transport:** WebSocket on `wss://{gateway-host}/api/ide/ws`

**Authentication:** Token-based (JWT or API key in initial handshake).

**Messages follow the same shape**, relayed between plugin and gateway with the bridge acting as a transparent proxy for most messages, and intercepting `local.*` tool calls for local execution.

### 9.3 Structured Edits

```typescript
interface StructuredEdit {
  path: string;              // relative to workspace root
  type: "create" | "modify" | "delete";
  diff?: string;             // unified diff for modifications
  content?: string;          // full content for creates
}
```

---

## 10. Security

### 10.1 Local Security

- Bridge listens on **localhost only** — no network exposure
- Workspace sandbox — bridge can only access the configured project root
- Read-only tools auto-allowed; write/exec require explicit user approval via IDE UI
- No secrets, credentials, or environment variables sent to remote agent
- Bridge process runs under the user's own OS permissions

### 10.2 Network Security

- Outbound-only connection from laptop to AWS (no inbound ports)
- WSS (TLS) for all bridge ↔ gateway communication
- Authentication required on WebSocket handshake
- Gateway validates session ownership on every message

### 10.3 Data Boundaries

- CA-LTM stores institutional knowledge — never raw code, customer data, or credentials
- File contents sent to the remote agent for reasoning are ephemeral (not persisted beyond the session transcript)
- Promotion pipeline ensures only high-signal, non-sensitive content becomes durable memory

---

## 11. CA-LTM Module Reuse

### 11.1 Reused Directly

| Module | Role in Code Assist |
|--------|-------------------|
| `@ca-ltm/domain` | Shared types for records, SessionState, events |
| `@ca-ltm/storage` | Durable records, SessionState, checkpoints, event log |
| `@ca-ltm/orchestrator` | Recall / working-state / checkpoint / resume flows |
| `@ca-ltm/sourcegraph` | Cross-repo code queries from memory records |
| `@ca-ltm/openclaw-plugin` | Register CA-LTM tools inside OpenClaw |
| `@ca-ltm/api` | Extended with IDE gateway endpoints |
| `@ca-ltm/admin-ui` | Memory inspection, review, audit |
| `@ca-ltm/harness` | Testing the full flow without OpenClaw |

### 11.2 Phased Integration

| Module | Phase | Reason |
|--------|-------|--------|
| `@ca-ltm/retrieval` | Phase 2 | Needs validated storage + classifier first |
| `@ca-ltm/classifier` | Phase 2 | Embedding provider must be tested against real data |
| `@ca-ltm/context-assembler` | Phase 2 | Needs retrieval results to assemble |
| `@ca-ltm/promotion` | Phase 3 | Needs working sessions generating real candidates |
| `@ca-ltm/importer` | Phase 4 | Backfill once the system is stable |

---

## 12. New Components

### 12.1 `@ca-ltm/local-bridge`

**Language:** TypeScript (Node.js)
**Dependencies:** `ws`, `chokidar` (optional file watching), `simple-git`
**Responsibilities:** Local tool execution, WebSocket client, workspace security

### 12.2 `@ca-ltm/jetbrains-plugin`

**Language:** Kotlin (IntelliJ Platform SDK)
**Responsibilities:** Chat UI, editor context, patch preview/apply, approval UX

### 12.3 IDE Gateway Routes (within `@ca-ltm/api`)

**New route group:** `/api/ide/`
**Endpoints:**

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/api/ide/ws` | WebSocket | Main session channel |
| `/api/ide/sessions` | GET | List user's sessions |
| `/api/ide/sessions/:id` | GET | Session detail + metadata |
| `/api/ide/sessions/:id` | DELETE | Close/archive session |

### 12.4 Future: `@ca-ltm/vscode-extension`

Same capabilities as the JetBrains plugin, implemented against VS Code Extension API. Connects to the same bridge daemon.

---

## 13. Implementation Phases

### Phase 1 — MVP: Local Bridge + Gateway + Chat

**Build:**
- `@ca-ltm/local-bridge` — local tool daemon with read-only tools (readFile, listDir, grep, glob, gitStatus, gitDiff, gitLog)
- IDE Gateway — WebSocket endpoint in `@ca-ltm/api`, session create/resume, tool-call routing
- `@ca-ltm/jetbrains-plugin` — chat panel, user message input, streaming response display

**Integrate:**
- OpenClaw on NemoClaw — agent with local tools + Sourcegraph MCP
- No CA-LTM dependency yet

**Result:** A working Amp-like chat assistant running on own infra. Can read local files, search code, query git, and use Sourcegraph. No institutional memory yet.

**Example tasks this enables:**
- A developer selects a function in their IDE, clicks "Explain", and the remote agent reads the file via the local bridge, optionally queries Sourcegraph for cross-repo references, and streams back an explanation — all on CleverAlpha's own infrastructure with no vendor dependency.
- An engineer asks "what calls this endpoint?" The agent uses `local.grep` to find local references and `sourcegraph.search` to find cross-repo callers, combining both into a comprehensive answer.

### Phase 2 — Edits + Write Tools

**Add:**
- `local.writeFile` and `local.exec` tools in bridge (with approval gating)
- Patch preview/apply UX in JetBrains plugin
- Structured edit protocol

**Result:** The assistant can propose and apply code changes, run tests, and execute terminal commands — with user approval on every write.

**Example tasks this enables:**
- An engineer asks "add input validation to this handler." The agent reads the file, proposes a diff, and the IDE shows a side-by-side preview. The engineer clicks "Apply" and the edit is written to disk.
- A developer asks "run the unit tests for this module." The agent calls `local.exec("npm test")` — after the user approves — and analyzes the output, suggesting fixes for any failures.

### Phase 3 — CA-LTM Integration (Storage + Orchestrator)

**Integrate:**
- `@ca-ltm/storage` — record reads, SessionState persistence, checkpoints
- `@ca-ltm/orchestrator` — recall flow (load session, fetch tagged records, assemble basic context)
- `@ca-ltm/openclaw-plugin` — register `ltm.*` tools in OpenClaw

**Prerequisite:** CA-LTM storage and orchestrator modules have been run and tested via `@ca-ltm/harness`.

**Result:** The assistant now has session continuity (resume where you left off) and can query institutional memory (migrations, patterns, decisions). Context assembly is basic (tag-based, no hybrid retrieval).

**Example tasks this enables:**
- A developer asks "what do I need to know about the auth middleware before I change it?" The orchestrator fetches the active authz migration record (with its do/avoid rules) and surfaces it alongside the local code — institutional knowledge grounded in the current codebase.
- An engineer works on a debugging session for 30 minutes, then closes the IDE. The next morning, they reopen the chat and the SessionState is restored — objective, findings, open questions all intact.

### Phase 4 — Hybrid Retrieval + Classification

**Integrate:**
- `@ca-ltm/retrieval` — full hybrid search (keyword + semantic + freshness)
- `@ca-ltm/classifier` — task classification, record embeddings
- `@ca-ltm/context-assembler` — 8-bucket priority assembly with token budgets

**Result:** The assistant surfaces the most relevant institutional memory for each turn, scoped by task type. Semantic search catches matches that keywords miss.

**Example tasks this enables:**
- An engineer asks "what do we know about cross-account access?" The retrieval pipeline finds the IAM AssumeRole pattern (semantic match — different vocabulary, same concept), the recent cross-account S3 incident, and a detection caveat about noisy auth alerts during migration — ranked by relevance and freshness.

### Phase 5 — Promotion + Durable Learning

**Integrate:**
- `@ca-ltm/promotion` — candidate extraction, type classification, create-vs-patch
- `@ca-ltm/admin-ui` — review flow for promotion candidates

**Result:** The assistant learns from sessions. Reusable patterns, incident lessons, and migration updates are proposed as durable records and reviewed by the team.

**Example tasks this enables:**
- During a debugging session, the agent and engineer discover that a specific SSM parameter must be in the target account for cross-account deploys. The promotion pipeline proposes an `incident` record with symptoms, root cause, and fix. A team lead reviews and approves it — the lesson is now institutional memory.

### Phase 6 — Power Features

**Add:**
- `@ca-ltm/importer` — backfill CA-LTM from existing ADRs, playbooks, incident docs
- Verification and staleness workflows
- Citations in chat responses linking to CA-LTM records and Sourcegraph results
- Session history browser in IDE plugin
- VS Code extension (`@ca-ltm/vscode-extension`)

---

## 14. Latency Strategy

To feel responsive for interactive coding:

| Strategy | Detail |
|----------|--------|
| **Persistent WebSocket** | No per-turn HTTP polling. Single long-lived connection. |
| **Small bootstrap context** | Load only top-relevant memory at session start, not full dump. |
| **Active file first** | Send active file + selection immediately. Fetch more on demand. |
| **Sourcegraph for broad search** | Cross-repo discovery via Sourcegraph is faster than crawling local FS for other repos. |
| **Streaming responses** | First token as fast as possible. Background enrichment if needed. |
| **Local tool caching** | Bridge caches file tree, recent reads, git status for repeated queries. |

**Practical targets:**

| Metric | Target |
|--------|--------|
| First token | 1–2 seconds |
| Simple turns | 2–5 seconds |
| Tool-heavy turns | 5–12 seconds |

---

## 15. Forward Considerations

### 15.1 VS Code Extension

Same architecture. Different IDE plugin implementation. Same bridge, same gateway, same remote stack. The bridge's IDE-agnostic protocol makes this a UI-only effort.

### 15.2 Multi-Repo Editing

Out of scope for v1. Would require:
- Bridge supporting multiple workspace roots
- Session metadata tracking multiple repos
- Context assembly aware of cross-repo relationships
- Potentially a different client model (CLI-based or workspace-aware)

### 15.3 Hybrid Local Co-Pilot

Only consider if:
- Remote latency is consistently too slow for interactive editing
- Offline/local-only mode is needed
- Keystroke-level autocomplete is required
- Continuous local background analysis is needed

In that case, add a small local model for autocomplete/quick refactors while keeping remote OpenClaw for deeper reasoning and CA-LTM-grounded decisions.

---

## 16. What Code Assist Does NOT Do

- **Does not run a local agent.** The agent brain is remote (OpenClaw on NemoClaw).
- **Does not index or store code.** Code truth is the local filesystem. Cross-repo search is Sourcegraph.
- **Does not auto-write files.** All writes require user approval.
- **Does not replace Amp/Cursor.** It is purpose-built for CleverAlpha's infrastructure and institutional memory needs.
- **Does not require CA-LTM to function.** MVP works with just OpenClaw + local tools + Sourcegraph.
