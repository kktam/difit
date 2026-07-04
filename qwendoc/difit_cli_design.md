# difit — CLI Design

## Command Structure

```
difit [options] [target] [compare-with]
difit --pr <url>
difit -                           # Explicit stdin mode
difit comment <subcommand>        # Comment management
difit --help                      # Usage information
difit --version                   # Version number
```

## Positional Arguments

| Position | Parameter | Default | Description |
|----------|-----------|---------|-------------|
| 1 | `<target>` | `HEAD` | Commit hash, tag, `HEAD~n`, branch name, or special keyword |
| 2 | `[compare-with]` | — | Second commit/branch to compare against (shows diff between target and this) |

### Special Keywords (as target)

| Keyword | Effect | Diff Mode |
|---------|--------|-----------|
| `.` | All uncommitted changes (staging + working) | `DOT` |
| `staged` | Staging area changes only | `STAGED` |
| `working` | Unstaged/working directory changes only | `WORKING` |
| `-` | Read diff from stdin (explicit) | — |

### Revision Resolution

The CLI resolves special arguments into a `DiffSelection` (base + target commit-ish):

```typescript
// Default (no args): HEAD vs HEAD^
baseCommitish: 'HEAD^', targetCommitish: 'HEAD'

// Single commit: <hash> vs <hash>^
baseCommitish: '<hash>^', targetCommitish: '<hash>'

// Two commits: <compare-with> vs <target>
baseCommitish: '<compare-with>', targetCommitish: '<target>'

// Working: staged vs working
baseCommitish: 'staged', targetCommitish: 'working'

// Staged: HEAD vs staged
baseCommitish: 'HEAD', targetCommitish: 'staged'

// Dot: HEAD vs working directory
baseCommitish: 'HEAD', targetCommitish: '.'
```

## Options

| Flag | Type | Default | Description |
|------|------|---------|-------------|
| `--port <port>` | number | 4966 | Preferred HTTP port (falls back to +1 if occupied) |
| `--host <host>` | string | `127.0.0.1` | Bind address (`0.0.0.0` for external access) |
| `--no-open` | boolean | false | Don't auto-open browser |
| `--comment <json>` | string[] | [] | Inject initial review comments (repeatable) |
| `--pr <url>` | string | — | GitHub PR URL to review |
| `--clean` | boolean | false | Clear all existing comments and viewed files |
| `--include-untracked` | boolean | false | Auto-include untracked files in diff |
| `--keep-alive` | boolean | false | Keep server alive after browser disconnect |
| `--background` | boolean | false | Run server in background, output JSON info |
| `--context <lines>` | number | git default (3) | Limit context lines per change (0 = changes only) |
| `--merge-base` | boolean | false | Resolve base via `git merge-base` |
| `-v, --version` | flag | — | Output version number |

### Constraint Matrix

Some option combinations are invalid:

| Combination | Error |
|-------------|-------|
| `--pr` + positional args | "cannot be used with positional arguments" |
| `--pr` + `--merge-base` | "cannot be used with --pr" |
| `--pr` + `--context` | "cannot be used with --pr" |
| `--context` + stdin diff | "cannot be used with stdin diff" |
| `--merge-base` + stdin diff | "cannot be used with stdin diff" |
| `--merge-base` + special arg | "requires a commit-ish base" |

## Comment JSON Format (`--comment`)

Supports both single objects and JSON arrays. Repeatable — can appear multiple times.

### Thread Comment Import
```json
{
  "type": "thread",
  "filePath": "src/example.ts",
  "position": {
    "side": "new",
    "line": 10
  },
  "body": "The background for this change is..."
}
```

### Reply Comment Import
```json
{
  "type": "reply",
  "filePath": "src/example.ts",
  "position": {
    "side": "new",
    "line": 10
  },
  "body": "I agree, let's fix this."
}
```

### Line Range Support
```json
{
  "line": { "start": 10, "end": 15 }
}
```

## PR Mode (`--pr`)

```
difit --pr https://github.com/owner/repo/pull/123
```

- Uses `gh pr diff --patch` to fetch the PR diff as a unified diff string
- Also imports unresolved inline review threads from the PR as startup comments
- Authentication via:
  1. `gh auth login` (interactive, recommended)
  2. `GH_TOKEN` / `GITHUB_TOKEN` environment variable (CI/non-interactive)
- GitHub Enterprise Server: `gh auth login --hostname YOUR-SERVER` or `GH_HOST` env var

## Background Mode (`--background`)

```
difit --background
```

Spawns a detached child process that inherits all CLI flags (auto-injects `--keep-alive` and `--no-open`). The parent process outputs a single JSON line to stdout, then exits:

```json
{"port": 4966, "url": "http://localhost:4966", "pid": 12345}
```

The child process runs independently until Ctrl+C or tab-close (when not in keep-alive mode). Scripts can consume the JSON output for automation.

## Comment Subcommand

```
difit comment <action> [options]
```

For programmatic comment management without starting the server. (Defined in `src/cli/comment.ts`.)

## Stdin Mode

Three ways to pipe diffs into difit:

```bash
# 1. Explicit stdin (positional '-')
git diff --cached | difit -

# 2. Auto-detect (pipe/file/socket detected)
diff -u file1.txt file2.txt | difit

# 3. Explicit via file redirect
cat changes.patch | difit
```

Intent-first detection rules:

1. `-` as argument → explicit stdin mode
2. Positional args or `--pr` → Git/PR mode (no auto stdin)
3. No explicit mode + stdin is pipe/file/socket → auto stdin mode

## Comment Output on Shutdown

When the server shuts down (Ctrl+C), it prints formatted comments to stdout:

```
src/components/Button.tsx:L42   # This line is automatically added
Make this variable name more descriptive

src/utils/helpers.ts:L12-L16   # This section is unnecessary
This code duplicates the logic in calculateTotal()
```

This format is optimized for copying as AI agent context — each comment includes the file path and line number as a comment line, followed by the comment body.

## Exit Codes

| Code | Condition |
|------|-----------|
| 0 | Success |
| 1 | Error (invalid args, git error, PR fetch failure, timeout) |

## Environment Variables

| Variable | Purpose |
|----------|---------|
| `DIFIT_BACKGROUND_CHILD` | Set internally to detect background child process |
| `DIFIT_EDITOR` | Editor override (overrides `EDITOR`) |
| `EDITOR` | System editor preference |
| `GH_TOKEN` / `GITHUB_TOKEN` | GitHub API authentication (for `--pr` mode) |
| `GH_HOST` | GitHub Enterprise Server hostname |
| `NODE_ENV` | Set to `development` for dev mode (hot reload) |
