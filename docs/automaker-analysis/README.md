# Automaker Deep Dive Analysis

Comprehensive analysis of the Automaker codebase — architecture, implementation, security, Docker, and operational guides.

## Documents

| Document | Description |
|----------|-------------|
| [01-overview.md](./01-overview.md) | Project overview, tech stack, prerequisites |
| [02-architecture.md](./02-architecture.md) | Monorepo structure, package dependency chain, data flow |
| [03-server.md](./03-server.md) | Server deep dive: routes, services, providers, middleware, WebSocket |
| [04-ui.md](./04-ui.md) | UI deep dive: components, state management, routing, themes, DnD |
| [05-shared-packages.md](./05-shared-packages.md) | All 8 shared libraries: types, platform, utils, prompts, etc. |
| [06-features-workflow.md](./06-features-workflow.md) | Complete feature lifecycle, planning modes, auto-mode, agent chat |
| [07-security.md](./07-security.md) | Security model, authentication, path sandboxing, known gaps |
| [08-docker.md](./08-docker.md) | Docker setup, compose files, isolation, Docker-in-Docker analysis |
| [09-safe-running-guide.md](./09-safe-running-guide.md) | How to run Automaker safely in isolated environments |
| [10-api-reference.md](./10-api-reference.md) | Complete REST API endpoint catalog |

## Quick Navigation

- **Want to understand the codebase?** Start with [01-overview](./01-overview.md) then [02-architecture](./02-architecture.md)
- **Want to run it safely?** Go to [09-safe-running-guide](./09-safe-running-guide.md)
- **Worried about security?** Read [07-security](./07-security.md)
- **Docker issues?** Check [08-docker](./08-docker.md)
- **Building features on top?** Read [03-server](./03-server.md) and [04-ui](./04-ui.md)
