# 08 — Docker Setup

## Docker Files

| File | Purpose |
|------|---------|
| `Dockerfile` | Multi-stage production build (5 stages) |
| `Dockerfile.dev` | Development image (single stage, volume mount) |
| `docker-compose.yml` | Production compose (fully isolated) |
| `docker-compose.dev.yml` | Development compose (source mount + live reload) |
| `docker-compose.dev-server.yml` | Server-only dev (for local Electron frontend) |
| `docker-compose.override.yml.example` | Override template for project mounts |
| `docker-entrypoint.sh` | Permission fixing + user switching |
| `.dockerignore` | Build context exclusions |

---

## Dockerfile (Production, 5 Stages)

### Stage 1: `base`
- `node:22-slim` (Debian)
- Installs build deps: python3, make, g++ (for node-pty native compilation)

### Stage 2: `server-builder`
- `npm ci --ignore-scripts && npm rebuild node-pty`
- Builds all packages and server

### Stage 3: `server` (production)
- Fresh `node:22-slim` (build deps NOT carried over)
- Installs: git, curl, bash, gosu, ca-certificates, openssh-client
- GitHub CLI (gh) v2.63.2, pinned, multi-arch
- Playwright/Chromium system dependencies (27+ libs)
- Claude CLI (global), Cursor CLI + OpenCode CLI (as automaker user)
- Non-root user `automaker` with configurable UID/GID (default 1001:1001)
- `HEALTHCHECK: curl -f http://localhost:3008/api/health`

### Stage 4: `ui-builder`
- Builds UI with Vite (`VITE_SKIP_ELECTRON=true`)

### Stage 5: `ui`
- `nginx:alpine` serving static files
- Custom nginx.conf for SPA routing

---

## docker-compose.yml (Production)

```yaml
services:
  ui:
    build: { target: ui }
    ports: ["3007:80"]
    depends_on: [server]

  server:
    build:
      target: server
      args:
        UID: ${UID:-1001}
        GID: ${GID:-1001}
    ports: ["3008:3008"]
    environment:
      - ANTHROPIC_API_KEY
      - CLAUDE_OAUTH_CREDENTIALS
      - CURSOR_AUTH_TOKEN
      - AUTOMAKER_API_KEY
      - ALLOWED_ROOT_DIRECTORY=/projects
      - DATA_DIR=/data
      - CORS_ORIGIN=http://localhost:3007
      - IS_CONTAINERIZED=true
    volumes:
      - automaker-data:/data
      - automaker-claude-config:/home/automaker/.claude
      - automaker-cursor-config:/home/automaker/.cursor
      - automaker-opencode-data:/home/automaker/.local/share/opencode
      - automaker-opencode-config:/home/automaker/.config/opencode
      - automaker-opencode-cache:/home/automaker/.cache/opencode
```

**Key: ALL volumes are Docker named volumes. ZERO host bind mounts.**
The container cannot access any files on the host filesystem.

---

## docker-compose.dev.yml (Development)

Key differences:
- Mounts source code: `.:/app:cached` (host bind mount)
- Named volume for node_modules (avoids platform conflicts)
- Host bind: `./data:/data` (shared between modes)
- Runs `npm ci` as root, then switches to automaker via gosu
- `HUSKY=0` to disable git hooks

---

## docker-entrypoint.sh

Runs as root, then switches:
1. Create/fix `.claude` directory (700)
2. Write `CLAUDE_OAUTH_CREDENTIALS` → `.credentials.json` (600)
3. Create/fix `.cursor` directory
4. Write `CURSOR_AUTH_TOKEN` → `auth.json` (600)
5. Fix `.npm` cache permissions
6. **`exec gosu automaker "$@"`** — switch to non-root user

---

## Volume Architecture

### Production (Named Only)
| Volume | Mount | Purpose |
|--------|-------|---------|
| automaker-data | /data | Settings, sessions, credentials, API keys |
| automaker-claude-config | ~/.claude | Claude CLI OAuth |
| automaker-cursor-config | ~/.cursor | Cursor CLI auth |
| automaker-opencode-* | various | OpenCode CLI state |

### Override (Host Mounts)
```yaml
# docker-compose.override.yml
services:
  server:
    volumes:
      - /path/to/project:/projects/my-project:rw
      - ~/.claude:/home/automaker/.claude       # Optional: share host credentials
      - ~/.config/gh:/home/automaker/.config/gh # Optional: GitHub CLI
      - ~/.gitconfig:/home/automaker/.gitconfig:ro
```

---

## Docker-in-Docker Analysis

### Current State: NO Docker-in-Docker Support

- No Docker socket mounted in ANY compose file
- No Docker CLI installed inside the container
- No DinD sidecar configured
- No references to docker.sock in server code
- The override example does NOT include Docker socket

### If You Need Docker-in-Docker

You would need to manually add:
```yaml
volumes:
  - /var/run/docker.sock:/var/run/docker.sock
```
And install Docker CLI in the container.

**This is DooD (Docker-outside-of-Docker)** — gives the container full control over the host Docker daemon. This effectively gives root access to the host. **Not recommended.**

### The Previous Issue

Running Automaker in Docker while working on a project that uses Docker services (e.g., docker-compose for databases) fails because:
1. No Docker CLI in the container
2. No Docker socket access
3. AI agent Bash commands cannot run `docker` or `docker-compose`

**Current fix status:** This is NOT fixed. Automaker in Docker still cannot manage Docker services inside projects. The workaround is:
- Run Automaker natively (not in Docker) if your project needs Docker
- Or mount the Docker socket (security risk)
- Or have the AI agent interact with external Docker hosts via API

---

## Architecture Support

The Docker image supports both AMD64 and ARM64. GitHub CLI and Claude CLI are downloaded for the correct architecture during build.

---

## Git History: Docker-Related Fixes

| Commit | Fix |
|--------|-----|
| ef6b9ac2 | Add --force to npm ci (platform-specific deps) |
| 92afbeb6 | Run npm install as root (permission issues) |
| aa8caeae | Make UID/GID configurable (volume permission mismatches) |
| 1a1517df | Fix npm cache permissions (npx MCP servers failing) |
| 5e330b76 | Add Playwright Chromium deps |
| 8ff4b591 | Implement ALLOWED_ROOT_DIRECTORY |

**Recurring themes:** Permission issues with volumes, platform-specific dependencies, expanding CLI tool support.
