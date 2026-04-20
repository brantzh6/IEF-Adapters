# 模型注册表 (Model Registry)

> 统一管理所有可用模型的能力标签、评分和推荐场景。
> 路由策略基于能力标签匹配，而非硬编码具体模型名。
> 更新依据见 `leaderboard-sources.md`。

---

## 1. 能力标签体系

每个模型标注以下能力维度，评分 1-5：

| 维度 | 含义 | 评分说明 |
|------|------|----------|
| coding | 代码生成与补全能力 | 5=顶级编码，1=基本能力 |
| review | 代码审查、缺陷发现 | 5=深度审查，1=表面检查 |
| architecture | 系统设计、方案权衡 | 5=资深架构师，1=简单建议 |
| long-context | 长上下文理解与保持 | 5=200k+ token，1=8k以内 |
| speed | 响应速度 | 5=极快(<2s)，1=很慢(>30s) |
| reasoning | 深度推理、逻辑链 | 5=强推理，1=浅层回答 |
| instruction | 指令遵从度 | 5=精确遵从，1=经常偏离 |

---

## 2. Provider-Model 映射表

### 2.1 z-ai

| Model | coding | review | architecture | long-context | speed | reasoning | instruction | 推荐场景 |
|-------|--------|--------|-------------|-------------|-------|-----------|-------------|----------|
| glm-5.1 | 4 | 5 | 4 | 4 | 3 | 5 | 4 | 高风险终审、核心逻辑 review |
| glm-5 | 4 | 4 | 4 | 4 | 3 | 4 | 4 | 后端编码、标准 review |
| glm-4.7 | 3 | 3 | 3 | 3 | 4 | 3 | 3 | 日常辅助、快速任务 |

### 2.2 qwen-bailian-coding

| Model | coding | review | architecture | long-context | speed | reasoning | instruction | 推荐场景 |
|-------|--------|--------|-------------|-------------|-------|-----------|-------------|----------|
| qwen3.6-plus | 5 | 4 | 4 | 5 | 4 | 4 | 5 | 日常架构、前端、全栈编码 |
| qwen3-coder-plus | 5 | 3 | 3 | 4 | 4 | 3 | 4 | 高速编码、批量实施 |

### 2.3 qwen-bailian

| Model | coding | review | architecture | long-context | speed | reasoning | instruction | 推荐场景 |
|-------|--------|--------|-------------|-------------|-------|-----------|-------------|----------|
| qwen3.6-plus | 5 | 4 | 4 | 5 | 4 | 4 | 5 | 长上下文分析、大材料吸收 |
| qwen3-max | 4 | 4 | 5 | 5 | 3 | 5 | 4 | 深度架构、复杂推理 |

### 2.4 openrouter

| Model | coding | review | architecture | long-context | speed | reasoning | instruction | 推荐场景 |
|-------|--------|--------|-------------|-------------|-------|-----------|-------------|----------|
| claude-opus | 5 | 5 | 5 | 5 | 2 | 5 | 5 | 外部独立视角、终极仲裁 |
| claude-sonnet | 4 | 4 | 4 | 5 | 3 | 4 | 5 | 独立 review、平衡性价比 |
| gpt-4o | 4 | 4 | 4 | 4 | 4 | 4 | 4 | 跨模型交叉验证 |

### 2.5 kimi

| Model | coding | review | architecture | long-context | speed | reasoning | instruction | 推荐场景 |
|-------|--------|--------|-------------|-------------|-------|-----------|-------------|----------|
| kimi-k2.5 | 4 | 4 | 3 | 5 | 4 | 4 | 4 | Review、长文档分析 |

### 2.6 minimax

| Model | coding | review | architecture | long-context | speed | reasoning | instruction | 推荐场景 |
|-------|--------|--------|-------------|-------------|-------|-----------|-------------|----------|
| MiniMax-M2.5 | 3 | 3 | 3 | 4 | 5 | 3 | 3 | 高速铺量、批量简单任务 |

### 2.7 deepseek

| Model | coding | review | architecture | long-context | speed | reasoning | instruction | 推荐场景 |
|-------|--------|--------|-------------|-------------|-------|-----------|-------------|----------|
| deepseek-chat | 4 | 3 | 3 | 4 | 4 | 4 | 3 | 日常编码备选 |
| deepseek-reasoner | 3 | 4 | 4 | 3 | 2 | 5 | 4 | 深度推理、复杂逻辑 |

### 2.8 anthropic

| Model | coding | review | architecture | long-context | speed | reasoning | instruction | 推荐场景 |
|-------|--------|--------|-------------|-------------|-------|-----------|-------------|----------|
| claude-sonnet | 4 | 4 | 4 | 5 | 3 | 4 | 5 | 原生 Claude 直连 |
| claude-opus | 5 | 5 | 5 | 5 | 2 | 5 | 5 | 最高质量输出 |

---

## 3. 选型规则

### 3.1 能力需求映射

| 任务类型 | 必需能力（权重） | 最低评分要求 |
|----------|-----------------|-------------|
| coding (日常) | coding(0.4) + instruction(0.3) + speed(0.3) | coding >= 4 |
| coding (高风险) | coding(0.4) + reasoning(0.3) + instruction(0.3) | coding >= 4, reasoning >= 4 |
| review (标准) | review(0.4) + reasoning(0.3) + instruction(0.3) | review >= 4 |
| review (高风险) | review(0.3) + reasoning(0.3) + architecture(0.2) + instruction(0.2) | review >= 4, reasoning >= 4 |
| architecture | architecture(0.4) + reasoning(0.3) + long-context(0.3) | architecture >= 4 |
| testing | coding(0.3) + review(0.3) + instruction(0.4) | coding >= 3, instruction >= 4 |
| long-context | long-context(0.5) + instruction(0.3) + reasoning(0.2) | long-context >= 5 |
| bulk/batch | speed(0.5) + coding(0.3) + instruction(0.2) | speed >= 4 |

### 3.2 风险等级加权

| Risk | 额外要求 |
|------|----------|
| R1 | 无额外要求，允许使用 speed 优先模型 |
| R2 | 主能力评分 >= 4 |
| R3 | 主能力评分 >= 4，reasoning >= 4，优先选择独立 provider |

### 3.3 Fallback 策略

1. 首选模型不可用 → 同 provider 内降级（如 glm-5.1 → glm-5）
2. Provider 不可用 → 切换到同能力等级的备用 provider
3. 所有备选不可用 → 上报用户，不猜测执行

---

## 4. 新模型接入流程

```
1. EVALUATE — 在关注的 leaderboard 上确认模型能力
   - 查看 LiveCodeBench / SWE-bench / Aider Polyglot 等评分
   - 运行 2-3 个标准化 prompt（coding、review、architecture 各一个）评估实际表现

2. REGISTER — 添加到本注册表
   - 填写 provider、model name、7 维能力评分
   - 标注推荐场景
   - 记录评估日期和依据

3. ROUTE — 更新路由策略
   - 确认新模型是否替代现有默认
   - 如果能力更强，更新 model-routing-policy 中的默认选择
   - 如果是补充，添加为备选

4. VERIFY — 在实际项目中验证
   - 用新模型跑 3-5 个真实任务
   - 对比结果质量与原模型
   - 确认无退化后正式启用
```

---

## 5. 注册表更新记录

| 日期 | 变更 | 依据 |
|------|------|------|
| 2026-04-20 | 初始注册表创建 | 基于现有 provider 配置和实际使用经验 |
