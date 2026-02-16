# Project Notes — eyeondrive/ai-coders-context

Fork-specific notes for David's adoption of ai-coders-context. Records discoveries, decisions, and changes made to the fork.

---

## 2026-02-15: Initial Setup and Abacus AI Support

### What This Project Is

ai-coders-context is an MCP server that manages a `.context/` directory as a single source of truth for AI coding context, then syncs it to tool-specific formats (Claude Code, Cursor, Antigravity, etc.). We forked it from `vinilana/ai-coders-context` because David works across three AI environments: Claude Code, Google Antigravity, and Abacus AI (DeepAgent Desktop).

The fork already supported Claude Code and Antigravity out of the box. We needed to add Abacus AI.

### Security / Architecture Review

Before building, we reviewed the MCP server's architecture to understand what it does and where data goes:

- **Runs 100% locally.** The MCP server communicates with Claude Code via stdin/stdout only — no network sockets, no HTTP listeners.
- **All 9 tools are local filesystem operations** — reading/writing files in `.context/`, syncing between AI tool formats. No data leaves the machine in MCP mode.
- **No telemetry, no analytics, no phoning home.**
- **Two optional external calls** (neither triggered in MCP server mode):
  - **Version check:** CLI startup pings the npm registry. Disabled with `AI_CONTEXT_DISABLE_UPDATE_CHECK=true`.
  - **LLM provider calls:** Only when using the CLI directly for AI-powered content generation. When used through Claude Code as an MCP server, Claude Code itself is the LLM — no separate API keys or external calls needed.

### What We Did

1. **Fixed broken clone.** The existing `ai-coders-context` directory had a stale `.git/config.lock` preventing all git operations. Deleted it and re-cloned fresh from `eyeondrive/ai-coders-context`.

2. **Added Abacus AI Desktop (DeepAgent) to the tool registry** (`src/services/shared/toolRegistry.ts`). New entry:
   - ID: `abacus`
   - Directory prefix: `.deepagent-desktop`
   - Capabilities: rules only (agents and skills not yet documented for the platform)
   - Rules stored in `.deepagent-desktop/rules/` as markdown files with YAML frontmatter
   - Inserted before the Aider entry in the registry array

3. **Built and verified.** `npm run build` succeeded cleanly. Confirmed `abacus` and `deepagent-desktop` appear in compiled output (`dist/services/shared/toolRegistry.js`).

4. **Built and committed** as `feat: add Abacus AI Desktop (DeepAgent) to tool registry` (`944cf9e`).

5. **Added these project notes** to `.context/docs/project-notes.md` and updated the docs README index with a "Fork-Specific" section. Committed as `docs: add fork-specific project notes to .context/docs` (`c672aaf`).

6. **Pushed both commits** to `eyeondrive/ai-coders-context` on GitHub.

### MCP Server Configuration — Resolved

Configured at the workspace level in `eyeondrive/.mcp.json` (not `~/.claude.json`), pointing to the local fork build. This scopes the MCP to the eyeondrive workspace rather than polluting global config. Documented in the workspace CLAUDE.md under "AI Tool Ecosystem" so future sessions know where it lives and how to fix it if the MCP fails to connect.

---

## 2026-02-15: Sync Workflow Test & Workspace Infrastructure

### Sync Workflow — Verified

Tested the full export pipeline from `.context/` to all tool formats:

1. **`export-rules --preset all --force`** — exported docs to 14 targets (CLAUDE.md, .cursorrules, .agent/rules/, .deepagent-desktop/rules/, etc.). All succeeded.
2. **`sync-agents --preset all`** — synced 8 agent playbooks to 8 tool targets (64 files total) via symlinks back to `.context/agents/`. Zero failures.
3. **`skill init`** — scaffolded 10 built-in skill templates (commit-message, pr-review, code-review, test-generation, documentation, refactoring, bug-investigation, feature-breakdown, api-design, security-audit).
4. **`skill export --preset all`** — exported 10 skills to 4 targets (Claude Code, Antigravity, Gemini, Codex).

All three of David's tools confirmed working:
- **Claude Code** (`.claude/`) — CLAUDE.md + agents/ (symlinks) + skills/
- **Antigravity** (`.agent/`) — rules/ + agents/ (symlinks) + workflows/
- **DeepAgent** (`.deepagent-desktop/`) — rules/ only (as expected)

### Non-Destructive Export Convention

Discovered that `export-rules` overwrites CLAUDE.md entirely — destructive for hand-written project rules. Established a marker convention:

- `<!-- GENERATED:AI-CONTEXT:START -->` / `<!-- GENERATED:AI-CONTEXT:END -->` wraps auto-generated content
- Hand-written content stays outside the markers and survives re-export
- For Antigravity/DeepAgent: workspace rules go in a separate `workspace-rules.md` alongside the generated `README.md` — no collision

Applied this to ai-coders-context as the reference template.

### Workspace-Level Changes

- **Session Start picker** added to workspace CLAUDE.md — asks which project at session start, lists active projects only, offers new/one-off/archive options
- **AGENT-ROLES.md instruction** made imperative — "you MUST read" instead of "read for the full breakdown"
- **Archived dead projects** — Applescript Printer Refresh and mmi.us moved to `old-projects/`
- **Course projects** (bookbot, Asteroids) kept but excluded from the picker — on hold

### MCP Server — Confirmed Working

The `.mcp.json` at workspace root was picked up by Claude Code. The ai-context MCP server connects and tools are available in sessions. No manual `claude mcp add` needed — the `.mcp.json` approach works.

### Open Items

- Skills are generic templates — `skill fill` would customize them to the project via LLM
- Extra tool directories from `--preset all` export (`.cursor/`, `.windsurf/`, etc.) could be cleaned up — David only uses three tools
- Consider configuring the MCP server for Antigravity as well (`~/.gemini/mcp_config.json`)
- Abacus AI agents/skills support — revisit when DeepAgent Desktop documents those features
- The generated content convention is manual — could eventually be built into ai-coders-context as a merge-aware export mode

---

## 2026-02-15: Fix venv Exclude Bug in Semantic Analysis

### The Problem

When using `fillSingle` on the mbti_sorter_v1 project, the semantic context builder was scanning `venv/` (Python virtual environment) and returning thousands of `site-packages/` paths instead of actual project code. Passing `exclude: ["venv"]` to `init` had no effect on `fillSingle`.

### Root Causes (Three Bugs)

1. **`DEFAULT_EXCLUDE_PATTERNS` missing Python venvs.** The default exclude list in `types.ts` covered `node_modules`, `__pycache__`, etc. but not `venv` or `.venv` — the standard Python virtual environment directories.

2. **`SemanticContextBuilder` had its own shorter hardcoded exclude list.** `contextBuilder.ts` line 34 defined `exclude: ['node_modules', 'dist', 'build', '.git', 'coverage']` instead of using `DEFAULT_EXCLUDE_PATTERNS` from `types.ts`. So even after fixing the defaults, the context builder wouldn't use them.

3. **User exclude patterns from `init` were never persisted.** `initializeContextTool.ts` accepted `exclude` and used it for the `FileMapper` during init, then threw it away. `fillScaffoldingTool.ts` created `new SemanticContextBuilder()` with zero options, so `fillSingle` always fell back to hardcoded defaults. There was no "chain of custody" between `init` and `fillSingle`.

### Fixes Applied

| File | Change |
|------|--------|
| `src/services/semantic/types.ts` | Added `'venv'` and `'.venv'` to `DEFAULT_EXCLUDE_PATTERNS` |
| `src/services/semantic/contextBuilder.ts` | Replaced hardcoded exclude list with `DEFAULT_EXCLUDE_PATTERNS` import |
| `src/services/ai/tools/initializeContextTool.ts` | Persists user `exclude` patterns to `.context/config.json` during init |
| `src/services/ai/tools/fillScaffoldingTool.ts` | `getOrBuildContext()` loads `.context/config.json` and merges user excludes with defaults before creating `SemanticContextBuilder` |

### Design Decisions

- **Config file location:** `.context/config.json` — lives alongside the scaffolding it configures. Created only when user provides custom excludes.
- **Merge, don't replace:** User excludes are merged with `DEFAULT_EXCLUDE_PATTERNS` in `getOrBuildContext()`, not passed as a replacement. This prevents users from accidentally removing default excludes like `node_modules`.
- **Backwards compatible:** Projects without a `config.json` behave exactly as before — defaults only. The `venv`/`.venv` addition to defaults is the only behavioral change for existing users.

### Testing

Built cleanly (`npm run build`). MCP server needs restart to pick up the new code (it's a long-running stdio process). Verified all three fixes present in compiled `dist/` output.

---

## 2026-02-15: Marker-Aware Merge for Export Rules

### The Problem

`exportRules` (both CLI and MCP) uses `fs.writeFile` to overwrite single-format targets like `CLAUDE.md` and `AGENTS.md`. This destroys any hand-written content in those files. The `<!-- GENERATED:AI-CONTEXT:START/END -->` marker convention was established as a workaround but the tool itself didn't honor it.

### The Fix

Modified `exportRulesService.ts` to detect and respect markers:

1. **New file** — generated content is wrapped in markers automatically
2. **Existing file with markers** — only the content between markers is replaced. Hand-written content outside markers is preserved. No `--force` needed (always safe).
3. **Existing file without markers, no force** — skipped (existing behavior)
4. **Existing file without markers, with force** — full overwrite, but content is now wrapped in markers so future exports will merge safely

Added `wrapWithMarkers()` and `mergeWithMarkers()` private methods to `ExportRulesService`.

### Also: Gitignore + Cleanup for Generated Tool Exports

Added all tool-specific output directories to `.gitignore`. These are generated artifacts from `export-rules --preset all` and shouldn't be committed. Each user/fork generates their own.

Removed `AGENTS.md` from git tracking (`git rm --cached`) — it's 100% generated content, now covered by `.gitignore`. Committed the updated `CLAUDE.md` which has the proper format: hand-written project rules at top, generated section inside `GENERATED:AI-CONTEXT` markers at bottom. This is the reference implementation of the marker-aware merge pattern.
