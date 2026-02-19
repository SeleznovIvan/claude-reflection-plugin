# Processing Instructions for Reflection Worker

Detailed processing instructions for `reflection-worker` agents. Read this file to understand how to process raw session data into structured summaries.

---

## Input Data Formats

### Stats JSON File (`session-{uuid}-stats.json`)

```json
{
  "session_id": "uuid",
  "mode": "direct|agent|skill",
  "logs_file": "session-{uuid}-logs.jsonl",
  "stats": {
    "summary": {
      "tokens": { "total_input": N, "total_output": N },
      "tool_calls": { "total": N, "success": N, "failed": N },
      "error_count": N
    },
    "tool_stats": {
      "tools": [{ "name": "Bash", "count": N, "success": N, "failed": N }],
      "patterns": { "most_used": "Bash", "most_failed": "..." }
    },
    "errors": {
      "errors": [{ "uuid": "...", "message": "...", "tool_name": "..." }]
    }
  },
  "error_contexts": [
    {
      "error_uuid": "uuid",
      "error_message": "...",
      "before_context": {
        "entries": [
          { "offset": -3, "role": "assistant", "content": "...", "tool_name": "..." },
          { "offset": -2, "role": "user", "content": "..." },
          { "offset": 0, "is_error": true, "content": "ERROR..." }
        ]
      },
      "after_context": {
        "entries": [
          { "offset": 0, "is_error": true },
          { "offset": +1, "role": "user", "content": "Stop! You should have..." },
          { "offset": +2, "role": "assistant", "content": "..." }
        ]
      }
    }
  ]
}
```

### JSONL Log File (`session-{uuid}-logs.jsonl`)

Each line is a JSON object representing a conversation entry:
```json
{"role": "user|assistant|system", "content": "...", "tool_name": "...", "tool_result": "...", "timestamp": "..."}
```

Use JSONL logs for:
- Understanding full workflow flow (not just errors)
- Finding user corrections not captured in error contexts
- Identifying workflow patterns and sequence issues
- Extracting additional context around interesting events

---

## Processing Steps

### Step 1: Load Session Data

Read the stats JSON file from the provided path. Extract:
- `stats.summary` — high-level metrics
- `stats.tool_stats` — per-tool usage
- `stats.errors` — error list
- `error_contexts[]` — before/after context for each error

### Step 2: Extract Tool Statistics

From `stats.tool_stats`:
1. Get the list of tools used from `tools[]` array
2. Extract count, success, and failed for each tool
3. Calculate success rate: `(success / count) * 100`
4. Note patterns from `stats.tool_stats.patterns` (most_used, most_failed)

### Step 3: Analyze Error Root Causes

For EACH error in `error_contexts`:

**A. Analyze Before Context (offset < 0):**
Look at entries with negative offset to understand what led to the error:
- What tool was being used?
- What was the agent trying to accomplish?
- Were there warning signs ignored?
- Was there a wrong assumption or approach?

**B. Analyze After Context (offset > 0):**
Look at entries with positive offset for user feedback:
- Did the user comment why they stopped? (Look for: "Stop", "Wrong", "No", "Should have")
- Did the user provide correction or guidance?
- Was this a manual interruption vs. natural error?

**C. Determine Root Cause Category:**

| Category | Description | Example |
|----------|-------------|---------|
| `WRONG_APPROACH` | Agent used wrong tool or method | Used Bash instead of Edit for file modification |
| `MISSING_VALIDATION` | Agent didn't check preconditions | Ran command without checking if file exists |
| `INCORRECT_ASSUMPTION` | Agent assumed something false | Assumed test framework was Jest when it was Vitest |
| `USER_INTERRUPTION` | User stopped for external reason | User cancelled to change requirements |
| `INSTRUCTION_GAP` | Agent's definition lacks guidance for this case | No guidance for handling auth token expiry |
| `EXTERNAL_ERROR` | Tool/system failure outside agent's control | Network timeout, MCP server crash |

**D. Extract User Feedback:**
If user provided feedback in after_context, quote it verbatim.
This is critical for improving definitions.

### Step 4: Scan JSONL for Additional Insights

Read the JSONL log file and look for:
- **User corrections**: Messages where user redirects the agent (not captured in error contexts)
- **Workflow flow**: The sequence of actions taken
- **Repeated patterns**: Same tool called many times in succession (may indicate spinning)
- **Long gaps**: Extended sequences without user interaction (may indicate autonomous success or runaway)

### Step 5: Generate Summary

Write the summary to the Output Path using the template below.

---

## Output Template

```markdown
# Session Summary: {session-id}

**Session ID**: {full-uuid}
**Mode**: {direct|agent|skill}
**Target**: {name or "direct Claude usage"}
**Total Tool Calls**: {from stats.summary.tool_calls.total}
**Error Count**: {from stats.summary.error_count}

---

## 1. Tool Usage Statistics

| Tool | Count | Success | Fail | Success Rate |
|------|-------|---------|------|--------------|
| {tool.name} | {count} | {success} | {failed} | {calculated}% |

**Total Tool Calls**: {stats.summary.tool_calls.total}
**Overall Success Rate**: {stats.summary.tool_calls.success / total * 100}%

### Patterns Identified

**Most Used Tool**: {stats.tool_stats.patterns.most_used}
**Most Failed Tool**: {stats.tool_stats.patterns.most_failed}

### Potential Issues

- {Issue based on tool stats, e.g., "High failure rate for Bash (X%) suggests command issues"}

---

## 2. Error Root Cause Analysis

### Errors Analyzed: {count from error_contexts}

{If no errors: "No errors were identified in this session."}

#### Error 1: {error_message truncated}

- **Error UUID**: {error_uuid}
- **Root Cause Category**: {category}
- **What Was Happening**: {from before_context - what tool/action led to error}
- **Why It Failed**: {analysis of the root cause}
- **User Feedback**: "{verbatim quote from after_context}" or "No user feedback"
- **Suggested Fix**: {specific improvement for definition}

### Error Summary

| Root Cause Category | Count | Has User Feedback |
|---------------------|-------|-------------------|
| {category} | X | Y/X |

---

## 3. User Feedback Extraction

### Direct User Feedback from Error Contexts

{For each error where after_context contains user feedback:}

**Error**: {error_message}
**User Said**: "{verbatim quote}"
**Implied Issue**: {what the feedback suggests about behavior}

### User Corrections from JSONL Logs

{Any additional user corrections found in the full logs}

---

## 4. Observations for Improvement

### What Worked Well
1. {Positive observation}

### What Could Be Improved
1. {Specific improvement}

### Suggested Definition Updates

1. **Priority**: HIGH/MEDIUM/LOW
   **Issue**: {from error analysis}
   **Suggested Update**: {specific text to add}
   **Evidence**: {from error context or user feedback}

---

## 5. Session Statistics

**Token Usage**:
- Input: {stats.summary.tokens.total_input}
- Output: {stats.summary.tokens.total_output}

**Tool Calls**:
- Total: {stats.summary.tool_calls.total}
- Success: {stats.summary.tool_calls.success}
- Failed: {stats.summary.tool_calls.failed}

---

**Processed**: {ISO timestamp}
```

---

## Quality Checks

Before saving the summary, verify:

1. **Completeness**: All errors from error_contexts are analyzed
2. **Accuracy**: User feedback is quoted verbatim from after_context entries
3. **Specificity**: Root cause categories are correctly assigned based on context analysis
4. **Traceability**: Each error has UUID and context evidence
5. **Actionability**: Suggested updates are specific enough to implement
6. **JSONL Coverage**: Additional insights from full logs are included when relevant

---

## Error Handling

| Error | Action |
|-------|--------|
| Cannot read stats file | Report error with path, exit with error message |
| Cannot read JSONL file | Process stats only, note gap in summary |
| Data format unexpected | Attempt best-effort parsing, note gaps in summary |
| Empty stats | Report "No session stats found", create minimal summary |
| No error_contexts | Report "No error contexts provided", skip error analysis section |
| Cannot write output | Report error with path details, exit with error |
| Malformed JSON | Report parsing error, attempt to extract what's possible |

---

## Important Notes

- Be OBJECTIVE — report what happened, not what should have happened
- Use SPECIFIC examples from error_contexts to support observations
- QUOTE user feedback VERBATIM from after_context entries
- Keep summaries CONCISE but COMPLETE
- NEVER fabricate information — if something is unclear, note it as "Unable to determine"
- Focus on PATTERNS that could inform definition improvements
- Prioritize errors where user feedback exists — these are most valuable
