# Use Cases

> Real-world application scenarios for Security Audit MCP Server

---

## Pre-Release Security Gate

### Scenario

Before shipping a release, ensure the codebase meets security standards.

### Flow

```
Feature Complete
       │
       ▼
┌──────────────┐
│  Full Audit  │
└──────┬───────┘
       │
       ▼
   ┌───────┐
   │Verdict│
   └───┬───┘
       │
   ┌───┴───┐
   ▼       ▼
  GO     NO-GO
   │       │
   ▼       ▼
Release  Fix Issues
         & Re-scan
```

### Benefits

- **Automated gate**: No manual security review bottleneck
- **Consistent criteria**: Same checks every release
- **Clear action items**: Specific findings to address
- **Audit trail**: Documented security posture

---

## Dependency Adoption Review

### Scenario

Evaluating a third-party library before adding it to your project.

### Flow

```
New Library Candidate
       │
       ▼
┌──────────────────┐
│ Clone Library    │
│ Repository       │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ audit.full_scan  │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ Review Findings: │
│ • Known vulns?   │
│ • Secrets?       │
│ • Code quality?  │
│ • Dependencies?  │
└────────┬─────────┘
         │
    ┌────┴────┐
    ▼         ▼
  Adopt    Reject/
           Find Alternative
```

### Questions Answered

- Does this library have known vulnerabilities?
- Are there hardcoded secrets in the source?
- What's the transitive dependency risk?
- Is the code quality acceptable?

---

## Continuous Security Monitoring

### Scenario

Regular security checks during development sprints.

### Flow

```
┌─────────────────────────────────────────────────────┐
│                   Sprint Cycle                       │
│                                                      │
│  Day 1    Day 3    Day 5    Day 7    Day 10        │
│    │        │        │        │         │           │
│    ▼        ▼        ▼        ▼         ▼           │
│  Start   Quick    Quick   Full      Release         │
│          Scan     Scan    Audit     Review          │
│            │        │        │                      │
│            └────────┴────────┘                      │
│                    │                                │
│                    ▼                                │
│            Track Trends                             │
│            Address Issues                           │
└─────────────────────────────────────────────────────┘
```

### Benefits

- **Early detection**: Find issues before they accumulate
- **Trend tracking**: Monitor security posture over time
- **Developer awareness**: Regular security feedback

---

## CI/CD Integration

### Scenario

Automated security checks in your deployment pipeline.

### Flow

```
┌──────────────────────────────────────────────────────┐
│                   CI/CD Pipeline                      │
│                                                       │
│  ┌─────────┐   ┌─────────┐   ┌─────────┐   ┌──────┐ │
│  │  Build  │──▶│  Test   │──▶│ Security│──▶│Deploy│ │
│  └─────────┘   └─────────┘   │  Audit  │   └──────┘ │
│                              └────┬────┘            │
│                                   │                  │
│                              ┌────┴────┐            │
│                              ▼         ▼            │
│                            PASS      FAIL           │
│                              │         │            │
│                              ▼         ▼            │
│                          Continue   Block &         │
│                                     Alert           │
└──────────────────────────────────────────────────────┘
```

### Configuration

| Environment | Policy | Action on Fail |
|-------------|--------|----------------|
| Development | Lenient | Warn only |
| Staging | Standard | Block deploy |
| Production | Strict | Block + alert |

---

## Open Source Project Audit

### Scenario

Auditing an open source project before forking or contributing.

### Flow

```
Open Source Repository
         │
         ▼
┌──────────────────┐
│  Clone Locally   │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ audit.detect_stack│
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ audit.full_scan  │
└────────┬─────────┘
         │
         ▼
┌──────────────────────────────────────┐
│           Assessment Report           │
│                                       │
│ • Known vulnerabilities               │
│ • Dependency health                   │
│ • Code security issues                │
│ • Maintenance indicators              │
│ • Overall viability verdict           │
└──────────────────────────────────────┘
```

### Assessment Factors

| Factor | Indicator |
|--------|-----------|
| Vulnerability count | Direct security risk |
| KEV presence | Active exploitation risk |
| Dependency freshness | Maintenance activity |
| Secret findings | Code hygiene |

---

## Compliance Reporting

### Scenario

Generating security documentation for compliance requirements.

### Flow

```
Compliance Requirement
(SOC 2, ISO 27001, etc.)
         │
         ▼
┌──────────────────┐
│ audit.full_scan  │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ audit.generate_  │
│      sbom        │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ report.generate  │
│ (format: markdown)│
└────────┬─────────┘
         │
         ▼
┌──────────────────────────────────────┐
│           Compliance Package          │
│                                       │
│ • Full audit report                   │
│ • SBOM (CycloneDX)                   │
│ • Vulnerability inventory             │
│ • Policy evaluation                   │
│ • Remediation status                  │
└──────────────────────────────────────┘
```

### Deliverables

| Document | Format | Purpose |
|----------|--------|---------|
| Audit Report | Markdown/JSON | Security assessment |
| SBOM | CycloneDX JSON | Component inventory |
| Vulnerability List | JSON | Risk inventory |
| Policy Result | JSON | Compliance status |

---

## Incident Response

### Scenario

A new vulnerability is disclosed. Check if your projects are affected.

### Flow

```
New CVE Announced
       │
       ▼
┌──────────────────┐
│ Update vuln DBs  │
│ (cache.update)   │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ Scan all projects│
│ in portfolio     │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ Filter by CVE    │
│ in findings      │
└────────┬─────────┘
         │
         ▼
┌──────────────────────────────────────┐
│           Affected Projects           │
│                                       │
│ project-a: vulnerable (v1.2.3)       │
│ project-b: patched (v1.2.5)          │
│ project-c: vulnerable (v1.2.3)       │
└──────────────────────────────────────┘
         │
         ▼
   Prioritize Patching
```

### Response Actions

1. **Identify scope**: Which projects are affected?
2. **Assess severity**: Is it in KEV? Critical?
3. **Plan remediation**: What version fixes it?
4. **Verify fix**: Re-scan after patching

---

## Multi-Project Portfolio Audit

### Scenario

Auditing multiple projects to assess overall security posture.

### Flow

```
┌─────────────────────────────────────────────────────┐
│               Project Portfolio                      │
│                                                      │
│  project-a/  project-b/  project-c/  project-d/    │
│      │           │           │           │          │
│      └───────────┴─────┬─────┴───────────┘          │
│                        │                             │
│                        ▼                             │
│              ┌──────────────────┐                   │
│              │  Batch Audit     │                   │
│              └────────┬─────────┘                   │
│                       │                             │
│                       ▼                             │
│    ┌─────────────────────────────────────┐         │
│    │         Portfolio Report             │         │
│    │                                      │         │
│    │  Project    Score   Verdict  Issues │         │
│    │  ────────   ─────   ───────  ────── │         │
│    │  project-a    92      GO        2   │         │
│    │  project-b    78     COND.      8   │         │
│    │  project-c    45     NO-GO     23   │         │
│    │  project-d    88      GO        4   │         │
│    └─────────────────────────────────────┘         │
└─────────────────────────────────────────────────────┘
```

### Portfolio Metrics

- **Average score**: Overall security health
- **NO-GO count**: Projects needing attention
- **Common issues**: Patterns across projects
- **Trend over time**: Improving or degrading?
