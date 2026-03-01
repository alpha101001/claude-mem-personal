# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

# Claude-Mem: AI Development Instructions

Claude-mem is a Claude Code plugin providing persistent memory across sessions. It captures tool usage, compresses observations using the Claude Agent SDK, and injects relevant context into future sessions.

## Architecture Overview

### Core Components

**5 Lifecycle Hooks**: Setup → SessionStart → UserPromptSubmit → PostToolUse → Stop

**Worker Service** (`src/services/worker-service.ts`) - Refactored orchestrator (~300 lines) managing 8 service layers:
- DatabaseManager - SQLite operations
- SessionManager - Session lifecycle
- SearchManager - Semantic search & Chroma
- SessionEventBroadcaster - SSE events
- SDKAgent - Claude Agent SDK compression
- GeminiAgent/OpenRouterAgent - Alternative AI providers
- SettingsManager - Configuration
- TimelineService - Context formatting

**Database** (`src/services/sqlite/`) - SQLite3 at `~/.claude-mem/claude-mem.db`, Bun's built-in `bun:sqlite`, WAL mode, 7 tables (sessions, memories, overviews, diagnostics, transcript_events, prompts, observations)

**Vector Search** (`src/services/sync/ChromaSync.ts`) - Chroma vector database at `~/.claude-mem/chroma/` for semantic embeddings

**MCP Server** (`src/servers/mcp-server.ts`) - Model Context Protocol server for mem-search tool integration with Claude Desktop

**Search Skill** (`plugin/skills/mem-search/SKILL.md`) - HTTP API for querying past work, auto-invoked when users ask about history

**Planning Skill** (`plugin/skills/make-plan/SKILL.md`) - Orchestrator for phased implementation plans

**Execution Skill** (`plugin/skills/do/SKILL.md`) - Orchestrator for executing phased plans using subagents

**Viewer UI** (`src/ui/viewer/`) - React interface at http://localhost:37777, built to `plugin/ui/viewer.html`

### Data Flow

1. **Capture**: UserPromptSubmit hook → PostToolUse hook calls worker API
2. **Storage**: Worker stores observations in SQLite via SessionStore
3. **Vector Sync**: ChromaSync embeds observations into Chroma
4. **Retrieval**: SessionStart hook requests context from worker
5. **Compression**: SDKAgent compresses using Claude Agent SDK
6. **Injection**: Context injected into Claude system prompt
7. **UI**: React viewer displays memory stream at localhost:37777

### Build System

The build process (`scripts/build-hooks.js`) uses esbuild to bundle 3 executables:

1. **worker-service.cjs** - Express HTTP API on port 37777, Bun-managed
2. **mcp-server.cjs** - MCP server for mem-search integration
3. **context-generator.cjs** - Context building service

Build steps:
- Bundle TypeScript → CommonJS with esbuild
- Build React viewer UI → single HTML file
- Generate `plugin/package.json` with Tree-Sitter dependencies (required for code parsing)
- Make executables with shebangs (`#!/usr/bin/env bun` or `#!/usr/bin/env node`)
- External dependencies: `bun:sqlite`, Tree-Sitter parsers, Chroma embedding providers

## Development Commands

### Build & Sync
```bash
npm run build-and-sync        # Build, sync to marketplace, restart worker
npm run build                 # Build hooks and executables only
npm run sync-marketplace      # Sync plugin/ to ~/.claude/plugins/marketplaces/thedotmack/
```

### Testing
```bash
npm test                      # Run all tests with Bun
npm run test:sqlite           # Database layer tests
npm run test:agents           # AI agent tests (SDK, Gemini, OpenRouter)
npm run test:search           # Search functionality tests
npm run test:context          # Context injection tests
npm run test:infra            # Process management tests
npm run test:server           # HTTP server tests
```

**Test Structure**: `tests/sqlite/`, `tests/worker/`, `tests/context/`, `tests/infrastructure/`, `tests/server/`, `tests/hooks/`

### Worker Management
```bash
npm run worker:start          # Start worker service
npm run worker:stop           # Stop worker service
npm run worker:restart        # Restart worker service
npm run worker:status         # Check worker status
npm run worker:logs           # View last 50 lines of today's log
npm run worker:tail           # Tail worker logs in real-time
```

### Queue Management
```bash
npm run queue                 # Check pending queue status
npm run queue:process         # Process pending queue items
npm run queue:clear           # Clear failed queue items
```

### Other
```bash
npm run bug-report            # Generate comprehensive bug report
npm run changelog:generate    # Generate CHANGELOG.md (automated)
```

## Marketplace Sync Process

When you run `npm run sync-marketplace`:

1. Syncs files from `/plugin/` to `~/.claude/plugins/marketplaces/thedotmack/`
2. Respects `.gitignore` exclusions
3. Installs dependencies in `~/.claude/plugins/cache/thedotmack/claude-mem/{version}/`
4. Triggers worker restart via HTTP call to localhost:37777

## File Locations

- **Source**: `<project-root>/src/`
- **Built Plugin**: `<project-root>/plugin/`
- **Installed Plugin**: `~/.claude/plugins/marketplaces/thedotmack/`
- **Plugin Cache**: `~/.claude/plugins/cache/thedotmack/claude-mem/{version}/`
- **Database**: `~/.claude-mem/claude-mem.db`
- **Chroma**: `~/.claude-mem/chroma/`
- **Settings**: `~/.claude-mem/settings.json`
- **Logs**: `~/.claude-mem/logs/worker-*.log`

## Privacy Tags

`<private>content</private>` - User-level privacy control (manual, prevents storage)

**Implementation**: Tag stripping happens at hook layer (edge processing) before data reaches worker/database. See `src/utils/tag-stripping.ts` for shared utilities.

## Exit Code Strategy

Claude-mem hooks use specific exit codes per Claude Code's hook contract:

- **Exit 0**: Success or graceful shutdown (prevents Windows Terminal tab accumulation)
- **Exit 1**: Non-blocking error (stderr shown to user, continues)
- **Exit 2**: Blocking error (stderr fed to Claude for processing)

**Philosophy**: Worker/hook errors exit with code 0 to prevent Windows Terminal tab accumulation. The wrapper/plugin layer handles restart logic. ERROR-level logging is maintained for diagnostics.

See `private/context/claude-code/exit-codes.md` for full hook behavior matrix.

## Configuration

Settings are managed in `~/.claude-mem/settings.json`. The file is auto-created with defaults on first run.

Configure: AI model, worker port, data directory, log level, context injection settings.

## Requirements

- **Node.js**: 18.0.0 or higher
- **Bun**: 1.0.0 or higher (auto-installed if missing)
- **uv**: Python package manager (auto-installed if missing, provides Python for Chroma)

## Documentation

**Public Docs**: https://docs.claude-mem.ai (Mintlify)
**Source**: `docs/public/` - MDX files, edit `docs.json` for navigation
**Deploy**: Auto-deploys from GitHub on push to main

## Pro Features Architecture

Claude-mem is designed with clean separation between open-source core and optional Pro features.

**Open-Source Core** (this repository):
- All worker API endpoints on localhost:37777 remain fully open and accessible
- Pro features are headless - no proprietary UI elements in this codebase
- Pro integration points are minimal: settings for license keys, tunnel provisioning logic
- Architecture ensures Pro features extend rather than replace core functionality

**Pro Features** (coming soon, external):
- Enhanced UI (Memory Stream) connects to same localhost:37777 endpoints as open viewer
- Additional features: advanced filtering, timeline scrubbing, search tools
- Access gated by license validation, not by modifying core endpoints
- Users without Pro licenses continue using full open-source viewer UI without limitation

This architecture preserves the open-source nature while enabling sustainable development through optional paid features.

## Important Notes

- **Changelog**: Auto-generated, never edit manually
- **Built Files**: Never read `plugin/scripts/*` or `dist/*` - always read source files in `src/`
- **Database Runtime**: Uses Bun's built-in `bun:sqlite`, no external SQLite dependencies
- **Tree-Sitter**: Dependencies installed in cache directory for code parsing
