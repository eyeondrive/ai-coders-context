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

4. **Committed locally** as `feat: add Abacus AI Desktop (DeepAgent) to tool registry`. Not pushed — awaiting review.

### Open Items

- Push to remote once David reviews the commit
- Configure the MCP server in Claude Code's settings (add to `.claude/settings.json` or equivalent)
- Test the actual sync workflow: create `.context/` content and export to all three tools
- Abacus AI agents/skills support — revisit when DeepAgent Desktop documents those features
