---
description: Add or update features in existing application. Used for iterative development.
---

# /enhance - Update Application

$ARGUMENTS

---

## Purpose

This command adds features or updates to an existing application using an impact-first approach. Always understand before acting, plan before coding.

---

## Steps

### Step 1: Understand Current State

Load the execution context before touching anything:

```bash
python .agent/scripts/session_manager.py info
```

Map the affected area:
- What tech stack is in use?
- What files are likely affected?
- Are there existing tests for the area being changed?
- Are there conflicting patterns? (e.g., "add Firebase" when project uses Supabase)

### Step 2: Impact Analysis

Before coding, classify the scope:

| Scope | Definition | Action |
|-------|------------|--------|
| **Small** | 1-3 files, no new deps | Proceed with inline plan |
| **Medium** | 4-10 files, or new dependency | Present plan, ask approval |
| **Large** | 10+ files, architecture change | Use `/plan` workflow first |

### Step 3: Present Plan (for Medium/Large changes)

```
"To add [feature]:
📁 New files: [N] (list key ones)
✏️  Modified files: [N] (list key ones)
📦 New dependencies: [list if any]
⏱️  Estimated effort: ~[N] minutes

⚠️  Watch out: [any potential conflicts]

Should I start? (y/n)"
```

> 🔴 **STOP** for large changes — use `/plan` first to create a PLAN-{feature}.md file.

### Step 4: Apply Changes

Activate the appropriate specialist:
- UI changes → `frontend-specialist` + `frontend-design` skill
- API changes → `backend-specialist` + `api-patterns` skill
- DB changes → `database-architect` + `database-design` skill
- Mobile → `mobile-developer` + `mobile-design` skill

### Step 5: Verify & Preview

```bash
# Run linting
python .agent/skills/lint-and-validate/scripts/lint_runner.py .

# If tests exist
npm test

# Restart preview
python .agent/scripts/auto_preview.py restart
```

### Step 6: Commit

```bash
git add -A
git commit -m "feat: [brief description of feature added]"
```

---

## Output Format

```markdown
## ✨ Enhancement Complete: [Feature Name]

### What Changed
- Created: [N files]
- Modified: [N files]
- Dependencies added: [list or none]

### How to Use
[2-3 sentence description of the new feature]

### Known Limitations / Next Steps
- [Any caveats or follow-up work]

### Preview
[URL if preview server is running]
```

---

## Conflict Detection

Before applying, warn if:
- Adding a new auth library when one already exists
- Switching databases mid-project
- Installing a UI framework that conflicts with existing one
- Adding state management that conflicts with current pattern

---

## Usage Examples

```
/enhance add dark mode
/enhance build admin panel
/enhance integrate Stripe payment
/enhance add search feature with filters
/enhance make landing page responsive
/enhance add email notifications
```

---

## Key Principles

- **Understand before acting** - never modify code without reading it first
- **Classify scope** - small = go, large = plan first
- **Get approval for breaking changes** - especially new dependencies
- **Commit each feature** - one feature per commit
- **Test after every change** - don't accumulate untested code
