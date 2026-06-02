Generate UI regression tests (manual case + Maestro YAML) and auto-run them, via the `maestro-testgen` skill.

Invoke the **maestro-testgen** skill (via the Skill tool) and follow its workflow exactly.

Input handling:
- If `$ARGUMENTS` describes a business flow in natural language, treat it as input mode B and pass it as the description.
- If `$ARGUMENTS` is empty or says "diff", derive the behaviors from the current `git diff` (input mode A).

The skill will: emit a Test Routing Decision -> write the manual test case -> judge automation feasibility -> (if `ready`) generate the Maestro YAML -> auto-run when a device is online and emit an HTML report + screenshots into the qa dir.

Output conventions (or the project's existing dirs):
- Manual case  -> qa/manual-cases/<case_id>.md
- Maestro flow -> qa/manual-cases/maestro-flows/<case_id>.yaml
- Report       -> qa/manual-cases/reports/<case_id>.html   (always pass --format; default NOOP = no report)
- Screenshots  -> qa/manual-cases/evidence/<case_id>/        (via --test-output-dir + in-flow takeScreenshot)

Arguments: $ARGUMENTS
