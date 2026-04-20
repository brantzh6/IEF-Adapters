# 监控反馈闭环机制

> 定义 Agent 工作过程中的日志规范、审计流程和 finding 输出机制。
> 按 Gate Profile 分级（G-Lite / G-Std / G-Full），项目根据治理等级选择适用层级。
> Harness 职责边界：日志 → 发现问题 → 输出标准 finding_report。任务创建和调度由外部任务体系负责。

---

## 1. 结构化日志规范

### 1.1 Gate 通过日志（所有层级必需）

Agent 在每个 Gate 通过时必须产出结构化日志：

```yaml
# Gate Pass Log
timestamp: <ISO 8601>
task_id: <task identifier>
gate: G1 / G2 / G3 / G4 / G5 / G6 / G7
gate_profile: G-Lite / G-Std / G-Full
verdict: pass / conditional_pass / reject
model_used: <provider>/<model>  # 如使用 Claude Worker
duration_seconds: <number>
evidence_summary: <一句话总结通过证据>
conditions: <如 conditional_pass，列出待满足条件>
```

### 1.2 失败日志

```yaml
# Gate Failure Log
timestamp: <ISO 8601>
task_id: <task identifier>
gate: <gate>
failure_type: review_reject / test_fail / provider_error / timeout / scope_expansion
failure_reason: <详细原因>
model_used: <provider>/<model>
retry_count: <number>
action_taken: retry / escalate / abort / switch_provider
```

### 1.3 路由决策日志

```yaml
# Routing Decision Log
timestamp: <ISO 8601>
task_id: <task identifier>
task_type: architecture / coding / review / test / incident
risk_level: R1 / R2 / R3
change_class: A / B / C / D
executor: host_agent / claude_worker
model_selected: <provider>/<model>
selection_reason: capability_match / user_request / fallback / independence_requirement
```

### 1.4 日志存储位置

| Agent | 日志目录 | 格式 |
|-------|---------|------|
| Qoder | `.qoder/logs/sdlc/` | YAML per task |
| Hermes | `logs/sdlc/` 或 `~/.hermes/logs/` | YAML per task |
| OpenClaw | `logs/sdlc/` 或 `~/.openclaw/logs/` | YAML per task |

文件命名：`{task_id}_{gate}_{timestamp}.yaml`

---

## 2. 分级监控策略

### 2.1 G-Lite (T1) — 基本日志

**要求**：
- 每个 Gate 通过/失败时记录结构化日志
- 任务完成时记录路由决策
- 无主动扫描，无自动分析

**适用场景**：
- L1 项目
- R1 低风险任务
- 一次性脚本和 demo

**产出**：
- 日志文件存储供事后查阅

---

### 2.2 G-Std (T2) — 日志 + 偏差检测

**要求**：
- G-Lite 全部 +
- 手动或定期触发日志审计
- 检测以下模式并输出报告：

**检测模式**：

| 模式 | 检测条件 | 建议动作 |
|------|----------|----------|
| 反复 reject | 同一文件/模块在 G4 被 reject >= 3 次 | 建议 refactor 该模块 |
| Provider 连续失败 | 同一 provider 连续 >= 3 次 error | 建议切换默认 provider |
| Review 通过率异常 | 某阶段通过率 < 60% | 分析根因，可能是设计质量问题 |
| 测试覆盖率下降 | 新增代码无对应测试 | 建议补充测试 |
| 超时频发 | 同一任务类型超时 >= 3 次 | 建议调整 max-turns 或拆分任务 |

**产出**：
- 结构化审计报告（markdown）
- 问题清单（含严重度标记）

---

### 2.3 G-Full (T3) — 完整闭环含偏差检测

**要求**：
- G-Std 全部 +
- 设计-实现偏差检测
- 输出标准 finding_report（交由外部任务体系处理，见 `integration-interfaces.md` §4）
- 趋势追踪

**设计-实现偏差检测**：

```
1. 读取任务的设计 brief（G1 产物）
2. 读取实际实现代码
3. 对比关键决策点：
   - 设计中声明的接口是否与实现一致？
   - 设计中的约束是否在代码中体现？
   - 非目标（non-goals）是否被意外引入？
4. 如发现偏差 → 创建偏差分析报告
```

**发现输出**：

当检测到问题时，输出标准 `finding_report` 格式（详见 `integration-interfaces.md` §4）：

```yaml
# Finding Report Output
schema_version: "1.0"
type: finding_report
generated_by: monitoring_loop
finding:
  title: "<问题标题>"
  severity: HIGH / MEDIUM / LOW
  pattern: "<匹配到的检测模式>"
  evidence:
    - log_ref: "<日志文件路径>"
suggested_action:
  type: refactor / fix / investigate / update_config
  suggested_classification:
    risk: R2
    class: B
    template: T2
```

> 注意：Harness 只输出发现，不负责任务创建和调度。
> 当外部任务体系未接入时，直接呈现给用户决策。

**趋势追踪指标**：

| 指标 | 计算方式 | 健康阈值 |
|------|----------|----------|
| Gate 通过率 | pass_count / total_attempts per gate | >= 80% |
| 平均 Gate 耗时 | mean(duration) per gate | 无硬限制，关注趋势 |
| Claude Worker 使用率 | cw_tasks / total_tasks | 10-30%（过高说明 Host Agent 能力不足） |
| 首次通过率 | first_attempt_pass / total | >= 70% |
| Reject-to-fix 周期 | time(reject → re-pass) | 趋势下降为健康 |

---

## 3. 反馈闭环流程

```
日志采集 → 模式检测 → 问题识别 → [输出 finding_report] → 外部任务体系 → SDLC Design
     ↑                                                                        |
     |________________________________________________________________________|
                              (持续改进循环)
```

> Harness 负责到 finding_report 输出为止。后续调度由任务体系负责。

### 3.1 G-Std 闭环（手动触发）

```
1. 用户或 Agent 触发 log-audit skill
2. Skill 扫描日志目录，检测模式
3. 输出审计报告（问题 + 严重度 + 建议动作）
4. 用户决策：接受 / 忽略 / 调整
5. 如接受 → Agent 按 SDLC 流程处理（Design → Implement → ...）
```

### 3.2 G-Full 闭环（半自动）

```
1. 每完成 N 个任务（或定期），自动触发审计
2. 检测到问题 → 输出 finding_report
3. 外部任务体系接收 → 创建任务 → 进入 SDLC 管道
4. 无任务体系时 → 直接呈现给用户，用户决策后按 SDLC 流程处理
5. 处理完成后，日志中标记"finding 已关闭"
6. 持续追踪：如果同类问题 7 天内再次出现 → 自动升级严重度
```

---

## 4. 监控驱动的典型改进场景

### 4.1 代码逻辑与设计不一致

```
检测: 设计 brief 写"接口幂等"，但实现中无幂等检查
日志证据: G4 review 中多次发现类似问题
→ 创建任务: refactor, R2, Class B
→ 任务内容: "为 API 层添加幂等性检查，对齐设计 brief 要求"
```

### 4.2 Provider 稳定性下降

```
检测: z-ai provider 连续 5 次超时
日志证据: failure_log 中 timeout count >= 5
→ 创建任务: update_config, R1, Class A
→ 任务内容: "切换默认 provider 从 z-ai 到 qwen-bailian-coding"
```

### 4.3 测试债务积累

```
检测: 最近 10 个任务中，6 个在 G5 首次 fail
日志证据: gate G5 first_attempt_pass_rate = 40%
→ 创建任务: investigate, R2, Class B
→ 任务内容: "分析 G5 失败根因，是否为测试基础设施问题或代码质量下降"
```

### 4.4 模型能力退化

```
检测: 使用 model-X 的 review 通过率从 85% 降到 60%
日志证据: 趋势追踪数据
→ 创建任务: update_config, R2, Class A
→ 任务内容: "重新评估 model-X 能力评分，考虑降级或替换"
```

---

## 5. 日志保留策略

| 日志类型 | 保留时间 | 说明 |
|----------|----------|------|
| Gate pass/fail | 90 天 | 供趋势分析 |
| 路由决策 | 90 天 | 供模型选择优化 |
| 审计报告 | 永久 | 供长期回溯 |
| 反馈任务记录 | 永久 | 供闭环追踪 |

---

## 6. 与 SDLC 主合同的关系

本文件是 SDLC 主合同 §10（监控与事故规则）的实施细则：
- §10 定义了三条处理路径（Path A/B/C）
- 本文件定义了如何**产出**触发这些路径的信号
- Gate Profile 决定监控深度，与 §2.5 治理模板对齐
