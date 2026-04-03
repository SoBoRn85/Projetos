# Operational Protocol & Core Principles

> Standards and behaviors for AI agents working within the Antigravity environment.

---

## 💎 Core Principles

- **Simplicity First**: Make every change as simple as possible. Aim for minimal code impact and avoid over-engineering.
- **No Laziness**: Find root causes. No temporary workarounds. Maintain senior developer standards in every task.
- **Minimal Impact**: Changes should only touch what's necessary. Avoid "scope creep" in code edits to prevent regression bugs.
- **Elegance (Balanced)**: For non-trivial changes, pause and ask "is there a more elegant way?". If a fix feels hacky, implement the elegant solution knowing the full context.

---

## 🚀 Workflow Orchestration

### 1. Plan Mode Default
- **Trigger**: Enter plan mode for ANY non-trivial task (3+ steps or architectural decisions).
- **Pivot**: If an implementation goes sideways, STOP and re-plan immediately — don't keep pushing a failing strategy.
- **Verification**: Use plan mode to define verification steps, not just building steps.

### 2. Subagent Strategy
- **Isolation**: Use subagents liberally to keep the main context window clean.
- **Offloading**: Offload research, exploration, and parallel analysis to subagents.
- **Focus**: One clear task per subagent for focused and reliable execution.

### 3. Verification Before Done
- **Proof of Work**: Never mark a task complete without proving it works.
- **Full Suite**: Run relevant tests (or the full suite for structural changes) before considering work finished.
- **Standard**: Ask yourself: "Would a staff engineer approve this implementation?"

### 4. Autonomous Problem Solving
- **Bug Fixing**: When given a bug report, focus on fixing it without excessive hand-holding.
- **Root Cause**: Run tests to identify the root cause autonomously.
- **Context**: Aim for zero context switching required from the user during the fixing process.

---

## 🛠️ Skill Quality Standards

When creating or modifying skills, follow this description structure:

- **Structure**: `[What it does] + [Use when ...] + [Do NOT use for ...]`
- **"Use when"**: Mandatory. Include actual phrases or scenarios users would encounter.
- **"Do NOT use for"**: Mandatory. Add negative triggers to prevent overlap with similar skills.
- **Clarity**: Keep descriptions under 1024 characters and avoid XML angle brackets.
