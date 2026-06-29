# 01. Fugu 深度解析：它到底训练了什么

## 1. Fugu 的核心定义

Sakana Fugu Technical Report 把 Fugu 描述为一族 **learned LLM orchestrators**：用户像调用一个模型一样调用 Fugu，但内部它会根据请求动态组织一个 LLM worker team。

公开资料支持的关键点：

- Fugu 本身是语言模型，不是纯规则 router。
- 它面向一个 worker model pool 做编排。
- 它可以选择 worker、构造 agentic scaffold、分配子任务、组合或验证输出。
- Fugu 有两个公开产品取向：
  - **Fugu**：低延迟，主要做单 worker selection。
  - **Fugu Ultra**：高质量，使用多 agent workflow，牺牲更多延迟。

需要谨慎的地方：

- 公开报告没有给出完整训练数据、训练代码和线上执行器。
- 报告中的 worker 包含当时的 frontier models，但具体线上版本和策略可能变化。
- 我们能复现的是 **Fugu-like design pattern**，不是 Sakana 内部模型。

## 2. Fugu 低延迟版：选择 worker，而不是生成答案

Fugu 低延迟版最重要的设计是：orchestrator 不负责最终回答，而是快速决定“这个请求交给哪个 worker 模型”。

公开报告中的结构可以抽象成：

```text
input text
  -> orchestrator backbone LM
  -> hidden state h
  -> lightweight selection head
  -> logits over worker models
  -> select worker
  -> worker generates answer
```

这和常见 prompt router 的差异在于：

| 方案 | 决策依据 | 是否训练 | 延迟 |
|---|---|---|---|
| 手写规则 router | 关键词、任务类型、用户 tier | 否 | 很低 |
| 强模型 prompt router | 让 LLM 读请求后输出模型名 | 可少量 prompt 调参 | 中等，需生成文本 |
| Fugu-like selection head | 小/中型 LM hidden state + 训练过的 head | 是 | 低，理论上可少解码或不解码 |

Fugu 报告明确说它使用 orchestrator logits，而不是依赖生成文本。这是降低延迟的关键：如果只是让 GPT/Claude 临时判断该选谁，就会多一次完整 LLM 调用。

## 3. Fugu 训练为什么分两段

Fugu 低延迟路线公开了两个训练阶段：

### 3.1 单步任务 SFT

先构建可验证任务集合 `D`。对每个问题 `q_i`：

1. 让 worker pool 中每个模型 `M_j` 多次作答。
2. 用 ground truth 或 verifier 得到 reward。
3. 对每个 worker 求平均 reward。
4. 把 reward 向量用 softmax 变成 soft target distribution。
5. 训练 selection head 预测这个分布。

抽象公式：

```text
score_i = [avg_reward(worker_1), ..., avg_reward(worker_K)]
target_i = softmax(score_i / temperature)
loss = KL(target_i || orchestrator_policy(q_i))
```

为什么不用 hard label？

- 多个 worker 可能都能答对。
- 有些任务 worker 之间只差一点。
- soft distribution 能保留“第二好模型也可用”的信息。
- 这对线上 fallback 和成本约束更友好。

### 3.2 端到端任务上的演化优化

单步任务不能覆盖真实 agent 场景。Fugu 报告进一步使用 multi-turn end-to-end tasks，例如代码助手环境中的仓库上下文、工具调用、执行反馈、最终是否完成任务。

这类任务的问题是：

- reward 稀疏，通常只有最终成功/失败。
- worker 是闭源 API，不可微。
- 每次评估都要花真实推理成本。
- 任务成功依赖完整 trajectory，而不是单轮答案。

因此报告采用 sep-CMA-ES 这类黑盒演化策略，直接优化终端 reward。

工程含义：

> 当 worker 是闭源模型，且 reward 来自整个执行轨迹时，不要强行用普通监督学习拟合每一步。黑盒优化、bandit、离线 replay 和在线 A/B 更贴合问题。

## 4. Fugu Ultra：生成 workflow，而不是只选 worker

Fugu Ultra 建立在 Conductor 思路上。Conductor 不直接回答问题，而是输出一个自然语言 workflow。

一个 workflow step 至少包含：

```text
subtask: 给 worker 的自然语言子任务
worker_id: 分配给哪个模型
access_list: 这个 worker 能看到哪些前置步骤输出
```

这带来两个重要能力：

- 可以动态构造拓扑：best-of-N、链式、树状、debate、verify/refine。
- 可以做 prompt engineering：不是只说“让模型 A 回答”，而是给模型 A 一个针对性的子任务。

### 4.1 Conductor/Fugu Ultra 的 RL 信号

Conductor 和 Fugu Ultra 的公开设计都强调 reward 至少包括两层：

1. **格式可解析**：workflow 输出必须能被解析。
2. **任务正确性**：执行 workflow 后最终答案是否正确。

这点非常关键。训练 workflow model 时，模型不是只学“怎么写得像计划”，而是学“哪种计划执行后能赢”。

### 4.2 为什么自然语言 workflow 难但强

有限动作 head 的优点是快、稳定、容易验证。自然语言 workflow 的优点是表达力强。

| 维度 | Fugu-like selection head | Fugu-Ultra/Conductor-like workflow |
|---|---|---|
| 动作 | 选择一个 worker | 生成多步工作流 |
| 训练信号 | worker reward distribution + end-to-end reward | workflow format + execution reward |
| 优点 | 低延迟、工程可控 | 表达力强，适合复杂任务 |
| 缺点 | 无法自由拆任务 | 解析、训练、成本和安全更难 |
| 适用 | 日常请求、coding turn、科学问答路由 | 深度研究、复杂代码任务、工具长链 |

## 5. Fugu Ultra 的关键工程扩展：状态和隔离

Fugu 报告特别强调 function calling multi-agent workflow 会带来 memory 问题。

原因是：在单 agent 系统里，工具调用 transcript 自然属于同一个 agent。但在多 agent workflow 里，任何 agent 都可能调用工具。系统必须知道：

- 哪个 agent 发起了调用。
- 它属于哪个 workflow step。
- 工具结果应该回给哪个 agent。
- 其他 agent 是否能看到这个工具结果。

Fugu Ultra 的公开设计包含两个相互制衡的原则：

1. **intra-workflow agent isolation**
   - 当前 workflow 内，agent 不能无条件看到其他 agent 的工具轨迹。
   - 防止第一个 agent 的路径污染后续 agent，导致所有模型沿同一路线重复。

2. **inter-workflow shared memory**
   - 跨 workflow/多轮对话，agent 可以继承已有环境背景。
   - 防止重复探索同一个文件、网页、实验结果。

工程上可落成：

```text
workflow_state:
  workflow_id
  steps[]
  agent_private_transcripts[step_id]
  shared_memory
  access_edges
  tool_call_routes
```

没有这层状态，Fugu-like 系统会退化成普通多模型聊天。

## 6. Fugu 不是什么

### 6.1 不是简单 OpenRouter/LiteLLM 网关

网关解决统一调用和日志。Fugu 解决的是“如何学习组织多个 worker”。

### 6.2 不是简单 Mixture-of-Agents

MoA 是固定层状结构。Fugu Ultra 的公开目标是 query-adaptive workflow：不同任务可用不同拓扑。

### 6.3 不是 prompt-only coordinator

TRINITY 和 Fugu 报告都强调训练过的 coordinator/orchestrator。单靠“请你选择最合适模型”这种 prompt，通常不能稳定学习每个 worker 在真实任务中的能力边界。

### 6.4 不是参数合并

Fugu 是 behavioral-level composition：把 closed-source frontier models 当黑盒 worker 组合，不需要访问它们的权重。

## 7. 复现目标应该怎么定

不要把目标写成：

```text
复刻 Sakana Fugu
```

应该写成：

```text
训练一个 Fugu-like orchestrator：
  在统一入口下，根据任务、成本、延迟和风险，
  动态选择模型或生成 workflow，
  并通过真实任务 reward 持续优化。
```

第一版成功标准：

- 在内部任务集上，router/cascade 比固定强模型成本更低。
- 在复杂任务上，workflow/fusion 比 strong-direct 更高质量。
- 能解释每次请求为什么选了某个策略和 worker。
- 能记录轨迹并用于下一轮训练。

