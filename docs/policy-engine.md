# Policy Engine

> Configurable security gates, scoring, and verdicts

---

## Overview

The Policy Engine evaluates security findings to determine project viability:

```
        Findings
            │
            ▼
    ┌───────────────┐
    │  Gate Check   │──────► Any fail? ──► NO-GO
    └───────────────┘
            │
            │ All pass
            ▼
    ┌───────────────┐
    │ Score Calc    │──────► Score
    └───────────────┘
            │
            ▼
    ┌───────────────┐
    │   Threshold   │
    │    Check      │
    └───────────────┘
            │
     ┌──────┼──────┐
     ▼      ▼      ▼
   ≥80    50-79   <50
    │       │       │
    ▼       ▼       ▼
   GO   COND.   NO-GO
```

---

## Gates (Hard Blockers)

Gates are non-negotiable security requirements. Any gate failure triggers NO-GO.

### Default Gates

| Gate | Condition | Rationale |
|------|-----------|-----------|
| `no_secrets` | Any secret finding | Credentials in code are critical risks |
| `no_kev_vulnerabilities` | Any CVE in CISA KEV | Actively exploited = immediate threat |
| `no_critical_vulnerabilities` | Any critical CVE | Critical severity requires action |

### Gate Evaluation

```
For each enabled gate:
    │
    ▼
┌──────────────────────────────────────┐
│ Check condition against findings     │
│                                      │
│ condition_type:                      │
│   • finding_present                  │
│   • severity_present                 │
│   • count_exceeds                    │
└──────────────────────────────────────┘
    │
    ├── Triggered? ──► Add to failed gates
    │
    └── Not triggered? ──► Add to passed gates
```

### Custom Gates

```yaml
# Example: Block if more than 5 high-severity issues
gates:
  - name: max_high_severity
    description: "Limit high severity findings"
    enabled: true
    condition_type: count_exceeds
    condition_params:
      severity: high
      threshold: 5
    action: block
    message: "Too many high severity issues (>5)"
```

---

## Scoring System

### Category Weights

Score starts at 100 and deducts based on findings:

| Category | Weight | Finding Types |
|----------|--------|---------------|
| Secrets | 25% | Hardcoded credentials |
| Vulnerabilities | 35% | CVEs in dependencies |
| Code Issues | 25% | SAST findings |
| Supply Chain | 15% | Dependency health |

### Severity Multipliers

Each finding's impact is scaled by severity:

| Severity | Multiplier | Impact on Score |
|----------|------------|-----------------|
| Critical | 1.0 | Full weight |
| High | 0.7 | 70% weight |
| Medium | 0.4 | 40% weight |
| Low | 0.1 | 10% weight |
| Info | 0.0 | No impact |

### Score Calculation

```
For each category:
    │
    ▼
┌──────────────────────────────────────┐
│ deduction = Σ (severity_mult × 10)   │
│                                      │
│ category_deduction = min(deduction,  │
│                          weight×100) │
└──────────────────────────────────────┘
    │
    ▼
total_deduction = Σ category_deductions

score = max(0, 100 - total_deduction)
```

### Example Calculation

```
Findings:
  • 1 secret (high)
  • 2 vulnerabilities (1 critical, 1 medium)
  • 3 code issues (all medium)

Deductions:
  Secrets:       1 × 0.7 × 10 = 7.0
  Vulnerabilities: (1.0 + 0.4) × 10 = 14.0
  Code Issues:   3 × 0.4 × 10 = 12.0

Total Deduction: 7 + 14 + 12 = 33

Final Score: 100 - 33 = 67 (CONDITIONAL)
```

---

## Verdicts

### Verdict Determination

```
┌─────────────────────────────────────────────────────┐
│                  Verdict Logic                       │
├─────────────────────────────────────────────────────┤
│                                                      │
│  IF any gate failed:                                │
│      verdict = NO-GO                                │
│                                                      │
│  ELSE IF score >= 80:                               │
│      verdict = GO                                   │
│                                                      │
│  ELSE IF score >= 50:                               │
│      verdict = CONDITIONAL                          │
│                                                      │
│  ELSE:                                              │
│      verdict = NO-GO                                │
│                                                      │
└─────────────────────────────────────────────────────┘
```

### Verdict Meanings

| Verdict | Score | Gates | Action |
|---------|-------|-------|--------|
| **GO** | ≥ 80 | All pass | Safe to proceed |
| **CONDITIONAL** | 50-79 | All pass | Review findings first |
| **NO-GO** | < 50 OR any gate fail | — | Must address issues |

---

## Policy Configuration

### Default Policy

```yaml
name: default
version: "1.0.0"

gates:
  - name: no_secrets
    condition_type: finding_present
    condition_params:
      finding_type: secret
    action: block
    message: "Remove secrets and rotate credentials"

  - name: no_kev_vulnerabilities
    condition_type: finding_present
    condition_params:
      in_kev: true
    action: block
    message: "Patch actively exploited vulnerabilities"

  - name: no_critical_vulnerabilities
    condition_type: severity_present
    condition_params:
      severity: critical
    action: block
    message: "Update dependencies with critical CVEs"

score_weights:
  - category: secrets
    weight: 0.25
  - category: vulnerabilities
    weight: 0.35
  - category: code_issues
    weight: 0.25
  - category: supply_chain
    weight: 0.15

score_threshold_go: 80.0
score_threshold_conditional: 50.0
```

### Custom Policy Example

```yaml
# Stricter policy for production releases
name: production
version: "1.0.0"

gates:
  - name: no_secrets
    condition_type: finding_present
    condition_params:
      finding_type: secret
    action: block

  - name: no_high_or_above
    condition_type: severity_present
    condition_params:
      severity: high
    action: block
    message: "No high severity issues in production"

score_threshold_go: 90.0
score_threshold_conditional: 70.0
```

---

## Recommendations

The policy engine generates actionable recommendations:

```
┌─────────────────────────────────────────────────────┐
│              Recommendation Sources                  │
├─────────────────────────────────────────────────────┤
│                                                      │
│ 1. Failed gates                                     │
│    → Gate-specific remediation message              │
│                                                      │
│ 2. Critical/High findings                           │
│    → "Address N critical/high issue(s)"            │
│                                                      │
│ 3. Secret findings                                  │
│    → "Remove secrets and rotate credentials"        │
│                                                      │
│ 4. KEV findings                                     │
│    → "Patch actively exploited vulnerabilities"     │
│                                                      │
└─────────────────────────────────────────────────────┘
```

---

## Policy Output

### PolicyResult Schema

```
┌─────────────────────────────────────────────────────┐
│                  PolicyResult                        │
├─────────────────────────────────────────────────────┤
│ policy_name: "default"                              │
│ policy_version: "1.0.0"                             │
│                                                      │
│ verdict: "GO" | "NO-GO" | "CONDITIONAL"             │
│ score: 85.0                                         │
│                                                      │
│ gates_evaluated: 3                                  │
│ gates_passed: ["no_kev", "no_critical"]            │
│ gates_failed: [                                     │
│   {name: "no_secrets", message: "..."}             │
│ ]                                                   │
│                                                      │
│ score_breakdown: {                                  │
│   "secrets": 15.0,                                  │
│   "vulnerabilities": 0.0                           │
│ }                                                   │
│                                                      │
│ recommendations: [                                  │
│   "Remove secrets and rotate credentials"          │
│ ]                                                   │
└─────────────────────────────────────────────────────┘
```
