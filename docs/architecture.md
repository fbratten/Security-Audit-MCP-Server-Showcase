# Architecture

> System design and component overview for Security Audit MCP Server

---

## System Layers

The Security Audit MCP Server follows a layered architecture:

```
┌─────────────────────────────────────────────────────────────────────┐
│                          MCP Protocol Layer                         │
│              Handles tool registration and communication            │
└─────────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────┐
│                           Tool Layer                                │
│                   Individual MCP tool implementations               │
│   detect_stack │ scan_secrets │ scan_deps │ scan_code │ full_scan  │
└─────────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         Scanner Layer                               │
│                    Wrapper classes for each scanner                 │
│        Gitleaks │ OSV-Scanner │ Semgrep │ Bandit │ Syft            │
└─────────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        Execution Layer                              │
│                  Docker-first, local-fallback strategy              │
└─────────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         Policy Layer                                │
│                   Gates, scoring, and verdicts                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Core Components

### 1. MCP Server

The central component that:
- Registers available tools
- Handles tool invocations
- Routes requests to appropriate handlers
- Returns structured responses

### 2. Stack Detector

Analyzes project structure to identify:
- Programming languages (Python, JavaScript)
- Package managers (pip, npm, yarn, poetry)
- Ecosystems (PyPI, npm)
- Applicable scanners

### 3. Scanner Wrappers

Each scanner has a dedicated wrapper that:
- Manages Docker container lifecycle
- Falls back to local binary if Docker unavailable
- Parses scanner-specific output
- Normalizes to unified Finding schema

### 4. Policy Engine

Evaluates findings against configurable rules:
- **Gates**: Hard blockers (NO-GO triggers)
- **Weights**: Category importance for scoring
- **Thresholds**: Score boundaries for verdicts

---

## Data Flow

```
Project Path
     │
     ▼
┌─────────────┐
│   Stack     │─────────────────────────────────┐
│  Detection  │                                 │
└─────────────┘                                 │
     │                                          │
     ▼                                          │
┌─────────────┐     ┌─────────────┐            │
│   Secret    │     │ Dependency  │            │
│   Scanner   │     │   Scanner   │            │
└─────────────┘     └─────────────┘            │
     │                    │                     │
     ▼                    ▼                     │
┌─────────────┐     ┌─────────────┐            │
│    Code     │     │    SBOM     │            │
│   Scanner   │     │  Generator  │            │
└─────────────┘     └─────────────┘            │
     │                    │                     │
     └────────┬───────────┘                     │
              │                                 │
              ▼                                 │
     ┌─────────────────┐                       │
     │    Findings     │◄──────────────────────┘
     │   Aggregation   │     (Stack Info)
     └─────────────────┘
              │
              ▼
     ┌─────────────────┐
     │     Policy      │
     │   Evaluation    │
     └─────────────────┘
              │
              ▼
     ┌─────────────────┐
     │     Verdict     │
     │   + Report      │
     └─────────────────┘
```

---

## Execution Strategy

### Docker-First Approach

```
Scanner Request
       │
       ▼
┌──────────────────┐
│ Docker Available?│
└────────┬─────────┘
         │
    ┌────┴────┐
    ▼         ▼
  [Yes]      [No]
    │         │
    ▼         ▼
┌────────┐ ┌────────┐
│ Docker │ │ Local  │
│  Run   │ │ Binary │
└────────┘ └────────┘
    │         │
    └────┬────┘
         │
         ▼
  ┌─────────────┐
  │ Parse Output│
  │ to Findings │
  └─────────────┘
```

### Container Configuration

| Setting | Value | Purpose |
|---------|-------|---------|
| Mount mode | Read-only | Security isolation |
| Network | Disabled | Prevent data exfiltration |
| Memory | 1GB limit | Resource control |
| Timeout | 5-10 min | Prevent hangs |

---

## Offline Capability

### Cache Structure

```
~/.cache/security-audit-mcp/
├── osv/
│   ├── pypi/           # PyPI vulnerabilities
│   └── npm/            # npm vulnerabilities
└── kev/
    └── known_exploited_vulnerabilities.json
```

### Update Flow

```
Internet ──────► Cache Update Tool ──────► Local Database
                      │
                      ▼
              ┌──────────────┐
              │ Verify Hashes│
              │ Extract Data │
              │ Index CVEs   │
              └──────────────┘
```

---

## Error Handling

### Fail-Loud Philosophy

Every error produces clear, actionable messages:

```
┌─────────────────────────────────────────────────┐
│               Error Hierarchy                    │
├─────────────────────────────────────────────────┤
│ SecurityAuditError (base)                       │
│   ├── ScannerError                              │
│   │     ├── ScannerTimeoutError                 │
│   │     └── ScannerUnavailableError             │
│   ├── DockerError                               │
│   │     └── DockerNotAvailableError             │
│   ├── CacheError                                │
│   │     └── CacheNotFoundError                  │
│   ├── PolicyError                               │
│   └── ConfigurationError                        │
└─────────────────────────────────────────────────┘
```

---

## Extensibility

### Adding New Scanners

```
1. Create scanner wrapper (extends BaseScanner)
2. Implement: scan(), is_available(), parse_output()
3. Register in scanner runner
4. Add MCP tool definition
5. Update policy weights if needed
```

### Custom Policies

```yaml
# Custom policy configuration
gates:
  - name: custom_gate
    condition_type: severity_present
    condition_params:
      severity: high
      count: 5
    action: block
    message: "Too many high severity issues"

score_weights:
  - category: custom_category
    weight: 0.20
```
