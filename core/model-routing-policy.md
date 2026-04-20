# 模型路由策略 — 通用模板

> 本模板指导 AI Agent 如何在自身工作与 Claude Worker 外部审查之间分配任务。
> 模型选择基于 `model-registry.md` 中的能力标签，而非硬编码。
> 项目使用时需根据实际情况调整。

---

## 1. 总体原则

### 1.1 Host Agent 是日常主力
Host Agent 默认承担所有日常任务：架构设计、编码、测试、基本 review。
不需要为每个任务都调用外部 worker。

### 1.2 Claude Worker 只用于高价值场景
Claude Worker 是稀缺资源，只在以下场景启用：
- 高风险独立审查
- 架构争议仲裁
- 信心不足时的外部验证
- Class C/D 独立复核

### 1.3 Review 与 Implementation 分离
T2+ 任务中，Agent 不能只用"实施者视角"做 review。
必须进行显式 role-switching（T2）或调用 Claude Worker（T3/高风险）。

### 1.4 最强资源不是默认资源
Claude Worker 用于定盘，不用于常规吞吐。

---

## 2. 默认任务分配规则

### 2.1 架构任务

| 场景 | 执行方 | Review 方 |
|------|--------|-----------|
| 日常架构设计 | Host Agent | Host Agent (role-switch) |
| 高风险架构 | Host Agent 出初稿 | Claude Worker review → Host Agent 终审 |
| 架构争议 | Host Agent 出方案 | Claude Worker 仲裁 |

### 2.2 编码任务

| 场景 | 执行方 | Review 方 |
|------|--------|-----------|
| 日常后端/前端 | Host Agent | Host Agent (role-switch) |
| 批量重构/补测试 | Host Agent | Host Agent (role-switch) |
| 高风险编码 | Host Agent | Claude Worker review |
| 核心链路变更 | Host Agent | Claude Worker review → Host Agent 终审 |

### 2.3 Review 任务

| 场景 | 执行方 |
|------|--------|
| T1 任务 review | Host Agent self-check |
| T2 日常 review | Host Agent role-switching |
| T2 高风险 review | Claude Worker |
| T3 / Class C/D review | Claude Worker（必需） |

### 2.4 测试任务

| 场景 | 执行方 | Review 方 |
|------|--------|-----------|
| 日常测试 | Host Agent | Host Agent |
| 批量补测试 | Host Agent | Host Agent (role-switch) |
| 测试盲点检查 | Claude Worker | — |
| 高风险测试策略 | Host Agent 出方案 | Claude Worker review |

---

## 3. 升级规则

### 3.1 必须调用 Claude Worker 的情况

出现以下任一情况，Host Agent 必须调用 Claude Worker：
- 涉及 memory / state / scheduler / orchestration 的 review
- 属于 R3 的代码 review
- 属于 Class C / D 的任何 review
- Agent 自身对当前结论信心不足
- 架构方案存在重大争议
- 事故后的永久修复方案需要外部复核
- 用户显式要求

### 3.2 可选调用 Claude Worker 的情况

以下情况 Agent 可选择是否调用：
- T2 标准 review 中发现复杂度高于预期
- 需要检查是否只覆盖 happy path
- 需要额外的反方视角

### 3.3 不应调用 Claude Worker 的情况

- T1 任务（自查即可）
- Trivial 且低风险的 T2 任务
- 纯文档/注释变更
- 常规铺量编码

---

## 4. 标准执行流程

### 4.1 日常功能开发

```
1. Agent 读需求并出方案 (G1)
2. Agent role-switch review 方案 (G2)
3. Agent 实施 (G3)
4. Agent role-switch review 代码 (G4)
5. Agent 编写和运行测试 (G5)
6. 部署 (G6)
7. 监控 (G7)
```

### 4.2 高风险任务

```
1. Agent 出方案初稿 (G1)
2. Claude Worker 做独立 review (G2)
3. Agent 综合反馈决策
4. Agent 实施 (G3)
5. Claude Worker 做独立代码 review (G4)
6. Agent 测试 (G5)
7. Agent 终审后部署 (G6/G7)
```

### 4.3 事故修复

```
1. Agent 快速实施临时修复
2. Claude Worker 检查是否只修表面
3. Agent 整理事故上下文与长期方案
4. Claude Worker review 永久修复方案
5. Agent 实施永久修复
6. Agent 测试
7. Agent 审核发布
```

---

## 5. 禁止规则

Agent 不得：
- 让 Claude Worker 成为每个任务的默认 reviewer
- 在高风险任务中跳过独立 review
- 在 Class C/D 任务中只做自查
- 在信心不足时仍然自行通过
- 在模型结论冲突时不做升级仲裁
- 无限循环调用 Claude Worker 重试（最多 2 次失败后上报用户）

---

## 6. 模型选择算法

当 Claude Worker 被触发时，按以下算法选择具体模型：

### 6.1 选择流程

```
输入: task_type + risk_level + change_class

1. 确定 required_capabilities
   - task_type → 查 model-registry.md §3.1 能力需求映射表
   - risk_level → 查 model-registry.md §3.2 风险等级加权

2. 匹配 registry 中的模型
   - 过滤：满足最低评分要求的模型
   - 排序：按加权总分从高到低
   - 选择：取排名第一

3. 独立性检查（仅 review 场景）
   - R3 / Class C/D：优先选择与 Host Agent 不同的 provider
   - 确保 reviewer 与 implementer 视角独立

4. Fallback
   - 首选不可用 → 同 provider 降级
   - Provider 不可用 → 切换到同能力等级的备用
   - 全部不可用 → 上报用户
```

### 6.2 快速参考表

| 场景 | 推荐能力组合 | 典型匹配 |
|------|-------------|----------|
| 日常编码 | coding + speed | qwen3.6-plus |
| 高风险编码 | coding + reasoning | glm-5, qwen3-max |
| 标准 review | review + reasoning | qwen3.6-plus, kimi-k2.5 |
| 高风险 review | review + reasoning + architecture | glm-5.1, claude-opus |
| 架构设计 | architecture + reasoning + long-context | qwen3-max, claude-opus |
| 批量任务 | speed + coding | MiniMax-M2.5, qwen3-coder-plus |
| 仲裁 | 独立 provider + review + reasoning | openrouter/claude-opus |

> 注意：以上「典型匹配」仅为当前 registry 快照，实际选择应查 model-registry.md 实时数据。

---

## 7. 路由记录格式

每个非 trivial 任务，Agent 必须记录路由决策：

```
Task Type: architecture / backend / frontend / review / test / incident
Risk Level: R1 / R2 / R3
Change Class: A / B / C / D
Executor: Host Agent / Claude Worker
Review Model: Agent self-check / Agent role-switch / Claude Worker
Claude Worker Used: yes / no
Model Selected: <provider>/<model> (如使用 Claude Worker)
Selection Reason: <基于能力标签匹配 / 指定 / fallback>
Reason: <升级理由或不升级理由>
```
