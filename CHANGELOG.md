# Changelog

## [5.1.9] - 2026-05-17

### Changed

- **code-review skill: unified severity taxonomy to High / Medium / Low.** Previously the
  skill prescribed Critical / Warning / Minor (in `SKILL.md`) and Critical / Important / Minor
  (in `code-reviewer.md`), which caused orchestrators to silently invent High / Medium / Low
  on the user-facing summary while subagents kept emitting Critical. The label drift killed
  the diagram trigger (keyed to "Critical" specifically) — diagrams almost never reached the
  posted summary. Unified taxonomy now used consistently in: Agent 2 severity rubric,
  conditional checklist, Consolidator handoff, Phase 3 scorer, Phase 4 inline-comment body,
  Phase 4 summary table, local-mode output, and `code-reviewer.md` headers.
- **Phase 3 scorer now emits `{severity, score, justification}` per finding.** Severity passes
  through from Phase 2 verbatim — the scorer no longer reclassifies. Severity and score are
  declared orthogonal: a High finding can score 60 (drops at gate); a Low finding can score 90
  (kept, low priority). Removes the leak point where numeric scores got translated back into
  invented severity labels at output time.
- **Phase 4 summary table now carries a Severity column.** Was `| File | Summary |` (free-text
  Summary was where label drift happened); now `| Severity | File | Note |` with explicit
  constraint that Severity is one of `High` / `Medium` / `Low` / `Clean` plus an anti-pattern
  list of labels not to invent (Critical, Warning, Minor, Important, Nit, P0/P1/P2, Major,
  Severe).
- **Diagram trigger widened from Critical-only to High OR Medium.** The Critical bar was so
  narrow (data loss / incorrect behavior / security only) that most genuine bugs landed in
  Warning and lost the diagram. Applied at Agent 2 prompt, Consolidator, Phase 4 line rules,
  and PR Comment Rules.
- **Consolidator gained light schema validation.** Off-rubric specialist severities (Critical,
  Warning, P1, etc.) get normalized in place with a `[normalized from <original>]` audit note
  so drift becomes visible rather than silent.
- **Summary body now ends with a REQUIRED, agent-agnostic attribution line.** Was
  `Generated with [Claude Code]` at the very bottom with no MUST clause; now
  `**Reviewed by:** <orchestrator-model> · superpowers:code-review · <harness-link>` with
  explicit instructions for the orchestrator to self-identify. Works for Claude, Grok,
  ChatGPT, or any future runtime — the orchestrator fills in its own model and harness link.

### Added

- **code-review skill: parallel reviewer spawning on Grok via `run_in_background: true`**
  ([b1993eb](https://github.com/jambrose/superpowers/commit/b1993eb)) — `.claude-plugin/plugin.json`
  was bumped to 5.1.9 at that commit but `package.json` + CHANGELOG were not. 5.1.9 now consolidates
  both that change and the taxonomy work above.

## [5.1.8] - 2026-05-17

### Fixed / Improved

- **code-review skill (Grok compatibility)**: Major fixes to make the `/code-review` skill work reliably on Grok:
  - Corrected subagent launch parameters (`subagent_type: "general-purpose" + persona: "reviewer"`, `background: false`).
  - Replaced impossible "hard synchronization barrier" rules with practical Grok-aware guidance.
  - Added explicit requirement to disclose when subagents fail to return output (prevents false claims that parallel reviewers ran).
  - Added "Grok vs Claude Execution Notes" section with recommended settings.
  - Updated Consolidator instructions to handle subagent retrieval failures gracefully.

This resolves the root cause of misleading review reports when using the code-review skill on Grok.

## [5.0.5] - 2026-03-17

### Fixed

- **Brainstorm server ESM fix**: Renamed `server.js` → `server.cjs` so the brainstorming server starts correctly on Node.js 22+ where the root `package.json` `"type": "module"` caused `require()` to fail. ([PR #784](https://github.com/obra/superpowers/pull/784) by @sarbojitrana, fixes [#774](https://github.com/obra/superpowers/issues/774), [#780](https://github.com/obra/superpowers/issues/780), [#783](https://github.com/obra/superpowers/issues/783))
- **Brainstorm owner-PID on Windows**: Skip `BRAINSTORM_OWNER_PID` lifecycle monitoring on Windows/MSYS2 where the PID namespace is invisible to Node.js. Prevents the server from self-terminating after 60 seconds. The 30-minute idle timeout remains as the safety net. ([#770](https://github.com/obra/superpowers/issues/770), docs from [PR #768](https://github.com/obra/superpowers/pull/768) by @lucasyhzhu-debug)
- **stop-server.sh reliability**: Verify the server process actually died before reporting success. Waits up to 2 seconds for graceful shutdown, escalates to `SIGKILL`, and reports failure if the process survives. ([#723](https://github.com/obra/superpowers/issues/723))

### Changed

- **Execution handoff**: Restore user choice between subagent-driven-development and executing-plans after plan writing. Subagent-driven is recommended but no longer mandatory. (Reverts `5e51c3e`)
