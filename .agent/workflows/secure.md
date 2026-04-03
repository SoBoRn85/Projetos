---
description: Security audit workflow. Full vulnerability scan using OWASP 2025, supply chain analysis, and risk prioritization.
---

# /secure - Security Audit

$ARGUMENTS

---

## Purpose

This command activates a comprehensive security audit using the `vulnerability-scanner` skill. Use before deploys, after major changes, or when suspicious behavior is detected.

---

## Sub-commands

```
/secure              - Full audit of current project
/secure scan         - Quick automated scan only
/secure deps         - Dependency vulnerability analysis
/secure [file/path]  - Audit specific file or directory
/secure report       - Generate security report
```

---

## Behavior

When `/secure` is triggered:

### Phase 1: Reconnaissance

Understand the target before scanning:

```bash
# Map the attack surface
python .agent/skills/vulnerability-scanner/scripts/security_scan.py .

# Dependency audit
python .agent/skills/dependency-analyzer/scripts/dependency_analyzer.py .
```

Identify:
- Technology stack and entry points
- Data flows and trust boundaries
- External integrations and APIs
- User input sources

### Phase 2: OWASP 2025 Analysis

Evaluate against OWASP Top 10:

| # | Category | Check |
|---|----------|-------|
| A01 | Broken Access Control | IDOR, SSRF, auth checks |
| A02 | Security Misconfiguration | Headers, defaults, exposed services |
| A03 | Software Supply Chain 🆕 | Dependencies, CI/CD integrity |
| A04 | Cryptographic Failures | Weak crypto, exposed secrets |
| A05 | Injection | SQL, NoSQL, command injection |
| A06 | Insecure Design | Flawed auth architecture |
| A07 | Authentication Failures | Session, credential management |
| A08 | Integrity Failures | Unsigned updates |
| A09 | Logging & Alerting | Blind spots, monitoring |
| A10 | Exceptional Conditions 🆕 | Fail-open states, error handling |

### Phase 3: Risk Prioritization

Using CVSS + EPSS scoring:

```
Critical → Immediate action required
High     → Fix before next deploy
Medium   → Schedule for next sprint
Low      → Backlog
```

### Phase 4: Reporting

Structure findings with:
1. **What?** - Clear vulnerability description
2. **Where?** - Exact file, line, endpoint
3. **Why?** - Root cause
4. **Impact?** - Business consequence
5. **Fix?** - Specific remediation steps

---

## Output Format

```markdown
## 🔐 Security Audit: [Project/Target]

### Threat Model
- Assets: [what we're protecting]
- Actors: [who might attack]
- Vectors: [how they'd attack]

### Findings

#### 🔴 Critical (fix immediately)
1. **[Vulnerability Name]**
   - Where: `file.ts:45`
   - Why: [Root cause]
   - Impact: [Business risk]
   - Fix: [Remediation steps]

#### 🟠 High (fix before deploy)
[...]

#### 🟡 Medium (next sprint)
[...]

#### 🟢 Low (backlog)
[...]

### Automated Scan Results
- security_scan.py: [PASS/FAIL - N issues]
- dependency_analyzer.py: [PASS/FAIL - N vulnerabilities]

### Remediation Priority
1. [Most critical fix first]
2. [Second priority]

### Supply Chain Status
- Dependencies audited: [N]
- Known vulnerabilities: [N]
- Lock files committed: [YES/NO]
```

---

## Examples

```
/secure
/secure scan
/secure deps
/secure src/auth/
/secure report
```

---

## Key Principles

- **Think like an attacker** - what would they target?
- **Zero Trust** - never trust, always verify
- **Prioritize by exploitability** - EPSS score matters more than CVSS alone
- **Fix root causes** - not just symptoms
- **Continuous scanning** - not just before deploy
