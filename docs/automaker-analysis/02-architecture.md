# 02 — Architecture

## Monorepo Structure

```
autodev/
├── apps/
│   ├── ui/               # React + Vite + Electron frontend
│   │   ├── src/
│   │   │   ├── routes/          # TanStack Router (22 routes)
│   │   │   ├── components/
│   │   │   │   ├── views/       # Main view components
│   │   │   │   ├── ui/          # 51 base UI components
│   │   │   │   ├── layout/      # Sidebar, ProjectSwitcher
│   │   │   │   └── dialogs/     # Global dialogs
│   │   │   ├── store/           # 5 Zustand stores
│   │   │   ├── hooks/           # 40+ custom hooks
│   │   │   ├── lib/             # API client, query keys
│   │   │   ├── styles/          # Tailwind + 40 theme CSS files
│   │   │   ├── config/          # Theme options, constants
│   │   │   └── utils/           # Router, helpers
│   │   ├── main.ts              # Electron main process (1007 lines)
│   │   ├── preload.ts           # Electron preload/IPC bridge
│   │   └── vite.config.mts      # Vite + Electron + TanStack + Tailwind
│   └── server/           # Express + WebSocket backend
│       └── src/
│           ├── routes/          # ~20 route modules
│           ├── services/        # ~15 service classes
│           ├── providers/       # AI provider abstraction
│           ├── middleware/      # Auth, path validation
│           └── lib/             # Auth, events, SDK options
└── libs/                 # 8 shared packages
    ├── types/            # Domain model (no dependencies)
    ├── platform/         # Paths, security, subprocess, CLI detection
    ├── utils/            # Logging, errors, images, context, atomic I/O
    ├── prompts/          # 40+ AI prompt templates
    ├── model-resolver/   # Model alias → full ID resolution
    ├── dependency-resolver/  # Topological sort for features
    ├── spec-parser/      # XML spec ↔ object conversion
    └── git-utils/        # Diffs, status parsing, repo detection
```

## Package Dependency Chain

Strict layering — packages can only depend on packages above them:

```
@automaker/types                          (no dependencies)
       ↓
@automaker/platform                       (depends on types)
       ↓
@automaker/utils, @automaker/spec-parser  (depend on types, platform)
       ↓
@automaker/prompts                        (depends on types)
@automaker/model-resolver                 (depends on types)
@automaker/dependency-resolver            (depends on types)
       ↓
@automaker/git-utils                      (depends on types, platform)
       ↓
apps/server, apps/ui                      (depend on all libs)
```

Build order enforced by `npm run build:packages`:
```
types → platform → utils + spec-parser → prompts + model-resolver + dependency-resolver → git-utils
```

## Data Flow

```
┌─────────────────────────────────────────────────────────┐
│                     USER BROWSER                         │
│                                                          │
│  React 19 + TanStack Router + Zustand stores            │
│      │                    ▲                              │
│      │ HTTP (REST)        │ WebSocket (events)           │
│      ▼                    │                              │
│  API Client ──────────────┤                              │
│  (http-api-client.ts)     │                              │
└──────────┬────────────────┼──────────────────────────────┘
           │                │
    Vite proxy /api    WS /api/events
           │           WS /api/terminal/ws
           ▼                │
┌──────────────────────────────────────────────────────────┐
│                    EXPRESS SERVER (:3008)                  │
│                                                           │
│  ┌─── Middleware Chain ──────────────────────────────┐    │
│  │ morgan → CORS → JSON parser → cookies → auth      │    │
│  └───────────────────────────────────────────────────┘    │
│                                                           │
│  ┌─── Routes (~160 endpoints) ──────────────────────┐    │
│  │ /api/auth, /api/features, /api/auto-mode,        │    │
│  │ /api/agent, /api/worktree, /api/settings, ...    │    │
│  └───────────────────────────────────────────────────┘    │
│                                                           │
│  ┌─── Services ─────────────────────────────────────┐    │
│  │ AgentService     AutoModeService   FeatureLoader  │    │
│  │ SettingsService  TerminalService   PipelineService│    │
│  │ IdeationService  NotificationService  ...         │    │
│  └──────────┬────────────────────────────────────────┘    │
│             │                                             │
│  ┌─── Providers ────────────────────────────────────┐    │
│  │ ClaudeProvider (SDK) │ CursorProvider (CLI)       │    │
│  │ CodexProvider (CLI)  │ OpencodeProvider (CLI)     │    │
│  └──────────┬────────────────────────────────────────┘    │
│             │                                             │
│  ┌─── EventEmitter ────────────────────────────────┐     │
│  │ Broadcasts all events to WebSocket clients       │     │
│  └──────────────────────────────────────────────────┘     │
└──────────────────────────────────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────────────────────┐
│              AI AGENT (in git worktree)                    │
│                                                           │
│  Claude Agent SDK with tools:                            │
│  Read, Write, Edit, Glob, Grep, Bash,                    │
│  WebSearch, WebFetch, TodoWrite                          │
│                                                           │
│  + MCP servers (configurable)                            │
│  + Skills & Subagents (configurable)                     │
└──────────────────────────────────────────────────────────┘
```

## Key Architectural Patterns

1. **Event-driven architecture** — All operations emit events via shared `EventEmitter`, streamed to UI via WebSocket
2. **Provider pattern** — AI providers (Claude, Cursor, Codex, OpenCode) behind abstract `BaseProvider` with `ProviderFactory` routing
3. **Service-oriented backend** — Each concern is a separate service class
4. **File-based storage** — No database; JSON files in `.automaker/` and `DATA_DIR`
5. **Atomic writes** — Feature data uses write-temp-then-rename with backup rotation
6. **Path sandboxing** — All server file I/O goes through `secure-fs.ts` → `validatePath()`
7. **Dual-mode frontend** — Same React app runs in web browser or Electron, with adapter layer (`http-api-client.ts`)
