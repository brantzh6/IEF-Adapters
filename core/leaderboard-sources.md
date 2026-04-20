# Leaderboard 数据源与更新流程

> 定义模型注册表的更新依据、关注的 Leaderboard、评估维度和检查清单。
> 确保模型选择始终基于客观数据，而非主观印象。

---

## 1. 关注的 Leaderboard

| Leaderboard | URL | 关注维度 | 更新频率 |
|-------------|-----|----------|----------|
| LiveCodeBench | https://livecodebench.github.io | 代码生成、补全、修复 | 月度 |
| Aider Polyglot | https://aider.chat/docs/leaderboards | 多语言编码、指令遵从 | 实时 |
| SWE-bench | https://www.swebench.com | 真实 issue 修复能力 | 季度 |
| LMSYS Chatbot Arena | https://chat.lmsys.org | 综合对话、推理、指令遵从 | 实时 |
| BigCodeBench | https://bigcode-bench.github.io | 代码生成基准 | 月度 |
| HumanEval+ | https://evalplus.github.io | 代码正确性严格测试 | 季度 |
| MMLU-Pro | — | 知识推理基准 | 季度 |

---

## 2. 关注指标

### 2.1 编码能力

| 指标 | 来源 | 权重 | 说明 |
|------|------|------|------|
| pass@1 (code generation) | LiveCodeBench, HumanEval+ | 高 | 一次生成正确率 |
| Polyglot benchmark | Aider | 高 | 多语言编码实际表现 |
| SWE-bench resolved | SWE-bench | 高 | 真实项目 bug 修复率 |
| Edit format compliance | Aider | 中 | 编辑格式遵从率（影响工具链集成） |

### 2.2 Review 能力

| 指标 | 来源 | 权重 | 说明 |
|------|------|------|------|
| Bug detection rate | 自测 | 高 | 已知 bug 的检出率 |
| False positive rate | 自测 | 中 | 误报率（越低越好） |
| Reasoning depth | Arena | 中 | 能否给出深层原因分析 |

### 2.3 架构能力

| 指标 | 来源 | 权重 | 说明 |
|------|------|------|------|
| Design quality | 自测 | 高 | 方案完整性、权衡合理性 |
| Long-context retention | Needle-in-haystack | 高 | 长文档中信息保持 |
| Alternative generation | 自测 | 中 | 能否提出有价值的替代方案 |

### 2.4 通用能力

| 指标 | 来源 | 权重 | 说明 |
|------|------|------|------|
| Instruction following | Arena, IFEval | 高 | 精确执行复杂指令 |
| Response latency | 实测 | 中 | TTFT + 生成速度 |
| Context window | 官方文档 | 中 | 有效上下文长度 |

---

## 3. 评估维度与权重（按任务场景）

| 任务场景 | 编码能力 | Review 能力 | 架构能力 | 速度 | 推理 | 指令遵从 |
|----------|---------|------------|---------|------|------|----------|
| 日常编码 | 40% | — | — | 30% | — | 30% |
| 高风险编码 | 35% | — | 15% | — | 30% | 20% |
| 标准 Review | — | 40% | — | — | 30% | 30% |
| 高风险 Review | — | 30% | 20% | — | 30% | 20% |
| 架构设计 | — | — | 40% | — | 30% | 30% |
| 批量任务 | 30% | — | — | 50% | — | 20% |

---

## 4. 更新节奏

| 触发条件 | 动作 |
|----------|------|
| 每月第一周 | 例行检查：浏览关注的 leaderboard，记录变化 |
| 重大模型发布 | 即时评估：新模型上线 1-2 周内完成评估 |
| Provider 新增模型 | 按需评估：确认能力后决定是否接入 |
| 实际使用中发现退化 | 紧急复查：对比 leaderboard 数据确认是否降级 |
| 季度末 | 全面审查：所有模型重新评分，清理弃用模型 |

---

## 5. 更新检查清单

每次更新模型注册表时，逐项完成：

```
□ 1. 查 Leaderboard
  - 浏览 LiveCodeBench、Aider、SWE-bench 最新数据
  - 记录关注模型的排名变化
  - 标记新进入 top-10 的模型

□ 2. 对比现有配置
  - 当前默认模型的排名是否下降？
  - 是否有新模型在关键指标上超越当前默认？
  - 现有模型是否被官方弃用或有已知问题？

□ 3. 决策
  - 无变化 → 记录"已检查，无需更新"
  - 有更优模型 → 进入接入流程（EVALUATE → REGISTER → ROUTE → VERIFY）
  - 现有模型退化 → 降级或替换

□ 4. 更新 Registry
  - 修改 model-registry.md 中的评分
  - 添加/移除模型条目
  - 更新"推荐场景"列

□ 5. 更新路由
  - 如果默认模型发生变化，更新 model-routing-policy.md
  - 通知使用 Harness 的项目更新本地配置

□ 6. 记录
  - 在 model-registry.md §5 添加更新记录
  - 记录变更原因和依据
```

---

## 6. 自测 Prompt 标准化

用于评估新模型时的标准化测试 prompt：

### 6.1 Coding Test

```
实现一个带超时重试的 HTTP 客户端包装，要求：
1. 支持 GET/POST
2. 可配置重试次数和退避策略
3. 超时后抛出明确错误
4. 支持请求/响应拦截器
用 TypeScript 实现，包含类型定义。
```

### 6.2 Review Test

```
Review 以下代码，找出所有问题（包含一个并发bug、一个资源泄露、一个逻辑错误）：
[提供预设的含 bug 代码片段]
```

### 6.3 Architecture Test

```
设计一个事件驱动的任务调度系统，支持：
- 优先级队列
- 任务依赖（DAG）
- 失败重试与死信队列
- 水平扩展
给出至少 2 个方案，逐一权衡利弊，推荐一个并说明理由。
```

评估标准：完整性、正确性、深度、是否遵从格式要求。
