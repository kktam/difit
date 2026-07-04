# difit — Architecture

## Overview

difit follows a **three-tier local-first architecture**: a CLI entry point that parses user intent, an Express backend that serves a REST API and static web assets, and a React SPA frontend that renders the diff UI in the browser. All processes run on localhost with no external network dependencies (except `--pr` mode which invokes `gh` CLI).

```
┌───────────────────────────────────────────────────────────┐
│                       USER TERMINAL                        │
│  $ difit [options] <target> [compare-with]                │
│  $ diff -u a b | difit                                    │
└──────────────────┬────────────────────────────────────────┘
                   │
                   ▼
┌───────────────────────────────────────────────────────────┐
│                   CLI LAYER (commander.js)                  │
│                                                             │
│  ┌──────────┐  ┌───────────────┐  ┌────────────────────┐  │
│  │ Argument │  │ Diff Selection│  │ Git/PR Resolution  │  │
│  │ Parsing  │──▶ Resolution    │──▶ (simple-git / gh)  │  │
│  └──────────┘  └───────────────┘  └────────┬───────────┘  │
│                                             │              │
│  ┌──────────────────────────────────────────┘              │
│  │                                                          │
│  ▼                                                          │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              Server Startup (startServer)             │  │
│  └──────────────────────┬───────────────────────────────┘  │
└─────────────────────────┼──────────────────────────────────┘
                          │
                          ▼
┌────────────────────────────────────────────────────────────┐
│               EXPRESS SERVER (localhost:4966)               │
│                                                             │
│  ┌─────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │ Static Files│  │  REST API   │  │  SSE Endpoints   │  │
│  │ (React SPA) │  │  /api/*     │  │  /api/watch      │  │
│  └─────────────┘  └──────┬───────┘  │  /api/heartbeat  │  │
│                          │          └──────────────────┘  │
│                          ▼                                 │
│              ┌───────────────────────────┐                 │
│              │      GitDiffParser        │                 │
│              │  (simple-git + execSync)  │                 │
│              └───────────────────────────┘                 │
│                          │                                 │
│              ┌───────────────────────────┐                 │
│              │    FileWatcherService     │                 │
│              │  (@parcel/watcher + SSE)  │                 │
│              └───────────────────────────┘                 │
└──────────────────────────┬────────────────────────────────┘
                           │
                           ▼
┌────────────────────────────────────────────────────────────┐
│               REACT SPA (Vite + Tailwind CSS)              │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │                     App.tsx                           │  │
│  │  ┌───────────┐ ┌──────────┐ ┌────────────────────┐  │  │
│  │  │   FileList │ │ DiffView │ │ Comment System    │  │  │
│  │  │  (sidebar) │ │  (main)  │ │  (threads/messages)│  │  │
│  │  └───────────┘ └──────────┘ └────────────────────┘  │  │
│  │                                                      │  │
│  │  Viewers: TextDiff | ImageDiff | MarkdownDiff |      │  │
│  │           NotebookDiff                               │  │
│  │                                                      │  │
│  │  Services: StorageService (localStorage)             │  │
│  └──────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────┘
```

## Module Organization (`src/`)

```
src/
├── cli/               # CLI entry point & argument processing
│   ├── index.ts       # Commander program definition, action handler
│   ├── utils.ts       # Git root detection, stdin detection, validation
│   ├── github.ts      # gh CLI integration (PR patch, comment import)
│   └── comment.ts     # difit comment subcommand
│
├── server/            # Express backend
│   ├── server.ts      # Express app, route handlers, server lifecycle
│   ├── git-diff.ts    # GitDiffParser (unified diff parsing, blob retrieval)
│   ├── file-watcher.ts # FileWatcherService (@parcel/watcher + SSE)
│   └── generated-file-check.ts # Auto-generated file detection
│
├── client/            # React SPA frontend
│   ├── App.tsx        # Root component, state orchestration
│   ├── main.tsx       # React entry point
│   ├── themeBootstrap.ts # Theme initialization from URL params
│   ├── components/    # 40+ React components
│   │   ├── DiffViewer.tsx, DiffViewerHeader.tsx
│   │   ├── FileList.tsx, DiffChunk.tsx
│   │   ├── CommentForm.tsx, CommentThreadCard.tsx
│   │   ├── DiffQuickMenu.tsx (revision selector)
│   │   ├── SettingsModal.tsx, HelpModal.tsx
│   │   └── ... (40+ total)
│   ├── viewers/       # Specialized diff renderers
│   │   ├── registry.ts  # Viewer dispatch (match file → viewer)
│   │   ├── TextDiffViewer.tsx     # Default: unified/split code diff
│   │   ├── ImageDiffViewer.tsx    # PNG/JPG/GIF/etc.
│   │   ├── MarkdownDiffViewer.tsx # Markdown rendered preview
│   │   └── NotebookDiffViewer.tsx # .ipynb cell diff
│   ├── hooks/         # 14+ custom React hooks
│   ├── contexts/      # React context providers
│   ├── services/      # StorageService (localStorage persistence)
│   ├── utils/         # Client-side utilities
│   ├── constants/     # Theme/appearance constants
│   └── styles/        # Global CSS (Tailwind v4 + custom properties)
│
├── utils/             # Shared utilities (CLI + server)
│   ├── diffSelection.ts    # DiffSelection creation, equality, keying
│   ├── commentFormatting.ts # Prompt generation from comments
│   ├── commentImports.ts   # Comment import merge/normalize logic
│   ├── editorOptions.ts    # External editor preset definitions
│   ├── fileUtils.ts        # File extension helpers
│   ├── suggestionUtils.ts  # Suggestion template logic
│   └── createId.ts         # Unique ID generation
│
├── types/             # Shared TypeScript type definitions
│   ├── diff.ts        # Core data types (DiffFile, DiffChunk, Comment types)
│   └── watch.ts       # Watch event types and DiffMode enum
│
└── site/              # Marketing/landing page (Vite static site)
    ├── StaticDiffApp.tsx
    ├── SitePage.tsx
    └── ...
```

## Data Flow

### 1. CLI → Server Startup

```
CLI args → resolveDiffSelection() → startServer()
                │
                ├── Git path:     startServer({ selection, ... })
                ├── Stdin path:   startServer({ stdinDiff, ... })
                ├── PR path:      getPrPatch() → startServer({ stdinDiff, ... })
                └── Background:   spawn detached child → startServer({ ... })
```

### 2. Diff Data Pipeline

```
GitDiffParser.parseDiff(selection)
    │
    ├── simple-git diff (single git invocation)
    ├── parseUnifiedDiff() → DiffFile[] (header, chunks, lines)
    ├── isGeneratedFile() per file (path patterns + gitattributes + content scan)
    └── markGitattributesGeneratedFiles() via git check-attr

    Returns: DiffResponse { commit, files[], isEmpty, baseCommitish, ... }
```

### 3. Comment Data Flow (Client ↔ Server)

```
Client (browser)                          Server (Express)
     │                                          │
     ├── GET  /api/diff?ignoreWhitespace=...    │
     │     ← DiffResponse (with commentImports) │
     │                                          │
     ├── GET  /api/comments-json?base=...&target=...
     │     ← { version, threads[] }             │
     │                                          │
     ├── POST /api/comments?base=...&target=... │
     │     { threads, baseVersion }             │
     │     → merge/overwrite → { version, merged, threads }
     │                                          │
     ├── POST /api/comment-imports              │
     │     → mergeImports → { importId, ... }   │
     │                                          │
     ├── navigate/close tab → sendBeacon        │
     │     → updateCommentSession()             │
     │                                          │
     ├── SSE /api/watch ← { type: 'reload' }   │
     └── SSE /api/heartbeat ← keepalive         │
```

### 4. File Watching Flow

```
FileWatcherService.start(diffMode, repoPath)
    │
    ├── @parcel/watcher subscribes to filesystem
    ├── Changes debounced (300ms)
    ├── git rev-parse HEAD to detect commit changes
    ├── git diff-index to detect working/staging changes
    ├── Broadcasts via SSE: { type: 'reload', diffMode, changeType }
    │
    Client receives → fetchDiffData() → merge comments → re-render
```

## Key Design Decisions

### Single Git Invocation
The entire diff for a selection is obtained via a single `git diff` command with `--no-ext-diff --color=never`. This avoids multiple round-trips and provides better startup latency on large repositories. The raw unified diff text is then parsed character-by-character by `GitDiffParser`.

### Unified Diff Parsing
`GitDiffParser.parseUnifiedDiff()` splits the raw `git diff` output on `^diff --git ` markers, then processes each file block independently. Git paths are decoded (octal escape sequences for special characters), and file status is inferred from header patterns (`new file mode`, `/dev/null`, rename lines).

### Viewer Registry Pattern
The client uses a registry-based dispatch system (`viewers/registry.ts`) that matches file types to specialized viewer components:

1. Image files → `ImageDiffViewer` (side-by-side with swipe/before-after)
2. Markdown files → `MarkdownDiffViewer` (rendered preview)
3. Notebook files → `NotebookDiffViewer` (cell-level additions/deletions)
4. Everything else → `TextDiffViewer` (default code diff)

### Comment Storage Strategy
Comments are stored in **two places** with different scopes:

- **Client-side** (`localStorage` via `StorageService`): Per-diff persistence indexed by `(repositoryId, baseCommitish, targetCommitish, baseMode)`. Survives browser restarts.
- **Server-side** (in-memory `Map`): Session-scoped comment state with versioning for concurrent-write detection. Comments are exported to stdout on server shutdown.

The server acts as the source of truth during a session; clients bootstrap from the server on connect and sync via POST/POST-with-versioning. The `baseVersion` field enables the server to detect stale writes and merge rather than overwrite.

### SSE-Based File Watching
Uses `@parcel/watcher` (native filesystem events) for change detection, debounced at 300ms, then broadcasts reload events via Server-Sent Events (`/api/watch`). The watch mode is determined by `DiffMode`:

- `WORKING` / `DOT`: Watch `.` (all files) + `.git` HEAD
- `STAGED`: Watch `.git` (index changes)
- `DEFAULT`: Watch `.git` (commit changes)
- `SPECIFIC`: No watching (static comparison)

### Background Process Model
`--background` spawns a detached child process that inherits the CLI arguments (with `--keep-alive` + `--no-open` auto-injected). The parent waits for a JSON line from the child's stdout (`{ port, url, pid }`) then exits, leaving the server running in the background.

## Security & Safety

- **No external network calls**: All traffic is localhost (default `127.0.0.1:4966`)
- **Path traversal protection**: All file-path endpoints normalize and validate paths relative to the Git repository root
- **CORS restricted**: `Access-Control-Allow-Origin: http://localhost:*`
- **Command injection prevention**: Git commands use `execFileSync` (not `execSync`) for blob retrieval, and `simple-git` (parameterized) for diff operations
- **Host binding warning**: Prints a security warning when binding to non-localhost addresses
- **Large file limits**: 10MB max buffer for blob content
