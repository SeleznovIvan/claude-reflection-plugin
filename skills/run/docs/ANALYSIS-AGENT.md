# Analysis Strategy: Agent Sessions

## What to Look For

- Tool usage compliance: does agent use tools as defined in frontmatter?
- Sequence correctness: does agent follow prescribed workflow steps?
- Error handling: does agent recover from errors properly?
- Task completion: does agent finish tasks without user intervention?
- Missing instructions: are there definition gaps that cause failures?
- Anti-pattern violations: does agent do things explicitly forbidden?

## Suggestion Routing

All suggestions target the agent's `.claude/agents/{name}.md` definition file.

## Key Analysis Points

### 1. Tool Compliance
- Compare tools listed in frontmatter `tools:` field vs tools actually used in sessions
- Identify tools used that aren't declared (may need adding to frontmatter)
- Identify declared tools never used (may indicate unnecessary permissions)

### 2. Anti-Pattern Adherence
- Check if anti-patterns defined in the agent are being followed
- Look for recurring violations of "never do" rules
- Identify new anti-patterns that should be documented

### 3. Instruction Coverage
- Identify recurring blockers and their root causes
- Look for user interruptions → these indicate definition gaps
- Check if examples in definition cover observed scenarios
- Verify model specification matches task complexity

### 4. Error Recovery
- How does the agent handle tool failures?
- Does it retry appropriately vs. spinning?
- Are there error scenarios the definition should address?

### 5. Workflow Patterns
- Does the agent follow a consistent workflow?
- Are there common deviations from the defined steps?
- Should the definition prescribe a more specific sequence?

### 6. Tool Sequence Compliance (from Worker Summaries)
Worker summaries now include a **"Tool Sequence Analysis"** section (section 2.5) with:
- Ordered tool call sequences extracted from session timelines
- Detected patterns: loops, clusters, unexpected transitions
- Deviations from the agent's prescribed workflow sequence

Use this data to:
- Identify if the agent consistently follows its prescribed tool order
- Detect common deviations that suggest the workflow definition needs updating
- Find repeated loops that indicate the agent is spinning/retrying inefficiently
- Recommend specific sequence improvements to the agent definition

### 7. Definition Compliance (from Worker Summaries)
Worker summaries now include a **"Definition Compliance"** section (section 2.6) with:
- Tool compliance table (declared vs. used tools)
- Workflow adherence assessment
- Anti-pattern violation checks

Use this data to:
- Aggregate compliance findings across all sessions
- Identify persistent undeclared tool usage (tools should be added to frontmatter)
- Identify consistently unused declared tools (may be removed from frontmatter)
- Find recurring anti-pattern violations that need stronger definition language
- Assess overall workflow adherence rates across sessions

## Analysis Methodology

### Step 1: Parse Agent Definition
Extract from the `.md` file:
- Frontmatter: name, description, tools, model, color
- Body sections: instructions, steps, patterns, anti-patterns, examples
- Any referenced files or external docs

### Step 2: Cross-Reference with Sessions
For each session summary:
- Map tool usage to frontmatter declarations
- Map workflow steps to defined instructions
- Map errors to missing guidance
- Map user feedback to definition gaps

### Step 3: Pattern Detection
Across all sessions:
- Find recurring tool usage patterns
- Find recurring error patterns
- Find recurring user corrections
- Calculate compliance rates

## Report Structure

When writing the analysis report for agent mode:

1. **Executive Summary** with health rating
2. **Tool Usage Analysis** — compliance table, undeclared tools, unused tools
3. **Tool Sequence Compliance** — sequence patterns, loops, deviations from prescribed workflow (from worker section 2.5)
4. **Definition Compliance Summary** — aggregated compliance findings across sessions (from worker section 2.6)
5. **Workflow Compliance** — step adherence, common deviations
6. **Error Pattern Analysis** — recurring errors, missing error handling
7. **Definition Gap Analysis** — coverage assessment table
8. **Detailed Recommendations** — grouped by priority, all targeting agent .md file
9. **Appendix** — session evidence table
