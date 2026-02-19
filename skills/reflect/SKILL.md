---
name: reflect
description: Analyze Claude session logs and propose improvements to definitions, CLAUDE.md, or memory files
allowed-tools: [Read, Write, Edit, Glob, Grep, Bash, Task, TodoWrite, AskUserQuestion,
                mcp__cclogviewer__list_sessions, mcp__cclogviewer__list_projects,
                mcp__cclogviewer__get_session_stats, mcp__cclogviewer__get_session_logs,
                mcp__cclogviewer__get_logs_around_entry,
                mcp__cclogviewer__list_agents, mcp__cclogviewer__get_agent_sessions,
                mcp__cclogviewer__search_logs, mcp__cclogviewer__get_session_errors,
                mcp__cclogviewer__get_session_timeline, mcp__cclogviewer__get_tool_usage_stats]
argument-hint: "<direct|agent|skill> [name] [--sessions N] [--days N]"
---

# Reflection Loop: $ARGUMENTS

You are starting a reflection loop to analyze Claude Code session logs and propose improvements.

---

## CRITICAL: MCP Call Requirements

**ALWAYS include `project` parameter in ALL MCP calls.**
The project name is stored in `reflection-setup.json` (see Phase 1).

**For raw log extraction:** Use `get_session_logs` with `output_path` parameter to save JSONL directly to file (this avoids returning 69K+ tokens to context).

**For stats (returned to context):** Use `get_session_stats` (~5K tokens).

**For error context:** Use `get_logs_around_entry` (~2K tokens).

---

## Phase 1: Parse Arguments & Load Setup

### 1.1 Parse Arguments

Extract from `$ARGUMENTS`:
- `mode`: Required — one of `direct`, `agent`, or `skill`
- `name`: Required for agent/skill mode (e.g., `playwright-e2e-specialist`). Not needed for `direct`.
- `--sessions N`: Optional — max sessions to analyze (default: 5)
- `--days N`: Optional — look back N days (default: 7)

If arguments are missing or malformed, ask the user using AskUserQuestion.

### 1.2 Load reflection-setup.json

Look for `reflection-setup.json` in the **current project root** (git root or cwd):

```bash
PROJECT_ROOT=$(git rev-parse --show-toplevel 2>/dev/null || pwd)
SETUP_FILE="$PROJECT_ROOT/.claude/reflection-setup.json"
```

**If file exists**: Read it and extract `project_name`.

**If file doesn't exist**: Tell the user to run `/reflect:setup` first and stop execution. Do NOT attempt to create it here.

### 1.3 Validate Target (agent/skill modes only)

**For `agent` mode**: Check `.claude/agents/{name}.md` exists in project root
**For `skill` mode**: Check `.claude/skills/{name}/SKILL.md` exists in project root
**For `direct` mode**: No validation needed

If target not found, list available agents/skills and ask user to select.

### 1.4 Create Workspace Folder

```bash
TIMESTAMP=$(date +%Y%m%d-%H%M%S)
WORKSPACE="$PROJECT_ROOT/.claude/claude-workspace/reflection/${MODE}-${NAME:-general}-${TIMESTAMP}"
mkdir -p "$WORKSPACE/raw-data" "$WORKSPACE/summaries" "$WORKSPACE/analysis"
```

### 1.5 Save Run Config

Write `$WORKSPACE/config.json`:
```json
{
  "mode": "direct|agent|skill",
  "name": "target-name or null",
  "project": "project-name-from-setup",
  "definition_path": "path-or-null",
  "created_at": "ISO timestamp",
  "days": 7,
  "sessions_limit": 5,
  "status": "initialized"
}
```

---

## Phase 2: Fetch Session Data

### 2.1 Find Sessions

**For `agent` mode:**
```
mcp__cclogviewer__get_agent_sessions(
  agent_type: "{name}",
  limit: {sessions_limit},
  days: {days},
  project: "{project_name}"
)
```

**For `skill` mode:**
Use the `Skill` tool name as the agent_type identifier:
```
mcp__cclogviewer__get_agent_sessions(
  agent_type: "{name}",
  limit: {sessions_limit},
  days: {days},
  project: "{project_name}"
)
```
Note: Skills are invoked via the `Skill` tool, so sessions where `Skill` tool was called with the target skill name will be found.

**For `direct` mode:**
```
mcp__cclogviewer__list_sessions(
  project: "{project_name}",
  limit: {sessions_limit},
  days: {days},
  include_agent_types: false
)
```
All sessions are included — we analyze the main user↔Claude conversation thread only (no sidechains).

If no sessions found: report to user, suggest expanding date range.

### 2.2 Extract Full Session Logs to JSONL Files

For EACH session, save the full parsed JSONL log file to disk using `output_path` (this does NOT return content to context — it writes directly to file):

```
mcp__cclogviewer__get_session_logs(
  session_id: "{id}",
  project: "{project_name}",
  include_sidechains: false,  // For direct mode: main thread only
  // include_sidechains: true  // For agent/skill mode: include subagent data
  output_path: "{WORKSPACE}/raw-data/session-{uuid}-logs.jsonl"
)
```

### 2.3 Fetch Session Stats (for quick reference)

Also fetch lightweight stats for each session (returned to context for orchestrator use):
```
mcp__cclogviewer__get_session_stats(
  session_id: "{id}",
  project: "{project_name}",
  include_sidechains: <same as above>,
  errors_limit: 20
)
```

### 2.4 Fetch Error Context

For EACH error UUID in stats.errors.errors[]:
1. `get_logs_around_entry(session_id, uuid, project, offset=-5)` → before context
2. `get_logs_around_entry(session_id, uuid, project, offset=+5)` → after context

### 2.5 Save Session Metadata

Write each session's stats + error contexts to `$WORKSPACE/raw-data/session-{uuid}-stats.json`:
```json
{
  "session_id": "uuid",
  "mode": "direct|agent|skill",
  "logs_file": "session-{uuid}-logs.jsonl",
  "stats": { ...get_session_stats result... },
  "error_contexts": [ { "error_uuid": "...", "before_context": {...}, "after_context": {...} } ]
}
```

---

## Phase 3: Process Sessions (Parallel Workers)

### 3.1 Launch `reflection-worker` Agents

For EACH session, launch a **`reflection-worker`** agent using the Task tool.

**Launch ALL workers in PARALLEL.**

Prompt for each worker:
```
Process session data and create a structured summary.

**Session ID**: {uuid}
**Mode**: {direct|agent|skill}
**Stats File**: {WORKSPACE}/raw-data/session-{uuid}-stats.json
**Logs File**: {WORKSPACE}/raw-data/session-{uuid}-logs.jsonl
**Output Path**: {WORKSPACE}/summaries/session-{uuid}-summary.md
**Target**: {name or "direct Claude usage"}

Read the processing instructions at: {PLUGIN_ROOT}/skills/reflect/docs/PROCESSING.md
```

Use `subagent_type: "reflection-worker"` and `model: "sonnet"`.

**IMPORTANT**: Use the exact agent name `reflection-worker` in the Task tool call.

### 3.2 Collect Results

Wait for all `reflection-worker` agents to complete. Count successful/failed.
Continue even if some fail.

---

## Phase 4: Analyze Summaries (`reflection-analyzer` Agent)

### 4.1 Launch `reflection-analyzer` Agent

Use Task tool to launch the **`reflection-analyzer`** agent on Opus.

**IMPORTANT**: Use the exact agent name `reflection-analyzer` in the Task tool call.

The prompt varies by mode:

**For `direct` mode:**
```
Analyze session summaries for direct Claude usage patterns.

**Mode**: direct
**Project Root**: {PROJECT_ROOT}
**Summaries Folder**: {WORKSPACE}/summaries/
**Output Path**: {WORKSPACE}/analysis/analysis-report.md

**Reference Files to Read:**
- {PROJECT_ROOT}/CLAUDE.md (project instructions)
- {PROJECT_ROOT}/.claude/CLAUDE.md (project-local, if exists)
- ~/.claude/CLAUDE.md (global user instructions, if exists)
- {PROJECT_ROOT}/.claude/memory/ (memory files directory, if exists)
- ~/.claude/projects/{project-slug}/memory/ (auto-memory directory, if exists)

**Analysis Strategy**: Read {PLUGIN_ROOT}/skills/reflect/docs/ANALYSIS-DIRECT.md

Analyze patterns and suggest improvements to CLAUDE.md or memory files.
Route each suggestion to the correct target using the routing rules in the strategy doc.
```

**For `agent` mode:**
```
Analyze session summaries against the agent definition.

**Mode**: agent
**Target Name**: {name}
**Definition Path**: {definition_path}
**Summaries Folder**: {WORKSPACE}/summaries/
**Output Path**: {WORKSPACE}/analysis/analysis-report.md

**Analysis Strategy**: Read {PLUGIN_ROOT}/skills/reflect/docs/ANALYSIS-AGENT.md

Read the agent definition file and ALL session summaries.
Propose specific edits to the agent .md definition file.
```

**For `skill` mode:**
```
Analyze session summaries against the skill definition and its outputs.

**Mode**: skill
**Target Name**: {name}
**Skill Path**: {PROJECT_ROOT}/.claude/skills/{name}/SKILL.md
**Summaries Folder**: {WORKSPACE}/summaries/
**Output Path**: {WORKSPACE}/analysis/analysis-report.md

**Analysis Strategy**: Read {PLUGIN_ROOT}/skills/reflect/docs/ANALYSIS-SKILL.md

IMPORTANT: Skills are loaded into context when invoked via the Skill tool — the SKILL.md content
stays in context for the entire session. Therefore, do NOT focus on tool compliance (the skill
orchestrates tool calls, not the other way around). Instead focus on:
- Output quality: Did the skill produce correct/complete results?
- Workflow completeness: Did all phases execute successfully?
- Error recovery: Did the skill handle failures gracefully?
- User satisfaction: Did users need to intervene or correct the skill?
- Referenced file quality: Are @docs/ and @templates/ adequate?

Read the SKILL.md file AND any files it references via @docs/ or @templates/ paths.
Propose edits to SKILL.md and/or its referenced docs.
```

Use `subagent_type: "reflection-analyzer"` and `model: "opus"`.

### 4.2 Verify Report

Read `$WORKSPACE/analysis/analysis-report.md` to confirm it was generated.

---

## Phase 5: User Approval & Apply Changes

### 5.1 Parse Recommendations

Read the analysis report and extract each recommendation with:
- Priority (CRITICAL/MEDIUM/LOW)
- Target file path (CLAUDE.md / memory/{topic}.md / agents/{name}.md / skills/{name}/SKILL.md)
- Current state (quote from existing file)
- Proposed change (exact text)
- Rationale + evidence

### 5.2 Present Each Recommendation

For EACH recommendation, ask user via AskUserQuestion:
- Show the recommendation details including target file
- Options: "Yes - Apply", "No - Skip", "Edit - Modify before applying"

### 5.3 Apply Approved Changes

For each approved change, determine target and apply:

**`direct` mode targets:**
- CLAUDE.md → Use Edit tool on `{PROJECT_ROOT}/CLAUDE.md`
- Memory file → Write/Edit to `{PROJECT_ROOT}/.claude/memory/{topic}.md` (create dir if needed, link from MEMORY.md)

**`agent` mode targets:**
- Agent definition → Use Edit tool on `.claude/agents/{name}.md`

**`skill` mode targets:**
- SKILL.md → Use Edit tool on `.claude/skills/{name}/SKILL.md`
- Referenced docs → Use Edit tool on the specific @docs/ or @templates/ file

Log each change to `$WORKSPACE/applied-changes.md`.

### 5.4 Update Reflection History

Append to `reflection-setup.json`'s `reflection_history`:
```json
{
  "timestamp": "ISO",
  "mode": "direct|agent|skill",
  "target": "name or null",
  "sessions_analyzed": 3,
  "recommendations_total": 6,
  "recommendations_approved": 4,
  "workspace_path": "relative/path/to/workspace"
}
```

---

## Phase 6: Final Summary

Present completion summary with table of recommendations and their status.
Update `$WORKSPACE/config.json` with `"status": "completed"`.

---

## Error Handling

| Phase | Error | Action |
|-------|-------|--------|
| Setup | reflection-setup.json missing | Tell user to run `/reflect:setup` first; stop |
| Setup | cclogviewer MCP not available | Tell user to run `/reflect:setup` first; stop |
| Fetch | No sessions found | Suggest expanding date range |
| Fetch | get_session_logs fails | Fall back to get_session_stats only |
| Process | `reflection-worker` agent fails | Continue with other workers |
| Analyze | `reflection-analyzer` fails | Save partial results, ask user |
| Apply | Target file not found | Create it (for memory files) or report error |
| Apply | Edit conflict | Show diff, ask user to resolve |
