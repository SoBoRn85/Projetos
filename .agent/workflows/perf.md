---
description: Performance profiling and optimization workflow. Core Web Vitals audit, bundle analysis, runtime profiling, and optimization recommendations.
---

# /perf - Performance Profiling

$ARGUMENTS

---

## Purpose

This command activates performance analysis using the `performance-profiling` skill. Measures first, then optimizes. Use before deploys, when users report slowness, or after major feature additions.

---

## Sub-commands

```
/perf                      - Full performance audit
/perf lighthouse <url>     - Lighthouse audit for specific URL
/perf bundle               - Bundle size analysis
/perf profile              - Runtime performance profiling
/perf vitals               - Core Web Vitals check
```

---

## Behavior

### The 4-Step Process (ALWAYS follow this order)

```
1. BASELINE  → Measure current state (never skip!)
2. IDENTIFY  → Find the real bottleneck
3. FIX       → Make targeted change only
4. VALIDATE  → Confirm improvement with metrics
```

> ⚠️ **Never skip step 1.** Optimizing without measuring is guessing.

---

### Step 1: Baseline Measurement

```bash
# Lighthouse audit (requires URL)
python .agent/skills/performance-profiling/scripts/lighthouse_audit.py <url>

# Bundle analysis
python .agent/skills/performance-profiling/scripts/bundle_analyzer.py .
```

### Step 2: Core Web Vitals Targets

| Metric | Good | Poor | Measures |
|--------|------|------|----------|
| **LCP** | < 2.5s | > 4.0s | Loading performance |
| **INP** | < 200ms | > 500ms | Interactivity |
| **CLS** | < 0.1 | > 0.25 | Visual stability |

### Step 3: Identify Bottleneck

| Symptom | Likely Cause | Tool |
|---------|--------------|------|
| Slow initial load | Large JS, render blocking | Lighthouse |
| Slow interactions | Heavy event handlers | DevTools Performance |
| Layout jank | Layout thrashing | DevTools Performance |
| Growing memory | Memory leaks | DevTools Memory |

### Step 4: Quick Win Priorities

| Priority | Action | Impact |
|----------|--------|--------|
| 1 | Enable compression (gzip/brotli) | High |
| 2 | Lazy load images | High |
| 3 | Code split routes | High |
| 4 | Cache static assets | Medium |
| 5 | Optimize images (WebP/AVIF) | Medium |
| 6 | Remove unused JS | Medium |

---

## Output Format

```markdown
## ⚡ Performance Report: [Target]

### Baseline Metrics
| Metric | Current | Target | Status |
|--------|---------|--------|--------|
| LCP    | Xs      | <2.5s  | ✅/❌  |
| INP    | Xms     | <200ms | ✅/❌  |
| CLS    | X.X     | <0.1   | ✅/❌  |
| Score  | XX/100  | >90    | ✅/❌  |

### Bundle Analysis
- Total size: [X MB]
- Largest dependencies: [list]
- Code splitting opportunities: [list]

### 🔴 Critical Issues (blocking good score)
1. [Issue] → [Fix]

### 🟡 Optimizations (high impact)
1. [Optimization] → [Expected improvement]

### 🟢 Quick Wins (low effort, measurable gain)
1. [Quick win]

### Implementation Plan
Phase 1 (this sprint): [quick wins]
Phase 2 (next sprint): [bigger optimizations]

### Expected Impact
After implementing: LCP ~Xs, Score ~XX/100
```

---

## Examples

```
/perf
/perf lighthouse https://myapp.com
/perf bundle
/perf profile
/perf vitals
```

---

## Key Principles

- **Measure first, always** - never optimize without data
- **Fix the biggest bottleneck** - not the easiest thing
- **Use real user data** - RUM > synthetic tests
- **The fastest code doesn't run** - remove before optimizing
- **Validate every change** - confirm improvement with metrics
