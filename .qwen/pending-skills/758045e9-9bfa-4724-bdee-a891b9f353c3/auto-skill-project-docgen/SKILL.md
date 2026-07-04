---
name: project-docgen
description: Analyze any software project and produce structured documentation — business summary, architecture, design summary, UI design, CLI design, and workflow — saved as separate Markdown files.
source: auto-skill
extracted_at: '2026-07-04T12:26:11.441Z'
---

# Project DocGen Skill

Analyze a software project codebase and produce six structured Markdown documents covering business context, architecture, design decisions, UI/UX, CLI interface, and user/developer workflows.

## When to Use

Invoke this skill when the user asks to:
- "Analyze this project and write documentation"
- "Generate docs for this codebase"
- "Document the project architecture"
- "Write a business/architecture/design summary"
- "Save project analysis into files"

## Output

Six Markdown files are saved to a user-specified output directory (default: `./qwendoc/`):

| File | Content |
|------|---------|
| `<project>_business_summary.md` | Tagline, executive overview, market opportunity, business model, traction, metrics, differentiators, feature catalog |
| `<project>_architecture.md` | System architecture diagram, module tree, data flow pipelines, key design decisions, security model |
| `<project>_design_summary.md` | Full tech stack, design patterns with code examples, state management, CSS architecture, performance considerations |
| `<project>_ui_design.md` | ASCII wireframe layout, header/sidebar/main descriptions, component details, all pages, modals, dialogs, keyboard shortcuts, responsive behavior |
| `<project>_cli_design.md` | Command structure, arguments/options, special keywords, exit codes, environment variables, JSON output formats |
| `<project>_workflow.md` | User workflows, development workflows, data lifecycle diagrams, API endpoint table, state machines |

## How to Use

### Step 1 — Explore the project thoroughly

Read these files in order to build a complete mental model:

1. **`README.md`** — Project description, quick start, usage, architecture overview
2. **`package.json`** — Dependencies, scripts, binary entry points, metadata
3. **`tsconfig.json` / `pyproject.toml` / `Cargo.toml`** — Language/framework config
4. **CLI entry point** — Command definitions, argument parsing, action handlers
5. **Server entry point** — Route definitions, middleware, lifecycle
6. **Frontend entry point** — Component tree, routing, state management
7. **Type definitions** — Core data models, interfaces, enums

For each area, read enough to understand patterns, not every line. Focus on:
- File and directory organization
- Key data structures and their relationships
- Build/run/test scripts
- Configuration files

### Step 2 — Identify architectural layers

Map the project to a 3+ tier structure:

```
CLI/Tool Layer → Server/Backend Layer → Client/Frontend Layer
```

Document:
- Module boundaries and file organization
- Data flow between layers
- Entry points and lifecycle hooks
- External dependencies and integration points

### Step 3 — Extract design patterns

Look for:
- **Structural patterns** — Registry, strategy, adapter, pipeline
- **State management** — Where state lives, how it's persisted, how it's synchronized
- **Concurrency** — File watching, SSE, background processes, versioning
- **Optimization** — Caching, lazy loading, debouncing, batching
- **Security** — Path traversal protection, command injection prevention, CORS

### Step 4 — Document the user interface

For projects with a UI:
- Draw ASCII wireframe layouts (header, sidebar, main content)
- Describe each page/component with purpose and key actions
- List all dialog/modal components
- Document responsive behavior and keyboard shortcuts

### Step 5 — Document the CLI

For CLI tools:
- Full command structure tree
- Every argument, option, flag with type and default
- Special keywords or shorthand notations
- Exit codes and error handling
- Environment variables

### Step 6 — Write workflow diagrams

Cover:
- **User workflows** — Common task flows from start to finish
- **Development workflows** — Build, test, deploy, benchmark
- **Data lifecycle** — How state flows through the system
- **State machines** — For reactive/watching components

### Constraints

**MUST DO:**
- Read actual source code — do not fabricate endpoints, components, or features
- Reference actual file paths for all source materials
- Include quantitative metrics (files, endpoints, components, tests)
- Use ASCII art for architecture and UI diagrams

**MUST NOT DO:**
- Do not use placeholder text or "TBD" — leave sections empty if you can't determine details
- Do not generate generic template content — every detail must come from code analysis
- Do not skip error handling or edge cases

### Output format

All files use Markdown with:
- A top-level `# Title` heading
- Second-level headings per major section (`## Section`)
- Third-level headings for subsections (`### Subsection`)
- Code blocks for examples, diagrams, and configuration
- Tables where structured data is presented (metrics, options, endpoints)
