# Harness-4-AIAgents

SDLC governance harness for AI agents — powered by [AgentSDLC](https://github.com/brantzh6/AgentSDLC) + [Claude Worker](https://github.com/brantzh6/claude-worker).

## What is this?

一个 **skills 项目**，用于约束 AI 编码 Agent 在结构化的 SDLC 治理下工作。安装后，你的 AI Agent 会：

- 在编码前先对任务进行 **四维分类**（Level/Type/Risk/Class）
- 遵循 **7 阶段生命周期**（设计 → 设计审查 → 实施 → 代码审查 → 测试 → 上线 → 监控）
- 在高风险任务中自动调用 **Claude Worker** 进行独立外部 review
- 根据 **模型路由策略** 决定何时自审、何时调外部审查

## Supported Agents

| Agent | 类型 | Skill 格式 | 上下文注入方式 |
|-------|------|-----------|---------------|
| [Qoder](https://qoder.ai) | IDE 集成 | `.qoder/rules/` + `.qoder/agents/` | 项目 `.qoder/` 自动加载 |
| [Hermes](https://github.com/NousResearch/hermes-agent) | Python CLI | `~/.hermes/skills/<name>/SKILL.md` | 项目根目录 `AGENTS.md` |
| [OpenClaw](https://github.com/openclaw/openclaw) | Node.js agent | `~/.openclaw/skills/<name>/SKILL.md` | 项目根目录 `AGENTS.md` |

## Prerequisites

使用前请确保：

1. **Claude Code CLI** 已安装（`claude` 命令可用）
   ```bash
   # 验证安装
   claude --version
   ```

2. **worker.py** 已获取（Claude Worker 的核心运行时）
   ```bash
   # 从 GitHub 下载
   curl -o worker.py https://raw.githubusercontent.com/brantzh6/claude-worker/main/code/services/api/claude_worker/worker.py
   ```

3. **至少一个 Provider 已配置**（API key 已设置）
   ```bash
   # 配置 provider
   python worker.py provider set-key qwen-bailian-coding
   # 验证连通性
   python worker.py provider verify qwen-bailian-coding
   ```

## Project Structure

```
Harness-4-AIAgents/
├── core/                              # 通用核心规约（Agent 无关）
│   ├── sdlc-governance.md             # SDLC v3.1 完整治理合同
│   ├── claude-worker-skill.md         # Claude Worker 调用参考
│   ├── model-routing-policy.md        # 模型路由策略（基于能力标签）
│   ├── model-registry.md              # 模型注册表（能力评分 + Provider 映射）
│   ├── leaderboard-sources.md         # Leaderboard 数据源与更新流程
│   └── monitoring-feedback-loop.md    # 监控反馈闭环机制
├── adapters/
│   ├── qoder/                         # Qoder IDE 适配包
│   │   ├── rules/sdlc-governance.md   # Qoder 版 SDLC 规则
│   │   ├── agents/claude-worker.md    # Claude Worker Agent 定义
│   │   └── skills/                    # sdlc-init + log-audit
│   ├── hermes/                        # Hermes Agent 适配包
│   │   ├── skills/                    # sdlc-harness + claude-worker + log-audit
│   │   ├── AGENTS.md                  # 项目上下文模板
│   │   └── init.sh                    # 一键初始化脚本
│   └── openclaw/                      # OpenClaw Agent 适配包
│       ├── skills/                    # sdlc-harness + claude-worker + log-audit
│       ├── AGENTS.md                  # 项目上下文模板
│       └── init.sh                    # 一键初始化脚本
└── samples/
    └── project-profile.md             # Project Profile 模板
```

---

## Usage: Qoder

### 方式一：一键初始化（推荐）

在任意项目中，对 Qoder 说：

> 初始化 SDLC 规约

Qoder 会调用 `sdlc-init` skill，自动完成所有部署。

### 方式二：手动安装

**Step 1: 安装全局 Skills**

将以下文件部署到 Qoder 全局目录：

```powershell
# Windows
$QODER_HOME = "$env:USERPROFILE\.qoder"

# 全局 SDLC 治理规约
New-Item -ItemType Directory -Path "$QODER_HOME\rules" -Force
Copy-Item "core\sdlc-governance.md" -Destination "$QODER_HOME\rules\sdlc-governance.md"

# 全局 Claude Worker Skill
New-Item -ItemType Directory -Path "$QODER_HOME\skills\claude-worker" -Force
Copy-Item "core\claude-worker-skill.md" -Destination "$QODER_HOME\skills\claude-worker\SKILL.md"
# 部署 worker.py（从 claude-worker 仓库下载）
# curl -o "$QODER_HOME\skills\claude-worker\worker.py" https://raw.githubusercontent.com/brantzh6/claude-worker/main/code/services/api/claude_worker/worker.py

# 全局 sdlc-init Skill
New-Item -ItemType Directory -Path "$QODER_HOME\skills\sdlc-init" -Force
Copy-Item "adapters\qoder\skills\sdlc-init\SKILL.md" -Destination "$QODER_HOME\skills\sdlc-init\SKILL.md"
```

**Step 2: 初始化项目**

在目标项目中部署项目级文件：

```powershell
$PROJECT = "your-project-path"

# 创建目录结构
New-Item -ItemType Directory -Path "$PROJECT\.qoder\rules" -Force
New-Item -ItemType Directory -Path "$PROJECT\.qoder\agents" -Force
New-Item -ItemType Directory -Path "$PROJECT\.qoder\claude-worker" -Force

# 拷贝规约到项目（Qoder 仅加载项目级 rules/）
Copy-Item "$QODER_HOME\rules\sdlc-governance.md" -Destination "$PROJECT\.qoder\rules\"
Copy-Item "core\model-routing-policy.md" -Destination "$PROJECT\.qoder\rules\model-routing-policy.md"
Copy-Item "samples\project-profile.md" -Destination "$PROJECT\.qoder\rules\project-profile.md"

# 拷贝 Claude Worker Agent 定义
Copy-Item "adapters\qoder\agents\claude-worker.md" -Destination "$PROJECT\.qoder\agents\"

# 拷贝 worker.py
Copy-Item "$QODER_HOME\skills\claude-worker\worker.py" -Destination "$PROJECT\.qoder\claude-worker\"
```

**Step 3: 配置 Project Profile**

编辑 `.qoder/rules/project-profile.md`，替换为项目实际值：
- 项目名称和描述
- Project Level（L1/L2/L3）
- Project Type（A/B/C/D）
- 环境矩阵
- 构建/测试/Lint 命令
- Claude Worker 默认 provider

### 部署后效果

安装完成后，Qoder 会自动：
- 对非 trivial 任务进行 4 维分类（Level/Type/Risk/Class）
- 根据分类选择治理模板（T1/T2/T3）和 Gate Profile
- T2+ 任务自动进行 role-switching review
- T3 / Class C/D 任务调用 Claude Worker 独立审查
- 每个任务产出结构化完成证据

> **注意**: Qoder 仅自动加载项目级 `.qoder/rules/` 目录，全局 `~/.qoder/rules/` 不会自动注入。因此规约必须部署到每个项目中。

---

## Usage: Hermes

### 方式一：一键初始化

```bash
# 在目标项目目录下执行
bash <(curl -sL https://raw.githubusercontent.com/brantzh6/Harness-4-AIAgents/main/adapters/hermes/init.sh)
```

脚本会自动：
1. 安装 `sdlc-harness` skill 到 `~/.hermes/skills/sdlc-harness/`
2. 安装 `claude-worker` skill 到 `~/.hermes/skills/claude-worker/`
3. 下载 `worker.py` 到 `~/.hermes/skills/claude-worker/`
4. 部署 `AGENTS.md` 到项目根目录

### 方式二：手动安装

```bash
HERMES_HOME="${HERMES_HOME:-$HOME/.hermes}"

# 安装 SDLC 治理 Skill
mkdir -p "$HERMES_HOME/skills/sdlc-harness"
cp adapters/hermes/skills/sdlc-harness/SKILL.md "$HERMES_HOME/skills/sdlc-harness/SKILL.md"

# 安装 Claude Worker Skill
mkdir -p "$HERMES_HOME/skills/claude-worker"
cp adapters/hermes/skills/claude-worker/SKILL.md "$HERMES_HOME/skills/claude-worker/SKILL.md"
curl -o "$HERMES_HOME/skills/claude-worker/worker.py" \
  https://raw.githubusercontent.com/brantzh6/claude-worker/main/code/services/api/claude_worker/worker.py

# 部署项目上下文
cp adapters/hermes/AGENTS.md ./AGENTS.md
```

### 安装后配置

编辑项目根目录的 `AGENTS.md`，修改 `Project Defaults` 部分：

```yaml
project_level: L2      # 改为你的项目等级
project_type: B        # 改为你的项目类型
default_risk: R2       # 改为默认风险等级
default_class: B       # 改为默认变更类别
default_template: T2   # 改为默认治理模板
default_gate_profile: G-Std
```

---

## Usage: OpenClaw

### 方式一：一键初始化

```bash
# 在目标项目目录下执行
bash <(curl -sL https://raw.githubusercontent.com/brantzh6/Harness-4-AIAgents/main/adapters/openclaw/init.sh)
```

脚本会自动：
1. 安装 `sdlc-harness` skill 到 `~/.openclaw/skills/sdlc-harness/`
2. 安装 `claude-worker` skill 到 `~/.openclaw/skills/claude-worker/`
3. 下载 `worker.py` 到 `~/.openclaw/skills/claude-worker/`
4. 部署 `AGENTS.md` 到项目根目录

### 方式二：手动安装

```bash
OPENCLAW_HOME="${OPENCLAW_HOME:-$HOME/.openclaw}"

# 安装 SDLC 治理 Skill
mkdir -p "$OPENCLAW_HOME/skills/sdlc-harness"
cp adapters/openclaw/skills/sdlc-harness/SKILL.md "$OPENCLAW_HOME/skills/sdlc-harness/SKILL.md"

# 安装 Claude Worker Skill
mkdir -p "$OPENCLAW_HOME/skills/claude-worker"
cp adapters/openclaw/skills/claude-worker/SKILL.md "$OPENCLAW_HOME/skills/claude-worker/SKILL.md"
curl -o "$OPENCLAW_HOME/skills/claude-worker/worker.py" \
  https://raw.githubusercontent.com/brantzh6/claude-worker/main/code/services/api/claude_worker/worker.py

# 部署项目上下文
cp adapters/openclaw/AGENTS.md ./AGENTS.md
```

### 安装后配置

同 Hermes，编辑 `AGENTS.md` 中的 `Project Defaults`。

---

## How It Works

### 1. 任务分类

Agent 在开始非 trivial 任务前，先进行四维分类：

```
## Task Classification
- Project Level: L2 (持久个人项目)
- Project Type: B (App/API)
- Change Risk: R2 (中风险)
- Change Class: B (运行时行为变更)
- Governance Template: T2 (标准)
- Gate Profile: G-Std
```

### 2. 生命周期执行

Agent 按 7 阶段生命周期推进，每个阶段都有对应的 Gate：

```
Design (G1) → Design Review (G2) → Implementation (G3) → Code Review (G4) → Testing (G5) → Deploy (G6) → Monitor (G7)
```

Gate Profile 决定每个 Gate 的严格程度：
- **G-Lite (T1)**: 自查即可
- **G-Std (T2)**: Role-switching review（Agent 切换视角自审）
- **G-Full (T3)**: 必须调用 Claude Worker 独立审查

### 3. Claude Worker 独立审查

高风险任务（R3、Class C/D、T3）时，Agent 调用 Claude Worker：

```bash
# 启动独立 review
python worker.py start --kind review \
  --prompt "Review the following changes for correctness and security: ..." \
  --provider qwen-bailian-coding

# 等待完成
python worker.py wait --run-id <run-id>

# 获取结构化结果
python worker.py fetch --run-id <run-id>
```

### 4. 模型路由决策

| 场景 | 执行方 | Review 方 |
|------|--------|-----------|
| 日常开发 (T1/T2) | Host Agent | Host Agent (role-switch) |
| 高风险编码 (R3) | Host Agent | Claude Worker |
| 架构争议 | Host Agent 出方案 | Claude Worker 仲裁 |
| T3 / Class C/D | Host Agent 实施 | Claude Worker review（**必需**） |

---

## Model Registry 与智能路由

模型选择不再硬编码，而是基于 **能力标签系统** 动态匹配：

- 每个模型标注 7 维能力评分（coding / review / architecture / long-context / speed / reasoning / instruction）
- 路由策略根据任务类型 + 风险等级自动匹配最优模型
- 支持 Fallback：首选不可用时自动降级

详见：
- [`core/model-registry.md`](core/model-registry.md) — 模型能力评分表
- [`core/leaderboard-sources.md`](core/leaderboard-sources.md) — Leaderboard 数据源与更新流程
- [`core/model-routing-policy.md`](core/model-routing-policy.md) — 基于能力标签的路由算法

### 模型更新机制

采用**半自动 Leaderboard Tracking**：
- 月度例行检查 LiveCodeBench、Aider、SWE-bench 等数据源
- 重大模型发布时即时评估
- 新模型接入流程：EVALUATE → REGISTER → ROUTE → VERIFY
- 详见 [`core/leaderboard-sources.md`](core/leaderboard-sources.md) §5 更新检查清单

### Provider 配置

```bash
python worker.py provider list          # 查看所有 provider
python worker.py provider set-key <name> # 设置 API key
python worker.py provider verify <name>  # 验证连通性
```

---

## Monitoring Feedback Loop

监控不再是“写完就忘”，而是持续反馈驱动改进的闭环：

```
日志采集 → 模式检测 → 问题识别 → 任务创建 → 回到 SDLC Design 阶段
```

按 Gate Profile 分三级：

| 层级 | 内容 | 适用 |
|------|------|------|
| **G-Lite (T1)** | 基本结构化日志 | 低风险项目 |
| **G-Std (T2)** | 日志 + 偏差检测 + 审计报告 | 标准项目 |
| **G-Full (T3)** | 完整闭环: 设计偏差检测 + 自动任务创建 + 趋势追踪 | 高风险项目 |

检测的典型模式：
- 同一模块反复被 reject → 建议 refactor
- Provider 连续失败 → 建议切换
- 测试首次通过率低 → 分析根因
- 设计-实现不一致 → 创建对齐任务

使用 `log-audit` skill 触发审计：
- Qoder: “审计日志” / “检查项目健康度”
- Hermes/OpenClaw: “audit logs” / “check project health”

详见：[`core/monitoring-feedback-loop.md`](core/monitoring-feedback-loop.md)

---

## Customization

### 修改默认分类

编辑 `samples/project-profile.md`（Qoder 版在 `.qoder/rules/project-profile.md`，Hermes/OpenClaw 版在 `AGENTS.md`），调整默认的 Level/Type/Risk/Class。

### 调整路由策略

编辑 `core/model-routing-policy.md`（或 Qoder 的 `.qoder/rules/model-routing-policy.md`），修改何时必须/可选/不应调用 Claude Worker。

### 添加项目特定的 R3 触发条件

在 Project Profile 的 §7 中添加项目特定的自动升级规则。

### 切换默认 Provider

在 Project Profile 的 Claude Worker 配置部分修改 `default_provider` 和 `default_model`。

---

## FAQ

**Q: Qoder 的全局 rules 为什么不自动加载？**
A: Qoder 目前仅自动加载项目级 `.qoder/rules/`。因此需要将规约部署到每个项目中。`sdlc-init` skill 可以一键完成这个过程。

**Q: Claude Worker 调用失败怎么办？**
A: Agent 会按以下策略处理：1) 检查 exitcode 和 stderr 诊断原因；2) 切换到备用 provider 重试；3) 最多重试 2 次，之后上报用户。

**Q: 我可以不用 Claude Worker 吗？**
A: 可以。对于 T1 和低风险 T2 任务，Agent 默认只进行 self-check 或 role-switching review，不调用 Claude Worker。Claude Worker 只在高风险场景必需。

**Q: 如何确认安装成功？**
A: 给 Agent 一个非 trivial 任务，观察它是否在开始前进行了任务分类（输出 Level/Type/Risk/Class/Template/Gate Profile）。如果有，说明 SDLC 治理已生效。

---

## License

MIT
