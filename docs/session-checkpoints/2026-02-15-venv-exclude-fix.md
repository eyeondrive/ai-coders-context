# Session Checkpoint — 2026-02-15: Semantic Exclude Fix + Marker-Aware Merge

## Context

While setting up `.context/` scaffolding for the mbti_sorter_v1 project to hand off to Antigravity (for building `developer-demo.html`), two bugs surfaced:

1. The `fillSingle` semantic analysis was scanning `venv/` and returning thousands of `site-packages/` paths instead of project code. User-provided excludes from `init` were silently dropped.
2. The `exportRules` command was wiping hand-written content in CLAUDE.md and AGENTS.md on every export — the `<!-- GENERATED:AI-CONTEXT:START/END -->` marker convention existed but the tool didn't honor it.

## Commit 1: venv Exclude Fix (`39a9411`)

Fixed three related bugs in the semantic analysis pipeline:

| File | Change |
|------|--------|
| `src/services/semantic/types.ts` | Added `'venv'`, `'.venv'` to `DEFAULT_EXCLUDE_PATTERNS` |
| `src/services/semantic/contextBuilder.ts` | Replaced hardcoded exclude list with `DEFAULT_EXCLUDE_PATTERNS` import |
| `src/services/ai/tools/initializeContextTool.ts` | Persists user `exclude` patterns to `.context/config.json` |
| `src/services/ai/tools/fillScaffoldingTool.ts` | Loads `.context/config.json` and merges user excludes with defaults |

## Commit 2: Marker-Aware Merge + Gitignore

Made `exportRules` preserve hand-written content outside marker boundaries:

| File | Change |
|------|--------|
| `src/services/export/exportRulesService.ts` | Marker detection, `wrapWithMarkers()`, `mergeWithMarkers()` |
| `.gitignore` | Added all generated tool export directories |
| `.context/docs/project-notes.md` | Updated with both fixes |

### Merge behavior:
- **New file** → content wrapped in markers automatically
- **Existing file with markers** → replace between markers only (always safe, no --force needed)
- **Existing file without markers, no force** → skip
- **Existing file without markers, with force** → overwrite with markers (future exports safe)

## Build Status

Both commits build cleanly. Not pushed to remote yet.

## Upstream Contribution Candidates

Both fixes are generic improvements worth contributing to `vinilana/ai-coders-context`:
- venv exclude: affects any Python project user
- Marker-aware merge: prevents data loss for all users with hand-written rules files

The Abacus AI/DeepAgent addition from the earlier session (`944cf9e`) is also a candidate.

## Not Yet Done

- MCP server restart needed to pick up new builds
- End-to-end test on mbti_sorter_v1 after restart
- The mbti_sorter_v1 Antigravity handoff is still pending
- PRs to upstream not yet opened
