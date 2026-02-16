# Session Checkpoint — 2026-02-15: Fix venv Exclude in Semantic Analysis

## Context

While setting up `.context/` scaffolding for the mbti_sorter_v1 project to hand off to Antigravity (for building `developer-demo.html`), the `fillSingle` semantic analysis was returning thousands of `venv/lib/python3.12/site-packages/` paths instead of actual project code. The `exclude: ["venv"]` parameter passed to `init` had no effect on `fillSingle`.

## What Was Done

Fixed three related bugs in the semantic analysis pipeline:

1. **`types.ts`** — Added `'venv'` and `'.venv'` to `DEFAULT_EXCLUDE_PATTERNS`
2. **`contextBuilder.ts`** — Replaced its own shorter hardcoded exclude list with `DEFAULT_EXCLUDE_PATTERNS` import (was inconsistent with `codebaseAnalyzer.ts`)
3. **`initializeContextTool.ts`** — Persists user-provided `exclude` patterns to `.context/config.json` during init
4. **`fillScaffoldingTool.ts`** — `getOrBuildContext()` loads `.context/config.json` and merges user excludes with defaults before creating `SemanticContextBuilder`

## Files Changed

- `src/services/semantic/types.ts`
- `src/services/semantic/contextBuilder.ts`
- `src/services/ai/tools/initializeContextTool.ts`
- `src/services/ai/tools/fillScaffoldingTool.ts`
- `.context/docs/project-notes.md`

## Build Status

Clean build (`npm run build`). All fixes confirmed in `dist/` output.

## Not Yet Done

- MCP server restart needed to pick up new build (long-running stdio process)
- End-to-end test: re-run `init` + `fillSingle` on mbti_sorter_v1 after restart
- The mbti_sorter_v1 Antigravity handoff is still pending — blocked on this fix
- Not pushed to remote yet

## Next Steps

1. Restart Claude Code to reload the MCP server with new build
2. Re-init `.context/` on mbti_sorter_v1 with `exclude: ["venv"]`
3. Run `fillSingle` on key docs to verify venv is excluded
4. Export to Antigravity format and hand off `developer-demo.html` build
