# Analysis Strategy: Skill Sessions

## Critical Context

Skills are loaded into context when invoked via the `Skill` tool. The SKILL.md content
stays in the main conversation context for the entire session. This means:
- The skill orchestrates the workflow, NOT individual tool calls
- Focus on WHETHER the skill achieves its goals, not HOW it calls tools
- Analyze the OUTPUTS and RESULTS, not the tool-call-level behavior

## What to Look For

- Output quality: Did the skill produce correct and complete results?
- Workflow completeness: Did all defined phases execute successfully?
- Phase failures: Which phases commonly fail or produce poor results?
- Error recovery: Did the skill handle MCP/tool failures gracefully?
- User satisfaction: Did users need to intervene, correct, or restart?
- Argument handling: Did the skill parse all input variations correctly?
- Referenced file quality: Are @docs/ and @templates/ adequate and up-to-date?

## Suggestion Routing

Suggestions may target:
- `SKILL.md` — Main skill definition (workflow, phases, instructions)
- `@docs/` files — Referenced documentation (processing rules, strategies)
- `@templates/` files — Output templates (format, sections)

## Key Analysis Points

### 1. Phase Execution
- Check if all phases execute successfully across sessions
- Identify phases that commonly fail or produce poor results
- Look for phases that are skipped or executed out of order
- Check if phase transitions are handled correctly

### 2. Output Quality
- Did the skill produce the expected output format?
- Were outputs complete (all sections filled)?
- Were outputs accurate (correct information)?
- Did users accept or need to modify outputs?

### 3. Template Adequacy
- Identify template gaps (sections consistently empty or wrong)
- Check if templates match what users actually need
- Look for missing template sections users frequently request
- Verify template formatting is correct

### 4. Error Handling
- How does the skill handle tool/MCP failures?
- Are there error scenarios not covered by the skill's error handling section?
- Does the skill degrade gracefully or fail catastrophically?

### 5. Argument Parsing
- Does the skill handle all documented argument formats?
- Are there argument combinations that cause failures?
- Are default values appropriate?

### 6. Referenced File Quality
- Read all files referenced via @docs/ or @templates/ in the SKILL.md
- Check if referenced docs contain sufficient guidance
- Identify referenced files that are outdated or incomplete
- Look for guidance that should be in @docs/ but is inline in SKILL.md

## Analysis Methodology

### Step 1: Parse Skill Definition
Extract from SKILL.md:
- Frontmatter: name, description, allowed-tools, argument-hint
- Phases/steps defined in the body
- @docs/ and @templates/ references
- Error handling section

### Step 2: Read Referenced Files
Read ALL files referenced via @docs/ or @templates/ paths in the SKILL.md.
These are part of the skill's "definition surface" and may need updates too.

### Step 3: Evaluate Per-Session
For each session summary:
- Did each phase complete?
- What was the output quality?
- Were there user interventions?
- What errors occurred and how were they handled?

### Step 4: Cross-Session Patterns
- Which phases fail most often?
- What are common user complaints?
- Are there systematic output quality issues?

## Report Structure

When writing the analysis report for skill mode:

1. **Executive Summary** with health rating
2. **Phase Execution Analysis** — per-phase success rates
3. **Output Quality Assessment** — completeness, accuracy, user acceptance
4. **Error Handling Analysis** — coverage, graceful degradation
5. **Referenced File Assessment** — @docs/ and @templates/ quality
6. **Detailed Recommendations** — grouped by priority, targeting SKILL.md or referenced files
7. **Appendix** — session evidence table
