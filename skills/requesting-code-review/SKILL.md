---
name: requesting-code-review
description: Use when completing tasks, implementing major features, or before merging to verify work meets requirements
---

# Requesting Code Review

Dispatch a **real** `superpowers:code-reviewer` subagent after completing work. The goal is to catch issues early before they compound. The reviewer must receive clean, focused context — never your full session history.

**Core principle:** Review early, review often. The review step is mandatory, not optional.

## When to Request Review

**Mandatory:**
- After each task in subagent-driven development or atomic prompts
- After completing a major feature or chunk
- Before merging to main

**Optional but valuable:**
- When stuck
- Before a big refactor
- After fixing a complex bug

## How to Request a Real Review (Explicit Instructions)

**1. Capture the git SHAs**
```bash
BASE_SHA=$(git rev-parse HEAD~1)
HEAD_SHA=$(git rev-parse HEAD)
```

**2. Dispatch the reviewer sub-agent — do this literally**

When your prompt reaches either of these triggers:
- "Code review: dispatch `superpowers:code-reviewer` subagent"
- A block beginning with `Agent tool (superpowers:code-reviewer):`

You **must** output the following exact block (this is the syntax that actually creates a real sub-agent in Grok Build):

```
Spawn sub-agent: superpowers:code-reviewer
WHAT_WAS_IMPLEMENTED: [one-sentence summary of what was just implemented in this chunk]
PLAN_OR_REQUIREMENTS: [e.g. "Chunk 4 (Tasks 4.1–4.6) of docs/plans/2026-05-14-async-admin-tasks.md — FEAT-002 handler routing"]
BASE_SHA: ${BASE_SHA}
HEAD_SHA: ${HEAD_SHA}
DESCRIPTION: [short description of the changes]
```

**3. Wait for the sub-agent to complete**

Do **not** write your final Report or mark the task as COMPLETE until the sub-agent has finished and returned its structured review (with Critical / Important / Minor issues).

**Critical Rule:**  
You are not allowed to claim “Review: PASS”, “subagent dispatched”, or “findings addressed” until you have actually received and processed output from a **real** spawned `superpowers:code-reviewer` sub-agent. If you have not yet received the reviewer’s report, you must continue waiting or explicitly spawn it.

**4. Act on the feedback**
- Fix all **Critical** issues immediately.
- Fix all **Important** issues before moving to the next task.
- Note **Minor** issues for later.
- You may push back on any finding with clear technical evidence.

See the reviewer template at: `requesting-code-review/code-reviewer.md`
