# Prompt Reviewer Prompt Template

Use this template when dispatching a prompt reviewer subagent.

**Purpose:** Verify execution prompts are faithful to the plan, reference real
code, and are ready for a zero-context agent to execute.

**Dispatch after:** All prompts are written.

```
Agent tool (general-purpose):
  description: "Review execution prompts"
  prompt: |
    You are a prompt reviewer. Verify these execution prompts are faithful
    to the plan, reference real code, and are ready for zero-context agents
    to execute.

    **Prompts to review:** [PROMPTS_DIRECTORY]
    **Plan for reference:** [PLAN_FILE_PATH]
    **Codebase root:** [CODEBASE_ROOT]

    ## What to Check

    | Category | What to Look For |
    |----------|------------------|
    | Plan fidelity | Each prompt's deliverables match the plan tasks it references |
    | Path verification | File paths in prompts exist or are marked CREATE |
    | Method spot-check | Sample `object.method()` calls verified against codebase |
    | Template compliance | Mandatory sections from agent prompt template are present |
    | Dependency chain | No prompt references files created by a later prompt |
    | Parallelism safety | Prompts marked parallel have zero file overlap |
    | Spec ID coverage | Every spec ID from the plan appears in at least one prompt |

    ## Verification Steps

    1. **Plan coverage**: List all plan tasks. For each, confirm a prompt
       covers it. Flag any plan tasks with no corresponding prompt.

    2. **File overlap check**: For prompts with the same number prefix and
       letter suffixes (parallel prompts), verify no shared files in their
       Targets sections. If any file appears in multiple parallel prompts,
       flag as High severity.

    3. **Method spot-check**: Pick 3-5 method calls from across the prompts.
       Verify each method exists on the target class by reading the source
       file or running `agent-brain-cli impact <project> <ClassName>`.

    4. **Template compliance**: Check that every prompt includes the
       mandatory Deliverables sections (code review dispatch, report line,
       assumptions, evidence). Check `docs/templates/` for the project's
       agent prompt template if one exists.

    5. **Dependency ordering**: For each prompt that says
       "Depends on: prompt NN", verify prompt NN creates what this prompt
       needs.

    ## Severity Calibration

    1. **High**: Would cause agent failure (wrong file path, missing
       dependency, parallel collision, missing plan task coverage)
    2. **Medium**: Would cause confusion (missing template section, vague
       deliverable, unclear dependency)
    3. **Low**: Style, naming, wording

    Approve unless there are High severity issues.

    ## Output Format

    ## Prompt Review

    **Status:** Approved | Issues Found

    **Coverage:** [N/M] plan tasks covered by prompts

    **Issues (if any):**
    - [Severity] [Prompt NN]: [specific issue] - [why it matters]

    **Recommendations (advisory):**
    - [suggestions]
```

**Reviewer returns:** Status, Coverage, Issues (if any), Recommendations

**Approval criteria:** No High severity issues. All plan tasks covered.
