---
name: start-reflection
description: Start a reflection loop to analyze agent/skill session logs and propose improvements
allowed-tools: [Read, Write, Edit, Glob, Grep, Bash, Task, TodoWrite, AskUserQuestion, mcp__cclogviewer__list_sessions, mcp__cclogviewer__list_projects, mcp__cclogviewer__get_session_stats, mcp__cclogviewer__get_logs_around_entry, mcp__cclogviewer__list_agents, mcp__cclogviewer__get_agent_sessions, mcp__cclogviewer__search_logs]
argument-hint: "<name> --type agent|skill [--from YYYY-MM-DD] [--to YYYY-MM-DD] [--sessions N]"
---

# Reflection Loop: $ARGUMENTS

You are starting a reflection loop to analyze agent/skill session logs and propose improvements to the definition.

---

## CRITICAL: MCP Call Requirements

**ALWAYS include `project` parameter in ALL MCP calls to avoid crashes:**

```
mcp__cclogviewer__get_session_stats(
  session_id: "xxx",
  project: "promova-aurum",  // REQUIRED - prevents crash!
  include_sidechains: true
)
```

**NEVER use `get_session_logs`** - returns 69K+ tokens, exceeds context limits.

**Use these tools instead:**
- `get_session_stats` (~5K tokens) - Combined summary, tool stats, and errors
- `get_logs_around_entry` (~2K tokens) - Context around specific log entries

**Error Root Cause Analysis:**
For each error UUID in stats.errors.errors[]:
1. Call `get_logs_around_entry(uuid, offset=-5)` → What agent did BEFORE error
2. Call `get_logs_around_entry(uuid, offset=+5)` → User comments AFTER error

---

## Phase 1: Parse Arguments and Initialize

### 1.1 Parse Arguments

Extract from `$ARGUMENTS`:
- `name`: Required - the agent or skill name to analyze (e.g., `playwright-e2e-specialist`, `managing-storage`)
- `--type`: Required - either `agent` or `skill`
- `--from YYYY-MM-DD`: Optional - start date for session filtering (default: 7 days ago)
- `--to YYYY-MM-DD`: Optional - end date for session filtering (default: today)
- `--sessions N`: Optional - maximum number of sessions to analyze (default: 5)

If arguments are missing or malformed, ask the user to provide them using AskUserQuestion.

### 1.2 Validate Target Exists

Based on `--type`:

**For agents**:
```bash
ls .claude/agents/{name}.md
```

**For skills**:
```bash
ls .claude/skills/{name}/SKILL.md
```

If the target doesn't exist:
1. List available agents/skills
2. Ask user to select one using AskUserQuestion

### 1.3 Create Reflection Folder

```bash
TIMESTAMP=$(date +%Y%m%d-%H%M%S)
REFLECTION_FOLDER=".claude/claude-workspace/reflection/${NAME}-${TIMESTAMP}"
mkdir -p "$REFLECTION_FOLDER/raw-data"
mkdir -p "$REFLECTION_FOLDER/summaries"
mkdir -p "$REFLECTION_FOLDER/analysis"
```

### 1.4 Save Configuration

Write `$REFLECTION_FOLDER/config.json`:
```json
{
  "name": "<name>",
  "type": "agent|skill",
  "definition_path": ".claude/agents/<name>.md | .claude/skills/<name>/SKILL.md",
  "created_at": "<ISO timestamp>",
  "from_date": "<YYYY-MM-DD>",
  "to_date": "<YYYY-MM-DD>",
  "sessions_limit": <N>,
  "status": "initialized"
}
```

**Output**: `Reflection loop initialized at $REFLECTION_FOLDER`

---

## Phase 2: Fetch Session Data

### 2.1 Find Sessions for Agent/Skill

Use MCP tool to find sessions where this agent/skill was used:

```
mcp__cclogviewer__get_agent_sessions(
  agent_type: "{name}",
  limit: {sessions_limit},
  days: {calculated from date range},
  project: "promova-aurum"  // CRITICAL: Always include!
)
```

**If no sessions found**: Report to user that no sessions were found for this agent/skill in the specified date range. Suggest expanding the date range or checking the agent/skill name.

### 2.2 Fetch Session Stats (ONE call per session)

For EACH session_id returned from step 2.1, call `get_session_stats` which returns combined data:

```
mcp__cclogviewer__get_session_stats(
  session_id: "{id}",
  project: "promova-aurum",  // CRITICAL!
  include_sidechains: true,
  errors_limit: 20
)
```

Returns combined: summary + tool_stats + errors in ONE response.

**DO NOT use `get_session_logs`** - causes token overflow (69K+ tokens).

### 2.3 Fetch Error Context (for root cause analysis)

For EACH error in `stats.errors.errors[]`, fetch context using `get_logs_around_entry`:

**Search BEFORE error (what went wrong):**
```
mcp__cclogviewer__get_logs_around_entry(
  session_id: "{id}",
  uuid: "{error.uuid}",
  project: "promova-aurum",
  offset: -5  // 5 entries BEFORE the error
)
```

**Search AFTER error (user comments about manual stop):**
```
mcp__cclogviewer__get_logs_around_entry(
  session_id: "{id}",
  uuid: "{error.uuid}",
  project: "promova-aurum",
  offset: +5  // 5 entries AFTER the error
)
```

### 2.4 Save Session Data

For each session, write to `$REFLECTION_FOLDER/raw-data/session-{uuid}-data.json`:

```json
{
  "session_id": "{uuid}",
  "stats": { ...get_session_stats result... },
  "error_contexts": [
    {
      "error_uuid": "{uuid}",
      "error_message": "{message}",
      "before_context": { ...logs_around_entry with offset=-5... },
      "after_context": { ...logs_around_entry with offset=+5... }
    }
  ]
}
```

**Output**: `Fetched data for X sessions with Y errors analyzed`

---

## Phase 3: Process Sessions (Parallel)

### 3.1 Launch logs-processor Agents

For EACH session data file, launch a `logs-processor` agent using the Task tool.

**IMPORTANT**: Launch ALL processors in PARALLEL using multiple Task tool calls in a single message.

For each session, use this prompt:

```
Process session data and create a structured summary with error root cause analysis.

**Session ID**: {session-uuid}
**Data Path**: {REFLECTION_FOLDER}/raw-data/session-{uuid}-data.json
**Output Path**: {REFLECTION_FOLDER}/summaries/session-{uuid}-summary.md
**Target Being Analyzed**: {name} ({type})

Read the session data file which contains:
1. `stats` - Combined session summary, tool stats, and errors
2. `error_contexts[]` - Before/after context for each error

For EACH error in error_contexts:
- Analyze `before_context` (offset < 0): What was agent doing wrong?
- Analyze `after_context` (offset > 0): Did user comment why they stopped?
- Look for patterns: wrong tool choice, missing validation, incorrect assumptions

Create summary with:
1. Tool execution statistics from stats.tool_stats
2. Error root cause analysis from error_contexts
3. User feedback extraction from after_context entries
4. Actionable improvement suggestions
```

Use `subagent_type: "logs-processor"` and `model: "sonnet"`.

### 3.2 Collect Results

After all parallel processors complete:
- Count successful summaries
- Note any failed processing with error reasons
- Continue even if some processors fail

**Output**: `Processed X/Y sessions successfully`

---

## Phase 4: Analyze Summaries

### 4.1 Verify Summaries Exist

Check that at least one summary was created:
```bash
ls $REFLECTION_FOLDER/summaries/*.md
```

If no summaries exist, report error and exit.

### 4.2 Launch logs-analyzer Agent

Use the Task tool to launch the `logs-analyzer` agent:

```
Analyze session summaries against the agent/skill definition.

**Target Name**: {name}
**Target Type**: {type}
**Definition Path**: {definition_path}
**Summaries Folder**: {REFLECTION_FOLDER}/summaries/
**Output Path**: {REFLECTION_FOLDER}/analysis/analysis-report.md

Read the definition file and ALL session summaries from the summaries folder.
Analyze patterns across sessions and propose improvements.

Generate an analysis report with:
1. Executive summary with overall health assessment
2. Tool usage analysis - patterns and compliance
3. Input-output analysis - completion metrics
4. Blocker analysis - recurring patterns
5. Detailed recommendations categorized as CRITICAL, MEDIUM, or LOW priority
6. Each recommendation must include current state, proposed change, rationale, and evidence
```

Use `subagent_type: "logs-analyzer"` and `model: "opus"`.

### 4.3 Verify Analysis Complete

Read the analysis report to confirm it was generated:
```
Read: $REFLECTION_FOLDER/analysis/analysis-report.md
```

**Output**: `Analysis complete - X recommendations generated`

---

## Phase 5: User Approval

### 5.1 Parse Recommendations

Read the analysis report and extract each recommendation. Parse:
- Priority (CRITICAL, MEDIUM, LOW)
- Category (Tool Usage, Input-Output, Blockers, Instructions)
- Current state (quote from definition)
- Proposed change (new text)
- Rationale
- Evidence from sessions

### 5.2 Present Recommendations for Approval

For EACH recommendation, present to the user using AskUserQuestion:

**Format**:
```
## Recommendation #{N} [{PRIORITY}]

**Category**: {category}

**Current State**:
> {quote from current definition, or "Not addressed"}

**Proposed Change**:
> {exact text to add or modify}

**Rationale**:
{explanation based on session analysis}

**Evidence**:
- Session {uuid1}: {specific example}
- Session {uuid2}: {specific example}
```

Ask: "Apply this change to the {type} definition?"
Options: "Yes - Apply change", "No - Skip this change"

### 5.3 Apply Approved Changes

For each APPROVED change:
1. Use the Edit tool to modify the definition file
2. Log the change to `$REFLECTION_FOLDER/applied-changes.md`

For each REJECTED change:
1. Log the rejection to `$REFLECTION_FOLDER/applied-changes.md`

### 5.4 Update Config Status

Update `$REFLECTION_FOLDER/config.json` with final status:
```json
{
  ...
  "status": "completed",
  "completed_at": "<ISO timestamp>",
  "recommendations_total": N,
  "recommendations_approved": X,
  "recommendations_rejected": Y
}
```

---

## Phase 6: Final Summary

Present the completion summary:

```markdown
# Reflection Loop Complete

**Target**: {name} ({type})
**Sessions Analyzed**: X
**Date Range**: {from_date} to {to_date}

## Recommendations Summary

| Priority | Total | Approved | Rejected |
|----------|-------|----------|----------|
| CRITICAL | A     | B        | C        |
| MEDIUM   | D     | E        | F        |
| LOW      | G     | H        | I        |

## Changes Applied

1. {Brief description of approved change 1}
2. {Brief description of approved change 2}
...

## Artifacts

- **Raw data**: {REFLECTION_FOLDER}/raw-data/
- **Summaries**: {REFLECTION_FOLDER}/summaries/
- **Analysis**: {REFLECTION_FOLDER}/analysis/analysis-report.md
- **Change log**: {REFLECTION_FOLDER}/applied-changes.md

---

**Next Steps**: Review the updated definition and test with new sessions.
```

---

## Error Handling

| Phase | Error | Handling |
|-------|-------|----------|
| Parse | Missing arguments | Ask user with AskUserQuestion |
| Parse | Invalid type | Show valid options, ask user |
| Initialize | Target not found | List available targets, ask user to select |
| Initialize | Folder creation fails | Report error with path, exit |
| Fetch | MCP tool unavailable | Check .mcp.json config, report error |
| Fetch | No sessions found | Suggest expanding date range, exit |
| Fetch | Session fetch fails | Retry once, skip if still fails |
| Process | Processor fails | Continue with other sessions, note in report |
| Process | All processors fail | Report error, suggest manual inspection |
| Analyze | Cannot read summaries | Report which files missing |
| Analyze | Analyzer fails | Save partial results, ask user how to proceed |
| Approve | Edit fails | Report error, allow manual edit |
| Approve | User cancels mid-flow | Save current state, allow resume later |

---

## Usage Examples

```bash
# Analyze an agent (default: 5 sessions, last 7 days)
/reflection:start-reflection playwright-e2e-specialist --type agent

# Analyze a skill
/reflection:start-reflection managing-storage --type skill

# Analyze with specific date range
/reflection:start-reflection admin-ui-specialist --type agent --from 2026-01-01 --to 2026-01-14

# Analyze more sessions
/reflection:start-reflection storage-specialist --type agent --sessions 10

# Combined options
/reflection:start-reflection validating-curriculum --type skill --from 2026-01-07 --sessions 8
```
