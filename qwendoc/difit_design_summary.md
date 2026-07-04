# difit — Design Summary

## Technical Stack

| Layer | Technology | Version | Purpose |
|-------|-----------|---------|---------|
| Runtime | Node.js | ≥ 21.0.0 | Cross-platform CLI & server |
| Language | TypeScript | 6.0 | Strict type safety (`@tsconfig/strictest`) |
| Package Manager | pnpm | 11.6.0 | Workspace monorepo |
| CLI Framework | commander.js | 15.x | Argument parsing, subcommands, help |
| HTTP Server | Express | 5.x | REST API + static file serving |
| Git Integration | simple-git | 3.x | Git operations (diff, log, branch, rev-parse) |
| File Watching | @parcel/watcher | 2.x | Native filesystem change events |
| Frontend | React | 19.x | UI rendering |
| Build Tool | Vite | 8.x | Dev server + production bundling |
| CSS Framework | Tailwind CSS | 4.x | Utility-first styling |
| Syntax Highlighting | prism-react-renderer | 2.x | Code rendering |
| Testing | Vitest | 4.x | Unit & integration tests |
| Linting | oxlint | 1.x | Type-aware linting |
| Formatting | oxfmt | 0.x | Code formatting |
| Pre-commit | lefthook | 2.x | Git hooks |
| Dead Code | knip | 6.x | Unused file/export detection |
| Package | lucide-react | 1.x | SVG icons |
| Floating UI | @floating-ui/react | 0.x | Tooltip/dropdown positioning |
| Diagram | mermaid | 11.x | Mermaid diagram rendering |

## Design Patterns

### 1. Intent-First Stdin Detection

The CLI uses a priority-based decision tree for determining the diff source:

```
User intent ──► --pr provided?       → PR patch mode
             ├── - argument?         → Explicit stdin mode
             ├── positional args?    → Git revision mode
             └── stdin is pipe/file? → Auto stdin mode
```

`detectStdinSource()` inspects the stdin file descriptor (`fs.fstatSync(0)`) to distinguish between pipe, file redirect, socket, and TTY. This avoids ambiguous behavior when diffs are piped vs. when arguments are provided.

### 2. Diff Selection Resolution

Diff comparisons are represented as a `DiffSelection` object:

```typescript
interface DiffSelection {
  baseCommitish: string;    // The base/base side of the diff
  targetCommitish: string;  // The target/new side of the diff
  baseMode?: 'direct' | 'merge-base';
}
```

Special commit-ish values are normalized:
- `HEAD` → compared against `HEAD^` (default)
- `.` → working directory compared against `HEAD`
- `staged` → staging area compared against `HEAD` (or base)
- `working` → working tree compared against staging area

The `DiffMode` enum (`DEFAULT | WORKING | STAGED | DOT | SPECIFIC`) drives file watching behavior after selection resolution.

### 3. Viewer Registry Pattern

Diff file rendering uses a strategy pattern via a registry:

```typescript
const viewers: DiffViewerRegistration[] = [
  { id: 'image',    match: isImageFile,    Component: ImageDiffViewer },
  { id: 'markdown', match: isMarkdownFile, Component: MarkdownDiffViewer },
  { id: 'notebook', match: isNotebookFile, Component: NotebookDiffViewer },
  { id: 'default',  match: () => true,     Component: TextDiffViewer },
];
```

The `getViewerForFile(file)` function returns the first matching registration (highest priority first), with the `default` entry at the end as fallback. Each viewer receives the same `DiffViewerBodyProps` interface.

### 4. Comment System with Versioning

The comment system uses an **optimistic concurrency** model:

- Each comment session has a monotonically increasing `version` number
- Clients send their last-known `baseVersion` with POST requests
- Server detects stale writes: if `baseVersion !== session.version`, the server **merges** rather than overwrites
- Merging uses semantic identity (thread ID) to reconcile concurrent edits
- Conflict resolution favors the union of both sides' threads
- `sendBeacon` on `beforeunload` ensures server state is up-to-date on tab close

### 5. Lazy Diff Rendering

Large diffs are rendered lazily using a visibility-based approach:

- `useLazyDiffRendering` hook tracks which files are in/visible via `IntersectionObserver` on file container elements
- Files are rendered top-to-bottom with an optional "up-to" target (for keyboard navigation jumps)
- A `renderedFilePaths` Set tracks which files have been rendered
- The diff data response (`DiffResponse`) is set once; individual file rendering is controlled by the lazy rendering hook

### 6. File Collapse as Viewed State Sync

File collapse state synchronizes bidirectionally with the "viewed" (reviewed) state:

- Marking a file as viewed → auto-collapses it
- Unmarking → auto-expands it
- Folder-level review toggling cascades to all files in that folder
- Initial collapse state is populated from persisted viewed files
- In keyboard navigation, collapsing a viewed file scrolls to the next unviewed file

### 7. Auto-Generated File Detection

Three-layer detection strategy for generated files:

1. **Path patterns**: Regex patterns for lockfiles, minified files, source maps, generated code patterns (`*.g.dart`, `*.pb.go`, etc.)
2. **Git attributes**: `git check-attr linguist-generated` (batched in chunks of 200)
3. **Content scanning**: Reads first 4KB of file, checks for `@generated` / `DO NOT EDIT` markers and language-specific generated headers

Results are cached with a 60-second TTL. The detection is available via a dedicated API endpoint (`/api/generated-status/:path`).

## TypeScript Architecture

- **Strict mode** via `@tsconfig/strictest` with selected relaxations (`exactOptionalPropertyTypes: false`, `noPropertyAccessFromIndexSignature: false`)
- **Project references**: Root `tsconfig.json` references `tsconfig.cli.json` for the CLI bundle; Vite config references root for the client
- **Path aliases**: `@/` maps to `./src/*` for cleaner imports
- **ESM only**: `"type": "module"` in package.json, `import.meta.url` for `__dirname` equivalent
- **Dual build target**: `ES2022` (Vite client) + Node-compatible CLI output

## State Management (Client)

No external state management library — the app uses React hooks and context:

- `App.tsx` is the single source of truth for:
  - Diff data and version (`diffData`, `diffDataVersion`)
  - View mode (`split`/`unified`)
  - Ignore whitespace toggle
  - Theme/appearance settings
  - Collapsed files set
  - Revision selector state
  - Modal toggles (settings, help, comments list, revision detail)
- Custom hooks encapsulate domain state:
  - `useDiffComments` → comment threads CRUD with localStorage persistence
  - `useViewedFiles` → viewed file tracking with localStorage
  - `useKeyboardNavigation` → cursor position within diff structure
  - `useLazyDiffRendering` → visibility-based file rendering
  - `useExpandedLines` → expand/collapse hidden context lines
  - `useFileWatch` → SSE connection and reload coordination
  - `useAppearanceSettings` → theme, syntax theme, font, editor config
  - `useViewport` → responsive breakpoints

## CSS Architecture

- **Tailwind CSS v4** with the new `@import "tailwindcss"` syntax
- **Custom theme tokens** via `@theme` directive (GitHub-like color palette): `github-bg-primary`, `github-border`, `diff-addition-bg`, etc.
- **CSS custom properties** for dynamic theming: `--app-font-size`, `--app-font-family`
- **Theme switching**: `data-theme` attribute (`dark`/`light`) + `data-color-vision` (`deuteranopia`) on root element
- **Transition system**: `transition: background-color 0.3s ease, color 0.3s ease, border-color 0.3s ease` on themed elements
- **Keyboard cursor overlay**: CSS `::before`/`::after` pseudo-elements on `<tr>` and `<td>` for keyboard navigation highlight (no JS-driven overlay divs)
- **Word-level diff highlighting**: CSS custom properties for addition/removal colors, including deuteranopia variants

## Performance Considerations

- Diff data is cached server-side (LRU cache, max 8 entries, most-recently-used eviction)
- Comment prompt generation is client-side (no server round-trip)
- Generated status checks are cached with 60s TTL
- Blob content retrieval uses 10MB max buffer
- File tree building uses a shared tree-node structure with memoized computation
- Keyboard navigation uses `useRef` for cursor state to avoid re-renders on every keypress
- Lazy rendering prevents rendering files below the fold
- Commit resolution is cached with 5s TTL (avoids repeated `git rev-parse` calls)
- @parcel/watcher uses native platform filesystem events (FSEvents on macOS, inotify on Linux)
