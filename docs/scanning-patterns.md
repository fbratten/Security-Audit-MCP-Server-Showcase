# Scanning Patterns

> How security scanners work together in the Security Audit MCP Server

---

## Scanner Overview

| Scanner | Type | Target | Output |
|---------|------|--------|--------|
| **Gitleaks** | Secret Detection | All files + git history | Hardcoded credentials |
| **OSV-Scanner** | SCA | Dependency manifests | CVE/OSV vulnerabilities |
| **Semgrep** | SAST | Source code | Security patterns |
| **Bandit** | SAST | Python code | Python-specific issues |
| **Syft** | SBOM | Entire project | Component inventory |

---

## Secret Detection (Gitleaks)

### What It Finds

```
┌────────────────────────────────────────────┐
│            Secret Categories               │
├────────────────────────────────────────────┤
│ • API Keys (AWS, GCP, Azure, etc.)        │
│ • Database credentials                     │
│ • Private keys (SSH, PGP)                 │
│ • OAuth tokens                            │
│ • JWT secrets                             │
│ • Password strings                        │
│ • Connection strings                      │
└────────────────────────────────────────────┘
```

### Detection Flow

```
Project Files
     │
     ├──► Regex Patterns ──► Potential Secrets
     │
     ├──► Entropy Analysis ──► High-entropy strings
     │
     └──► Git History ──► Historical secrets
              │
              ▼
       ┌─────────────┐
       │ Fingerprint │
       │   + Rule ID │
       └─────────────┘
```

### Configuration Options

| Option | Default | Description |
|--------|---------|-------------|
| `include_git_history` | false | Scan commit history |
| `timeout` | 300s | Maximum scan duration |

---

## Dependency Scanning (OSV-Scanner)

### Supported Manifests

```
Python                    Node.js
───────                   ───────
pyproject.toml           package.json
requirements.txt         package-lock.json
Pipfile.lock             yarn.lock
poetry.lock              pnpm-lock.yaml
uv.lock
```

### Vulnerability Matching

```
Manifest Parsing
       │
       ▼
┌──────────────────┐
│ Package + Version│
└────────┬─────────┘
         │
         ▼
┌──────────────────┐     ┌──────────────────┐
│   OSV Database   │────►│  Match CVEs      │
└──────────────────┘     └──────────────────┘
         │
         ▼
┌──────────────────┐     ┌──────────────────┐
│   CISA KEV       │────►│ Flag Exploited   │
└──────────────────┘     └──────────────────┘
         │
         ▼
┌──────────────────┐
│   Finding with   │
│ CVE + KEV status │
└──────────────────┘
```

### KEV Integration

Findings are cross-referenced with CISA's Known Exploited Vulnerabilities:

| KEV Status | Meaning | Policy Impact |
|------------|---------|---------------|
| `in_kev: true` | Actively exploited | Default: NO-GO |
| `in_kev: false` | Not in KEV | Normal scoring |

---

## Static Analysis (Semgrep + Bandit)

### Semgrep Coverage

```
┌─────────────────────────────────────────────┐
│              Rule Categories                │
├─────────────────────────────────────────────┤
│ • Security audit patterns                   │
│ • Secret detection (backup)                 │
│ • Python security rules                     │
│ • JavaScript security rules                 │
│ • SQL injection patterns                    │
│ • XSS vulnerabilities                       │
│ • Command injection                         │
│ • Path traversal                            │
└─────────────────────────────────────────────┘
```

### Bandit (Python-Specific)

Focused analysis for Python code:

```
Python Source
     │
     ├──► Hardcoded passwords
     │
     ├──► SQL injection (string formatting)
     │
     ├──► Shell injection (subprocess)
     │
     ├──► Insecure functions (eval, exec)
     │
     ├──► Weak cryptography
     │
     └──► Assert statements in production
```

### CWE Mapping

All code findings include CWE identifiers:

| CWE | Description | Example |
|-----|-------------|---------|
| CWE-78 | OS Command Injection | `os.system(user_input)` |
| CWE-89 | SQL Injection | `f"SELECT * WHERE id={id}"` |
| CWE-79 | XSS | Unescaped HTML output |
| CWE-798 | Hardcoded Credentials | `password = "secret123"` |

---

## SBOM Generation (Syft)

### Output Format

CycloneDX JSON format with:

```json
{
  "bomFormat": "CycloneDX",
  "specVersion": "1.4",
  "components": [
    {
      "type": "library",
      "name": "package-name",
      "version": "1.2.3",
      "purl": "pkg:pypi/package-name@1.2.3"
    }
  ]
}
```

### Use Cases

```
SBOM Output
     │
     ├──► Compliance reporting
     │
     ├──► License analysis
     │
     ├──► Vulnerability enrichment
     │
     └──► Supply chain visibility
```

---

## Parallel Execution

### Full Scan Orchestration

```
                    audit.full_scan
                          │
                          ▼
                  ┌───────────────┐
                  │ detect_stack  │
                  └───────┬───────┘
                          │
          ┌───────────────┼───────────────┐
          ▼               ▼               ▼
    ┌───────────┐   ┌───────────┐   ┌───────────┐
    │  secrets  │   │   deps    │   │   code    │
    │  scanner  │   │  scanner  │   │  scanner  │
    └─────┬─────┘   └─────┬─────┘   └─────┬─────┘
          │               │               │
          └───────────────┼───────────────┘
                          │
                          ▼
                  ┌───────────────┐
                  │   aggregate   │
                  │   findings    │
                  └───────┬───────┘
                          │
                          ▼
                  ┌───────────────┐
                  │    policy     │
                  │  evaluation   │
                  └───────────────┘
```

### Timeout Management

Each scanner has independent timeout:

| Scanner | Default Timeout |
|---------|-----------------|
| Gitleaks | 5 minutes |
| OSV-Scanner | 10 minutes |
| Semgrep | 10 minutes |
| Bandit | 5 minutes |
| Syft | 5 minutes |

---

## Finding Normalization

All scanner outputs are normalized to a unified schema:

```
┌─────────────────────────────────────────────────────┐
│                 Unified Finding                      │
├─────────────────────────────────────────────────────┤
│ Identity                                            │
│   • id: Unique identifier                           │
│   • fingerprint: Deduplication key                  │
├─────────────────────────────────────────────────────┤
│ Classification                                      │
│   • type: secret | vulnerability | code_issue       │
│   • severity: critical | high | medium | low | info │
│   • confidence: 0.0 - 1.0                          │
├─────────────────────────────────────────────────────┤
│ Identifiers                                         │
│   • cve_id, cwe_id, osv_id, ghsa_id               │
│   • in_kev: CISA KEV status                        │
├─────────────────────────────────────────────────────┤
│ Location                                            │
│   • file_path, line_start, line_end                │
│   • package_name, package_version (for deps)       │
├─────────────────────────────────────────────────────┤
│ Content                                             │
│   • title, description, recommendation             │
│   • scanner, scanner_rule_id                       │
└─────────────────────────────────────────────────────┘
```
