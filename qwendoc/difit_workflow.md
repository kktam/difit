# difit — Workflow Guide

## User Workflows

### 1. Quick Local Review (Basic)

```
$ npx difit
    │
    ▼
CLI: Resolves HEAD vs HEAD^
    │
    ▼
Server starts on localhost:4966
    │
    ├── GitDiffParser.parseDiff(HEAD^, HEAD)
    ├── Returns DiffResponse with files[]
    └── Browser opens automatically
           │
           ▼
     User browses diff in UI:
     ├── File tree sidebar shows all changed files
     ├── Split/Unified view toggle
     ├── Click comment buttons to add inline feedback
     ├── Mark files as reviewed (auto-collapses)
     └── Copy individual comments or all comments as AI prompts
           │
           ▼
     Close tab → server auto-shuts down
     (or Ctrl+C in terminal)
           │
           ▼
     Formatted comments printed to terminal
```

### 2. Compare Two Branches

```
$ difit feature-branch main
    │
    ├── Resolves: baseCommitish = 'main', targetCommitish = 'feature-branch'
    ├── Mode: SPECIFIC (no file watching, static comparison)
    └── Browser opens with diff of feature-branch vs main
```

### 3. Review All Uncommitted Changes

```
$ difit .
    │
    ├── Found untracked files? → Prompts user (unless --include-untracked)
    │   "Would you like to include these untracked files? (Y/n): "
    │   If yes: git add --intent-to-add <files>
    │
    ├── Mode: DOT (watches working directory + .git)
    ├── Diff: HEAD vs working directory (git diff HEAD)
    └── Browser opens → user reviews → edits saved → file watcher triggers reload
```

### 4. Review Specific Commit

```
$ difit 6f4a9b7
    │
    ├── Resolves: 6f4a9b7^ vs 6f4a9b7
    ├── Mode: DEFAULT (watches .git for changes)
    └── Shows diff of that single commit
```

### 5. GitHub PR Review

```
$ difit --pr https://github.com/owner/repo/pull/123
    │
    ├── CLI calls gh pr diff --patch <url>
    ├── Parses patch as stdin diff
    ├── Fetches unresolved inline review threads via gh pr view
    ├── Injects PR comments as startup comments
    ├── Mode: N/A (stdin-based, no watching)
    └── Browser opens with PR diff + imported comments
```

### 6. External Diff Tool Integration

```bash
# From any diff-generating tool
$ diff -u file1.txt file2.txt | difit

# Review merge base vs feature branch
$ git diff --merge-base main feature-branch | difit -

# Review a saved patch file
$ cat changes.patch | difit

# Review an entire new file
$ git diff -- /dev/null path/to/newfile.ts | difit
```

### 7. Background Server Mode (Script/CI)

```bash
# Start server, get JSON port info
$ difit --background --port 8080
{"port":8080,"url":"http://localhost:8080","pid":54321}

# Use in scripts:
$ PORT=$(difit --background | node -e "process.stdin.on('data',d=>{const j=JSON.parse(d);console.log(j.port);process.exit()})")
$ echo "Server running on port $PORT"

# Server stays alive until Ctrl+C or tab close
```

### 8. AI-Agent-Driven Review

```bash
# Agent injects initial findings as comments
$ difit --comment '{
    "type": "thread",
    "filePath": "src/auth.ts",
    "position": {"side": "new", "line": 42},
    "body": "This authentication flow doesn't handle token refresh"
  }' \
  --comment '{
    "type": "thread",
    "filePath": "src/db.ts",
    "position": {"side": "new", "line": 15},
    "body": "Missing connection pool error handling"
  }'

# Launch UI for human review with pre-loaded agent findings
$ difit --comment "$(cat agent-findings.json)"
```

### 9. Review with Merge Base

```bash
# Compare feature branch against its merge base with main
$ difit feature-branch --merge-base main

# Equivalent to: git diff $(git merge-base feature-branch main) feature-branch
```

## Development Workflows

### 1. Full Development Loop

```bash
# 1. Install dependencies (first time)
pnpm install

# 2. Start dev server with hot reload
pnpm run dev
    ├── Runs both Vite dev server + CLI server concurrently
    ├── Hot reloads React components on save
    ├── Re-starts CLI server on TS changes
    └── Open browser to view changes

# 3. Make code changes → see live updates

# 4. Run tests
pnpm test

# 5. Run lint + type check
pnpm run check

# 6. Run formatter
pnpm run format

# 7. Build production artifacts
pnpm run build
```

### 2. Production Build & Test

```bash
# Build everything (CLI bundle + Vite client)
pnpm run build
    ├── tsc --project tsconfig.cli.json  → dist/cli/
    └── vite build                        → dist/client/

# Run from built artifacts
pnpm run start <target>
```

### 3. Performance Benchmarking

```bash
# Measure diff rendering performance
pnpm run perf            # Default size
pnpm run perf:small      # Small repo
pnpm run perf:medium     # Medium repo
pnpm run perf:large      # Large repo
pnpm run perf:xlarge     # Very large repo

# Compare two performance runs
pnpm run perf:compare
```

### 4. Site Documentation

```bash
# Build marketing/demo site
pnpm run dev:site
    ├── Builds CLI (for data export)
    ├── Exports site data via node scripts/export-site-data.js
    ├── Starts Vite with vite.config.site.ts
    └── Opens browser with static site

# Build site for GitHub Pages
pnpm run build:site
    └── Outputs to dist/site/
```

### 5. VS Code Extension Packaging

```bash
pnpm run package:vscode
    └── Runs pnpm -C packages/vscode run package
```

## Data Lifecycle

### Comment Persistence

```
┌──────────────────────────────────────────────────────────┐
│                     BROWSER TAB                           │
│                                                           │
│  Memory (React state)                                     │
│      │                                                    │
│      ├── useDiffComments hook manages threads[]           │
│      │                                                    │
│      ├── On change → POST /api/comments (server sync)     │
│      ├── On change → localStorage (persistence)           │
│      ├── On tab close → sendBeacon POST /api/comments    │
│      └── On tab open → GET /api/comments-json (bootstrap)│
│                                                           │
│  localStorage (StorageService)                             │
│    keys: difit-storage-v1:<repoId>:<base>:<target>        │
│    Stores: version, threads[], viewedFiles[],             │
│            appliedCommentImportIds[]                       │
└──────────────────────────────────────────────────────────┘
         │
         ▼
┌──────────────────────────────────────────────────────────┐
│                  EXPRESS SERVER                            │
│                                                             │
│  In-memory Map<selectionKey, CommentSessionState>           │
│    ├── threads: DiffCommentThread[]                         │
│    ├── version: number (monotonic counter)                  │
│    │                                                        │
│    ├── On POST /api/comments                                │
│    │   ├── If baseVersion matches → replace                 │
│    │   ├── If baseVersion stale → merge by thread ID        │
│    │   └── Increment version, broadcast via SSE             │
│    │                                                        │
│    ├── On server shutdown (Ctrl+C)                          │
│    │   └── Fetch /api/comments-output → print to terminal   │
│    │                                                        │
│    └── No persistent server-side storage                    │
│         (localStorage is the durable store)                 │
└──────────────────────────────────────────────────────────┘
```

### Session Lifecycle

```
Server Start
    │
    ├── Parse diff → DiffResponse
    ├── Start file watcher (if applicable)
    ├── Start heartbeat SSE endpoint
    └── Open browser (unless --no-open)
    │
    ├── Client connects → GET /api/diff → renders UI
    ├── Client comments sync (bidirectional)
    ├── File watcher triggers reloads (working/staging mode)
    └── ...
    │
    Tab Close (without --keep-alive):
    ├── Client sendBeacon POST /api/comments
    ├── Server waits 100ms (for pending requests)
    ├── Prints comments to terminal
    └── process.exit(0)

Tab Close (with --keep-alive):
    ├── Server logs "Client disconnected, staying alive"
    └── Waits for Ctrl+C

Ctrl+C (terminal):
    ├── SIGINT handler fires
    ├── Fetch /api/comments-output (HTTP to self)
    ├── Print comments to terminal
    └── process.exit(0)
```

## File Watching State Machine

```
                    ┌──────────────┐
                    │   Idle       │
                    │ (watching)   │
                    └──────┬───────┘
                           │
               ┌───────────┴────────────┐
               │ File system change     │
               │ detected               │
               └───────────┬────────────┘
                           │
                    ┌──────▼───────┐
                    │  Debouncing  │  (300ms debounce)
                    │  collecting  │
                    │  changes     │
                    └──────┬───────┘
                           │
          ┌────────────────┼────────────────┐
          │                │                │
    ┌─────▼─────┐   ┌──────▼──────┐   ┌────▼─────┐
    │ Working   │   │ File change │   │ No real  │
    │ dir       │   │ but HEAD    │   │ change   │
    │ changed   │   │ also moved  │   │ detected │
    └─────┬─────┘   └──────┬──────┘   └────┬─────┘
          │                │                │
          ▼                ▼                ▼
    ┌──────────┐    ┌──────────┐     ┌──────────┐
    │ Broadcast│    │ Broadcast│     │  Return  │
    │ 'reload' │    │ 'reload' │     │ to Idle  │
    │ (working)│    │ (commit) │     │          │
    └──────────┘    └──────────┘     └──────────┘
          │                │
          ▼                ▼
    Client receives SSE → fetchDiffData()
    → merge previous comment threads into new diff
    → re-render UI
```

## Theme Initialization

```
User opens difit URL
    │
    ├── themeBootstrap.ts runs before React mounts
    │
    ├── Check URL query params:
    │   ?theme=dark|light
    │   ?syntax=github-dark|github-light|...
    │   ?colorVision=normal|deuteranopia
    │   ?fontSize=<number>
    │   ?fontFamily=<string>
    │
    ├── Set data-theme and data-color-vision on <html>
    ├── Load syntax theme CSS
    └── Set font properties
```

## Comment Prompt Generation

Each comment generates an AI-optimized prompt:

```
# Single line comment
src/components/Button.tsx:L42   # ← Automatically added
Make this variable name more descriptive

# Range comment
src/components/Button.tsx:L42-L48   # ← Automatically added
This section is unnecessary and can be removed

# Copy All output:
--- All Comments ---

src/components/Button.tsx:L42
Make this variable name more descriptive

src/utils/helpers.ts:L12-L16
This code duplicates the logic in calculateTotal()

src/auth.ts:L100
Add input validation here
```

## REST API Endpoints

| Method | Path | Purpose | Auth |
|--------|------|---------|------|
| GET | `/api/diff` | Main diff data with file list and chunks | None (localhost) |
| GET | `/api/revisions` | Available branches and recent commits | None |
| GET | `/api/generated-status/:path` | Check if file is auto-generated | None |
| GET | `/api/line-count/:path` | Get line counts for old/new refs | None |
| GET | `/api/blob/:path` | Get raw file content (including images) | None |
| GET | `/api/comments-json` | Get all comment threads | None |
| POST | `/api/comments` | Save/update comment threads | None |
| POST | `/api/comment-imports` | Import external comments (deduplication) | None |
| GET | `/api/comments-output` | Get formatted comment text for terminal | None |
| POST | `/api/open-in-editor` | Open a file in the user's editor at a line | None |
| GET | `/api/watch` | SSE endpoint for file watch events | None |
| GET | `/api/heartbeat` | SSE endpoint for connection keepalive | None |

## Development Scripts

| Script | Command | Purpose |
|--------|---------|---------|
| `dev` | `node scripts/dev.js` | Concurrent Vite dev server + CLI |
| `build` | `tsc + vite build` | Production build |
| `start` | `build + node dist/cli/index.js` | Full production test |
| `test` | `vitest run` | Run all tests |
| `test:watch` | `vitest` | Watch mode tests |
| `check` | `oxlint . --type-aware` | Lint + type check |
| `format` | `oxfmt --check .` | Check formatting |
| `format:fix` | `oxfmt --write .` | Auto-fix formatting |
| `knip` | `knip` | Detect unused files/exports |
| `perf:*` | Various scripts | Performance benchmarks |
