# Code Quality Reviewer Prompt Template

Use this template when dispatching a code quality reviewer subagent.

**Purpose:** Verify implementation is well-built (clean, tested, maintainable)

**Only dispatch after spec compliance review passes.**

```
Task tool (superpowers:code-reviewer):
  Use template at code-review/code-reviewer.md

  WHAT_WAS_IMPLEMENTED: [from implementer's report]
  PLAN_OR_REQUIREMENTS: Task N from [plan-file]
  BASE_SHA: [commit before task]
  HEAD_SHA: [current commit]
  DESCRIPTION: [task summary]
```

**Structural analysis:** The reviewer should use Agent Brain CLI if available:
- `agent-brain-cli impact <project> <symbol> [--depth N]` — enumerate all dependents of a changed interface
- `agent-brain-cli dead-code <project> [--kind function]` — symbols the change removed/renamed that still have callers
- `agent-brain-cli query <project> --file <path>` — verify expected symbols exist in a file

If CLI is not available, grep/glob serves the same purpose with more work. Either way, verifying that every caller of a changed interface is updated is the #1 guard against silent regressions — it's not optional when the change touches an interface.

(Note: older versions of this template referenced MCP tool names like `get_dependents` / `impact_analysis`. Those were the MCP-era API and have been replaced by the CLI shown above.)

**In addition to standard code quality concerns, the reviewer should check:**
- Does each file have one clear responsibility with a well-defined interface?
- Are units decomposed so they can be understood and tested independently?
- Is the implementation following the file structure from the plan?
- Did this implementation create new files that are already large, or significantly grow existing files? (Don't flag pre-existing file sizes — focus on what this change contributed.)

**Code reviewer returns:** Strengths, Issues (Critical/Important/Minor), Assessment
