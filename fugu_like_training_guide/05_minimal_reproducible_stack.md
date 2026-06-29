# 05. 最小可复现技术栈

## 1. 推荐栈

| 层 | 推荐工具 | 用途 |
|---|---|---|
| API gateway | LiteLLM / 自写 OpenAI-compatible adapter | 统一模型调用 |
| Orchestrator SFT/RL | Hugging Face Transformers + TRL | 训练 router/workflow model |
| Structured output | vLLM structured outputs / JSON schema / constrained decoding | 限制 workflow 格式 |
| Black-box optimization | pycma / CMA-ES | 优化 selection head 或策略参数 |
| Trace store | SQLite 起步，Postgres 生产 | 存请求、模型调用、reward |
| Evaluation | pytest / custom harness / LLM judge / rubric evaluator | 计算 reward |
| Experiment tracking | W&B / MLflow / plain JSONL | 对比版本 |

## 2. 目录结构

```text
fugu_like/
  configs/
    models.yaml
    strategies.yaml
  data/
    tasks.jsonl
    replay_results.jsonl
    workflow_traces.jsonl
  src/
    gateway.py
    model_registry.py
    trace_store.py
    router_train.py
    workflow_schema.py
    workflow_executor.py
    rewards.py
    eval_harness.py
  notebooks/
    analyze_worker_specialization.ipynb
  tests/
    test_workflow_schema.py
    test_rewards.py
```

## 3. ModelRegistry 配置

```yaml
workers:
  - id: cheap
    provider: openrouter
    model: provider/cheap-model
    cost_weight: 0.1
    capabilities: [rewrite, extract, simple_qa]

  - id: coding
    provider: anthropic
    model: claude-opus-4.8
    cost_weight: 1.0
    capabilities: [coding, debugging, security]

  - id: reasoning
    provider: openai
    model: gpt-5.5
    cost_weight: 1.0
    capabilities: [math, planning, synthesis]

  - id: science
    provider: google
    model: gemini-3.1-pro
    cost_weight: 1.0
    capabilities: [science, long_context, multimodal]
```

模型名需要按你实际 provider 替换。文档里的 frontier 名称来自公开 Fugu 报告和发布页，不代表你一定能访问。

## 4. TraceStore 表结构

SQLite 起步：

```sql
CREATE TABLE requests (
  request_id TEXT PRIMARY KEY,
  created_at TEXT NOT NULL,
  input_hash TEXT NOT NULL,
  task_type TEXT,
  risk_level TEXT,
  strategy TEXT,
  final_reward REAL,
  final_status TEXT
);

CREATE TABLE model_calls (
  call_id TEXT PRIMARY KEY,
  request_id TEXT NOT NULL,
  worker_id TEXT NOT NULL,
  prompt_tokens INTEGER,
  completion_tokens INTEGER,
  cost_usd REAL,
  latency_ms INTEGER,
  status TEXT,
  output_ref TEXT,
  FOREIGN KEY(request_id) REFERENCES requests(request_id)
);

CREATE TABLE workflow_steps (
  step_id TEXT PRIMARY KEY,
  request_id TEXT NOT NULL,
  workflow_id TEXT NOT NULL,
  step_index INTEGER NOT NULL,
  worker_id TEXT NOT NULL,
  subtask TEXT NOT NULL,
  access_json TEXT NOT NULL,
  reward REAL,
  FOREIGN KEY(request_id) REFERENCES requests(request_id)
);
```

## 5. Workflow JSON Schema

```json
{
  "type": "object",
  "required": ["steps", "final"],
  "properties": {
    "steps": {
      "type": "array",
      "minItems": 1,
      "maxItems": 5,
      "items": {
        "type": "object",
        "required": ["id", "worker", "subtask", "access"],
        "properties": {
          "id": {"type": "integer", "minimum": 1, "maximum": 5},
          "worker": {"type": "string"},
          "subtask": {"type": "string", "minLength": 20, "maxLength": 1000},
          "access": {
            "type": "array",
            "items": {"type": "integer", "minimum": 1, "maximum": 5}
          }
        }
      }
    },
    "final": {
      "type": "object",
      "required": ["worker", "subtask", "access"],
      "properties": {
        "worker": {"type": "string"},
        "subtask": {"type": "string"},
        "access": {
          "type": "array",
          "items": {"type": "integer", "minimum": 1, "maximum": 5}
        }
      }
    }
  }
}
```

用 vLLM structured outputs 或 provider 的 JSON schema constrained decoding 可以显著提高格式合法率；但仍然要在服务端二次验证，因为模型输出不能作为权限边界。

## 6. Router 训练最小代码形态

第一版可以不用微调 LLM，用 embedding + sklearn。

```python
from sklearn.linear_model import LogisticRegression
from sklearn.metrics import roc_auc_score

X_train = task_embeddings
y_train = strong_needed_labels

clf = LogisticRegression(max_iter=1000, class_weight="balanced")
clf.fit(X_train, y_train)

probs = clf.predict_proba(X_valid)[:, 1]
print("auc", roc_auc_score(y_valid, probs))
```

线上策略：

```python
def route(prob_strong_needed, threshold):
    if prob_strong_needed >= threshold:
        return "strong_direct"
    return "cheap_direct"
```

## 7. Selection-head 训练最小形态

先用 embedding/MLP 验证 worker specialization，再做 small LM hidden state。

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class WorkerSelector(nn.Module):
    def __init__(self, input_dim, num_workers):
        super().__init__()
        self.net = nn.Sequential(
            nn.Linear(input_dim, 512),
            nn.ReLU(),
            nn.Linear(512, num_workers),
        )

    def forward(self, x):
        return self.net(x)

def kl_loss(logits, target_probs):
    log_probs = F.log_softmax(logits, dim=-1)
    return F.kl_div(log_probs, target_probs, reduction="batchmean")
```

训练目标：

```text
input embedding -> worker logits
target -> soft reward distribution over workers
```

## 8. Workflow SFT 训练

使用 instruction tuning 格式：

```json
{
  "messages": [
    {
      "role": "system",
      "content": "You are an orchestration model. Return only valid JSON workflow."
    },
    {
      "role": "user",
      "content": "Task: ... Workers: ... Constraints: max 3 steps."
    },
    {
      "role": "assistant",
      "content": "{\"steps\":[...],\"final\":{...}}"
    }
  ]
}
```

训练前先做 data validation：

```python
for sample in sft_samples:
    workflow = json.loads(sample["messages"][-1]["content"])
    validate_workflow_schema(workflow)
```

如果 SFT 数据本身不合法，RL 只会放大混乱。

## 9. GRPO/RL 训练接口

TRL 的 GRPOTrainer 支持 reward functions。对 workflow model，reward function 的实际计算会很重：

```python
def reward_func(prompts, completions, **kwargs):
    rewards = []
    for prompt, completion in zip(prompts, completions):
        try:
            workflow = parse_and_validate(completion)
        except Exception:
            rewards.append(0.0)
            continue
        result = execute_workflow_in_sandbox(prompt, workflow)
        rewards.append(compute_reward(result))
    return rewards
```

生产建议：

- 先用小任务和模拟 worker。
- 再用离线 replay。
- 最后才接真实昂贵 worker。
- 每个 batch 严格预算。

## 10. CMA-ES/pycma 训练接口

适合优化小参数 policy，例如 worker selection head 的最后一层或策略阈值。

```python
import cma
import numpy as np

def evaluate(theta):
    rewards = []
    for task in eval_tasks:
        worker = policy_select(theta, task)
        result = run_task_with_worker(task, worker)
        rewards.append(result.reward)
    return -float(np.mean(rewards))  # cma minimizes

es = cma.CMAEvolutionStrategy(x0=np.zeros(num_params), sigma0=0.1)
while not es.stop():
    solutions = es.ask()
    losses = [evaluate(x) for x in solutions]
    es.tell(solutions, losses)
    es.disp()
```

注意：

- 真实 worker 调用很贵，要先做模拟和 replay。
- 每个候选 theta 需要多任务平均，降低噪声。
- 参数数量要小，否则评估预算爆炸。

## 11. 最小端到端验证

第一版只需要证明这三件事：

1. Router 在 200-500 个任务上能降低成本且质量不明显下降。
2. Selection-head 在 held-out tasks 上选择 worker 的 reward 高于规则 router。
3. Workflow generator 输出合法率高，并在少量复杂任务上超过 strong_direct。

如果这三件事不成立，先不要扩展到更大模型和 RL。

