# 10 — API Reference

Complete REST API endpoint catalog. All endpoints are at `http://localhost:3008` unless noted.

---

## Authentication Endpoints (Unauthenticated)

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/api/auth/status` | Check auth status; auto-login if `AUTOMAKER_AUTO_LOGIN=true` |
| POST | `/api/auth/login` | Validate API key, create session (rate limited: 5/min/IP) |
| GET | `/api/auth/token` | Generate single-use WebSocket token (5-min TTL) |
| POST | `/api/auth/logout` | Invalidate session, clear cookie |

## Health (Unauthenticated)

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/api/health` | Basic health check |
| GET | `/api/health/environment` | Containerization info |
| GET | `/api/health/detailed` | **Authenticated** — Detailed health with provider status |

## Setup (Unauthenticated)

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/api/setup/claude-status` | Claude CLI installation/auth status |
| POST | `/api/setup/install-claude` | Install Claude CLI |
| POST | `/api/setup/auth-claude` | Authenticate Claude CLI |
| POST | `/api/setup/deauth-claude` | Deauthenticate |
| POST | `/api/setup/store-api-key` | Store API key |
| POST | `/api/setup/delete-api-key` | Delete API key |
| GET | `/api/setup/api-keys` | List stored API keys |
| GET | `/api/setup/platform` | Platform info |
| POST | `/api/setup/verify-claude-auth` | Verify Claude auth |
| POST | `/api/setup/verify-codex-auth` | Verify Codex auth |
| GET | `/api/setup/gh-status` | GitHub CLI status |
| GET | `/api/setup/cursor-status` | Cursor CLI status |
| POST | `/api/setup/auth-cursor` | Auth Cursor |
| POST | `/api/setup/deauth-cursor` | Deauth Cursor |
| GET | `/api/setup/codex-status` | Codex CLI status |
| POST | `/api/setup/install-codex` | Install Codex |
| POST | `/api/setup/auth-codex` | Auth Codex |
| POST | `/api/setup/deauth-codex` | Deauth Codex |
| GET | `/api/setup/opencode-status` | OpenCode CLI status |
| POST | `/api/setup/auth-opencode` | Auth OpenCode |
| POST | `/api/setup/deauth-opencode` | Deauth OpenCode |
| GET | `/api/setup/opencode/models` | OpenCode models |
| POST | `/api/setup/opencode/models/refresh` | Refresh models |
| GET | `/api/setup/opencode/providers` | OpenCode providers |
| POST | `/api/setup/opencode/cache/clear` | Clear cache |
| GET | `/api/setup/cursor-config` | Cursor config |
| POST | `/api/setup/cursor-config/default-model` | Set Cursor default |
| POST | `/api/setup/cursor-config/models` | Set Cursor models |
| GET | `/api/setup/cursor-permissions` | Cursor permissions |
| POST | `/api/setup/cursor-permissions/profile` | Apply permission profile |
| POST | `/api/setup/cursor-permissions/custom` | Custom permissions |
| DELETE | `/api/setup/cursor-permissions` | Delete permissions |

---

## Authenticated Endpoints

### Agent (`/api/agent`)

| Method | Path | Purpose |
|--------|------|---------|
| POST | `/api/agent/start` | Start/resume agent session |
| POST | `/api/agent/send` | Send message, stream response |
| POST | `/api/agent/history` | Get conversation history |
| POST | `/api/agent/stop` | Stop current execution |
| POST | `/api/agent/clear` | Clear history |
| POST | `/api/agent/model` | Set session model |
| POST | `/api/agent/queue/add` | Add to prompt queue |
| POST | `/api/agent/queue/list` | List queued prompts |
| POST | `/api/agent/queue/remove` | Remove from queue |
| POST | `/api/agent/queue/clear` | Clear queue |

### Sessions (`/api/sessions`)

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/api/sessions/` | List all sessions |
| POST | `/api/sessions/` | Create session |
| PUT | `/api/sessions/:id` | Update metadata |
| POST | `/api/sessions/:id/archive` | Archive |
| POST | `/api/sessions/:id/unarchive` | Unarchive |
| DELETE | `/api/sessions/:id` | Delete |

### Features (`/api/features`)

| Method | Path | Purpose |
|--------|------|---------|
| POST | `/api/features/list` | List features for project |
| POST | `/api/features/get` | Get single feature |
| POST | `/api/features/create` | Create feature |
| POST | `/api/features/update` | Update feature |
| POST | `/api/features/bulk-update` | Bulk update |
| POST | `/api/features/bulk-delete` | Bulk delete |
| POST | `/api/features/delete` | Delete feature |
| POST | `/api/features/agent-output` | Get agent output |
| POST | `/api/features/raw-output` | Get raw JSONL output |
| POST | `/api/features/generate-title` | AI-generate title |

### Auto Mode (`/api/auto-mode`)

| Method | Path | Purpose |
|--------|------|---------|
| POST | `/api/auto-mode/start` | Start auto loop |
| POST | `/api/auto-mode/stop` | Stop auto loop |
| POST | `/api/auto-mode/stop-feature` | Stop specific feature |
| POST | `/api/auto-mode/status` | Get status |
| POST | `/api/auto-mode/run-feature` | Run single feature |
| POST | `/api/auto-mode/verify-feature` | Verify feature |
| POST | `/api/auto-mode/resume-feature` | Resume interrupted |
| POST | `/api/auto-mode/context-exists` | Check existing context |
| POST | `/api/auto-mode/analyze-project` | Analyze project |
| POST | `/api/auto-mode/follow-up-feature` | Send follow-up instructions |
| POST | `/api/auto-mode/commit-feature` | Commit changes |
| POST | `/api/auto-mode/approve-plan` | Approve/reject plan |
| POST | `/api/auto-mode/resume-interrupted` | Resume after restart |

### Worktree (`/api/worktree`)

| Method | Path | Purpose |
|--------|------|---------|
| POST | `/api/worktree/info` | Worktree info |
| POST | `/api/worktree/status` | Git status |
| POST | `/api/worktree/list` | List worktrees |
| POST | `/api/worktree/diffs` | Get diffs |
| POST | `/api/worktree/file-diff` | File-specific diff |
| POST | `/api/worktree/merge` | Merge branch |
| POST | `/api/worktree/create` | Create worktree |
| POST | `/api/worktree/delete` | Delete worktree |
| POST | `/api/worktree/create-pr` | Create pull request |
| POST | `/api/worktree/pr-info` | PR info |
| POST | `/api/worktree/commit` | Commit changes |
| POST | `/api/worktree/generate-commit-message` | AI commit message |
| POST | `/api/worktree/push` | Git push |
| POST | `/api/worktree/pull` | Git pull |
| POST | `/api/worktree/checkout-branch` | Checkout branch |
| POST | `/api/worktree/list-branches` | List branches |
| POST | `/api/worktree/switch-branch` | Switch branch |
| POST | `/api/worktree/open-in-editor` | Open in editor |
| POST | `/api/worktree/open-in-terminal` | Open in terminal |
| GET | `/api/worktree/default-editor` | Default editor |
| GET | `/api/worktree/available-editors` | Available editors |
| POST | `/api/worktree/refresh-editors` | Refresh editors |
| GET | `/api/worktree/available-terminals` | Available terminals |
| GET | `/api/worktree/default-terminal` | Default terminal |
| POST | `/api/worktree/refresh-terminals` | Refresh terminals |
| POST | `/api/worktree/open-in-external-terminal` | External terminal |
| POST | `/api/worktree/init-git` | Init git repo |
| POST | `/api/worktree/migrate` | Migrate format |
| POST | `/api/worktree/start-dev` | Start dev server |
| POST | `/api/worktree/stop-dev` | Stop dev server |
| POST | `/api/worktree/list-dev-servers` | List dev servers |
| GET | `/api/worktree/dev-server-logs` | Dev server logs |
| GET | `/api/worktree/init-script` | Get init script |
| PUT | `/api/worktree/init-script` | Save init script |
| DELETE | `/api/worktree/init-script` | Delete init script |
| POST | `/api/worktree/run-init-script` | Run init script |
| POST | `/api/worktree/discard-changes` | Discard all changes |
| POST | `/api/worktree/list-remotes` | List remotes |

### File System (`/api/fs`)

| Method | Path | Purpose |
|--------|------|---------|
| POST | `/api/fs/read` | Read file |
| POST | `/api/fs/write` | Write file |
| POST | `/api/fs/mkdir` | Create directory |
| POST | `/api/fs/readdir` | List directory |
| POST | `/api/fs/exists` | Check exists |
| POST | `/api/fs/stat` | File stats |
| POST | `/api/fs/delete` | Delete file/dir |
| POST | `/api/fs/validate-path` | Validate path |
| POST | `/api/fs/resolve-directory` | Resolve directory |
| POST | `/api/fs/save-image` | Save image |
| POST | `/api/fs/browse` | Browse filesystem |
| GET | `/api/fs/image` | Serve image |
| POST | `/api/fs/save-board-background` | Save background |
| POST | `/api/fs/delete-board-background` | Delete background |

### Settings (`/api/settings`)

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/api/settings/status` | Migration status |
| GET | `/api/settings/global` | Get global settings |
| PUT | `/api/settings/global` | Update global |
| GET | `/api/settings/credentials` | Get masked creds |
| PUT | `/api/settings/credentials` | Update creds |
| POST | `/api/settings/project` | Get project settings |
| PUT | `/api/settings/project` | Update project |
| POST | `/api/settings/migrate` | Migrate localStorage |
| POST | `/api/settings/agents/discover` | Discover agents |

### Terminal (`/api/terminal`)

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/api/terminal/status` | Status (enabled, password needed) |
| POST | `/api/terminal/auth` | Terminal auth |
| POST | `/api/terminal/logout` | Terminal logout |
| GET | `/api/terminal/sessions` | List sessions |
| POST | `/api/terminal/sessions` | Create session |
| DELETE | `/api/terminal/sessions/:id` | Delete session |
| POST | `/api/terminal/sessions/:id/resize` | Resize |
| GET | `/api/terminal/settings` | Terminal settings |
| PUT | `/api/terminal/settings` | Update settings |

### GitHub (`/api/github`)

| Method | Path | Purpose |
|--------|------|---------|
| POST | `/api/github/check-remote` | Check GitHub remote |
| POST | `/api/github/issues` | List issues |
| POST | `/api/github/prs` | List PRs |
| POST | `/api/github/issue-comments` | List comments |
| POST | `/api/github/validate-issue` | AI validate issue |
| POST | `/api/github/validation-status` | Validation status |
| POST | `/api/github/validation-stop` | Stop validation |
| POST | `/api/github/validations` | Get all validations |
| POST | `/api/github/validation-delete` | Delete validation |
| POST | `/api/github/validation-mark-viewed` | Mark viewed |

### Other Endpoints

| Method | Path | Purpose |
|--------|------|---------|
| POST | `/api/spec-regeneration/create` | Create app spec |
| POST | `/api/spec-regeneration/generate` | Generate spec (AI) |
| POST | `/api/spec-regeneration/generate-features` | Features from spec |
| POST | `/api/suggestions/generate` | Generate suggestions |
| POST | `/api/backlog-plan/generate` | Generate backlog plan |
| POST | `/api/backlog-plan/apply` | Apply plan |
| GET | `/api/models/available` | All available models |
| GET | `/api/models/providers` | Provider statuses |
| GET | `/api/running-agents/` | Active agents |
| GET | `/api/workspace/config` | Workspace config |
| POST | `/api/templates/clone` | Clone starter template |
| GET | `/api/claude/usage` | Claude usage stats |
| GET | `/api/codex/usage` | Codex usage stats |
| GET | `/api/codex/models` | Codex models |
| POST | `/api/enhance-prompt/` | AI enhance prompt |
| POST | `/api/context/describe-image` | AI describe image |
| POST | `/api/context/describe-file` | AI describe file |
| POST | `/api/mcp/test` | Test MCP server |
| POST | `/api/mcp/tools` | List MCP tools |
| POST | `/api/pipeline/config` | Get pipeline config |
| POST | `/api/pipeline/config/save` | Save pipeline |
| POST | `/api/ideation/session/start` | Start brainstorm |
| POST | `/api/ideation/session/message` | Send message |
| POST | `/api/ideation/analyze` | Analyze project |
| POST | `/api/ideation/convert` | Idea → feature |
| POST | `/api/notifications/list` | List notifications |
| POST | `/api/event-history/list` | List events |
| POST | `/api/event-history/replay` | Replay event |

---

## WebSocket Endpoints

| Path | Purpose | Auth |
|------|---------|------|
| `/api/events` | Real-time event streaming (broadcast) | Same as HTTP |
| `/api/terminal/ws?sessionId=X` | Terminal PTY I/O | Terminal auth token |
