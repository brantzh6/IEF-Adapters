---
name: log-audit
description: Scan SDLC structured logs to detect anomaly patterns (repeated rejections, provider failures, coverage drops, design-implementation drift). Outputs audit report with actionable findings. Use when asked to audit logs, check project health, or perform periodic review.
version: 1.0.0
author: brantzh6
tags:
  - monitoring
  - audit
  - feedback-loop
---

# SDLC Log Audit — Hermes

Scan `logs/sdlc/` directory for structured SDLC logs, detect patterns, and produce an audit report.

## When to Use

- User requests: "audit logs", "check project health", "log audit"
- Periodic trigger (suggested: every 10 completed tasks)
- After observing consecutive failures

## Execution Flow

### 1. Collect Logs

```bash
# Scan log directory
find logs/sdlc/ -name "*.yaml" -type f | sort -t_ -k3 -r | head -50
```

If directory doesn't exist or is empty: inform user that SDLC logs have not been generated yet.

### 2. Detect Patterns

| Pattern | Condition | Severity |
|---------|-----------|----------|
| Repeated reject | Same module G4 reject >= 3 times | HIGH |
| Provider consecutive failure | Same provider error >= 3 times | HIGH |
| Design-implementation drift | G4 review notes design inconsistency | HIGH |
| Low first-pass rate | G5 first_attempt_pass < 60% | MEDIUM |
| Frequent timeouts | Same task type timeout >= 3 times | MEDIUM |
| Abnormal CW usage | > 40% or < 5% (T2+ projects) | LOW |

### 3. Output Audit Report

```markdown
# SDLC Log Audit Report

- Audit time: <timestamp>
- Scan range: Last N tasks / Last N days
- Gate Profile: <project profile>

## Issues Found

### [HIGH] <issue title>
- Evidence: <log reference>
- Occurrences: N
- Suggested action: <refactor / fix / update_config>

## Health Metrics

| Metric | Current | Threshold | Status |
|--------|---------|-----------|--------|
| Gate pass rate | X% | >= 80% | OK/WARN |
| First-pass rate | X% | >= 70% | OK/WARN |
| CW usage rate | X% | 10-30% | OK/WARN |

## Recommendations

1. <specific recommendation>
```

### 4. Feedback Task Creation (G-Full only)

For G-Full projects, automatically create feedback tasks for HIGH severity issues:

```yaml
source: log_audit
trigger: <pattern name>
evidence: <log file path>
suggested_classification:
  risk: R2
  class: B
  template: T2
action: <refactor / fix / investigate>
```

### 5. G-Std Projects

Output report, then ask user whether to act on findings. Only create tasks after user confirmation.

## Rules

- Read-only: never modify log files
- Never self-decide to execute fixes (report → user decision → execute)
- G-Lite projects: do not proactively trigger audit
