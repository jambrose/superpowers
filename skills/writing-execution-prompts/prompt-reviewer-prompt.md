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
    **Spec for reference:** [SPEC_FILE_PATH] — the design spec the plan
    decomposes. The plan may have inherited mistakes; the spec captures
    original intent. Check prompts against BOTH.
    **Codebase root:** [CODEBASE_ROOT]

    **Agent Brain CLI:** if available in this project (run
    `agent-brain-cli --help` to confirm), USE IT as your primary
    verification tool before grep/read. Specifically:
    - `agent-brain-cli query <project> --file <path>` — list symbols
      and signatures in a file the prompt cites
    - `agent-brain-cli impact <project> <symbol> --depth 1` — enumerate
      ALL callers of a symbol the prompt modifies (catches transitive
      callsites the prompt's "migrate these N callsites" list misses)
    - `agent-brain-cli dead-code <project>` — catch symbols the prompt
      claims to remove that still have live callers
    The project name is the directory name of the codebase root.

    ## What to Check

    | Category | What to Look For |
    |----------|------------------|
    | Spec fidelity | Prompts honor the spec's scope boundaries (in-scope vs out-of-scope) and don't instruct work the spec forbids. Flag if a prompt expands scope beyond what the spec authorizes. |
    | Plan fidelity | Each prompt's deliverables match the plan tasks it references |
    | Path verification | File paths in prompts exist or are marked CREATE |
    | Method spot-check | Sample `object.method()` calls verified against codebase (use `agent-brain-cli query --file` / `impact` first) |
    | Template compliance | Mandatory sections from agent prompt template are present |
    | Dependency chain | No prompt references files created by a later prompt |
    | Parallelism safety | Prompts marked parallel have zero file overlap |
    | Spec ID coverage | Every spec ID from the plan appears in at least one prompt |

    ## Verification Steps

    1. **Spec scope check**: Read the spec's "Out of Scope" / "Non-Goals" section (if present). For each prompt, verify its Deliverables don't instruct work that the spec explicitly excludes. Line-range delete instructions are a common failure mode: a prompt inheriting a plan's "delete lines N-M" can sweep up out-of-scope symbols that happen to live in the same range. Flag any prompt whose surgical instructions touch spec-forbidden territory.

    2. **Plan coverage**: List all plan tasks. For each, confirm a prompt
       covers it. Flag any plan tasks with no corresponding prompt.

    3. **File overlap check**: For prompts with the same number prefix and
       letter suffixes (parallel prompts), verify no shared files in their
       Targets sections. If any file appears in multiple parallel prompts,
       flag as High severity.

    4. **Method spot-check** (use Agent Brain CLI first if available): Pick 3-5 method calls from across the prompts. Verify each method exists on the target class AND that the prompt's call matches the real signature (arg count, kwarg names, return type). `agent-brain-cli query <project> --file <path>` lists symbols and signatures; `agent-brain-cli impact <project> <ClassName>` enumerates callers to help pick non-trivial methods to verify (don't just sample leaf getters). Prompts that invent plausible-looking APIs are a top cause of zero-context execution failures.

    5. **Line-range sanity check** (when a prompt says "delete lines N-M" or cites a specific line range): open the file and confirm every line in the range is actually the symbol the prompt intends to remove. Ranges inherited from plans are a common source of off-by-one errors that can delete adjacent out-of-scope code.

    6. **Template compliance**: Check that every prompt includes the
       mandatory Deliverables sections (code review dispatch, report line,
       assumptions, evidence). Check `docs/templates/` for the project's
       agent prompt template if one exists.

    7. **Dependency ordering**: For each prompt that says
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
