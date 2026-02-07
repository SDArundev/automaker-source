# 01 — Project Overview

## What Is Automaker?

An autonomous AI development studio. You describe features on a Kanban board, AI agents (Claude Agent SDK) implement them in isolated git worktrees. Available as a desktop app (Electron) or web app.

**Version:** 0.13.0
**License:** Proprietary (free for internal/personal use, no redistribution)

## Tech Stack

### Frontend
| Technology | Version | Purpose |
|-----------|---------|---------|
| React | 19 | UI framework |
| Vite | 7 | Build tool + dev server |
| Electron | 39 | Desktop app framework |
| TypeScript | 5.9 | Type safety |
| TanStack Router | - | File-based routing |
| Zustand | 5 | State management with persistence |
| Tailwind CSS | 4 | Utility-first styling (40 themes) |
| Radix UI | - | Accessible component primitives |
| dnd-kit | - | Drag-and-drop for Kanban |
| @xyflow/react | - | Graph visualization |
| xterm.js | - | Terminal emulator |
| CodeMirror | 6 | Code/syntax highlighting |

### Backend
| Technology | Version | Purpose |
|-----------|---------|---------|
| Node.js | 22+ | Runtime (ESM) |
| Express | 5 | HTTP server |
| TypeScript | 5.9 | Type safety |
| Claude Agent SDK | 0.1.76 | AI agent execution |
| WebSocket (ws) | 8 | Real-time event streaming |
| node-pty | 1.1.0-beta41 | PTY terminal sessions |
| MCP SDK | 1.25.2 | Model Context Protocol |

### Testing
| Tool | Purpose |
|------|---------|
| Playwright | E2E testing |
| Vitest | Unit testing |
| ESLint 9 | Linting |
| Prettier 3 | Formatting |
| Husky | Pre-commit hooks |

## Prerequisites

| Requirement | Version | Notes |
|-------------|---------|-------|
| Node.js | >=22.0.0 <23.0.0 | Use nvm (`nvm use 22`) |
| npm | any | Bundled with Node.js |
| Claude Code CLI | latest | `npm install -g @anthropic-ai/claude-code` then `claude login` |
| Xcode CLI Tools | latest | macOS only, for `node-pty` compilation |

## Ports

| Port | Service |
|------|---------|
| 3007 | UI (Vite dev server / nginx in production) |
| 3008 | Server (Express API + WebSocket) |

## Environment Variables

### Required (one of)
| Variable | Purpose |
|----------|---------|
| `ANTHROPIC_API_KEY` | Anthropic API key |
| Claude CLI auth | Alternative: `claude login` |

### Optional
| Variable | Default | Purpose |
|----------|---------|---------|
| `PORT` | 3008 | Server port |
| `DATA_DIR` | `./data` | Data storage directory |
| `HOST` | `0.0.0.0` | Server bind host |
| `HOSTNAME` | `localhost` | User-facing hostname |
| `VITE_HOSTNAME` | `localhost` | Frontend API hostname |
| `CORS_ORIGIN` | computed | CORS allowed origins |
| `ALLOWED_ROOT_DIRECTORY` | none | Path sandboxing |
| `AUTOMAKER_API_KEY` | auto-generated | API authentication |
| `AUTOMAKER_AUTO_LOGIN` | `true` (dev) | Skip login in dev |
| `AUTOMAKER_MOCK_AGENT` | `false` | Mock agent for CI |
| `AUTOMAKER_DISABLE_AUTH` | `false` | Disable all auth (dangerous) |
| `TERMINAL_PASSWORD` | none | Terminal access password |
| `IS_CONTAINERIZED` | `false` | Set by Docker compose |
