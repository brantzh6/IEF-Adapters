# 外部系统对接接口

> Harness 定义治理规则和监控标准，但不实现知识积累和任务调度。
> 本文件定义 Harness 与外部知识体系、任务体系的对接协议。
> 只定义数据格式和交互点，不约束外部系统的实现方式。

---

## 1. 三层架构定位

```
Harness（治理约束）  ←→  知识体系（经验积累）  ←→  任务体系（执行驱动）
```

| 系统 | 职责 | 不管什么 |
|------|------|----------|
| Harness | SDLC Gate、模型路由、质量标准、监控规范 | 经验存储、任务调度 |
| 知识体系 | 经验积累、规则提炼、模式匹配、知识检索 | 质量判定、流程控制 |
| 任务体系 | 任务分解、优先级排序、进度追踪、依赖管理 | 经验存储、治理规则 |

---

## 2. Harness → 知识体系：经验条目输出格式

当 Harness 监控发现可提炼的经验时，输出以下格式供知识体系接收：

```yaml
# Experience Entry (Harness Output)
schema_version: "1.0"
type: experience_entry

id: "exp-{timestamp}-{short_hash}"
discovered_at: <ISO 8601>
project: <project identifier>

# 经验内容
pattern:
  description: "<用自然语言描述发现的模式>"
  detection_rule: "<触发条件>"
  severity: HIGH / MEDIUM / LOW

root_cause: "<根因分析，如已知>"
resolution: "<修复方案，如已执行>"
outcome: "<修复效果，如已验证>"

# 元数据
tags: [<领域标签>]
related_modules: [<涉及文件/模块>]
gate_context: <发现于哪个 Gate>
model_context: <使用什么模型分析得出>

# 生命周期
status: draft / confirmed / obsolete
expires_when: "<失效条件，如相关代码被重构>"
```

### 2.1 输出时机

| 时机 | 条件 |
|------|------|
| 修复验证后 | 反馈任务完成且 Gate 通过 |
| 审计发现确认后 | 用户确认审计报告中的问题为真 |
| 模型评估后 | 模型能力评分变更 |

### 2.2 Harness 不关心的

- 知识体系如何存储（YAML / 向量库 / 图数据库）
- 知识体系如何检索（语义 / tag / 全文）
- 知识体系如何过期淘汰
- 知识是否跨项目共享

---

## 3. 知识体系 → Harness：规则注入格式

知识体系可向 Harness 注入提炼出的规则，影响 Gate 判定或模型选择：

```yaml
# Injected Rule (Knowledge System Input)
schema_version: "1.0"
type: injected_rule

id: "rule-{domain}-{seq}"
source: knowledge_system
confidence: high / medium / low
based_on: ["exp-xxx", "exp-yyy"]  # 关联的经验条目

# 规则内容
rule:
  scope: gate_check / model_routing / review_focus
  condition: "<触发条件>"
  action: "<建议动作>"
  
# 示例
# rule:
#   scope: review_focus
#   condition: "修改涉及 auth 模块"
#   action: "G4 review 重点检查幂等性和 token 过期处理"

# 生命周期
active: true
review_after: <ISO 8601 date>  # 建议复审日期
```

### 3.1 注入点

| 注入目标 | 影响 |
|----------|------|
| `gate_check` | G4 review 时额外关注点 |
| `model_routing` | 特定场景的模型偏好调整 |
| `review_focus` | Code Review 的重点检查清单 |

### 3.2 Harness 如何消费

- Agent 在执行 Gate 前检查是否有 active 的 injected_rule
- 匹配 condition → 将 action 纳入当前 Gate 的检查项
- Harness 不负责规则的正确性（由知识体系保证）

---

## 4. Harness → 任务体系：发现输出格式

当 Harness 监控检测到问题时，输出标准格式供任务体系决策：

```yaml
# Finding Report (Harness Output)
schema_version: "1.0"
type: finding_report

id: "finding-{timestamp}-{short_hash}"
generated_at: <ISO 8601>
generated_by: log_audit / monitoring_loop / manual_review

# 发现内容
finding:
  title: "<问题标题>"
  severity: HIGH / MEDIUM / LOW
  pattern: "<匹配到的检测模式>"
  evidence:
    - log_ref: "<日志文件路径>"
      excerpt: "<关键内容摘录>"
  occurrences: <number>

# 建议（非强制）
suggested_action:
  type: refactor / fix / investigate / update_config
  description: "<建议动作描述>"
  suggested_classification:
    risk: R1 / R2 / R3
    class: A / B / C / D
    template: T1 / T2 / T3

# 上下文
context:
  gate_profile: G-Lite / G-Std / G-Full
  affected_modules: [<模块列表>]
  trend: improving / stable / degrading
```

### 4.1 输出时机

| 触发 | 条件 |
|------|------|
| log-audit skill | 用户触发或定期触发审计 |
| G-Full 自动检测 | 达到检测阈值 |
| 趋势告警 | 健康指标持续下降 |

### 4.2 Harness 不关心的

- 任务体系是否采纳建议
- 任务的优先级排序和调度
- 任务的分配和执行进度
- 任务之间的依赖关系

---

## 5. 任务体系 → Harness：任务执行遵循

任务体系创建的任务在执行时，仍然遵循 Harness 的 SDLC 流程：

```yaml
# Task Handoff (Task System → Harness)
schema_version: "1.0"
type: task_handoff

task_id: "<任务体系分配的 ID>"
origin: finding_report / user_created / knowledge_triggered
origin_ref: "<关联的 finding/rule ID>"

# 要求 Harness 管控的参数
sdlc_entry_point: design / implement  # 从哪个阶段开始
classification:
  risk: R1 / R2 / R3
  class: A / B / C / D
  template: T1 / T2 / T3
```

任务进入 SDLC 管道后，Harness 正常管控全部 Gate。

---

## 6. 当前状态

| 接口 | 状态 | 说明 |
|------|------|------|
| Harness → 知识体系 | **格式已定义，未对接** | 等知识体系工程启动 |
| 知识体系 → Harness | **格式已定义，未对接** | 等知识体系工程启动 |
| Harness → 任务体系 | **格式已定义，未对接** | 等任务体系工程启动 |
| 任务体系 → Harness | **格式已定义，未对接** | 等任务体系工程启动 |

当外部系统未接入时，Harness 按当前行为工作：
- 审计结果直接输出给用户（替代任务体系）
- 经验由用户手动管理（替代知识体系）
- 无规则注入（使用 Harness 内置规则）
