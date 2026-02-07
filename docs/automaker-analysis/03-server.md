# 03 — Server Deep Dive

## Boot Sequence (`apps/server/src/index.ts`)

1. Load env vars via `dotenv.config()`
2. Parse `PORT` (3008), `HOST` (0.0.0.0), `HOSTNAME` (localhost), `DATA_DIR` (./data)
3. Initialize path security via `initAllowedPaths()`
4. Create Express app with middleware chain:
   - `morgan` (HTTP logging, skips `/api/health`)
   - `cors` (dynamic origin validation)
   - `express.json({ limit: '50mb' })`
   - `cookieParser()`
5. Create shared `EventEmitter` for WebSocket streaming
6. Instantiate services (order matters for dependency injection)
7. Async initialization (migrate settings, create directories)
8. Mount routes (unauthenticated first, then authenticated after `authMiddleware`)
9. Create HTTP server + 2 WebSocket servers (`noServer` mode)
10. Graceful shutdown handlers (SIGTERM, SIGINT)

## Middleware Chain

| Order | Middleware | Purpose |
|-------|-----------|---------|
| 1 | `morgan` | HTTP request logging (toggleable, color-coded) |
| 2 | `cors` | Dynamic origin validation (localhost/LAN/configured) |
| 3 | `express.json` | JSON body parsing (50MB limit) |
| 4 | `cookieParser` | Cookie parsing |
| 5 | `requireJsonContentType` | Anti-CSRF: requires JSON content-type for POST/PUT/PATCH |
| 6 | `authMiddleware` | API key / session token / cookie authentication |
| 7 | `validatePathParams` | Per-route path validation against `ALLOWED_ROOT_DIRECTORY` |

## Services

### AgentService
**File:** `services/agent-service.ts`
**Purpose:** Interactive AI chat sessions (Agent Runner view)

- Session lifecycle: create → start → send messages → stop/clear
- Conversation history persistence to `{DATA_DIR}/agent-sessions/{sessionId}.json`
- Prompt queue: add → auto-process after current message completes
- Provider routing: resolves model → picks Claude/Cursor/Codex/OpenCode
- Context loading: CLAUDE.md, memory files, MCP servers, Skills, Subagents

### AutoModeService
**File:** `services/auto-mode-service.ts` (167KB, the core engine)
**Purpose:** Autonomous feature implementation

Key methods:
- `startAutoLoopForProject()` — background loop polling for pending features
- `executeFeature()` — full execution: worktree → agent → pipeline → status update
- `resumeFeature()` — resume interrupted features
- `followUpFeature()` — send additional instructions to re-run
- `approvePlan()` — resolve pending plan approval
- `commitFeature()` — commit worktree changes
- `verifyFeature()` — run lint/typecheck/test/build

Failure tracking: 3 consecutive failures in 60s → auto-pause. Rate limit → immediate pause.

### FeatureLoader
**File:** `services/feature-loader.ts`
**Purpose:** CRUD for features

- Each feature: `.automaker/features/{featureId}/feature.json`
- Atomic writes with backup rotation for corruption recovery
- Image migration, description history tracking
- Sync completed features to `app_spec.txt`

### TerminalService
**File:** `services/terminal-service.ts`
**Purpose:** PTY sessions via node-pty

- Cross-platform shell detection (macOS/Linux/Windows/WSL)
- Output throttling: 4ms intervals (~250fps), 4KB batches
- 50KB scrollback buffer per session (circular)
- Max 1000 concurrent sessions
- Environment cleaning: removes PORT, DATA_DIR, AUTOMAKER_API_KEY

### SettingsService
**File:** `services/settings-service.ts`
**Purpose:** Global + project settings persistence

- Global: `{DATA_DIR}/settings.json`
- Credentials: `{DATA_DIR}/credentials.json`
- Project: `{projectPath}/.automaker/settings.json`
- Migration from legacy Electron paths

### Other Services

| Service | Purpose |
|---------|---------|
| PipelineService | Custom pipeline step configuration |
| IdeationService | AI brainstorming, idea management, project analysis |
| EventHookService | Execute shell/HTTP webhooks on system events |
| NotificationService | Project notifications with read/unread state |
| EventHistoryService | Event persistence for debugging/replay |
| DevServerService | Manage dev servers for worktrees |
| InitScriptService | Worktree initialization scripts |
| ClaudeUsageService | Claude CLI usage statistics |
| CodexUsageService | Codex CLI usage statistics |
| MCPTestService | Test MCP server connections |

## AI Providers

Registry pattern with priority-based routing:

| Provider | Priority | Method | Models |
|----------|----------|--------|--------|
| CursorProvider | 10 | CLI | 21 Cursor models |
| CodexProvider | 5 | CLI/SaaS | GPT models, o-series |
| OpencodeProvider | 3 | CLI | 5 free models + dynamic |
| ClaudeProvider | 0 (default) | SDK | Opus 4.5, Sonnet 4, 3.5 Sonnet, Haiku 4.5 |

### ClaudeProvider Details
- Uses `@anthropic-ai/claude-agent-sdk` `query()` function
- **Always runs `permissionMode: 'bypassPermissions'`** (fully autonomous)
- Environment allowlist: ANTHROPIC_API_KEY, PATH, HOME, USER, etc.
- Supports: MCP servers, thinking tokens, subagents, session resume
- API key resolution: inline → env → credentials sources

## WebSocket Architecture

Two WebSocket servers on the same HTTP server:

### Events WebSocket (`/api/events`)
- Broadcast model: every client gets ALL events
- JSON format: `{ type: EventType, payload: unknown }`
- 50+ event types covering agent, auto-mode, features, planning, notifications
- Used for: real-time agent streaming, auto-mode progress, feature updates

### Terminal WebSocket (`/api/terminal/ws`)
- Per-session connections (`?sessionId=`)
- Client → Server: `input`, `resize`, `ping`
- Server → Client: `connected`, `scrollback`, `data`, `exit`, `pong`, `error`
- Rate-limited resize (100ms minimum interval)
- Scrollback buffer sent on connect before subscribing to live data

### WebSocket Authentication
Same as HTTP: API key header, session token, cookie, or single-use wsToken (5-min TTL)

## Agent Execution Flow

When a feature moves to "In Progress":

```
1. Validate feature not already running
2. Load feature from .automaker/features/{id}/feature.json
3. Check for existing context (resume vs fresh start)
4. Check for approved plan (skip planning if so)
5. Find/create git worktree for feature's branch
6. Update status to in_progress
7. Load context: CLAUDE.md, memory files, custom context
8. Build prompt: planning prefix + feature description + images
9. Resolve model → route to provider (Claude/Cursor/Codex/OpenCode)
10. Build SDK options: tools, thinking tokens, MCP servers
11. Run agent (streams ProviderMessage events)
    ├── If planning mode: pause for approval
    ├── Agent output saved to agent-output.md + raw-output.jsonl
    └── Progress events streamed to frontend via WebSocket
12. Execute pipeline steps (if configured)
13. Set final status: verified or waiting_approval
14. Record learnings and memory usage
15. Emit auto_mode_feature_complete
```

**Agent tools:** Read, Write, Edit, Glob, Grep, Bash, WebSearch, WebFetch, TodoWrite
**Max turns:** 1000 (auto-mode), 100 (chat), 250 (suggestions), 50 (quick ops)

## Error Handling

- Regex-based error classification: AUTHENTICATION, BILLING, RATE_LIMIT, NETWORK, TIMEOUT, VALIDATION, PERMISSION, CLI_NOT_FOUND, MODEL_NOT_SUPPORTED, SERVER_ERROR, UNKNOWN
- `createRetryHandler(maxRetries=3, baseDelay=1000)` with exponential backoff + jitter
- Auto-mode failure tracking with auto-pause
- `unhandledRejection`: log but continue
- `uncaughtException`: log and exit (process.exit(1))
