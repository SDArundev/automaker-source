# 07 — Security Analysis

## Security Model Overview

Automaker has **defense-in-depth** with multiple layers, but significant gaps exist in the AI agent sandboxing.

```
Layer 1: Docker container boundary     (strongest)
Layer 2: Non-root user (automaker)     (prevents privilege escalation)
Layer 3: ALLOWED_ROOT_DIRECTORY        (Node.js file operations only)
Layer 4: API authentication            (session/key based)
Layer 5: CORS restrictions             (origin validation)
Layer 6: AI agent permissions          (NONE - bypassPermissions)
```

---

## Authentication (`apps/server/src/lib/auth.ts`)

### Methods (Priority Order)
1. `X-API-Key` header (Electron mode)
2. `X-Session-Token` header (web mode)
3. `apiKey` query parameter (fallback)
4. `token` query parameter (image loads)
5. `automaker_session` cookie (web mode)

### API Key
- Auto-generated UUID on first run
- Stored in `{DATA_DIR}/.api-key` with mode 0600
- Timing-safe comparison via `crypto.timingSafeEqual`

### Sessions
- 32 random bytes (hex), 30-day expiry
- Persisted to `{DATA_DIR}/.sessions` with mode 0600
- Cookie: HttpOnly, SameSite=lax, Secure in production

### WebSocket Tokens
- Single-use, 5-minute expiry, in-memory only
- Generated via `POST /api/auth/token`

### Rate Limiting
- `AuthRateLimiter` class exists: 5 attempts/minute/IP
- Applied to `POST /api/auth/login`

### Unauthenticated Endpoints
- `/api/health` — health check
- `/api/auth/*` — login/logout/status
- `/api/setup/*` — CLI detection, API key storage

### Danger: `AUTOMAKER_DISABLE_AUTH`
Setting this env var **completely bypasses all authentication**. Anyone with network access can control the server.

---

## Path Sandboxing

### How It Works
1. `initAllowedPaths(rootDir, dataDir)` called at startup
2. `ALLOWED_ROOT_DIRECTORY` env var sets the boundary
3. All Node.js file I/O goes through `secure-fs.ts` → `validatePath()`
4. `validatePath()` resolves paths and checks against allowed boundaries
5. `isPathWithinDirectory()` uses `path.relative()` to prevent `../` traversal
6. `DATA_DIR` is always allowed (for settings/credentials)

### What It Protects
- Server-side file reads/writes via Express routes
- Feature data access
- Context file operations
- Settings file operations

### What It Does NOT Protect
**AI agent Bash commands.** The agent can run `cat /etc/passwd`, `curl`, or access any file the `automaker` user can read. The SDK runs with `permissionMode: 'bypassPermissions'`.

---

## AI Agent Security — THE CRITICAL GAP

### Configuration
```typescript
// sdk-options.ts
function getBaseOptions() {
  return {
    permissionMode: 'bypassPermissions',
    allowDangerouslySkipPermissions: true,
  };
}
```

### Agent Capabilities
| Capability | Restricted? | Notes |
|-----------|------------|-------|
| Read files | NO | Can read anything `automaker` user can |
| Write files | NO | Can write anywhere `automaker` user can |
| Execute shell commands | NO | Full Bash access |
| Network access | NO | Can make HTTP requests, connect to services |
| Access credentials | NO | Can read `.claude/.credentials.json`, etc. |
| Install software | PARTIALLY | Can npm install, but not apt (no sudo) |
| Git operations | NO | Can push to remotes, create commits |
| Kill processes | PARTIALLY | Limited to `automaker` user's processes |

### What Mitigates This
- Docker container boundary (strongest protection)
- `automaker` user cannot escalate to root
- Named volumes in production prevent host filesystem access
- Network is Docker bridge (isolated from host network)

---

## CORS Configuration

```
If CORS_ORIGIN is set and not '*':
  → Only those origins allowed
If no CORS_ORIGIN:
  → Allow: localhost, 127.0.0.1, ::1, 0.0.0.0
  → Allow: 192.168.*, 10.*, 172.* (private networks)
  → Reject: everything else
No origin (curl, Electron):
  → Always allowed
credentials: true
```

**Issue:** `172.*` prefix check is overly broad — includes non-RFC1918 addresses beyond `172.16.0.0/12`.

---

## Known Vulnerabilities

From `SECURITY_TODO.md`:

### Critical: Command Injection
- `apps/server/src/routes/worktree/routes/merge.ts` — Branch names and commit messages interpolated into shell commands without sanitization
- `apps/server/src/routes/worktree/routes/push.ts` — Remote names interpolated into shell commands
- `start-automaker.sh` — Unsafe `.env` parsing: `export $(grep | xargs)`

**Mitigation:** `execGitCommand()` (safe, array-based) exists but is inconsistently used.

### Other Issues
- `.dockerignore` does not exclude `.env` files (build context leakage)
- OAuth credentials passed via env vars (visible in `docker inspect`)
- `git config --system --add safe.directory '*'` allows git ops on any directory in container
- Rate limiter class exists but may not be wired into all endpoints

---

## Terminal Security

- Full PTY access to container shell via `node-pty`
- Optional password protection via `TERMINAL_PASSWORD` env var
- Input validation: type checking and 1MB size limit
- Resize validation: integer bounds checking
- No command filtering or restriction

---

## Recommendations

### Immediate (No Code Changes)
1. Always run in Docker (production compose)
2. Set strong `AUTOMAKER_API_KEY`
3. Set explicit `CORS_ORIGIN`
4. Set `TERMINAL_PASSWORD` if network-exposed
5. Never set `AUTOMAKER_DISABLE_AUTH`
6. Never mount Docker socket

### Hardening (Code/Config Changes)
```yaml
# docker-compose.yml additions
server:
  cap_drop:
    - ALL
  cap_add:
    - CHOWN
    - SETUID
    - SETGID
    - DAC_OVERRIDE
  security_opt:
    - no-new-privileges:true
  read_only: true
  tmpfs:
    - /tmp
    - /home/automaker/.npm
  pids_limit: 100
```

### Longer Term
- Fix command injection in merge.ts and push.ts (use array-based exec consistently)
- Restrict agent Bash tool to `ALLOWED_ROOT_DIRECTORY` via cgroup/seccomp
- Add network policy to limit outbound connections
- Exclude `.env` from `.dockerignore`
- Fix CORS `172.*` prefix to only allow RFC1918 range
