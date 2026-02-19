# Claude Reflection Plugin

A globally installable [Claude Code](https://claude.ai/code) plugin that analyzes session logs and proposes improvements to your CLAUDE.md, memory files, agent definitions, and skill definitions.

## What It Does

The Reflection Plugin reads your Claude Code session logs (via the `cclogviewer` MCP server), identifies patterns — mistakes, user corrections, workflow inefficiencies — and generates specific, evidence-based recommendations to improve your setup.

### Three Reflection Modes

| Mode | Command | Analyzes | Suggests improvements to |
|------|---------|----------|--------------------------|
| **Direct** | `/reflect:run direct` | Direct Claude usage sessions | CLAUDE.md, memory files |
| **Agent** | `/reflect:run agent <name>` | Agent sessions | Agent `.md` definition |
| **Skill** | `/reflect:run skill <name>` | Skill sessions | SKILL.md, @docs/, @templates/ |

## Installation

### Option 1: Inside Claude Code (interactive)

```
/plugin marketplace add SeleznovIvan/claude-reflection-plugin
/plugin install reflect@claude-reflection-plugin
```

Restart Claude Code to load the plugin.

### Option 2: From terminal

```bash
claude plugin marketplace add SeleznovIvan/claude-reflection-plugin
claude plugin install reflect@claude-reflection-plugin --scope user
```

### Post-install setup

Run the setup skill to install the `cclogviewer` MCP dependency and configure your project:

```
/reflect:setup
```

This will:
1. Install `cclogviewer-mcp` binary
2. Register it as an MCP server with Claude Code
3. Auto-discover your project name from session logs
4. Create `.claude/reflection-setup.json` in your project

## Usage

```bash
# Analyze direct Claude usage (last 7 days, up to 5 sessions)
/reflect:run direct

# Analyze an agent with custom range
/reflect:run agent playwright-e2e-specialist --sessions 5 --days 14

# Analyze a skill
/reflect:run skill managing-storage --sessions 3
```

### Options

| Option | Default | Description |
|--------|---------|-------------|
| `--sessions N` | 5 | Maximum sessions to analyze |
| `--days N` | 7 | Look back N days |

## How It Works

The reflection loop runs in 6 phases:

1. **Parse & Initialize** — Validate arguments, load project config, create workspace
2. **Fetch Session Data** — Retrieve session logs, stats, timeline, and tool usage from `cclogviewer` MCP
3. **Process Sessions** — Parallel `reflection-worker` agents (Sonnet) produce per-session summaries with tool sequence analysis and definition compliance checks
4. **Analyze Summaries** — `reflection-analyzer` agent (Opus) performs cross-session analysis with mode-specific strategy
5. **User Approval** — Present each recommendation for approval/rejection, apply approved changes
6. **Final Summary** — Report results, update reflection history

```mermaid
flowchart TD
    Start(["/reflect:run agent my-agent --sessions 3"])

    subgraph Phase1["Phase 1: Parse & Initialize"]
        P1A[Parse mode, name, options]
        P1B[Load reflection-setup.json]
        P1C[Validate target exists]
        P1D[Create workspace folder]
    end

    subgraph Phase2["Phase 2: Fetch Session Data"]
        P2A[Find sessions via cclogviewer]
        P2B[Save JSONL logs to workspace]
        P2C[Fetch session stats]
        P2D[Fetch tool usage stats & timeline]
        P2E[Fetch error contexts]
        P2F[Save session metadata]
    end

    subgraph Phase3["Phase 3: Process Sessions (Parallel)"]
        W1["reflection-worker\n(Sonnet)"]
        W2["reflection-worker\n(Sonnet)"]
        W3["reflection-worker\n(Sonnet)"]
        W1 --> S1[Summary 1\n+ sequence analysis\n+ definition compliance]
        W2 --> S2[Summary 2\n+ sequence analysis\n+ definition compliance]
        W3 --> S3[Summary 3\n+ sequence analysis\n+ definition compliance]
    end

    subgraph Phase4["Phase 4: Cross-Session Analysis"]
        P4A["reflection-analyzer\n(Opus)"]
        P4B[Mode-specific strategy doc]
        P4A --- P4B
        P4C[Analysis report with\nprioritized recommendations]
    end

    subgraph Phase5["Phase 5: User Approval"]
        P5A{For each recommendation}
        P5B[/"Show diff in AskUserQuestion\n(inline markdown preview)"/]
        P5C[Apply approved changes\nvia Edit tool]
        P5D[Log to applied-changes.md]
    end

    Phase6([Phase 6: Final Summary\nUpdate history & config])

    Start --> P1A --> P1B --> P1C --> P1D
    P1D --> P2A --> P2B & P2C & P2D
    P2B & P2C & P2D --> P2E --> P2F
    P2F --> W1 & W2 & W3
    S1 & S2 & S3 --> P4A
    P4A --> P4C
    P4C --> P5A
    P5A --> P5B --> P5C --> P5D
    P5D --> P5A
    P5A -- done --> Phase6
```

### Suggestion Routing

Recommendations are automatically routed to the correct file:

- **Direct mode**: Prescriptive rules → `CLAUDE.md` | Descriptive learnings → memory files
- **Agent mode**: All suggestions → agent `.md` definition
- **Skill mode**: Suggestions → `SKILL.md`, `@docs/`, or `@templates/` files

## Plugin Structure

```
├── .claude-plugin/plugin.json      # Plugin manifest
├── .mcp.json                       # MCP dependency (cclogviewer)
├── CLAUDE.md                       # Plugin instructions
├── skills/
│   ├── run/
│   │   ├── SKILL.md                # Main orchestrator (/reflect:run)
│   │   └── docs/
│   │       ├── PROCESSING.md       # Worker instructions
│   │       ├── ANALYSIS-DIRECT.md  # Direct mode strategy
│   │       ├── ANALYSIS-AGENT.md   # Agent mode strategy
│   │       └── ANALYSIS-SKILL.md   # Skill mode strategy
│   └── setup/
│       └── SKILL.md                # /reflect:setup
└── agents/
    ├── reflection-worker.md        # Per-session processor (Sonnet)
    └── reflection-analyzer.md      # Cross-session analyzer (Opus)
```

## Requirements

- [Claude Code](https://claude.ai/code) CLI
- `cclogviewer-mcp` binary (installed via `/reflect:setup`)

## License

MIT
