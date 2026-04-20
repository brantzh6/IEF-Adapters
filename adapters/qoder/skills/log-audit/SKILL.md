---
name: log-audit
description: 扫描项目 SDLC 日志，检测异常模式（反复 reject、provider 失败、覆盖率下降、设计偏差），输出审计报告。按 Gate Profile 决定是否自动创建反馈任务。当用户要求审计日志、检查项目健康度、或定期 review 时使用。
---

# SDLC 日志审计 — Qoder

扫描 `.qoder/logs/sdlc/` 目录中的结构化日志，发现模式并输出审计报告。

## 触发条件

- 用户显式要求："审计日志"、"检查项目健康度"、"log audit"
- 定期触发（建议每完成 10 个任务后）
- 发现连续失败时主动建议

## 执行流程

### 1. 收集日志

```powershell
# 扫描日志目录
Get-ChildItem ".qoder/logs/sdlc/" -Filter "*.yaml" -Recurse | Sort-Object LastWriteTime -Descending
```

如日志目录不存在或为空，提示用户：SDLC 日志尚未产生，需先在项目中执行几个 SDLC 管控的任务。

### 2. 检测模式

按优先级检测以下模式：

| 模式 | 条件 | 严重度 |
|------|------|--------|
| 反复 reject | 同一模块 G4 reject >= 3 次 | HIGH |
| Provider 连续失败 | 同一 provider error >= 3 次 | HIGH |
| 设计-实现偏差 | G4 review 指出与设计不一致 | HIGH |
| 测试首次通过率低 | G5 first_attempt_pass < 60% | MEDIUM |
| 超时频发 | 同类任务 timeout >= 3 次 | MEDIUM |
| Claude Worker 使用率异常 | > 40% 或 < 5%（T2+项目） | LOW |

### 3. 输出审计报告

```markdown
# SDLC 日志审计报告

- 审计时间: <timestamp>
- 扫描范围: 最近 N 个任务 / 最近 N 天
- Gate Profile: <项目当前 profile>

## 发现的问题

### [HIGH] <问题标题>
- 证据: <日志引用>
- 出现次数: N
- 建议动作: <refactor / fix / update_config>

### [MEDIUM] <问题标题>
- ...

## 健康指标

| 指标 | 当前值 | 健康阈值 | 状态 |
|------|--------|----------|------|
| Gate 通过率 | X% | >= 80% | OK/WARN |
| 首次通过率 | X% | >= 70% | OK/WARN |
| Claude Worker 使用率 | X% | 10-30% | OK/WARN |

## 建议

1. <具体建议>
2. <具体建议>
```

### 4. 反馈任务创建（仅 G-Full）

如果项目 Gate Profile 为 G-Full，对 HIGH 严重度问题自动创建反馈任务：

```yaml
source: log_audit
trigger: <模式名称>
evidence: <日志文件路径>
suggested_classification:
  risk: R2
  class: B
  template: T2
action: <refactor / fix / investigate>
```

创建后按正常 SDLC 流程走（从 Design 开始）。

### 5. G-Std 项目

输出报告后，询问用户是否要处理发现的问题。用户确认后再创建任务。

## 注意事项

- 不修改日志文件（只读操作）
- 不自行决定是否执行修复（报告 → 用户决策 → 执行）
- G-Lite 项目不主动触发审计
