# Security Audit MCP Server

[![License](https://img.shields.io/badge/License-Proprietary-red.svg)](#license)
[![Status](https://img.shields.io/badge/Status-Active-brightgreen.svg)](#)
[![Version](https://img.shields.io/badge/Version-1.0.0-blue.svg)](#version-history)
[![MCP](https://img.shields.io/badge/MCP-Compatible-purple.svg)](#)
[![Demos](https://img.shields.io/badge/Interactive_Demos-4-orange.svg)](demos/)

> **Comprehensive security auditing through the Model Context Protocol**

An MCP server that answers the critical question: *"Is this software project viable to ship or adopt, given its code quality, dependency health, and security posture?"*

**This is a showcase repository.** Source code is maintained privately.

---

## Key Capabilities

- **Secret Detection** — Scan codebases for hardcoded credentials, API keys, and tokens
- **Dependency Vulnerability Analysis** — Identify CVEs with CISA KEV catalog integration
- **Static Code Analysis** — Multi-language SAST with CWE mapping
- **SBOM Generation** — CycloneDX-compliant Software Bill of Materials
- **Supply Chain Assessment** — Evaluate dependency health and project hygiene
- **Policy Engine** — Configurable gates with GO/NO-GO/CONDITIONAL verdicts

[**Try the Interactive Demos →**](demos/)

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                        MCP Client (Claude)                          │
└─────────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    Security Audit MCP Server                        │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │                      Tool Layer                                │  │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────────┐  │  │
│  │  │ detect   │ │  scan    │ │  scan    │ │   evaluate       │  │  │
│  │  │ _stack   │ │ _secrets │ │  _deps   │ │   _policy        │  │  │
│  │  └──────────┘ └──────────┘ └──────────┘ └──────────────────┘  │  │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────────┐  │  │
│  │  │  scan    │ │ generate │ │  check   │ │    full          │  │  │
│  │  │  _code   │ │  _sbom   │ │ _supply  │ │    _scan         │  │  │
│  │  └──────────┘ └──────────┘ └──────────┘ └──────────────────┘  │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                   │                                  │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │                    Scanner Layer                               │  │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────────┐  │  │
│  │  │ Gitleaks │ │   OSV    │ │ Semgrep  │ │     Bandit       │  │  │
│  │  │ (secrets)│ │ Scanner  │ │  (SAST)  │ │  (Python SAST)   │  │  │
│  │  └──────────┘ └──────────┘ └──────────┘ └──────────────────┘  │  │
│  │                    ┌──────────────────┐                        │  │
│  │                    │      Syft        │                        │  │
│  │                    │     (SBOM)       │                        │  │
│  │                    └──────────────────┘                        │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                   │                                  │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │                    Policy Engine                               │  │
│  │  ┌─────────────────────────────────────────────────────────┐  │  │
│  │  │  Gates (Hard Blockers)     │    Scoring (Weighted)      │  │  │
│  │  │  • no_secrets              │    • secrets: 25%          │  │  │
│  │  │  • no_kev_vulnerabilities  │    • vulnerabilities: 35%  │  │  │
│  │  │  • no_critical_cves        │    • code_issues: 25%      │  │  │
│  │  │                            │    • supply_chain: 15%     │  │  │
│  │  └─────────────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                   │                                  │
│                                   ▼                                  │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │                    Verdict Output                              │  │
│  │           GO  │  CONDITIONAL  │  NO-GO                         │  │
│  │         (≥80) │   (50-79)     │  (<50 or gate fail)           │  │
│  └───────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     Docker Container Layer                          │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐  │
│  │ gitleaks │ │   osv-   │ │ semgrep  │ │  bandit  │ │   syft   │  │
│  │          │ │ scanner  │ │          │ │          │ │          │  │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

---

## How It Works

### 1. Stack Detection

The server automatically detects your project's technology stack:

```
Project Analysis
       │
       ▼
┌──────────────────┐     ┌──────────────────┐
│  Manifest Scan   │────▶│  Stack Profile   │
│  • pyproject.toml│     │  • Languages     │
│  • package.json  │     │  • Pkg Managers  │
│  • requirements  │     │  • Ecosystems    │
└──────────────────┘     └──────────────────┘
       │
       ▼
┌──────────────────┐
│ Scanner Selection│
│ • Applicable     │
│ • Prioritized    │
└──────────────────┘
```

### 2. Security Scanning

Multiple scanners run in isolated Docker containers:

| Scanner | Purpose | Output |
|---------|---------|--------|
| **Gitleaks** | Secret detection | Hardcoded credentials, API keys |
| **OSV-Scanner** | Dependency vulnerabilities | CVEs, OSV IDs, GHSA |
| **Semgrep** | Static analysis | Code security issues, CWE mapping |
| **Bandit** | Python SAST | Python-specific vulnerabilities |
| **Syft** | SBOM generation | CycloneDX components |

### 3. Policy Evaluation

Findings are evaluated against configurable security policies:

```
                    Findings
                        │
        ┌───────────────┼───────────────┐
        ▼               ▼               ▼
   ┌─────────┐    ┌─────────┐    ┌─────────┐
   │  Gate   │    │  Gate   │    │  Gate   │
   │ Check 1 │    │ Check 2 │    │ Check 3 │
   └────┬────┘    └────┬────┘    └────┬────┘
        │              │              │
        ▼              ▼              ▼
   ┌─────────────────────────────────────┐
   │         Gate Results                 │
   │   PASS / FAIL for each gate         │
   └─────────────────────────────────────┘
                    │
                    ▼
   ┌─────────────────────────────────────┐
   │         Score Calculation            │
   │   100 - (weighted deductions)       │
   └─────────────────────────────────────┘
                    │
                    ▼
   ┌─────────────────────────────────────┐
   │              Verdict                 │
   │   GO │ CONDITIONAL │ NO-GO          │
   └─────────────────────────────────────┘
```

---

## Interactive Demos

Experience the Security Audit MCP Server capabilities:

| Demo | Description |
|------|-------------|
| [**Stack Detector**](demos/stack-detector/) | See how project analysis works |
| [**Secret Scanner**](demos/scan-secrets/) | Explore secret detection patterns |
| [**Policy Evaluator**](demos/policy-evaluator/) | Configure gates and see verdicts |
| [**Full Audit**](demos/full-audit/) | Complete security audit simulation |

[**View All Demos →**](demos/)

---

## Supported Technologies

### Languages & Ecosystems

| Language | Package Managers | Ecosystem |
|----------|------------------|-----------|
| Python | pip, poetry, pipenv, uv | PyPI |
| JavaScript/Node.js | npm, yarn, pnpm | npm |

### Vulnerability Databases

- **OSV** — Open Source Vulnerabilities database
- **CISA KEV** — Known Exploited Vulnerabilities catalog
- **NVD** — National Vulnerability Database (via CVE)
- **GHSA** — GitHub Security Advisories

---

## Key Features

### Offline-First Design

Vulnerability databases are cached locally, enabling:
- Air-gapped environment operation
- Consistent, reproducible scans
- No network dependency during audits

### Containerized Execution

All scanners run in isolated Docker containers:
- Security isolation from host system
- Consistent environments across platforms
- Resource limits and timeout enforcement
- Read-only project mounts

### Unified Finding Schema

All scanner outputs normalized to a single format:

```
┌─────────────────────────────────────────┐
│              Finding                     │
├─────────────────────────────────────────┤
│ • ID & Fingerprint (deduplication)      │
│ • Type (secret/vuln/code_issue/supply)  │
│ • Severity (critical/high/medium/low)   │
│ • Location (file, line, column)         │
│ • Identifiers (CVE, CWE, OSV, GHSA)     │
│ • KEV status (actively exploited?)      │
│ • Recommendation                         │
└─────────────────────────────────────────┘
```

### Configurable Policy Gates

Default gates that trigger NO-GO:

| Gate | Condition |
|------|-----------|
| `no_secrets` | Any hardcoded secret detected |
| `no_kev_vulnerabilities` | Any CISA KEV entry found |
| `no_critical_vulnerabilities` | Any critical severity CVE |

---

## Documentation

| Document | Description |
|----------|-------------|
| [Architecture](docs/architecture.md) | System design and components |
| [Scanning Patterns](docs/scanning-patterns.md) | How scanners work together |
| [Policy Engine](docs/policy-engine.md) | Gates, scoring, and verdicts |
| [Use Cases](docs/use-cases.md) | Real-world application scenarios |

---

## Use Cases

### Pre-Release Security Gate

Run comprehensive security audit before shipping:

```
Code Complete → Security Audit → Verdict → Release Decision
                     │
              ┌──────┴──────┐
              │   NO-GO?    │
              │ Fix issues  │
              └─────────────┘
```

### Dependency Adoption Review

Evaluate third-party libraries before integration:

```
New Dependency → Audit Package → Review Findings → Adopt/Reject
```

### Continuous Security Monitoring

Periodic audits during development:

```
Sprint Start → ... → Security Check → ... → Sprint End
                           │
                    Address findings
```

---

## Version History

### v1.0.0 (Current)

- Complete MCP server implementation
- Five integrated security scanners
- Configurable policy engine with gates and scoring
- GO/NO-GO/CONDITIONAL verdict system
- Offline vulnerability database support
- CycloneDX SBOM generation
- Supply chain health assessment

---

## About

This showcase demonstrates the capabilities of the Security Audit MCP Server, a comprehensive security auditing solution for software projects.

**Source code is maintained privately.**

### Contact

- **GitHub**: [fbratten](https://github.com/fbratten)
- **Email**: jack.rumble556@passfwd.com

---

## License

This showcase content is provided for demonstration purposes.

Source code is proprietary. All rights reserved.

© 2025 fbratten
