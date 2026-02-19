# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This repository contains the **Reflection Plugin** — a globally installable Claude Code plugin that analyzes session logs from the `cclogviewer` MCP server, identifies patterns and issues, and proposes improvements to CLAUDE.md, memory files, agent definitions, and skill definitions.

The plugin supports **3 reflection scenarios**:
- **`direct`** — Analyze direct Claude usage sessions → suggest improvements to CLAUDE.md and memory files
- **`agent`** — Analyze agent sessions → suggest improvements to agent `.md` definitions
- **`skill`** — Analyze skill sessions → suggest improvements to SKILL.md and referenced docs

## Plugin Structure

```
reflection-plugin/                          # Git repo root (installable via claude plugin install)
├── .claude-plugin/
│   └── plugin.json                         # Plugin metadata + manifest
├── .mcp.json                               # MCP server dependency (cclogviewer)
├── CLAUDE.md                               # This file — plugin instructions
├── skills/
│   ├── run/
│   │   ├── SKILL.md                        # Main entry point (/reflect:run)
│   │   └── docs/
│   │       ├── PROCESSING.md              # Worker agent instructions
│   │       ├── ANALYSIS-DIRECT.md         # Strategy for direct Claude sessions
│   │       ├── ANALYSIS-AGENT.md          # Strategy for agent sessions
│   │       └── ANALYSIS-SKILL.md          # Strategy for skill sessions
│   └── setup/
│       └── SKILL.md                        # /reflect:setup — install cclogviewer MCP
├── agents/
│   ├── reflection-worker.md                # Per-session summary producer (Sonnet)
│   └── reflection-analyzer.md              # Cross-session analyzer (Opus)
└── archive/
    └── v1/                                 # Archived old files (pre-plugin)
```

## Architecture

The reflection loop operates in 6 phases:

1. **Parse & Initialize** — Parse mode (direct/agent/skill), load `reflection-setup.json`, validate target, create workspace
2. **Fetch Session Data** — Use `cclogviewer` MCP tools to retrieve session logs (saved to file via `output_path`), stats, and error contexts
3. **Process Sessions** — Launch `reflection-worker` agents in parallel (Sonnet, one per session) to create structured summaries
4. **Analyze Summaries** — Launch `reflection-analyzer` agent (Opus) with mode-specific strategy doc to generate recommendations
5. **User Approval** — Present each recommendation with target file routing, apply approved changes via Edit tool
6. **Final Summary** — Report results, update `reflection-setup.json` history, save artifacts

### MCP Tool Usage (Critical)

- **Always include `project` parameter** in ALL `cclogviewer` MCP calls — the project name comes from `reflection-setup.json`
- **Use `get_session_logs` with `output_path`** to save JSONL directly to file (avoids returning 69K+ tokens to context)
- Use `get_session_stats` (~5K tokens) for lightweight stats returned to context
- Use `get_logs_around_entry` (~2K tokens) for error root cause analysis (before/after context)

### Suggestion Routing

Recommendations are routed to the correct target file based on mode:
- **`direct` mode**: CLAUDE.md (prescriptive instructions) or memory files (descriptive learnings)
- **`agent` mode**: Agent `.md` definition file
- **`skill` mode**: SKILL.md or its referenced `@docs/` and `@templates/` files

### State File

`{PROJECT_ROOT}/.claude/reflection-setup.json` stores:
- `project_name` — auto-discovered from cclogviewer
- `project_root` — absolute path
- `reflection_history[]` — log of past reflection runs with recommendations stats

Created via `/reflect:setup`.

## Workspace Output Structure

Each reflection run creates:
```
{PROJECT_ROOT}/.claude/claude-workspace/reflection/{mode}-{name}-{timestamp}/
├── raw-data/          # Session logs (JSONL) + stats JSON files from MCP
├── summaries/         # Processed session summaries (one per session)
├── analysis/          # Cross-session analysis report
├── config.json        # Run configuration and status
└── applied-changes.md # Change log (approved + rejected)
```

## Versioning

When releasing changes, **always bump the version in BOTH files**:
- `.claude-plugin/plugin.json` — the plugin metadata version
- `.claude-plugin/marketplace.json` — the marketplace manifest version (this is what the installer reads)

If only `plugin.json` is bumped, users will not see the update — `marketplace.json` is the source of truth for the displayed/installed version.

## Usage

```bash
# First time: install cclogviewer and configure project
/reflect:setup

# Analyze direct Claude usage
/reflect:run direct --sessions 3 --days 14

# Analyze an agent
/reflect:run agent playwright-e2e-specialist --sessions 5

# Analyze a skill
/reflect:run skill managing-storage --sessions 3
```
