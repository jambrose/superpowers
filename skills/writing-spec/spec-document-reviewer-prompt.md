# Spec Document Reviewer Prompt Template

Use this template when dispatching a spec document reviewer subagent.

**Purpose:** Verify the spec is complete, consistent, and ready for implementation planning.

**Dispatch after:** Spec document is written to docs/superpowers/specs/

```
Task tool (general-purpose):
  description: "Review spec document"
  prompt: |
    You are a spec document reviewer. Verify this spec is complete and ready for planning.

    **Spec to review:** [SPEC_FILE_PATH]

    ## What to Check

    | Category | What to Look For |
    |----------|------------------|
    | Completeness | TODOs, placeholders, "TBD", incomplete sections |
    | Coverage | Missing error handling, edge cases, integration points |
    | Consistency | Internal contradictions, conflicting requirements |
    | Clarity | Ambiguous requirements |
    | YAGNI | Unrequested features, over-engineering |
    | Scope | Focused enough for a single plan — not covering multiple independent subsystems |
    | Architecture | Units with clear boundaries, well-defined interfaces, independently understandable and testable |

    ## CRITICAL

    Look especially hard for:
    - Any TODO markers or placeholder text
    - Sections saying "to be defined later" or "will spec when X is done"
    - Sections noticeably less detailed than others
    - Units that lack clear boundaries or interfaces — can you understand what each unit does without reading its internals?

    ## Groundedness (verify against the codebase before flagging)

    Before flagging an issue, verify your claim against the actual codebase. Read the referenced files. If you cannot verify, state it as a question, not a defect.

    **If Agent Brain CLI is available, use it FIRST:**
    - `agent-brain-cli query <project> "search text"` — architectural decisions and prior patterns
    - `agent-brain-cli query <project> --file <path>` — verify symbols referenced in the spec
    - `agent-brain-cli impact <project> <symbol>` — FULL caller / blast radius for every symbol the spec names

    If Agent Brain CLI is not available, fall back to grep/glob to verify claims.

    ### Transitive completeness (catches blast-radius bugs)

    For every symbol the spec cites or proposes to modify, verify the FULL inventory — not just the cited lines:

    - Run `impact <symbol> --depth 1` (or grep the symbol name project-wide) to enumerate ALL callers / constructors. Does the spec's "migrate these N callsites" list match? Missed transitive callers are a common blast-radius bug.
    - For functions gaining new positional args — confirm every call site is listed, or flag that the new arg should be keyword-only.
    - For classes whose constructors the spec modifies — check who constructs them. Middle-layer factories (pipelines, registries, dispatchers) are silently skipped when you only look at the leaf.

    Verifying leaf citations is necessary but not sufficient. "Line 82 really says what the spec claims" does not prove "these are all the places that need to change."

    ## Output Format

    ## Spec Review

    **Status:** ✅ Approved | ❌ Issues Found

    **Issues (if any):**
    - [Section X]: [specific issue] - [why it matters]

    **Recommendations (advisory):**
    - [suggestions that don't block approval]
```

**Reviewer returns:** Status, Issues (if any), Recommendations
