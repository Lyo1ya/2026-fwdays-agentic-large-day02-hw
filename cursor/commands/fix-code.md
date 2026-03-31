---
description: "Apply fixes based on AI code review output"
globs: "**/*.{ts,tsx,js,jsx}"
---

# Command: Fix code

## Context

You are applying code fixes based on an AI code review report (for example, output from `.cursor/commands/review-code.md`).
Your goal is to safely implement high-value fixes with minimal regression risk, while preserving intended behavior and project conventions.

## Inputs

Required inputs:
- `Review Output`: the AI review result containing issues, severity, fix suggestions, and verdict.
- `Codebase State`: current working tree (staged/unstaged changes and relevant files).

Optional inputs:
- `Target Scope`: specific issue IDs to fix (for example `REV-001, REV-003`).
- `Constraints`: files to avoid, time limits, or risk limits.

If required input is missing, stop and request it before editing code.

## Instructions

Follow these steps in order:

1. Parse the `Review Output` and extract all issues with:
   - ID, severity, location, actionable fix, suggested test.
2. Validate each issue against current code:
   - Confirm the issue still exists.
   - Skip stale/invalid issues and record why.
3. Prioritize fixes:
   - Critical/High first, then Medium, then Low.
   - Prefer low-risk, high-impact fixes if time is constrained.
4. Implement fixes incrementally:
   - Make small, focused edits per issue.
   - Keep behavior backward compatible unless explicitly required.
5. Update/add tests for each fixed issue when feasible.
6. Run relevant verification (lint/tests/typecheck or targeted checks).
7. Produce final output in the mandatory format.

## Fix Strategy

Apply this strategy for each issue:

1. **Reproduce/Reason**
   - Identify failing path, unsafe logic, or missing guard.
2. **Minimal Safe Change**
   - Implement the smallest change that fully resolves root cause.
3. **Defense-in-Depth**
   - Add input validation, null/edge handling, and safe defaults.
4. **Test Reinforcement**
   - Add/adjust tests for the exact failure mode and one edge case.
5. **Regression Scan**
   - Check adjacent flows and API contracts for side effects.
6. **Document Decision**
   - Record why this approach was chosen (especially if alternatives exist).

## Output Format (MANDATORY)

Return output in this exact structure:

### Summary
- What was fixed, scope covered, and overall risk after fixes (2-5 bullets).

### Applied Fixes
- One entry per fixed issue:
  - `Issue ID`:
  - `Severity`:
  - `Status`: `Fixed` | `Partially Fixed` | `Skipped`
  - `Files Changed`:
  - `What changed`:
  - `Why this fix works`:
  - `Tests updated/added`:
  - `Follow-up needed` (if any):

### Updated Code
- Provide concise code snippets/diffs for key changes only.
- Include file paths for each snippet.
- Do not dump entire files.

## Behavior Rules

- Be deterministic and explicit: no vague "improved logic" statements.
- Do not claim fixes that were not actually implemented.
- Keep fixes scoped to reviewed issues unless a required dependency change is needed.
- Merge duplicate/overlapping issues into one coherent implementation.
- Prefer readability and maintainability over cleverness.
- If an issue cannot be safely fixed now, mark as `Skipped` with rationale.

## Safety Constraints

- Never expose or log secrets, tokens, keys, or sensitive payloads.
- Do not introduce unsafe execution (`eval`, dynamic code execution).
- Do not add new dependencies without explicit approval.
- Do not modify protected/core files unless explicitly authorized.
- Preserve backward compatibility for public interfaces unless explicitly requested.
- Avoid broad refactors unrelated to the reviewed issues.

## How To Verify

Use this checklist before final output:

1. Confirm every targeted issue is marked `Fixed`, `Partially Fixed`, or `Skipped`.
2. Confirm each `Fixed` issue maps to concrete file changes.
3. Confirm tests were added/updated for fixed behavior (or explain why not).
4. Confirm lint/typecheck/tests relevant to touched code were executed.
5. Confirm no sensitive data appears in diffs, logs, or output.
6. Confirm no out-of-scope/unapproved changes were introduced.
7. Confirm mandatory output sections are present exactly:
   - `Summary`
   - `Applied Fixes`
   - `Updated Code`
