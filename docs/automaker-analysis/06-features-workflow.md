# 06 — Features & Workflow

## Complete Feature Lifecycle

### 1. Feature Creation

Created via `POST /api/features/create` → `FeatureLoader.create()`.

**Feature fields:**

| Field | Type | Purpose |
|-------|------|---------|
| id | string | Auto-generated: `feature-{timestamp}-{random}` |
| title | string | Display name (can be AI-generated) |
| description | string | Full feature specification/prompt |
| status | string | Lifecycle state |
| priority | number | Execution order priority |
| dependencies | string[] | Feature IDs that must complete first |
| model | string | AI model override |
| imagePaths | array | Attached screenshots/mockups |
| textFilePaths | array | Attached text files |
| branchName | string | Git branch for worktree isolation |
| skipTests | boolean | Skip automated verification |
| thinkingLevel | ThinkingLevel | Extended thinking: none/low/medium/high/ultrathink |
| planningMode | PlanningMode | skip/lite/spec/full |
| requirePlanApproval | boolean | User must approve plan |
| planSpec | PlanSpec | Plan state, content, tasks |

### 2. Status Transitions

```
backlog/pending/ready
    │
    ▼ (drag to In Progress, or Auto Mode picks it)
in_progress
    │
    ├──▶ pipeline_{stepId}  (if custom pipeline steps exist)
    │         │
    │         ▼
    ├──▶ waiting_approval   (if skipTests=true, needs review)
    │         │
    │         ├── approve ──▶ verified
    │         └── reject  ──▶ backlog (with feedback)
    │
    └──▶ verified           (if skipTests=false, auto-verified)
              │
              ▼
         completed          (after commit/merge)

On error: in_progress ──▶ backlog (allows retry)
```

### 3. Auto Mode Feature Selection

`loadPendingFeatures()` process:
1. Load ALL features from `.automaker/features/`
2. Filter to pending/ready/backlog status (or approved plans with incomplete tasks)
3. Filter by worktree/branch scope
4. Apply dependency-aware ordering via `resolveDependencies()` (topological sort)
5. Auto-remove missing dependencies (deleted features)
6. Check `areDependenciesSatisfied()` — deps must be completed/verified
7. Return ordered list of ready features

### 4. Agent Execution

See `executeFeature()` in AutoModeService:

1. Add to `runningFeatures` map (prevents double-execution)
2. Load feature, check for existing context or approved plan
3. Derive working directory from worktree
4. Set status to `in_progress`
5. Load context: CLAUDE.md, memory files, custom context
6. Build prompt: planning prefix + feature description + images
7. Resolve model → route to provider
8. Run agent via Claude SDK (or Cursor/Codex/OpenCode CLI)
9. Execute pipeline steps (if configured)
10. Set final status
11. Record learnings

### 5. Verification

`verifyFeature()` runs in sequence:
1. `npm run lint`
2. `npm run typecheck`
3. `npm run test`
4. `npm run build`

Each check runs in the worktree directory. Any failure → verification fails.

### 6. Commit & PR

- **Commit:** Stages all changes (`git add -A`), commits with `feat: {title}`
- **PR:** Uses `gh pr create` CLI command
- **Merge:** Merges worktree branch into target

---

## Planning Modes

### Skip (Default)
No planning phase. Agent goes directly to implementation.

### Lite
Quick outline before implementation. Agent writes a brief plan, then implements.
With approval: generates plan → pauses → user approves/rejects → continues.

### Spec
Detailed task breakdown. Generates structured tasks:
```
```tasks
## Phase 1: Foundation
- [ ] T001: Create user model | File: src/models/user.ts
- [ ] T002: Add validation | File: src/validators/user.ts
```
```
Tasks executed sequentially. Progress tracked via `planSpec.tasksCompleted/tasksTotal`.

### Full
Comprehensive Software Design Document (SDD). Phased execution plan with detailed specifications. Most thorough but slowest.

### Plan Approval Flow

1. Agent generates plan based on planning mode
2. `planSpec.status` → `'generated'`
3. Execution pauses via Promise stored in `pendingApprovals` map
4. Event emitted: `planning_complete`
5. Frontend shows plan to user
6. User calls `POST /api/auto-mode/approve-plan`
7. Options: approve (optionally edit), reject (with feedback)
8. 30-minute timeout

---

## Auto Mode

### How It Works
Background loop (`runAutoLoopForProject()`) that polls every 2-5 seconds:
1. Check capacity (running count vs `maxConcurrency`)
2. Load pending features (filtered by branch, dependency-ordered)
3. Pick next ready feature
4. Start execution in background
5. Emit `auto_mode_idle` when all complete

### Configuration
- Global `maxConcurrency` (default: 1)
- Per-worktree concurrency via `autoModeByWorktree` settings
- Key format: `"${projectId}::${branchName ?? '__main__'}"`

### Failure Handling
- 3 consecutive failures within 60 seconds → auto-pause
- Rate limit / quota errors → immediate pause
- Event: `auto_mode_paused_failures`
- Manual restart resets failure tracking

### Execution State Recovery
Saved to `.automaker/execution-state.json`:
```json
{
  "version": 1,
  "autoLoopWasRunning": true,
  "maxConcurrency": 3,
  "projectPath": "/path/to/project",
  "branchName": null,
  "runningFeatureIds": ["feature-123"],
  "savedAt": "2024-01-01T00:00:00Z"
}
```
On server restart, `resumeInterrupted()` can restore auto-mode state.

---

## Agent Chat System

### Session Lifecycle
1. `createSession(name, projectPath, workingDirectory, model)`
2. `startConversation({sessionId, workingDirectory})`
3. `sendMessage({sessionId, message, imagePaths, model, thinkingLevel})`
4. Messages streamed via WebSocket events

### Persistence
- Metadata: `{DATA_DIR}/sessions-metadata.json`
- Messages: `{DATA_DIR}/agent-sessions/{sessionId}.json`
- Queue: `{DATA_DIR}/agent-sessions/{sessionId}-queue.json`

### Chat vs Feature Differences

| Aspect | Chat (AgentService) | Feature (AutoModeService) |
|--------|---------------------|---------------------------|
| System prompt | `prompts.agent.systemPrompt` | Context files as system content |
| History | Full conversation chain | No history (independent) |
| Session resume | SDK `sdkSessionId` | No resume |
| Planning | None | Based on planningMode |
| Images | User-attached | Feature-attached |

### Prompt Queue
Add prompts to auto-execute sequentially:
- `addToQueue()` → `processNextInQueue()` after current completes
- Queue persisted to disk for durability

---

## Context System

### Context Files (`.automaker/context/`)
- Markdown/text files added by user
- Metadata in `context-metadata.json` (name, description per file)
- Formatted with headers showing purpose

### Memory Files (`.automaker/memory/`)
YAML frontmatter format:
```yaml
---
tags: [auth, jwt, security]
summary: "JWT refresh token pattern"
importance: 0.8
usageStats:
  loaded: 5
  referenced: 3
  successfulFeatures: 2
---
# Learning content here...
```

Smart selection: tag matching + usage scores + importance ratings. Max 5 files by default.

### What Gets Fed to Agents
1. CLAUDE.md (via SDK `settingSources` if `autoLoadClaudeMd=true`)
2. Context files from `.automaker/context/`
3. Selected memory files (relevance-matched)
4. Feature description + images
5. Planning prefix (if applicable)

---

## Pipeline System

Custom steps between "in_progress" and "waiting_approval":

```json
{
  "steps": [
    { "id": "review", "name": "Code Review", "order": 1, "instructions": "..." },
    { "id": "test", "name": "Integration Tests", "order": 2, "instructions": "..." }
  ]
}
```

Feature status becomes `pipeline_{stepId}` during each step. Steps executed sequentially, each gets its own agent run.

---

## Other Systems

### AI Profiles (Claude-Compatible Providers)
Custom providers with models:
- Type: anthropic, glm, minimax, openrouter, custom
- Each exposes models[] to all dropdowns
- Per-phase model config: 12 phases (enhancement, validation, spec generation, etc.)

### GitHub Integration
- Import issues/PRs via `gh` CLI
- AI-powered issue validation (verdict, confidence, complexity, related files)
- PR creation from worktrees

### Ideation
- AI brainstorming sessions with streaming
- Guided prompts by category
- Project analysis for suggestions
- Ideas → features conversion

### Event Hooks
Triggers: feature_created, feature_success, feature_error, auto_mode_complete, auto_mode_error
Actions: shell commands or HTTP webhooks with variable substitution

### Notifications
Types: feature_waiting_approval, feature_verified, spec_regeneration_complete, agent_complete
Per-project with read/dismiss tracking.
