---
description: Debugging command. Activates DEBUG mode for systematic problem investigation.
---

# /debug - Systematic Problem Investigation

$ARGUMENTS

---

## Purpose

This command activates DEBUG mode using the `systematic-debugging` skill for evidence-based, 4-phase problem investigation. No random guesses — only structured evidence.

---

## Behavior

When `/debug` is triggered, follow the **4-Phase Systematic Debugging Process**:

### Phase 1: REPRODUCE

Before touching any code, reliably reproduce the issue.

```markdown
## Reproduction Steps
1. [Exact step to reproduce]
2. [Next step]
3. Expected: [what should happen]
4. Actual: [what actually happens]

## Reproduction Rate
- [ ] Always (100%)
- [ ] Often (50-90%)
- [ ] Sometimes (10-50%)
- [ ] Rare (<10%)
```

> ⚠️ **Never fix a bug you can't consistently reproduce.**

### Phase 2: ISOLATE

Narrow down the source with targeted questions:

- When did this start? → `git log --oneline -20`
- What changed recently? → `git diff HEAD~5`
- Does it happen in all environments? (dev vs prod)
- What's the minimal reproduction case?

### Phase 3: UNDERSTAND — Root Cause (The 5 Whys)

Find the **root cause**, not just the symptom:

```markdown
1. Why: [First observation — the visible symptom]
2. Why: [Deeper reason]
3. Why: [Still deeper]
4. Why: [Getting closer]
5. Why: [Root cause ← THIS is what we fix]
```

### Phase 4: FIX & VERIFY

Apply fix, then verify it's truly resolved:

```markdown
## Fix Verification
- [ ] Bug no longer reproduces
- [ ] Related functionality still works
- [ ] No new issues introduced
- [ ] Regression test added to prevent recurrence
```

---

## Output Format

```markdown
## 🔍 Debug: [Issue]

### 1. Symptom
[What's happening]

### 2. Reproduction
- Rate: [Always / 50% / Rare]
- Steps: [minimal reproduction]

### 3. Isolation
- Started after: [git commit/change]
- Affected: [environments/users/scope]
- NOT affected: [scope boundary]

### 4. Root Cause (5 Whys)
1. Why → [symptom]
2. Why → [deeper]
3. Why → [root]
🎯 Root cause: [Clear explanation]

### 5. Fix
// Before
[broken code]

// After
[fixed code]

### 6. Prevention
- 🛡️ Regression test added: [yes/no + file]
- 📋 Similar code checked: [yes/no]
- 📝 Root cause documented: [yes/no]
```

---

## Useful Debug Commands

```bash
# Recent changes (find when bug was introduced)
git log --oneline -20
git diff HEAD~5

# Search for pattern in codebase
grep -r "errorPattern" --include="*.ts"

# Check production logs
pm2 logs app-name --err --lines 100
```

---

## Examples

```
/debug login not working
/debug API returns 500
/debug form doesn't submit
/debug data not saving to database
/debug memory leak in production
/debug performance degradation after deploy
```

---

## Anti-Patterns (NEVER do these)

❌ **Random changes** — "Maybe if I change this..."
❌ **Ignoring evidence** — "That can't be the cause"
❌ **Assuming** — "It must be X" without proof
❌ **Skipping reproduction** — Fixing a bug you haven't reproduced
❌ **Stopping at symptoms** — Not finding the root cause
