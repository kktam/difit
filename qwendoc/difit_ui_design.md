# difit — User Interface Design

## Tech Stack

- **Framework**: React 19 with TypeScript
- **Build**: Vite 8 (dev server + production bundling)
- **Styling**: Tailwind CSS v4 with custom GitHub-like theme tokens
- **Icons**: lucide-react 1.x
- **Syntax Highlighting**: prism-react-renderer 2.x
- **Floating UI**: @floating-ui/react (tooltips, dropdowns)
- **Diagrams**: mermaid 11.x (Markdown preview)
- **Keyboard**: react-hotkeys-hook 5.x
- **Markdown**: react-markdown 10.x with remark-gfm

## Layout Structure

```
┌──────────────────────────────────────────────────────────┐
│  HEADER                                                  │
│  ┌────────────┬───────────────────────────────────────┐  │
│  │ Logo [◀▶]  │ [Split|Unified] ✓Ignore WS  [⟳]      │  │
│  │ Settings   │ [Comments] [3/10 files] [HEAD^..HEAD] │  │
│  └────────────┴───────────────────────────────────────┘  │
├──────────┬───────────────────────────────────────────────┤
│ SIDEBAR  │  MAIN CONTENT AREA (scrollable)               │
│          │                                                │
│ ┌──────┐ │  ┌─────────────────────────────────────────┐  │
│ │File  │ │  │  File Header                             │  │
│ │Tree  │ │  │  ┌──────────┐ ┌──────────┐              │  │
│ │      │ │  │  │ [▼] [✓]  │ │src/App.tsx│ [+12 -3]   │  │
│ │📁 src│ │  │  └──────────┴──────────────────┬────────┘  │
│ │ 📄 A │ │  │  Diff Chunk                     │           │
│ │ 📄 B │ │  │  ┌───┬────┬──────────────────┐ │           │
│ │📁 pkg│ │  │  │ L │ R │ Content          │ │           │
│ │ 📄 C │ │  │  ├───┼────┼──────────────────┤ │           │
│ │ 📄 D │ │  │  │42 │ 42 │ unchanged        │ │           │
│ │      │ │  │  │ - │ 43 │ + added line     │ │           │
│ │ ↕drag │ │  │  │41 │ -  │ - deleted line  │ │           │
│ │      │ │  │  └───┴────┴──────────────────┘ │           │
│ │[Help]│ │  │                                 │           │
│ │[GH★] │ │  │  ┌──────────────────────────┐   │           │
│ └──────┘ │  │  │ Expand hidden lines ↑15  │   │           │
│          │  │  └──────────────────────────┘   │           │
│          │  │                                 │           │
│          │  │  File Header (collapsed)        │           │
│          │  │  ┌──────────────────────────┐   │           │
│          │  │  │ [▶] [✓] src/utils.ts     │   │           │
│          │  │  └──────────────────────────┘   │           │
│          │  │                                 │           │
└──────────┴────────────────────────────────────┴───────────┘
```

## Header Bar

The header is a two-section bar at the top of the screen:

### Left Section (Logo + File Tree Toggle)
- **Logo**: "difit" text/logo (Lucide icons, secondary text color)
- **Toggle File Tree button**: PanelLeftClose / PanelLeft icons
- **Settings button**: Gear icon → opens SettingsModal
- On mobile: section spans full width, stacks vertically

### Right Section (Toolbar)
- **View Mode Toggle**: Segmented button group
  - `[Split]` — side-by-side diff (desktop only, auto-hides on mobile)
  - `[Unified]` — inline diff
  - Icons: Columns / AlignLeft
- **Ignore Whitespace**: Checkbox with label, toggles `git diff -w`
- **Reload Button**: Refresh icon, visible when file changes detected via SSE
- **Comments Dropdown**: Appears when comments exist
  - Shows comment count
  - Copy All Prompt, Delete All, View All actions
- **Progress Bar**: "3 / 10 files viewed" with colored progress bar
  - Green (>50% remaining), Yellow (20-50%), Red (<20%)
  - Sparkle animation when all files reviewed
- **Revision Selector**: Quick menu to switch between diff targets
  - Shows commit range (e.g., `abc1234...def5678`)
  - Dropdown with recent branches, commits, special targets
  - "Open Advanced" link opens RevisionDetailModal

## File Tree Sidebar (Left Panel)

### Structure
- Resizable panel (drag handle, 200-600px range, default 280px)
- Toggleable via header button or keyboard shortcut
- Tree view of all files in the diff, organized by directory
- Each directory has expand/collapse toggle + folder review checkbox
- Each file shows:
  - Status icon: green `+` (added), red `✕` (deleted), yellow `✎` (renamed), gray `⧉` (modified)
  - File name
  - Review checkbox (checked = viewed/collapsed)
  - Comment indicator dot (file has comments)

### Tree Behavior
- Directories show aggregate: folder icon + file/folder name
- `getTreeRowPaddingLeft(depth)` computes indentation (16px base + depth × 24px)
- Folder review toggle marks/unmarks all contained files
- Empty directories are omitted from the tree
- On mobile: renders as a full-screen slide-in panel with overlay backdrop

### Bottom Bar (Desktop Only)
- Keyboard Shortcuts link → HelpModal
- GitHub link → opens repository in browser

## Diff Content Area (Main)

### File Header (sticky, z-10)
- **Collapse Toggle**: ChevronDown (expanded) / ChevronRight (collapsed)
- **Status Icon**: Same as file tree
- **File Path**: Clickable, shows full path
- **Additions/Deletions**: `+12 -3` summary
- **Reviewed Checkbox**: Mark file as viewed
- **Copy Button**: Copy file path
- **Open in Editor**: Opens file in configured IDE
- **Keyboard focus indicator**: Green left-border glow when focused via keyboard nav

### Diff Chunks
Each file contains one or more diff chunks:

#### Split Mode (side-by-side)
```
┌─────────────────────┬─────────────────────┐
│      OLD (base)     │      NEW (target)   │
┌───┬─────────────────┬───┬─────────────────┐
│42 │ unchanged line  │42 │ unchanged line  │
├───┼─────────────────┼───┼─────────────────┤
│ - │ deleted line    │43 │ + added line    │ ← gutter
│   │   [comment btn] │   │   [comment btn] │
├───┼─────────────────┼───┼─────────────────┤
│41 │ old context     │ - │ (empty)         │
└───┴─────────────────┴───┴─────────────────┘
```

#### Unified Mode (inline)
```
┌───┬───┬─────────────────────────────┐
│ L │ R │ Content                     │
├───┼───┼─────────────────────────────┤
│42 │42 │  unchanged line             │
├───┼───┼─────────────────────────────┤
│ - │43 │+ added line                 │
│   │   │  [comment btn]              │
├───┼───┼─────────────────────────────┤
│41 │ - │- deleted line               │
│   │   │  [comment btn]              │
├───┼───┼─────────────────────────────┤
│40 │40 │  unchanged line             │
└───┴───┴─────────────────────────────┘
```

#### Expand Buttons
Between non-adjacent chunks, an expand button shows:
- Direction: `↑` (up), `↓` (down), `↕` (both)
- Hidden line count: "Expand hidden lines ↑15"
- Actions: expand by default amount, or expand all

### Gutter (Comment Button)
- Hover state: comment button appears on the gutter edge of each line
- Click: opens a comment form anchored to that line
- Drag: select a range of lines (highlighted in blue border)
  - Range-selected lines get a continuous left-border highlight
  - First selected line gets top-border highlight
  - Last selected line gets bottom-border highlight

## Comment System

### Comment Form
```
┌─────────────────────────────────────────┐
│  Write a comment...                      │
│  ┌─────────────────────────────────────┐ │
│  │ [textarea: multi-line input]        │ │
│  │                                     │ │
│  └─────────────────────────────────────┘ │
│  [Suggestion ▾] [Cancel] [Add comment]   │
└─────────────────────────────────────────┘
```
- Small icon buttons for key actions appear above the form
- "Suggestion" button opens a dropdown of comment templates
- "Add comment" submits, "Cancel" closes

### Comment Thread Card
```
┌─────────────────────────────────────┐
│  [Author Badge] 2 minutes ago  [⋮] │
│  The error handling here...         │
│                              [Copy] │
├─────────────────────────────────────┤
│  [Author Badge] 1 minute ago  [⋮]  │
│  Good catch, fixed!                 │
│                              [Copy] │
├─────────────────────────────────────┤
│  [Reply input box (condensed)]      │
│  ┌─────────────────────────────┐    │
│  │ Type a reply...             │    │
│  └─────────────────────────────┘    │
│                  [Send reply]       │
└─────────────────────────────────────┘
```
- Threaded: multiple messages in a conversation
- Each message shows author badge (when multiple authors), timestamp, body
- "Copy" button generates AI-optimized prompt from the thread
- "⋮" menu → Edit / Delete message
- Timeline indicator shows when reply is after a long gap
- Outdated badge when underlying code has changed

### Comments List Modal
- Full list of all comments across all files
- Grouped by file
- Click navigates to the comment's position in the diff
- "Copy All Prompt" and "Delete All" actions

## Specialized Viewers

### Image Diff Viewer
```
┌─────────────────────────────────────────┐
│  [Preview ▾]  [Before] [After] [Swipe]  │
├─────────────────────────────────────────┤
│                                         │
│   ┌─────────┐     ┌─────────┐           │
│   │  BEFORE  │     │  AFTER  │           │
│   │ (old)    │     │ (new)   │           │
│   └─────────┘     └─────────┘           │
│                                         │
│   Or Swipe mode: side-scroll reveal     │
└─────────────────────────────────────────┘
```
- Preview mode tabs: Before, After, Side-by-Side, Swipe, Onion Skin
- Supports common image formats (PNG, JPG, GIF, SVG, WebP, etc.)
- Blob content fetched via `/api/blob/:path`

### Markdown Diff Viewer
```
┌─────────────────────────────────────────┐
│  [Code ▾] [Preview] [Rich Diff]         │
├─────────────────────────────────────────┤
│  Rendered Markdown with diff            │
│  highlighting (green/red backgrounds)   │
│                                         │
│  # Heading (unchanged)                  │
│  ~~~                                    │
│  This text was **removed**              │
│  ++ This text was **added** ++          │
│  ~~~                                    │
│  Code blocks highlighted via Prism      │
│  Mermaid diagrams rendered inline       │
└─────────────────────────────────────────┘
```

### Notebook Diff Viewer
```
┌─────────────────────────────────────────┐
│  [Code ▾] [Preview] [Rich Diff]         │
├─────────────────────────────────────────┤
│  Cell 1 [added]                         │
│  ┌─────────────────────────────────┐    │
│  │ [2]: print("hello")            │    │
│  └─────────────────────────────────┘    │
│  Cell 2 [modified]                     │
│  ┌─────────────────────────────────┐    │
│  │ [-1]: import os                 │    │
│  │ [+1]: import os, sys           │    │
│  └─────────────────────────────────┘    │
│  Output cell differences highlighted    │
└─────────────────────────────────────────┘
```

## Modals

### Settings Modal
- **Syntax Theme**: Dropdown (GitHub Dark, Light, High Contrast, Dark Dimmed, Dark Tritanopia)
- **Appearance Theme**: Dark / Light toggle
- **Color Vision**: Normal / Deuteranopia
- **Font Size**: Slider or preset values
- **Font Family**: Text input
- **Editor**: Preset selector + custom command/template
- **Editor Options**: Custom command line + argument template (e.g., `code --goto "{file}:{line}"`)

### Help Modal
- Keyboard shortcuts reference table
- Quick usage guide

### Revision Detail Modal
- Full branch list with current-branch indicator
- Recent commits list with short hash + message
- Selection form with base/target commit inputs
- Base mode toggle (Direct / Merge Base)

## Responsive Behavior

| Viewport | Behavior |
|----------|----------|
| Desktop (>768px) | Split/unified toggle, resizable sidebar, full header |
| Mobile (<768px) | Auto Unified mode, slide-in file tree, compact header, bottom comments bar |

## Keyboard Navigation

| Shortcut | Action |
|----------|--------|
| `j` / `↓` | Move cursor to next diff line |
| `k` / `↑` | Move cursor to previous diff line |
| `n` | Move to next file |
| `p` | Move to previous file |
| `e` | Toggle file expanded/collapsed |
| `v` | Toggle file reviewed |
| `c` | Create comment at cursor position |
| `Shift+C` | Copy all comments as AI prompt |
| `Shift+D` | Delete all comments |
| `Shift+L` | Show comments list |
| `r` | Refresh/reload diff |
| `?` | Open help modal |
| `Ctrl+,` | Open settings |

Cursor navigation works across files and supports both split/unified modes. The cursor position is remembered per file. When a file is collapsed via keyboard, the cursor advances to the next unviewed file.

## Progressive Enhancement

- Loading state: Centered "Loading diff..." spinner
- Error state: Centered error message with error text
- Empty diff: "No differences found" message, server still runs
- No data: "No diff data available" fallback
- The server always returns a response even when the diff is empty; the browser just doesn't open automatically
