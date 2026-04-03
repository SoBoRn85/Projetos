---
description: Code review workflow. Systematic review of code quality, security, and best practices using structured checklist.
---

# /review - Code Review

$ARGUMENTS

---

## Purpose

This command activates a structured code review using the `code-review-checklist` skill. Use when reviewing PRs, files, or specific features before merging.

---

## Behavior

When `/review` is triggered:

### Step 1: Identify Scope

Determine what to review:
- Specific file(s): `/review src/auth.service.ts`
- Feature/PR: `/review user authentication feature`
- All recent changes: `/review` (uses `git diff`)

### Step 2: Gather Context

```bash
# Recent changes
git log --oneline -10
git diff HEAD~1 --name-only
```

### Step 3: Apply Review Checklist

Load `code-review-checklist` skill and systematically evaluate:

1. **Correctness** — Does the code do what it's supposed to?
2. **Security** — Input validation, injection risks, secrets exposure, AI-specific risks
3. **Performance** — N+1 queries, unnecessary loops, caching
4. **Code Quality** — Naming, DRY, SOLID principles
5. **Testing** — Coverage of new code, edge cases
6. **Documentation** — Complex logic commented, public APIs documented

### Step 4: Run Automated Checks

```bash
python .agent/skills/vulnerability-scanner/scripts/security_scan.py .
python .agent/skills/lint-and-validate/scripts/lint_runner.py .
```

---

## Output Format

```markdown
## 🔎 Code Review: [Target]

### Summary
[Brief overview of what was reviewed]

### 🔴 Blocking Issues (must fix before merge)
- [Issue with file:line reference]

### 🟡 Suggestions (should fix)
- [Recommendation with reasoning]

### 🟢 Nits (optional improvements)
- [Minor polish items]

### ✅ Passed Checks
- Correctness: OK
- Security: OK
- Performance: OK
- Tests: OK

### Verdict
[ ] ✅ Approved
[ ] 🔄 Approved with minor changes
[ ] 🚫 Changes requested
```

---

## Severity Guide

| Symbol | Meaning | Action Required |
|--------|---------|----------------|
| 🔴 BLOCKING | Security/correctness issue | Must fix before merge |
| 🟡 SUGGESTION | Best practice violation | Should fix |
| 🟢 NIT | Polish/style | Optional |
| ❓ QUESTION | Clarification needed | Discuss with author |

---

## AI-Specific Review (2025)

When reviewing AI/LLM code, also check:
- [ ] Prompt injection protection
- [ ] Outputs sanitized before critical sinks
- [ ] Chain-of-thought is verifiable
- [ ] External state assumptions are safe

---

## Examples

```
/review
/review src/services/payment.service.ts
/review authentication PR
/review all changes since last release
```

---

## Key Principles

- **Focus on blocking issues first** - don't let minor nits overshadow real problems
- **Explain why** - not just what's wrong
- **Be constructive** - suggest fixes, not just criticism
- **Check AI risks** - prompt injection is a real 2025 threat
