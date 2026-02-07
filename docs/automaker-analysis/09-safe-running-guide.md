# 09 — Safe Running Guide

## TL;DR

| Method | Safety | Functionality | Recommended For |
|--------|--------|---------------|-----------------|
| Docker production (default) | Highest | No host access, no project editing | Evaluation only |
| Docker + override mount | High | Edit mounted projects only | Most users |
| Native (web mode) | Medium | Full host access | Developers who trust the tool |
| Native (Electron) | Medium | Full host access + desktop integration | Power users |

---

## Method 1: Docker Production (Maximum Isolation)

The safest way. Container cannot access host files.

### Setup

```bash
# 1. Build
docker-compose build

# 2. Set credentials
export ANTHROPIC_API_KEY=sk-ant-...
# Or use Claude CLI OAuth:
export CLAUDE_OAUTH_CREDENTIALS='{"oauth_token":"..."}'

# 3. Run
docker-compose up -d

# 4. Access
open http://localhost:3007
```

### What's isolated
- No host filesystem access (named volumes only)
- `ALLOWED_ROOT_DIRECTORY=/projects` restricts server file ops
- Non-root user inside container
- Separate UI (nginx) and server containers

### Limitation
You can't work on any projects since the container can't see your files.

---

## Method 2: Docker + Project Mount (Recommended)

Add your project directories to the container.

### Setup

```bash
# 1. Create override file
cp docker-compose.override.yml.example docker-compose.override.yml

# 2. Edit to mount your projects
```

```yaml
# docker-compose.override.yml
services:
  server:
    volumes:
      # Mount your project(s)
      - /path/to/your/project:/projects/your-project:rw

      # Optional: share host Claude CLI credentials
      - ~/.claude:/home/automaker/.claude

      # Optional: GitHub CLI for PR operations
      - ~/.config/gh:/home/automaker/.config/gh
      - ~/.gitconfig:/home/automaker/.gitconfig:ro
    environment:
      - GH_TOKEN=${GH_TOKEN}
```

```bash
# 3. Match UID/GID for volume permissions
UID=$(id -u) GID=$(id -g) docker-compose build

# 4. Run
docker-compose up -d
```

### Security hardening (add to override)
```yaml
services:
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
    pids_limit: 100
```

### What's isolated
- Only mounted directories are accessible
- `ALLOWED_ROOT_DIRECTORY=/projects` for Node.js file ops
- Non-root user
- No Docker socket

### Known limitation: AI agent Bash commands can access anything inside the container
The `Bash` tool is not restricted to `ALLOWED_ROOT_DIRECTORY`. Inside the container, the agent can read credential files at `/home/automaker/.claude/`. This is a known gap.

---

## Method 3: Native (Web Mode)

Run directly on your machine. Lower isolation, full functionality.

### Setup

```bash
# 1. Prerequisites
node --version  # Must be 22.x
claude --version  # Claude CLI must be installed and authenticated

# 2. Create empty .env (prevents TUI crash)
touch .env

# 3. Install dependencies
npm install

# 4. Run
npm run dev
# Choose "Web Application" → opens at http://localhost:3007

# Or directly:
npm run dev:full  # Runs both server + UI
```

### Safety measures
- Set `ALLOWED_ROOT_DIRECTORY` to restrict file operations:
  ```bash
  echo "ALLOWED_ROOT_DIRECTORY=/path/to/workspace" >> .env
  ```
- Set `TERMINAL_PASSWORD` if network-accessible:
  ```bash
  echo "TERMINAL_PASSWORD=mypassword" >> .env
  ```
- Set `AUTOMAKER_API_KEY` for explicit auth:
  ```bash
  echo "AUTOMAKER_API_KEY=$(uuidgen)" >> .env
  ```

### What's NOT isolated
- AI agents have full access to your filesystem (as your user)
- Terminal gives full shell access
- Network connections unrestricted

---

## Method 4: Native (Electron)

Desktop app with native dialog integration.

```bash
npm run dev:electron        # Standard
npm run dev:electron:debug  # With DevTools
```

Same safety considerations as web mode, plus:
- Native file dialogs for project selection
- Window bounds persistence
- Auto-spawns backend server

---

## Docker-in-Docker: Working with Projects That Use Docker

### The Problem
If your project uses Docker (e.g., `docker-compose` for databases), the AI agent inside Automaker's Docker container cannot run Docker commands because:
1. No Docker CLI installed
2. No Docker socket access
3. Bash commands fail on `docker`, `docker-compose`

### Current Status: NOT FIXED
There is no built-in Docker-in-Docker support.

### Workarounds

**Option A: Run Automaker natively (safest for this use case)**
```bash
npm run dev:full
```
Agent can use your host Docker directly.

**Option B: Mount Docker socket (security risk)**
```yaml
# docker-compose.override.yml
services:
  server:
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    group_add:
      - ${DOCKER_GID:-999}  # Host's docker group ID
```
Then install Docker CLI inside the container. This gives the container full control over your host's Docker daemon.

**Option C: External Docker host**
Have the AI agent interact with Docker via TCP API:
```bash
export DOCKER_HOST=tcp://host.docker.internal:2375
```
Requires configuring Docker daemon to listen on TCP (also a security concern).

**Option D: Pre-start services before Automaker**
Start your project's Docker services on the host before running Automaker. The AI agent can then interact with the running services via network ports without needing Docker commands.

---

## Quick Reference: Environment Variables for Safety

```bash
# .env file

# Authentication (pick one)
ANTHROPIC_API_KEY=sk-ant-...
# OR rely on: claude login

# API security
AUTOMAKER_API_KEY=strong-random-key-here

# Path sandboxing
ALLOWED_ROOT_DIRECTORY=/path/to/workspace

# Terminal protection
TERMINAL_PASSWORD=your-password

# CORS restriction
CORS_ORIGIN=http://localhost:3007

# NEVER set these in production:
# AUTOMAKER_DISABLE_AUTH=true   ← removes ALL auth
# AUTOMAKER_AUTO_LOGIN=true     ← skips login (OK for dev)
```

---

## Checklist: Before First Run

```
□ Node.js 22.x installed (nvm use 22)
□ Claude CLI installed and authenticated (claude login)
□ .env file created (at minimum: touch .env)
□ npm install completed
□ Ports 3007 and 3008 are free
□ (Docker) UID/GID match for volume permissions
□ (Docker) Override file created for project mounts
□ (Optional) ALLOWED_ROOT_DIRECTORY set
□ (Optional) AUTOMAKER_API_KEY set
□ (Optional) TERMINAL_PASSWORD set
```
