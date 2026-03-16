---
name: code-review
description: Code review pull requests and local changes with structural dependency analysis. Use when asked to review code, review a PR, check changes, or do a code review. Also use when completing tasks, implementing major features, or before merging to verify work meets requirements.
allowed-tools: Bash(gh *), Bash(git diff:*), Bash(git status:*), Bash(git log:*), Bash(git blame:*), Bash(git show:*)
---

# Graph-Aware Code Review

Review code changes using parallel review agents with structural analysis.

**Two modes:**
- **PR mode:** `$ARGUMENTS` is a number (e.g., `86`, `#86`) → reviews a pull request, posts comment to PR
- **Local mode:** no arguments or `--staged` → reviews uncommitted local changes

## Instructions

Follow these phases precisely. Make a todo list first.

### Phase 1 — Pre-flight

1. **Detect mode** from `$ARGUMENTS`:
   - Numeric → PR mode. Strip `#` if present.
   - Empty or `--staged` → Local mode.
   - Other → use conversation context to resolve, or ask.

2. **Get the diff:**
   - PR mode: `gh pr diff <number>`
   - Local mode: `git diff` + `git diff --staged` (or `git diff --staged` only if `--staged`)

3. **If no changes found**, report that and stop.

4. **Get changed file list:**
   - PR mode: `gh pr diff <number> --name-only`
   - Local mode: `git diff --name-only` + `git diff --staged --name-only`

5. **Structural pre-flight** (if Agent Brain MCP tools are available):
   ```python
   # Get blast radius for changed symbols
   mcp__agent-brain__agent_brain_impact_analysis(
       symbols=[<key classes/functions from changed files>],
       project="<project-name>"
   )
   ```
   Extract 5-10 key symbol names from the diff (class names, function names that were modified).
   If Agent Brain is unavailable, skip this step and note it in the output.

6. **Find CLAUDE.md files:** root CLAUDE.md + any CLAUDE.md files in directories of changed files.

7. **Get PR metadata** (PR mode only): `gh pr view <number> --json title,body,author`

### Phase 2 — Parallel Review

Launch **4 parallel Sonnet agents**. Each agent receives:
- The diff (or relevant portions)
- The changed file list
- The structural context from Phase 1 (impact analysis results)

Each agent MUST include a `## Tools Used` section at the end listing every tool it called.

Tell each agent:

> **Structural analysis tools (use when investigating dependencies or blast radius):**
> If Agent Brain MCP tools are available in this project, you can use:
> - `mcp__agent-brain__agent_brain_get_dependents(name, project)` — what depends on a symbol
> - `mcp__agent-brain__agent_brain_get_dependencies(name, project)` — what a symbol depends on
> - `mcp__agent-brain__agent_brain_get_symbol(name, project)` — full symbol info with edges
> - `mcp__agent-brain__agent_brain_find_dead_code(project)` — unreferenced symbols
> - `mcp__agent-brain__agent_brain_impact_analysis(symbols, project)` — blast radius
> - `mcp__agent-brain__agent_brain_graph_query(cypher, project)` — raw Cypher queries
>
> Use these when investigating structural issues (dependency gaps, missing implementations,
> orphaned references). They are optional — use them when they would strengthen your analysis.
> If they are not available, fall back to grep/glob-based analysis.

#### Agent 1 — Structural Completeness

Focus: dependency gaps, missing interface implementations, orphaned references.

You received impact analysis results showing what depends on the changed symbols.
Your job is to verify that ALL dependents are properly handled. Specifically:

- Are all subclasses/implementations updated when a base class changes?
- Are all callers updated when a function signature changes?
- Are test fixtures and mocks updated to match new interfaces?
- Are there new symbols that nothing references (potential dead code)?

Use `get_dependents` and `get_symbol` to dig deeper on specific symbols.
Return a list of issues with file:line and explanation.

#### Agent 2 — Bugs & Logic

Focus: logic errors, race conditions, null/undefined handling, edge cases, resource leaks.

Read the diff AND the full modified files for context (not just the changed lines).
Focus on significant bugs that will impact functionality in practice. Avoid nitpicks.

You can use `get_dependencies` to check whether callers of a changed function handle
the new behavior correctly.

Return a list of issues with file:line and explanation.

#### Agent 3 — Compliance & Conventions

Focus: CLAUDE.md rules, code comment guidance, naming conventions, project patterns.

Read the CLAUDE.md files found in Phase 1. Note that CLAUDE.md is guidance for Claude
as it writes code — not all instructions are applicable during code review.

Check:
- Do changes comply with explicit CLAUDE.md rules?
- Do changes comply with guidance in code comments (TODOs, warnings, notes)?
- Are there patterns in the codebase that the changes violate?

Return a list of issues with file:line, the specific rule violated, and explanation.

#### Agent 4 — Historical Context

Focus: git blame, prior PRs, recurring patterns.

- Run git blame on modified sections to understand history
- Find prior PRs that touched these files and check their comments
- Look for issues that were flagged before and may apply here

Return a list of issues with file:line and explanation.

### Phase 3 — Scoring

For **each issue** found in Phase 2, launch a parallel **Haiku agent** to score it 0-100.

Give each scoring agent:
- The issue description
- The diff context
- The CLAUDE.md file paths

Scoring rubric (give this verbatim to each agent):

> Score this issue on a scale from 0-100:
> - 0: False positive. Doesn't hold up to scrutiny, or is a pre-existing issue.
> - 25: Might be real, might be false positive. Couldn't verify. If stylistic, not in CLAUDE.md.
> - 50: Real but nitpicky or rare in practice. Not very important relative to the rest.
> - 75: Verified real. Will impact functionality, or directly mentioned in CLAUDE.md.
> - 100: Definitely real. Will happen frequently. Evidence directly confirms this.
>
> For CLAUDE.md issues: double-check that CLAUDE.md actually calls out this issue specifically.

**Filter:** discard all issues scoring below 80.

**False positives to watch for:**
- Pre-existing issues not introduced by this change
- Issues a linter, typechecker, or compiler would catch
- Pedantic nitpicks a senior engineer wouldn't flag
- General quality issues unless explicitly required in CLAUDE.md
- Issues silenced by lint-ignore comments in the code
- Changes in functionality that are likely intentional
- Real issues but on lines not modified in this change

### Phase 4 — Output

**Always** output the full report to the terminal.

**PR mode additionally:** post the report as a comment on the PR using `gh pr comment <number>`.

#### Report Format

If issues were found:

```
### Code Review

**Summary:** [2-3 sentences on what changed]

**Structural Impact:** [blast radius — N dependents, key relationships, or "no structural concerns"]

Found N issues:

**Critical** (N issues)

1. **file.py:42** — brief description (reason)

   [explanation + suggested fix]
   [PR mode: link to file:line with full SHA]

**Warning** (N issues)

1. **file.ts:15** — brief description (reason)

   [explanation + suggested fix]

| File | Summary |
|------|---------|
| path/to/file.py | What changed, clean or has issues |

**Confidence: N/5** — [one sentence reasoning]
```

If no issues found after filtering:

```
### Code Review

No significant issues found. Checked for: structural completeness, bugs, CLAUDE.md compliance, historical context.

**Structural Impact:** [brief summary]
```

#### PR Comment Rules

- Keep output brief
- No emojis
- Link and cite relevant code, files, and URLs
- Use full git SHA in file links (not HEAD or short SHA)
- Line range format: `#L[start]-L[end]` with 1 line context before and after
- End with: `Generated with [Claude Code](https://claude.ai/code)`

## Integration with Workflows

**Subagent-Driven Development:** review after EACH task (lighter version via code-quality-reviewer-prompt.md)
**Executing Plans:** review after each batch
**Ad-Hoc Development:** review before merge, when stuck, after complex bug fix

**Related skills:**
- **superpowers:receiving-code-review** - How to handle review feedback
- **superpowers:verification-before-completion** - Evidence before claims
