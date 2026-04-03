---
description: Skill creation and improvement workflow. Create new skills from scratch, improve existing skills, run evaluations, benchmark performance, and optimize skill descriptions for better triggering.
---

# /skill - Skill Creator & Improver

$ARGUMENTS

---

## Purpose

This command activates the `skill-creator` skill for creating new skills, improving existing ones, running evaluations, and optimizing triggering accuracy. The end goal is a properly structured, tested, and packaged `.skill` file.

---

## Sub-commands

```
/skill create <name>      - Create a new skill from scratch
/skill improve <name>     - Improve an existing skill
/skill eval <name>        - Run evaluations on a skill
/skill describe <name>    - Optimize skill description for better triggering
/skill list               - List all available skills
/skill package <name>     - Package skill into .skill file
```

---

## Core Loop (always follow this order)

```
1. INTENT    → What should this skill enable Claude to do?
2. DRAFT     → Write SKILL.md with proper structure
3. TEST      → Create 2-3 realistic test prompts
4. EVALUATE  → Run with-skill vs without-skill (baseline)
5. REVIEW    → Human reviews outputs in viewer
6. IMPROVE   → Revise based on feedback
7. REPEAT    → Until user is happy
8. OPTIMIZE  → Run description optimizer
9. PACKAGE   → Create .skill file
```

---

## Behavior

### Step 1: Capture Intent

Before writing anything, ask:
1. What should this skill enable Claude to do?
2. When should it trigger? (what user phrases/contexts)
3. What's the expected output format?
4. Should we set up test cases?

### Step 2: SKILL.md Structure

Every skill must have:

```markdown
---
name: skill-name
description: [What it does] + [Use when...] + [Do NOT use for...]
allowed-tools: Read, Write, Edit, Glob, Grep
---

# Skill Name

## Overview
[Purpose and problem it solves]

## Usage Patterns
[When to apply this skill]

## Core Process
[Step-by-step instructions]

## Output Format
[Expected output structure]

## Anti-Patterns
[What NOT to do]
```

### Step 3: Test Cases

Create `evals/evals.json` with realistic prompts:

```json
{
  "skill_name": "my-skill",
  "evals": [
    {
      "id": 1,
      "prompt": "A real user prompt...",
      "expected_output": "Description of what good looks like",
      "files": []
    }
  ]
}
```

### Step 4: Run Evaluations

For each eval, run TWO versions in parallel:
- **With skill**: Claude uses the new SKILL.md
- **Without skill (baseline)**: Claude without skill

Save results to `<skill-name>-workspace/iteration-1/`

### Step 5: Description Optimization

After the skill content is solid, optimize the description:

```bash
python -m scripts.run_loop \
  --eval-set trigger-eval.json \
  --skill-path .agent/skills/<skill-name>/ \
  --max-iterations 5 \
  --verbose
```

---

## Skill File Structure

```
.agent/skills/<skill-name>/
├── SKILL.md               (required)
├── scripts/               (optional - executable helpers)
│   └── my_script.py
├── references/            (optional - docs loaded as needed)
│   └── patterns.md
└── assets/                (optional - templates, files)
    └── template.html
```

---

## Output Format

```markdown
## 🛠️ Skill Created: [skill-name]

### Skill Summary
- Name: `skill-name`
- Triggers when: [user phrases]
- Does NOT trigger for: [anti-triggers]
- Tools needed: [list]

### Test Results (Iteration N)
| Eval | With Skill | Baseline | Winner |
|------|-----------|----------|--------|
| eval-1 | ✅ 3/3 | ❌ 1/3 | Skill |
| eval-2 | ✅ 3/3 | ❌ 0/3 | Skill |

### Description Optimization
- Before: [old description]
- After: [optimized description]
- Trigger accuracy: X% → Y%

### File Location
`c:\Users\Robson\Projetos\.agent\skills\<skill-name>\`
```

---

## Examples

```
/skill create rust-debugging
/skill improve frontend-design
/skill eval brainstorming
/skill describe vulnerability-scanner
/skill list
/skill package mcp-builder
```

---

## Key Principles

- **Explain the WHY** - not just instructions, but reasoning
- **Keep SKILL.md under 500 lines** - add reference files for details
- **Make descriptions "pushy"** - skills tend to undertrigger
- **Always test with baseline** - prove the skill adds value
- **Generalize from examples** - avoid overfitting to specific test cases
