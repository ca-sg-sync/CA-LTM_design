# @ca-ltm/promotion — Module Specification

> **Version:** 0.1.0-draft
> **Phase:** 7 (manual) → 10 (auto-helpers)

---

## 1. Module Identity

| Field | Value |
|-------|-------|
| **Package** | `@ca-ltm/promotion` |
| **Role** | Session → durable memory promotion logic |
| **Dependencies** | `@ca-ltm/domain`, `@ca-ltm/storage` |
| **Dependents** | `@ca-ltm/orchestrator` |

---

## 2. Responsibilities

- Candidate extraction from session state and responses
- Record type classification (what type should a candidate become?)
- Create vs patch decisions (new record or update existing?)
- Conflict detection (duplicates, contradictions)
- Suggestion/proposal generation
- Review payload generation
- Promotion threshold enforcement
- Auto-promote criteria evaluation

---

## 3. Promotion Pipeline

```
SessionState + Recent Response
  → extractCandidates()         // identify promotable findings
  → classifyCandidate()         // assign RecordType
  → findExistingMatch()         // check for duplicates/updates
  → decideAction()              // create | patch | skip
  → generateProposal()          // build full proposal
  → shouldAutoPromote()         // auto or review-required?
```

---

## 4. Public API

```typescript
interface PromotionService {
  extractCandidates(input: PromotionInput): Promise<PromotionCandidate[]>;

  classifyCandidate(candidate: PromotionCandidate): RecordType;

  findExistingMatch(candidate: PromotionCandidate): Promise<AnyRecord | null>;

  decideAction(
    candidate: PromotionCandidate,
    existing?: AnyRecord
  ): PromotionAction;

  generateProposal(candidate: PromotionCandidate): RecordProposal;

  shouldAutoPromote(candidate: PromotionCandidate): boolean;
}

function createPromotionService(
  storage: StorageService,
  config?: PromotionConfig
): PromotionService;
```

### Configuration

```typescript
interface PromotionConfig {
  autoPromoteMinConfidence?: Confidence;   // default: "high"
  reviewRequiredByDefault?: boolean;       // default: true
  maxCandidatesPerCheckpoint?: number;     // default: 5
  duplicateThreshold?: number;             // 0-1, title similarity
}
```

### Types (from `@ca-ltm/domain`)

```typescript
interface PromotionInput {
  sessionState: SessionState;
  recentFindings?: string[];
  recentResponse?: string;
  existingRecords?: AnyRecord[];
}

interface PromotionCandidate {
  id: string;
  suggestedType: RecordType;
  title: string;
  summary: string;
  confidence: Confidence;
  evidence?: string[];
  sourceSessionId: string;
  extractedAt: string;
}

type PromotionAction =
  | { action: "create"; record: Partial<BaseRecord> }
  | { action: "patch"; recordId: string; patch: Partial<BaseRecord> }
  | { action: "skip"; reason: string };

interface RecordProposal {
  candidate: PromotionCandidate;
  action: PromotionAction;
  reviewRequired: boolean;
  conflictsWith?: string[];
}
```

---

## 5. Promotion Rules

### What Qualifies for Promotion

| Category | Example |
|----------|---------|
| Migration update | New blocker discovered, milestone reached, rule clarified |
| Stable pattern | "This is how we do cross-account access" |
| Debugging lesson | Root cause + fix that will recur |
| Architecture decision | "We chose X over Y because Z" |
| Domain term | "BD_ACCOUNT means broker-dealer handled account" |
| System caveat | "This service looks independent but depends on X" |
| Playbook | Repeatable investigation/response sequence |
| Alert pattern | "A + B + C usually means token issuer degradation" |
| Detection caveat | "During migration X, this alert is noisy" |

### Promotion Thresholds

A candidate must satisfy **at least 3** of:

1. Likely useful again (not just task-local)
2. Stable enough to survive weeks/months
3. Tied to evidence (session, Sourcegraph, docs)
4. Understandable out of context
5. Not redundant with existing records

### Promotion Modes

| Mode | Risk | When |
|------|------|------|
| Auto-store as `session_note` | Low | Always (default) |
| Auto-promote to durable | Medium | Only when confidence = "high" AND `reviewRequiredByDefault = false` |
| Review-required durable | Low | Default for early versions |

---

## 6. Candidate Extraction

### From SessionState

```typescript
function extractFromSession(state: SessionState): PromotionCandidate[] {
  const candidates: PromotionCandidate[] = [];

  // Findings that look like patterns, decisions, or incidents
  for (const finding of state.findings ?? []) {
    if (looksLikePattern(finding)) candidates.push(buildCandidate("pattern", finding));
    if (looksLikeDecision(finding)) candidates.push(buildCandidate("decision", finding));
    if (looksLikeIncident(finding)) candidates.push(buildCandidate("incident", finding));
    if (looksLikeMigrationUpdate(finding)) candidates.push(buildCandidate("migration", finding));
    if (looksLikeGlossary(finding)) candidates.push(buildCandidate("glossary", finding));
  }

  return candidates;
}
```

### Classification Signals

| Signal | Suggested Type |
|--------|---------------|
| "we always do X", "the pattern is", "the approved way" | `pattern` |
| "we decided to", "we chose", "the rationale" | `decision` |
| "the root cause was", "the fix was", "this broke because" | `incident` |
| "during the migration", "the old way", "the new way" | `migration` |
| "X means", "X refers to", "the definition of" | `glossary` |
| "when this happens, check", "first verify", "then escalate" | `playbook` |
| "this alert fires when", "A + B usually means" | `alert_pattern` |
| "ignore this signal during", "this metric is misleading" | `detection_caveat` |

---

## 7. Conflict Detection

```typescript
async function findExistingMatch(
  candidate: PromotionCandidate
): Promise<AnyRecord | null> {
  // 1. Search by exact title match within same type
  // 2. Search by related systems overlap + keyword similarity
  // 3. If similarity > duplicateThreshold, return match
  // 4. Otherwise return null (new record)
}
```

### Action Decision

```typescript
function decideAction(
  candidate: PromotionCandidate,
  existing?: AnyRecord
): PromotionAction {
  if (!existing) {
    return { action: "create", record: candidateToRecord(candidate) };
  }

  if (isSubsetOf(candidate, existing)) {
    return { action: "skip", reason: "already covered by existing record" };
  }

  return {
    action: "patch",
    recordId: existing.id,
    patch: computePatch(existing, candidate),
  };
}
```

---

## 8. Internal Structure

```
src/
  index.ts                    # barrel exports: createPromotionService
  service.ts                  # PromotionService implementation
  extraction/
    extractor.ts              # Candidate extraction from session
    classifier.ts             # Record type classification
    signals.ts                # Pattern/decision/incident signal detection
  matching/
    matcher.ts                # Find existing record matches
    conflict.ts               # Conflict/duplicate detection
    similarity.ts             # Title/content similarity scoring
  decisions/
    action.ts                 # Create vs patch vs skip
    thresholds.ts             # Promotion threshold rules
    auto-promote.ts           # Auto-promote criteria
  proposals/
    generator.ts              # Record proposal builder
    review.ts                 # Review payload generation
  config.ts                   # PromotionConfig defaults
```

---

## 9. What It Does NOT Do

- Does not persist records directly (returns proposals; orchestrator writes via storage)
- Does not assemble context
- Does not classify tasks
- Does not interact with Sourcegraph
- Does not manage sessions

---

## 10. Test Criteria

- Candidates are correctly extracted from session findings
- Classification assigns correct record types for known patterns
- `findExistingMatch` returns existing record when title/systems overlap
- `findExistingMatch` returns null for genuinely new knowledge
- `decideAction` returns "create" for new records
- `decideAction` returns "patch" for existing records with new information
- `decideAction` returns "skip" for redundant candidates
- `shouldAutoPromote` only returns true for high-confidence
- `shouldAutoPromote` returns false when `reviewRequiredByDefault = true`
- Review payloads contain sufficient context for human reviewer
- Conflict detection catches obvious duplicates
- `maxCandidatesPerCheckpoint` is respected
- Edge cases: empty session, no findings, all low-confidence, all duplicates
