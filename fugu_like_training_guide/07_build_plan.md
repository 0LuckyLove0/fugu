# 07. 落地路线图

## 1. 两周 PoC

目标：证明任务分布里存在多模型路由价值。

### 交付物

- 统一模型调用脚本。
- ModelRegistry。
- TraceStore。
- 200-500 个真实或半真实任务。
- cheap/strong/3-5 workers replay 结果。
- 成本-质量报告。

### 每日安排

```text
Day 1-2:
  建 ModelRegistry 和统一调用封装。

Day 3-4:
  整理任务样本，脱敏，打 task_type/risk_level。

Day 5-7:
  对每个任务跑 cheap/strong/若干 worker，保存输出、成本、延迟。

Day 8-9:
  写 reward/verifier：测试、exact match、rubric judge、人工抽查。

Day 10-11:
  训练简单 router：embedding + logistic regression。

Day 12-13:
  画成本-质量曲线，找出可降本任务类型。

Day 14:
  决策是否进入 6 周版本。
```

### 验收

- 能明确回答哪些任务 cheap 足够。
- 能明确回答哪些任务 worker 差异明显。
- 能明确回答是否值得训练 orchestrator。

## 2. 六周版本

目标：做出 Fugu-like low-latency router 和 workflow prototype。

### Week 1：Trace 基础设施

- API gateway。
- trace schema。
- model cost/latency logging。
- task profile。

验收：所有调用可回放。

### Week 2：Router v1

- 多 worker replay。
- cheap/strong label。
- router classifier。
- threshold tuning。

验收：低风险任务成本下降，质量可控。

### Week 3：Selection head v1

- worker soft label。
- embedding/MLP selector。
- 对比 hand-written rule。
- 分析 worker specialization。

验收：selector 在 held-out tasks 上高于规则。

### Week 4：Workflow schema + executor

- 定义 JSON schema。
- validator。
- access list 上下文构造。
- agent private transcript。
- budget guard。

验收：强模型生成的 workflow 可执行率 > 95%。

### Week 5：Workflow traces

- 用强模型/人工生成 100-300 条 teacher workflow。
- 执行并打 reward。
- 分析成功拓扑。

验收：高价值任务上至少一类 workflow 明显胜过 strong_direct。

### Week 6：Workflow SFT baseline

- 微调小模型或用 structured prompt baseline。
- 离线评测合法率、质量、成本。
- 和 OpenRouter Fusion/Fugu 外部 baseline 对比。

验收：给出是否继续训练 RL/GRPO 的证据。

## 3. 三个月版本

目标：进入真正 learned orchestration。

### Month 1：数据和评测稳定

- 任务集扩大到 2k-10k。
- 至少 5 个 task families。
- 建 hard set。
- 建 DRACO-like rubric。
- 每周 replay 新模型。

### Month 2：训练低延迟 orchestrator

- 训练 worker selector。
- 尝试 small LM hidden state + head。
- 用 soft labels 做 SFT。
- 用黑盒优化微调阈值/head。
- 做线上灰度。

### Month 3：训练 workflow orchestrator

- 1k-5k workflow traces。
- SFT workflow generator。
- structured output 强约束。
- 小规模 GRPO。
- 只对复杂任务灰度上线。

## 4. 算力和成本估计

粗略量级，实际取决于 worker 价格和任务长度。

| 阶段 | 主要成本 | 风险 |
|---|---|---|
| PoC replay | 多 worker API 调用 | 输出质量不可比，需要人工校准 |
| Router training | 低，本地 CPU/GPU 可做 | label 噪声 |
| Selection head | 中，小 GPU | worker pool 变更导致漂移 |
| Workflow SFT | 中到高 | teacher workflow 质量决定上限 |
| GRPO/RL | 高，执行每个 workflow 很贵 | reward 稀疏和训练不稳定 |

节约成本的方法：

- 先用离线 replay。
- 先训练小模型和浅层 head。
- 先在可验证任务上训练。
- 限制 workflow step 数。
- 用模拟 worker 做格式预训练。

## 5. 决策门槛

### 继续投入的信号

- worker 在不同任务上有明显 specialization。
- router 能在不明显降质的情况下降本。
- workflow 对复杂任务有可复现收益。
- reward 能稳定区分好坏。
- trace 能定位失败。

### 停止或降级的信号

- strong_direct 已接近满分。
- cheap/strong 差异无法预测。
- workflow 提升主要来自更长输出，而不是事实/测试通过。
- LLM judge 与人工评审分歧大。
- 成本或延迟超过业务可接受范围。

## 6. 最终形态

成熟系统应长这样：

```text
Unified AI Entry
  -> policy/risk gate
  -> router for cheap vs strong
  -> selection head for low-latency worker choice
  -> workflow orchestrator for complex tasks
  -> verifier/judge/tools
  -> trace/replay/training loop
```

真正的护城河不是某一个 orchestrator checkpoint，而是：

- 真实任务轨迹。
- worker 能力画像。
- 领域 reward。
- 失败样本。
- 成本-质量曲线。
- 可回放评测平台。

