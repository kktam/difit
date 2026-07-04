# difit — Business Summary

## Tagline
**GitHub-style diff viewer, launched from your terminal.** A lightweight CLI tool that spins up a local web server to display Git commit diffs in a GitHub-like "Files changed" view, purpose-built for the AI-assisted code review era.

## Executive Overview

**Problem:** Developers reviewing code locally must choose between raw `git diff` output in the terminal (no syntax highlighting, no file tree navigation, no inline commenting) or uploading code to remote platforms before it's ready to share. AI coding agents generate diffs that need structured human review, but there is no lightweight local tool that bridges the gap between terminal diffs and full-featured web review.

**Solution:** difit provides a single-command (`npx difit`) local web server that renders Git diffs with a polished GitHub-like interface. It adds inline commenting optimized for AI workflows — each comment includes file paths and line numbers formatted as ready-to-use AI prompts. Comments survive across sessions via `localStorage`, and AI agents can programmatically pre-load comments via CLI flags. The tool also supports stdin diffs, GitHub PR review mode, and real-time file watching.

**Key Differentiator:** difit is purpose-built for the AI era — comments are designed to be copied as context for AI coding agents, and the tool exposes a machine-readable Skill API (`npx skills add yoshiko-pg/difit`) so AI agents can invoke it autonomously.

## Market Opportunity

| Segment | Description | Relevance |
|---------|-------------|-----------|
| Solo developers & indie hackers | Need quick local code review without setting up CI/CD | High — `npx difit` zero-install workflow |
| AI-assisted development | Developers using Copilot, Cursor, Claude Code who need to review AI-generated diffs | Core target — comments-as-prompts feature |
| Remote/contract code reviewers | Review code offline without pushing to GitHub/GitLab | Medium — stdin mode works with any VCS |
| Team leads before PR | Pre-review local changes before pushing to shared repos | High — compare branches locally |
| CLI power users | Prefer terminal workflows but want rich diff visualization | High — composable CLI flags |

## Business Model

difit is an **MIT-licensed open-source project** published on npm (`difit`). There are no paid tiers, telemetry, or usage limits.

- **Distribution:** npm registry (`npm install -g difit`, `npx difit`), GitHub releases
- **Revenue model:** None (free and open-source). Community contributions via GitHub sponsors.
- **Ecosystem play:** The difit Skill API (`npx skills add yoshiko-pg/difit`) is a vector for AI agent adoption — agents can call difit for code review, creating a pull effect for developer toolchains.
- **VS Code extension:** A companion extension (`packages/vscode`) bundles difit directly into VS Code's diff editor context.

## Traction & Milestones

| Milestone | Status |
|-----------|--------|
| v1.0 — Basic commit diff viewer | ✅ Released |
| Split/Unified view modes | ✅ Released |
| Syntax highlighting (Prism.js) | ✅ Released |
| Inline commenting system | ✅ Released |
| Comment AI-prompt generation | ✅ Released |
| Stdin diff mode | ✅ Released |
| GitHub PR integration (`--pr`) | ✅ Released |
| File watching (live reload) | ✅ Released |
| Merge-base comparison (`--merge-base`) | ✅ Released |
| Keyboard navigation | ✅ Released |
| Revision selector | ✅ Released |
| File tree sidebar | ✅ Released |
| Image diff viewer | ✅ Released |
| Markdown diff preview | ✅ Released |
| Notebook (.ipynb) diff viewer | ✅ Released |
| VS Code extension | ✅ Released |
| Light theme support | ✅ Released |
| Color vision deficiency support | ✅ Released |
| Background server mode (`--background`) | ✅ Released |
| Auto-generated file detection | ✅ Released |
| Word-level diff highlighting | ✅ Released |
| Folder-level review marking | ✅ Released |

## Core Metrics

| Metric | Value |
|--------|-------|
| npm version | 5.0.6 |
| Source files | 171+ (TypeScript/TSX) |
| CLI arguments/flags | 15+ |
| REST API endpoints | 12+ |
| React components | 40+ custom |
| CSS theme properties | 40+ custom properties |
| Syntax-highlighted languages | 30+ |
| Test files | 25+ (Vitest) |
| Client-side hooks | 14+ custom React hooks |
| License | MIT |
| Node requirement | ≥ 21.0.0 |
| Package manager | pnpm 11.6+ |

## Key Differentiators

1. **AI-native comment system** — Every comment includes the file path and line range formatted as a ready-to-use AI prompt. "Copy All Prompt" outputs all comments in a structured format agents can consume.
2. **Agent-aware** — Ships with a `difit` Skill for `npx skills` that enables AI agents to autonomously launch code review sessions with programmatic comment injection.
3. **Zero configuration** — `npx difit` from anywhere inside a Git repo. No config files, no daemon, no setup.
4. **Multi-source diff** — Supports Git commits, working directory changes, stdin pipes, GitHub PR patches, and arbitrary diff-text from external tools.
5. **Real-time file watching** — In working-dir mode, difit watches filesystem changes and auto-reloads the diff view.
6. **Portable** — Single npm package, no external service dependencies. Runs entirely on localhost.

## Feature Summary

### Core Diff Viewing
- GitHub-style "Files changed" layout with file tree sidebar
- Split (side-by-side) and Unified (inline) view modes
- Ignore whitespace toggle
- Syntax highlighting via Prism.js (30+ languages)
- Word-level diff highlighting
- Context line count control (`--context`)
- File collapse/expand per file
- Auto-collapse generated files and deleted files
- Image diff viewer (side-by-side or swipe)
- Markdown rendered preview
- Jupyter Notebook (.ipynb) diff viewer

### Review & Comment System
- Inline line-level comments (click or drag-to-select line range)
- Multi-message thread comments
- Comment editing and deletion
- AI prompt generation from comments ("Copy Prompt" / "Copy All Prompt")
- Persistent storage (browser localStorage + server-side session)
- Comment author badges (multi-author detection)
- Outdated comment detection (when underlying code changes)
- Comment import from external sources (CLI `--comment` flag)
- Comment export on server shutdown
- Visual comment migration when switching revision scopes

### Git Integration
- Single commit, branch, tag, or `HEAD~n` reference
- Two-commit comparison (`difit A B`)
- Working directory, staging area, or both (`difit . / difit staged / difit working`)
- Merge-base resolution (`--merge-base`)
- GitHub PR review (`--pr <url>`) via `gh pr diff --patch`
- GitHub PR comment import (unresolved inline review threads)
- Stdin diff pipe (`diff -u a b | difit`)
- Untracked file auto-inclusion (`--include-untracked`)

### Revision Selector
- In-UI branch picker (all local branches, sorted by default→current→alpha)
- Recent commit picker (last 20 commits)
- Special targets (All Uncommitted, Staging Area, Working Directory)
- Resolved hash display

### File Watching & Live Reload
- Real-time diff refresh on file changes (working dir / staging area)
- SSE-based file watching via `@parcel/watcher`
- Comments-aware reload (comments preserved across reloads)
- Heartbeat-based tab-close detection (auto-shutdown unless `--keep-alive`)
- Background server mode (`--background` with JSON output for scripts)

### AI Agent Integration
- `npx skills add yoshiko-pg/difit` installs agent skills
- `difit` skill: ask user for review through difit after code changes
- `difit-review` skill: review a specific diff or PR with findings preloaded as comments
- `--comment` flag injects initial review threads programmatically
- Comment prompt format optimized for LLM context windows

### Accessibility & Appearance
- Dark theme (default, GitHub-like) and Light theme
- Deuteranopia (red-green colorblind) color scheme
- Resizable sidebar file tree (drag handle)
- Full keyboard navigation (Shift+? for shortcuts)
- Mobile-responsive layout (auto-switches to unified view)
- Sparkle animation on completing all reviews
- Configurable font size and font family
- External editor integration (VS Code, Cursor, any CLI editor)
- Syntax theme selection (GitHub Dark, Light, High Contrast, and more)
