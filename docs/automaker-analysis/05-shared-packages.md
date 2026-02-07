# 05 — Shared Packages

## @automaker/types (`libs/types/`)

The domain model. Zero dependencies.

### Core Types

| Type | Fields | Purpose |
|------|--------|---------|
| `BaseFeature` | id, title, category, description, status, priority, dependencies, spec, model, imagePaths, textFilePaths, branchName, skipTests, thinkingLevel, reasoningEffort, planningMode, requirePlanApproval, planSpec, error, summary, startedAt, descriptionHistory | Feature card data |
| `PlanSpec` | status, content, tasks[], tasksCompleted, tasksTotal | Plan state |
| `PlanTask` | id, description, filePath, phase, status | Individual task in spec |
| `FeatureStatusWithPipeline` | 'backlog' \| 'in_progress' \| 'waiting_approval' \| 'verified' \| 'completed' \| `pipeline_${string}` | Feature lifecycle |

### Settings Types

| Type | Purpose |
|------|---------|
| `GlobalSettings` (245 fields) | User prefs stored in `{DATA_DIR}/settings.json` |
| `ProjectSettings` | Per-project overrides in `.automaker/settings.json` |
| `Credentials` | API keys (anthropic, google, openai) |
| `PhaseModelConfig` | Per-task model selection (12 phases) |
| `PhaseModelEntry` | model + thinkingLevel + reasoningEffort + providerId |
| `KeyboardShortcuts` | 21 customizable shortcut bindings |
| `MCPServerConfig` | MCP server: command/args/env or URL/headers |
| `ClaudeCompatibleProvider` | Custom provider: baseUrl, apiKey, models[] |
| `PromptCustomization` | 12 prompt category overrides |

### Model Types

| Type | Values |
|------|--------|
| `ModelAlias` | 'haiku', 'sonnet', 'opus' |
| `CursorModelId` | 21 models (claude-*, gpt-*, o-*, gemini-*) |
| `OpencodeModelId` | 5 free models |
| `ThinkingLevel` | 'none', 'low', 'medium', 'high', 'ultrathink' |
| `ReasoningEffort` | 'none', 'minimal', 'low', 'medium', 'high', 'xhigh' |
| `PlanningMode` | 'skip', 'lite', 'spec', 'full' |

### Event System

`EventType` — 50+ event types:
- `auto_mode_*` (start, feature_start, progress, complete, error, idle, paused)
- `agent_*` (stream, complete, error)
- `planning_*` (started, complete, approval_required)
- `feature_*` (updated, created, completed, error)
- `suggestions_*`, `ideation_*`, `worktree_*`, `dev_server_*`, `notification_*`

---

## @automaker/platform (`libs/platform/`)

Path management, security, subprocess spawning, CLI detection.

### Path Functions (43 exports)
Project paths: `getAutomakerDir()`, `getFeaturesDir()`, `getFeatureDir()`, `getContextDir()`, `getAppSpecPath()`, `getExecutionStatePath()`, etc.
Global paths: `getGlobalSettingsPath()`, `getCredentialsPath()`
Ideation paths: `getIdeasDir()`, `getIdeationSessionPath()`, etc.

### Security (`security.ts`)
| Function | Purpose |
|----------|---------|
| `initAllowedPaths(rootDir, dataDir)` | Initialize security boundaries |
| `validatePath(filePath)` | Throws `PathNotAllowedError` if outside boundaries |
| `isPathAllowed(filePath)` | Boolean check |
| `isPathWithinDirectory(filePath, dirPath)` | Containment check via `path.relative()` |

### Secure Filesystem (`secure-fs.ts`)
Wraps ALL `fs` operations with `validatePath()` checks:
- Async: access, readFile, writeFile, mkdir, readdir, stat, rm, unlink, copyFile, appendFile, rename, lstat
- Sync: existsSync, readFileSync, writeFileSync, mkdirSync, readdirSync, statSync, accessSync, unlinkSync, rmSync
- Env file operations: readEnvFile, writeEnvKey, removeEnvKey
- Throttling: `p-limit` with configurable concurrency (default 100), retry with exponential backoff

### Other Modules
- **subprocess.ts** — `spawnProcess()`, `spawnJSONLProcess()` for CLI execution
- **node-finder.ts** — Cross-platform Node.js binary finder with nvm/fnm support
- **system-paths.ts** — ~45 exports for CLI tool paths (gh, claude, codex, opencode, shells)
- **editor.ts** — Editor detection: Cursor, VS Code, Zed, etc.
- **terminal.ts** — Terminal detection: iTerm2, Warp, Alacritty, Hyper, etc.
- **wsl.ts** — 9 exports for Windows Subsystem for Linux

---

## @automaker/utils (`libs/utils/`)

### Error Handler
- `classifyError()` — Regex-based pattern matching → ErrorInfo with type, severity, message, action
- `isAbortError()`, `isRateLimitError()`, `isQuotaExhaustedError()`, etc.
- `getUserFriendlyErrorMessage()` — Human-readable messages

### Logger
- `createLogger(name)` — Named logger with level filtering
- Levels: error, warn, info, debug
- Color and timestamp toggling

### Atomic Writer
- `atomicWriteJson()` — Write-temp-then-rename for corruption safety
- `readJsonWithRecovery()` — Tries backup files if main is corrupted
- `rotateBackups()` — Manages backup file rotation

### Context Loader
- `loadContextFiles(projectPath)` — Loads `.automaker/context/` files
- `getContextFilesSummary()` — Quick summary without reading content

### Memory Loader
- `loadRelevantMemory()` — Smart selection based on tags, usage scores, importance
- `appendLearning()` — Add learnings with file-level locks
- `recordMemoryUsage()` — Track successful usage for scoring
- `parseFrontmatter()` / `serializeFrontmatter()` — YAML frontmatter handling

### Other
- `convertImagesToContentBlocks()` — Images to Claude SDK format
- `buildPromptWithImages()` — Multimodal prompt building
- `normalizeContentBlocks()` — Content normalization

---

## @automaker/prompts (`libs/prompts/`)

40+ prompt templates organized by category:

| Category | Prompts |
|----------|---------|
| Auto Mode | planningLite, planningLiteWithApproval, planningSpec, planningFull, featurePrompt, followUp, continuation, pipelineStep |
| Agent | systemPrompt |
| Backlog Plan | systemPrompt, userPrompt |
| Enhancement | improve, technical, simplify, acceptance |
| Commit Messages | systemPrompt |
| Title Generation | systemPrompt |
| Issue Validation | systemPrompt |
| Ideation | systemPrompt, suggestionsSystemPrompt |
| App Spec | generateSystem, structuredInstructions, generateFeatures |
| Context Description | describeFile, describeImage |
| Suggestions | features, refactoring, security, performance |
| Task Execution | taskPrompt, implementationInstructions, playwrightVerification, learningExtraction, planRevision, continuationAfterApproval, resumeFeature, projectAnalysis |

All customizable via settings with merge functions: `mergeAllPrompts(customization)`.

---

## @automaker/model-resolver (`libs/model-resolver/`)

| Function | Purpose |
|----------|---------|
| `resolveModelString(key, default?)` | Main resolver: alias → full model ID |
| `getEffectiveModel(explicit?, session?, default?)` | Priority resolution chain |
| `resolvePhaseModel(entry, default?)` | Resolve PhaseModelEntry with provider routing |

Model mappings:
- `haiku` / `claude-haiku` → `claude-haiku-4-5-20251001`
- `sonnet` / `claude-sonnet` → `claude-sonnet-4-5-20250929`
- `opus` / `claude-opus` → `claude-opus-4-5-20251101`
- Cursor/Codex/OpenCode models pass through unchanged

---

## @automaker/dependency-resolver (`libs/dependency-resolver/`)

Modified Kahn's algorithm with priority-aware topological sort.

| Function | Purpose |
|----------|---------|
| `resolveDependencies(features)` | Topological sort → ordered features, circular deps, missing deps, blocked features |
| `areDependenciesSatisfied(feature, all, opts?)` | Check if deps completed/verified |
| `wouldCreateCircularDependency(features, src, tgt)` | DFS cycle detection |
| `getAncestors(feature, all, maxDepth?)` | Traverse dependency graph |
| `formatAncestorContextForPrompt(ancestors)` | Markdown for agent prompts |

---

## @automaker/spec-parser (`libs/spec-parser/`)

XML ↔ object conversion for `app_spec.txt`.

| Function | Purpose |
|----------|---------|
| `xmlToSpec(xml)` | Parse XML → SpecOutput |
| `specToXml(spec)` | SpecOutput → XML string |
| `validateSpec(spec)` | Validate structure |
| `extractXmlSection(xml, tag)` | Extract content between tags |
| `escapeXml()` / `unescapeXml()` | XML entity handling |

---

## @automaker/git-utils (`libs/git-utils/`)

| Function | Purpose |
|----------|---------|
| `getGitRepositoryDiffs(repoPath)` | Main entry: returns `{ diff, files, hasChanges }` |
| `parseGitStatus(output)` | Parse `git status --porcelain` → FileStatus[] |
| `isGitRepo(path)` | Check if path is a git repo |
| `generateSyntheticDiffForNewFile()` | Unified diff format for untracked files |
| `appendUntrackedFileDiffs()` | Merge git diff with synthetic diffs |
| `generateDiffsForNonGitDirectory()` | Handle non-git directories |

Handles: binary files (placeholder), >1MB files (truncated), recursive directory listing, 10MB git diff buffer.
