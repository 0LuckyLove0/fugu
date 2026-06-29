# 04. Fugu-like 训练 Playbook

## 总览

目标不是一步训练出 Fugu，而是分阶段把一个统一入口训练成 orchestrator。

```text
Stage 0: 可观测网关
Stage 1: RouteLLM-like router
Stage 2: Fugu/TRINITY-like selection head
Stage 3: Conductor-like workflow generator
Stage 4: 端到端优化和线上闭环
```

每阶段都有明确输入、训练目标、产物和验收标准。

## Stage 0：先做可观测网关，不训练模型

### 目标

把所有模型调用统一记录下来，为训练准备数据。

### 你要实现

```text
POST /v1/chat/completions
  -> normalize request
  -> classify task profile
  -> choose strategy by rule
  -> call provider
  -> record trace
  -> return response
```

### 最小策略

```yaml
rules:
  - if task_type in ["rewrite", "extract", "classify"]:
      strategy: cheap_direct
  - if risk_level == "high":
      strategy: strong_direct
  - if task_type in ["deep_research", "architecture_review"]:
      strategy: fusion_panel
  - else:
      strategy: strong_direct
```

### 验收

- 每次请求都有 trace。
- 每个模型调用都有 cost/latency/status。
- 可以离线 replay 同一批任务。
- 可以按 task_type 统计成本。

如果 Stage 0 没做好，不要进入训练。

## Stage 1：训练 RouteLLM-like router

### 目标

回答一个问题：

```text
这个请求是否真的需要强模型？
```

### 数据构造

对每个任务同时跑 cheap model 和 strong model：

```json
{
  "query": "...",
  "cheap_reward": 0.8,
  "strong_reward": 0.9,
  "cheap_cost": 0.01,
  "strong_cost": 0.20
}
```

构造 label：

```python
strong_margin = strong_reward - cheap_reward
label_strong_needed = strong_margin > 0.10
```

### 模型选择

第一版不用大模型：

- embedding + logistic regression。
- small transformer classifier。
- fine-tuned 1B-3B classifier。

### 决策阈值

Router 不应该输出一个死结论，而应该输出概率：

```text
p_strong_needed = router(query)
```

不同用户或任务用不同阈值：

```yaml
free_user: 0.75
paid_user: 0.50
high_risk: 0.20
```

### 验收

画成本-质量曲线：

```text
x-axis: average cost
y-axis: reward / pass rate
```

router 必须优于：

- always cheap。
- always strong。
- random routing。
- hand-written rules。

如果没有优于 strong baseline 或成本下降很小，不要进入 Stage 2。

## Stage 2：训练 Fugu/TRINITY-like selection head

### 目标

从“强/弱二选一”升级为“在 worker pool 中选择最合适 worker”。

### 训练样本

对每个任务 `q_i`，让每个 worker 多次作答并打分：

```json
{
  "query": "solve this GPQA question...",
  "worker_scores": {
    "gpt": 0.90,
    "gemini": 1.00,
    "opus": 0.70,
    "cheap": 0.40
  }
}
```

转成 soft target：

```python
def soft_target(scores, tau=0.2):
    values = np.array(list(scores.values()))
    probs = np.exp(values / tau)
    probs = probs / probs.sum()
    return dict(zip(scores.keys(), probs.tolist()))
```

### 模型结构

一个现实版本：

```text
backbone: Qwen2.5-0.5B / Qwen3-0.6B / Gemma small
input: user query + optional short context
hidden_state: final token or special <route> token
head: linear(hidden_dim -> num_workers)
loss: KL(target_distribution || predicted_distribution)
```

### 为什么先不用生成文本

如果让 orchestrator 生成：

```text
I choose model X because...
```

就多了完整解码延迟。Fugu 的低延迟路线强调从 hidden state/head logits 直接做选择。

### 简化实现

你不一定要先改 transformer 内部。第一版可以：

1. 用 embedding model 编码 query。
2. 训练 MLP 输出 worker logits。
3. 如果有效，再迁移到 small LM hidden state + head。

### 端到端优化

SFT 后，做 expensive end-to-end optimization：

```text
candidate policy theta
  -> run N real tasks
  -> compute terminal reward
  -> update theta with sep-CMA-ES / bandit / Bayesian optimization
```

为什么可行：

- worker 是黑盒 API。
- reward 稀疏。
- 评估成本高。
- selection head 参数可以做得很少。

### 验收

- 在 held-out tasks 上超过 best single worker 或接近 best oracle。
- 平均延迟接近 direct strong model。
- 不同领域有合理路由分布。
- worker 宕机或禁用时能 fallback。

## Stage 3：训练 Conductor-like workflow generator

### 目标

让 orchestrator 输出可执行 workflow：

```json
{
  "steps": [
    {
      "id": 1,
      "worker": "model_a",
      "subtask": "derive an algorithm",
      "access": []
    },
    {
      "id": 2,
      "worker": "model_b",
      "subtask": "implement the algorithm",
      "access": [1]
    }
  ]
}
```

### 为什么不能直接 RL

直接 RL 的问题：

- 格式经常错。
- 执行 workflow 很贵。
- reward 稀疏。
- 安全边界难控制。

正确顺序：

```text
1. 定义严格 JSON schema
2. 用规则/专家/强模型生成 teacher workflows
3. SFT 让模型学会输出合法 workflow
4. 离线执行 workflows，打 reward
5. 用 GRPO/PPO 做小步 RL
6. 只在离线评测胜出后灰度
```

### SFT 数据

样本：

```json
{
  "prompt": "Task: Fix a flaky parser test. Workers: ... Constraints: max 3 steps.",
  "completion": {
    "steps": [
      {
        "id": 1,
        "worker": "gpt",
        "subtask": "Inspect failure logs and isolate the parser behavior.",
        "access": []
      },
      {
        "id": 2,
        "worker": "opus",
        "subtask": "Propose a minimal patch and identify edge cases.",
        "access": [1]
      }
    ]
  }
}
```

### RL reward

```python
reward = 0.0
if valid_schema:
    reward += 0.2
if within_budget:
    reward += 0.1
if final_answer_correct:
    reward += 0.7
if policy_violation:
    reward -= 1.0
```

训练初期，格式和预算 reward 要明确。否则模型可能学会“调用所有强模型、多写步骤”来提高质量。

### GRPO 训练形态

每个 task 采样多个 workflow：

```text
task q
  -> workflow_1 reward 0.2
  -> workflow_2 reward 1.0
  -> workflow_3 reward 0.0
  -> workflow_4 reward 0.7
```

GRPO 用同组样本的相对表现计算 advantage，适合 verifiable reward 场景。TRL 提供 GRPOTrainer，可以接自定义 reward function；实际使用时要把 workflow 执行成本控制住。

### 验收

- JSON/schema 合法率 > 99%。
- 平均 step 数可控。
- 在复杂任务上超过 strong_direct 和固定 MoA/Fusion baseline。
- 失败时能定位是格式、路由、worker、工具还是合成问题。

## Stage 4：组合成 Fugu-like 系统

### 策略选择层

最终不应所有请求都进 workflow。

```python
def choose_strategy(profile):
    if profile.privacy_level == "restricted":
        return "approved_local_model"
    if profile.task_type in {"rewrite", "extract", "classify"}:
        return "cheap_or_router"
    if profile.latency_budget_ms < 5000:
        return "selection_head"
    if profile.task_type in {"deep_research", "complex_coding", "scientific_reasoning"}:
        return "workflow_orchestrator"
    return "router"
```

### 双路线结构

```text
Fugu-like low latency:
  task -> selection head -> one worker -> answer

Fugu-Ultra-like high quality:
  task -> workflow generator -> executor -> workers/tools -> verifier -> answer
```

### 线上持续学习

上线后每周做：

1. 抽样失败请求。
2. 给失败贴标签。
3. 对同一任务 replay 新模型/新策略。
4. 更新 router 数据集。
5. 更新 hard benchmark。
6. 小步训练或阈值调整。
7. 先离线回放，再灰度上线。

## 5. 不建议做的事

- 不要一开始训练 7B workflow model。
- 不要所有请求都走多 agent。
- 不要用 LLM judge 代替所有 ground truth。
- 不要让 workflow 自由调用任意工具。
- 不要让模型自己决定是否违反预算。
- 不要用公开 benchmark 分数替代内部任务评测。

## 6. 最短可行路线

如果只想 6 周内看到效果：

```text
Week 1: 统一网关 + TraceStore
Week 2: 500 个内部任务多 worker replay
Week 3: 训练 cheap/strong router
Week 4: 定义 workflow schema + 强模型生成 teacher workflows
Week 5: 执行 workflow，构建 100-300 个高质量 traces
Week 6: 训练 workflow SFT baseline，和 strong_direct/Fusion/Fugu 做离线对比
```

这个路线不会立刻得到真正 Fugu，但能证明你的任务分布里是否存在：

- 可降本空间。
- worker specialization。
- workflow 提质空间。
- 足够可靠的 reward。

