# 03. 训练数据与 Reward 设计

## 1. 训练 Fugu-like 模型的真正难点

训练 Fugu-like orchestrator 的难点不是模型结构，而是数据和 reward。

你需要的数据不是普通 instruction tuning 数据，而是：

```text
同一个任务
  -> 多个 worker 的输出
  -> 每个 worker 的成本和延迟
  -> 每个 worker 在该任务上的质量
  -> 某个 workflow 执行后的最终结果
  -> 失败原因
```

没有这些数据，orchestrator 不知道“哪个 worker 在什么条件下好”，也不知道“什么 workflow 会成功”。

## 2. 数据分层

### 2.1 Level 0：原始任务样本

字段：

```json
{
  "task_id": "task_001",
  "query": "Fix this failing pytest in the repository...",
  "domain": "coding",
  "task_type": "debugging",
  "risk_level": "medium",
  "language": "zh/en",
  "requires_tools": true,
  "requires_citation": false,
  "gold": {
    "type": "unit_tests",
    "command": "pytest tests/test_parser.py -q"
  }
}
```

来源：

- 真实用户请求。
- 内部 issue/PR/code review。
- 历史客服和运营任务。
- 可公开使用的 benchmark。
- 专家构造任务。

要求：

- 去除隐私和 token。
- 标注任务类型。
- 保留上下文依赖。
- 明确验收标准。

### 2.2 Level 1：多 worker replay 数据

对每个任务，让多个 worker 独立完成。

```json
{
  "task_id": "task_001",
  "worker_results": [
    {
      "worker": "cheap_model",
      "output": "...",
      "cost": 0.01,
      "latency_ms": 1200,
      "reward": 0.0,
      "failure": "test_failed"
    },
    {
      "worker": "strong_model",
      "output": "...",
      "cost": 0.18,
      "latency_ms": 6500,
      "reward": 1.0,
      "failure": null
    }
  ]
}
```

用途：

- 训练 router。
- 构造 Fugu selection-head 的 soft labels。
- 分析 worker specialization。
- 建立成本-质量 Pareto frontier。

### 2.3 Level 2：workflow trace 数据

记录多 agent workflow 的完整执行。

```json
{
  "workflow_id": "wf_001",
  "task_id": "task_001",
  "orchestrator": "teacher_model_or_rule_v1",
  "steps": [
    {
      "id": 1,
      "worker": "model_a",
      "subtask": "Inspect the failing traceback and identify the likely file.",
      "access": [],
      "output_ref": "blob_001"
    },
    {
      "id": 2,
      "worker": "model_b",
      "subtask": "Propose a minimal patch using step 1.",
      "access": [1],
      "output_ref": "blob_002"
    }
  ],
  "final_answer_ref": "blob_final",
  "reward": 1.0,
  "cost": 0.43,
  "latency_ms": 42000
}
```

用途：

- 训练 Conductor-like workflow generator。
- 做 offline policy evaluation。
- 找出常见成功 topology。
- 做失败归因。

### 2.4 Level 3：人工偏好与失败标签

自动 reward 不够时，需要人工或业务 owner。

```json
{
  "task_id": "task_123",
  "comparison": {
    "answer_a": "strong_direct",
    "answer_b": "workflow_v2",
    "winner": "workflow_v2",
    "reason": "better citation support and covered counterarguments"
  },
  "failure_labels": [
    "missing_primary_source",
    "overconfident_claim"
  ]
}
```

用途：

- 训练 judge/ranker。
- 校准 DRACO-like rubric。
- 找出 LLM judge 的偏差。

## 3. Reward 设计

### 3.1 可验证任务 reward

最适合训练 orchestrator。

| 任务 | Reward |
|---|---|
| 数学 | final answer exact match / symbolic check |
| 代码 | tests pass / benchmark score |
| 数据任务 | output schema valid + numeric accuracy |
| 检索问答 | claim 是否被引用支持 |
| 多轮工具任务 | final state 是否达到目标 |

例子：

```python
def reward_code_task(result):
    if not result.patch_applied:
        return 0.0
    if result.tests_passed:
        return 1.0
    if result.compiles:
        return 0.3
    return 0.0
```

### 3.2 Workflow 格式 reward

Conductor/Fugu Ultra 类模型必须先学会输出可执行 workflow。

```python
def reward_workflow_format(workflow):
    if not parseable_json(workflow):
        return 0.0
    if not valid_schema(workflow):
        return 0.0
    if has_disallowed_worker(workflow):
        return 0.0
    if exceeds_step_budget(workflow):
        return 0.0
    return 1.0
```

训练早期可以把格式 reward 权重设高，否则模型会生成看似聪明但不可执行的计划。

### 3.3 成本和延迟 reward

Fugu-like 系统必须考虑成本，否则最优策略会退化成“每次都调用所有最贵模型”。

```text
final_reward =
  quality_score
  - lambda_cost * normalized_cost
  - lambda_latency * normalized_latency
  - lambda_policy * policy_violation
```

示例：

```python
def total_reward(quality, cost_usd, latency_s, violation):
    return (
        quality
        - 0.10 * min(cost_usd / 1.00, 3.0)
        - 0.05 * min(latency_s / 60.0, 3.0)
        - 1.00 * violation
    )
```

注意：高价值任务可以降低成本惩罚，实时交互任务应提高延迟惩罚。

### 3.4 深度研究 reward

参考 DRACO 思路，不要只给整体分。

```yaml
rubric:
  factual_accuracy: 0.40
  breadth_depth: 0.25
  citation_quality: 0.20
  objectivity: 0.10
  clarity: 0.05
negative_criteria:
  unsupported_claim: -0.15
  fabricated_source: -0.30
  harmful_advice: -1.00
```

Claim-level citation check：

```text
claim -> cited source -> quoted passage -> support / contradict / insufficient
```

如果 citation 只是装饰，Fusion/Fugu-like 系统可能会写得更长但更不可靠。

## 4. 软标签：训练 Fugu selection head 的核心

对一个任务 `q_i`，假设 4 个 worker 的平均 reward：

```text
cheap_model: 0.20
coding_model: 0.95
reasoning_model: 0.75
general_model: 0.60
```

不要只标 `coding_model` 为 1，其他为 0。更好是：

```python
import numpy as np

scores = np.array([0.20, 0.95, 0.75, 0.60])
tau = 0.2
target = np.exp(scores / tau) / np.exp(scores / tau).sum()
```

这会让模型知道：

- coding_model 最好。
- reasoning_model 也不错。
- general_model 可作为 fallback。
- cheap_model 不适合这个任务。

这正是 Fugu 报告中 soft performance distribution 的思想。

## 5. 数据量建议

现实建议，不是论文复现规模：

| 阶段 | 数据量 | 目标 |
|---|---:|---|
| PoC | 200-500 tasks | 验证任务分布和 worker 差异 |
| Router v1 | 2k-10k tasks | 训练 cheap/strong/worker selection |
| Selection head v1 | 10k-50k replayed tasks | 学 worker soft distribution |
| Workflow v1 | 1k-5k high-quality traces | 学可执行 workflow schema |
| RL/black-box optimize | 200-2k expensive end-to-end tasks | 优化真实成功率 |

端到端 workflow 任务很贵，优先使用可验证高价值任务。

## 6. 失败标签体系

没有失败标签，后续无法改进。

推荐标签：

```yaml
failure_labels:
  - route_wrong_worker
  - cheap_model_overconfident
  - workflow_invalid_schema
  - workflow_too_expensive
  - tool_permission_denied
  - test_failed
  - hallucinated_citation
  - unsupported_claim
  - missed_requirement
  - incomplete_final_synthesis
  - timeout
  - user_rejected
```

每次线上失败至少要归到一类。后续训练可以把这些标签转成：

- router negative examples。
- workflow validator rules。
- reward penalty。
- benchmark hard set。

## 7. 数据采集顺序

推荐顺序：

```text
1. 先收集真实任务和强模型 baseline
2. 多 worker replay，得到 worker reward/cost/latency
3. 训练 cheap-vs-strong router
4. 上线小流量，继续收集 trace
5. 对高价值任务手工/强模型生成 workflow
6. 执行 workflow，保存成功和失败轨迹
7. 训练 workflow generator
8. 只在离线 benchmark 胜出后灰度上线
```

不要倒过来。没有 replay 和 trace，直接 RL 训练 workflow model 成本高且不可控。

