# Analysis Strategy: Direct Claude Sessions

## What to Look For

- Repeated mistakes Claude makes in this project
- Missing context that causes wrong assumptions
- Build/test commands Claude gets wrong
- Architecture patterns Claude doesn't follow
- Common user corrections ("No, you should...", "Stop", "Wrong")
- Inefficient workflows that could be streamlined with better context

## Suggestion Routing

### → CLAUDE.md (project root)

Prescriptive instructions the whole team benefits from:
- Build/test/deploy commands
- Code style and formatting rules
- Architecture overview and constraints
- Technology stack documentation
- Important gotchas and "never do" rules
- File naming conventions
- Required patterns (e.g., "always use X for Y")
- Tool preferences and workflow conventions

### → Memory Files (~/.claude/projects/{slug}/memory/)

Descriptive learnings for this user/project:
- Discovered code patterns ("the auth module loads config at startup")
- Debugging insights ("X test fails when Y service is down")
- File location notes ("payment logic is in /src/billing, not /src/payments")
- Performance characteristics ("building the full project takes 5 min")
- Workarounds ("use --legacy-peer-deps for npm install")
- Non-obvious dependencies between modules
- Common error solutions

## Routing Decision Table

| Suggestion Type | Target | Reasoning |
|----------------|--------|-----------|
| Build/test commands | CLAUDE.md | Prescriptive, shared with team |
| Code style rules | CLAUDE.md | Prescriptive convention |
| Architecture overview | CLAUDE.md | Shared context |
| "Never do X" rules | CLAUDE.md | Prescriptive constraint |
| Tool/framework preferences | CLAUDE.md | Shared convention |
| Discovered file locations | Memory | Descriptive learning |
| Debugging workarounds | Memory | Session-specific insight |
| Performance notes | Memory | Observed behavior |
| Non-obvious dependencies | Memory | Discovered pattern |
| Common error solutions | Memory | Learned fix |

## Analysis Focus Areas

### 1. Command & Build Errors
Look for patterns where Claude uses wrong commands:
- Wrong package manager (npm vs yarn vs pnpm vs bun)
- Wrong test runner or test flags
- Wrong build commands
- Wrong environment setup steps

### 2. Architecture Violations
Look for patterns where Claude violates project conventions:
- Putting files in wrong directories
- Using wrong import patterns
- Not following naming conventions
- Creating files that duplicate existing patterns

### 3. Context Gaps
Look for repeated instances where Claude lacks project knowledge:
- Asks questions that CLAUDE.md should answer
- Makes assumptions that are wrong for this project
- Doesn't know about project-specific tools or scripts
- Misunderstands the project structure

### 4. Workflow Inefficiencies
Look for patterns where better context would save time:
- Claude searches for files it should know about
- Claude tries multiple approaches before finding the right one
- Claude asks the user for information it should already have

## Report Structure

When writing the analysis report for direct mode:

1. **Executive Summary** with overall health rating
2. **CLAUDE.md Recommendations** — grouped by topic, each with exact text to add
3. **Memory File Recommendations** — grouped by topic, each specifying which memory file
4. **Gap Analysis** — what's missing from current CLAUDE.md
5. **Appendix** — session evidence table
