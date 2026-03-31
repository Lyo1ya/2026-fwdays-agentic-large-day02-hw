---
description: "Run full AI code review with scoring and actionable fix suggestions"
globs: "**/*.{ts,tsx,js,jsx}"
---

# Command: review code

## Instructions

Perform a full AI code review of the current branch/worktree changes.

Review scope:
- Staged and unstaged diffs
- Newly added files
- Related tests and impacted code paths

Assess and report:
- Correctness bugs and behavioral regressions
- Security and privacy risks
- Performance and scalability concerns
- Maintainability, readability, and architecture fit
- API/typing/contracts and edge-case handling
- Test coverage quality and missing scenarios

For every issue, include:
- Severity: `Critical`, `High`, `Medium`, or `Low`
- Impact: what can break and for whom
- Evidence: file path and relevant code reference
- Actionable fix suggestion: specific change proposal
- Test suggestion: what test should be added/updated

## Output Format

Return results in this exact structure:

### Summary
- Short review scope and overall risk statement (2-4 bullets)

### Issues
- List findings ordered by severity (highest first)
- Use one block per issue:
  - `ID`: short unique identifier (for example `REV-001`)
  - `Severity`: `Critical|High|Medium|Low`
  - `Category`: bug, security, performance, testing, design, etc.
  - `Location`: file path(s)
  - `Problem`: concise description
  - `Why it matters`: user/system impact
  - `Actionable fix`: concrete step-by-step suggestion
  - `Suggested test`: exact test scenario

If no issues are found, write: `No blocking issues found.`
Then include residual risks and testing gaps.

### Scores
- Provide numeric scores from `0-10`:
  - `Correctness`
  - `Security`
  - `Performance`
  - `Maintainability`
  - `Testing`
- Include `Overall Score` as the weighted average:
  - Correctness 30%
  - Security 25%
  - Performance 15%
  - Maintainability 15%
  - Testing 15%
- Explain each score briefly in one sentence.

### Verdict
- Return one:
  - `APPROVE`
  - `APPROVE WITH NITS`
  - `REQUEST CHANGES`
  - `BLOCK`
- Add a one-paragraph rationale.

## Behavior Rules

- Findings-first output: list issues before broad summary commentary.
- Be evidence-based: no speculation without clear uncertainty labels.
- Prefer high-signal issues over style-only comments.
- Avoid duplicate findings; merge related issues into one entry.
- Do not expose secrets, tokens, or sensitive payloads in the review.
- Keep recommendations practical and minimally disruptive where possible.
- If uncertain, state assumptions and what would confirm them.
- If code is good, explicitly say so and still report residual risk/testing gaps.

## How To Verify

Use this checklist before finalizing the review:

1. Confirm all changed files were considered (staged + unstaged + new files).
2. Confirm issues are ordered by severity and include locations.
3. Confirm each issue has an actionable fix and a test suggestion.
4. Confirm all five category scores and weighted overall score are present.
5. Confirm verdict is present and consistent with findings.
6. Confirm no sensitive data is quoted in output.
7. Confirm "no issue" case still includes residual risks/testing gaps.
