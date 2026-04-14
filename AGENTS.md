# AGENTS.md

Guidance for Coding agents (subagents, Explore, general-purpose) working in this repository.

## Rule: Always Use `/ia` Skill for iA Analysis

**MANDATORY: Invoke the `/ia` skill (via the Skill tool) before starting ANY iA-related analysis task.**

This applies to:
- Program dependency tracing
- Call hierarchy analysis
- Field impact analysis
- File dependency analysis
- Copybook impact analysis
- Data structure inspection
- Cross-reference lookups
- Complexity assessment
- Where-used queries
- Any query-driven investigation of IBM i programs using `ia_*` MCP tools

## Why

The `/ia` skill contains:
- Correct tool sequencing (prevents wasted calls)
- SQL patterns and query templates (prevents wrong queries)
- Interpretation guidance (prevents wrong conclusions)
- Context awareness (tool relationships and data flow)

Using `ia_*` tools directly without the skill's guidance leads to inefficient queries and misinterpretation of results.

## How

When delegating iA analysis to an agent:

```
Agent(
  description: "Analyze call hierarchy for program X",
  prompt: "Use the /ia skill to analyze... [rest of analysis request]"
)
```

The agent will:
1. Call the `/ia` skill (see `.claude/skills/ia/SKILL.md` for tool reference)
2. Follow the skill's sequencing and patterns
3. Return findings with proper context

## Reference

- **Full skill details**: `.claude/skills/ia/SKILL.md`
- **SQL patterns fallback**: `.claude/skills/ia/references/sql-patterns.md`
- **Repository tables**: `archive/FileInfo.md` (35+ iA tables with schemas)
- **Environment**: `IA_LIBRARY=IADEMODEV`
