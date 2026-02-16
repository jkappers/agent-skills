# Execution Protocol and Rollback Procedures

Append these sections to every sprint plan document.

## Execution Protocol

1. Read acceptance criteria before starting. Criteria define completion.
2. Use `reference_material` paths to locate source code. Do not guess or duplicate.
3. Complete ALL acceptance criteria for a sprint before proceeding to next sprint.
4. After each sprint, output:
   - Files created/modified (absolute paths)
   - Build command result (full error output if failure)
   - Verification results (commands run + output)
   - Deviations or blockers encountered
5. If a sprint fails:
   - Stop immediately. Do not continue to next sprint.
   - Output failure details: file path, line number, error message
   - Propose fix or request clarification
6. If a criterion is ambiguous, request clarification before starting sprint.

## Rollback on Failure

If Sprint N fails acceptance criteria:

1. Run `git status` to identify modified files
2. Run `git diff` to review changes
3. If changes are salvageable:
   - Fix specific errors
   - Re-run acceptance criteria
   - Proceed only if ALL criteria pass
4. If changes are not salvageable:
   - Run `git restore .` to revert all changes
   - Re-read sprint definition
   - Restart sprint from clean state
5. After 2 failed attempts on same sprint:
   - Stop execution
   - Request human review of sprint definition
   - Sprint may need scope reduction or clarification
