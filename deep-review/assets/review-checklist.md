# Code Review Quick Checklist

Quick reference checklist for code reviews.

## Pre-Review (2 min)

- [ ] Read PR description and linked issue
- [ ] Check PR size (<400 lines ideal)
- [ ] Verify CI/CD status (tests passing?)
- [ ] Understand the business requirement

## Repo Alignment (2 min)

- [ ] PR follows the repo's own rules: CLAUDE.md / CONTRIBUTING / ADRs / module BLUEPRINT-REQUIREMENTS checked (see reference/house-rules.md)
- [ ] No module-boundary violations: no imports of another module's internals, no shared DB tables (see reference/architecture-review-guide.md)

## Architecture & Design (5 min)

- [ ] Solution fits the problem
- [ ] Consistent with existing patterns
- [ ] No simpler approach exists
- [ ] Will it scale?
- [ ] Changes in right location

## Logic & Correctness (10 min)

- [ ] Edge cases handled
- [ ] Null/undefined checks present
- [ ] Off-by-one errors checked
- [ ] Race conditions considered
- [ ] Error handling complete
- [ ] New failure paths log with context and emit metrics where the module has them
- [ ] Correct data types used

## Security (5 min)

- [ ] No hardcoded secrets
- [ ] Input validated/sanitized
- [ ] SQL injection prevented
- [ ] XSS prevented
- [ ] Authorization checks present
- [ ] Sensitive data protected

## Performance (3 min)

- [ ] No N+1 queries
- [ ] Expensive operations optimized
- [ ] Large lists paginated
- [ ] No memory leaks
- [ ] Caching considered where appropriate
- [ ] DB migrations backward-compatible (expand/contract); no lock-heavy operations on hot tables

## Testing (5 min)

- [ ] Tests exist for new code
- [ ] Edge cases tested
- [ ] Error cases tested
- [ ] Tests are readable
- [ ] Tests are deterministic
- [ ] Feature flags: clear name, correct default, removal path, both paths tested

## Code Quality (3 min)

- [ ] Clear variable/function names
- [ ] No code duplication
- [ ] Functions do one thing
- [ ] Complex code commented
- [ ] No magic numbers
- [ ] All content in English (comments, strings, docs)

## Documentation (2 min)

- [ ] Public APIs documented
- [ ] README updated if needed
- [ ] Breaking changes noted
- [ ] Complex logic explained

---

## Severity Labels

| Label             | Meaning      | Action              |
| ----------------- | ------------ | ------------------- |
| 🔴 `[blocking]`   | Must fix     | Block merge         |
| 🟡 `[important]`  | Should fix   | Discuss if disagree |
| 🟢 `[nit]`        | Nice to have | Non-blocking        |
| 💡 `[suggestion]` | Alternative  | Consider            |
| ❓ `[question]`   | Need clarity | Respond             |

---

## Decision Matrix

| Situation                         | Decision                  |
| --------------------------------- | ------------------------- |
| Critical security issue           | 🔴 Block, fix immediately |
| Breaking change without migration | 🔴 Block                  |
| Missing error handling            | 🟡 Should fix             |
| No tests for new code             | 🟡 Should fix             |
| Style preference                  | 🟢 Non-blocking           |
| Minor naming improvement          | 🟢 Non-blocking           |
| Clever but working code           | 💡 Suggest simpler        |

---

## Time Budget

| PR Size       | Target Time  |
| ------------- | ------------ |
| < 100 lines   | 10-15 min    |
| 100-400 lines | 20-40 min    |
| > 400 lines   | Ask to split |

---

## Red Flags

Watch for these patterns:

- `// TODO` without a ticket reference
- `console.log` left in code
- Commented out code
- `any` type in TypeScript
- Empty catch blocks
- `unwrap()` in Rust production code
- Magic numbers/strings
- Copy-pasted code blocks
- Missing null checks
- Hardcoded URLs/credentials
