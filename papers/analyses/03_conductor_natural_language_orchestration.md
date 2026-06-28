# Learning to Orchestrate Agents in Natural Language with the Conductor 深度解析

## 基本信息

- 论文：Learning to Orchestrate Agents in Natural Language with the Conductor
- arXiv：2512.04388
- 本地 PDF：`papers/pdfs/conductor_natural_language_orchestration_2512.04388.pdf`
- 本地文本：`papers/text/conductor_natural_language_orchestration_2512.04388.clean.txt`
- 页数：39
- 定位：Fugu-Ultra 深度多 agent workflow 编排的核心基础。

## 一句话结论

Conductor 把“多 agent workflow”表示成自然语言，让一个 7B 模型通过强化学习学会拆任务、写子提示、分配 agent、设置通信拓扑，从而超过手写多 agent scaffold 和单模型 baseline。

## 研究问题

传统多 agent 方法常常有三个限制：

1. workflow 是人手写的，例如固定 debate、固定 MoA、固定反思。
2. router 只能从有限模板中选，不能自由生成新协作方式。
3. 大模型虽然能写 prompt，但没有经过 end-to-end reward 训练，不一定稳定。

Conductor 的问题是：

> 能否训练一个模型，让它直接以自然语言输出完整 agentic workflow，并通过最终任务 reward 学会更强的组织策略？

## 方法设计

### Conductor 输出什么

Conductor 不直接回答用户问题，而是输出一个 workflow。每个 workflow step 包含：

- assigned worker agent：这一轮由哪个模型执行。
- subtask instruction：给该模型的自然语言子任务提示。
- access / visibility：该 agent 能看到哪些前序 agent 的输出。

这相当于把自然语言变成 orchestration DSL。

### 为什么自然语言重要

自然语言 workflow 比固定 action space 更有表达力：

- 可以动态拆分任务。
- 可以给不同 agent 写针对性 prompt。
- 可以设计串行、树状、验证、聚合等拓扑。
- 可以根据问题难度分配更多或更少 compute。
- 可以把 prompt engineering 本身纳入学习对象。

### 训练方式

论文训练一个 7B Conductor，使用强化学习：

- Conductor 生成 workflow。
- 系统解析 workflow。
- 按 workflow 调用 worker LLM。
- 最终输出与可验证答案比较。
- reward 同时考虑格式是否可解析和答案是否正确。

论文还提到：

- 使用 randomized agent pools 进行短期 finetuning，使 Conductor 能适应不同可用模型集合。
- 允许 Conductor 把自己作为 worker，形成 recursive topology。
- 限制递归深度，避免无限循环。

## 实验结论

论文报告 Conductor 在多个高难 benchmark 上表现强：

- LiveCodeBench
- GPQA Diamond
- AIME
- MATH500
- BigCodeBench
- Medreason 等

主要结论：

- 7B Conductor 能超过所有 worker agent 和多种 self-reflection/multi-agent baseline。
- 与 MASRouter、Smoothie 等固定/半固定多 agent 方法相比，Conductor 更强。
- Conductor 的平均 agent call 数更低，说明它不是靠粗暴堆调用次数获胜。
- 3B Conductor 可以学到 agent selection，但 7B 的 prompt engineering 能力更强。
- Conductor 会对复杂任务分配更多 agent，对简单 factual recall 任务使用更简单 workflow。

## 典型 emergent strategies

论文观察到多类自然涌现的编排策略：

1. **Sequential planner-executor**
   - 先让一个模型规划，再让另一个模型执行。

2. **Tree topology**
   - 多个 agent 独立处理不同部分，再由后续 agent 聚合。

3. **Verifier / validator**
   - 让一个模型检查前一个模型的输出。

4. **Task abdication**
   - Conductor 识别到某个强模型更适合做 planner，于是把自己的部分编排职责交给它。

5. **Recursive revision**
   - Conductor 看完上一轮 workflow 结果后，决定是否调用额外 agent 修正。

这些策略说明：高质量编排不是固定模板，而是根据任务动态生成组织结构。

## 核心贡献

1. **自然语言作为 orchestration interface**
   - 比有限 action space 更灵活。
   - 也更容易解释和审计。

2. **把 prompt engineering 训练进模型**
   - Conductor 不只是选模型，还会写针对性子 prompt。

3. **把通信拓扑变成学习对象**
   - agent 之间谁能看见谁，不再固定。

4. **证明小型 orchestrator 可以放大大型 worker**
   - 7B Conductor 本身不一定比 worker 强，但能组织 worker 变强。

## 局限

1. **依赖可验证 reward**
   - 代码、数学、选择题更容易训练。
   - 开放式战略分析、商业判断、法律建议的 reward 更难定义。

2. **自然语言 workflow 可解析性是工程问题**
   - 格式错误会导致 workflow 无法执行。
   - 生产中需要 schema、parser、validator。

3. **延迟和成本仍然高**
   - 多 agent workflow 天然比 direct call 慢。
   - 适合复杂任务，不适合简单实时问答。

4. **工具和状态管理复杂**
   - agent 能看到哪些上下文、工具结果如何隔离，是生产系统难点。

## 与 Fugu 的关系

Fugu-Ultra 基本沿着 Conductor 思路发展：

- 使用自然语言 workflow。
- 使用 RL 训练 orchestrator。
- 支持多 agent、多步骤、复杂通信。
- 在生产中进一步加入 workflow state、agent isolation 和 memory。

可以说：

> Conductor 是 Fugu-Ultra 的研究底座。

## 对我们的启示

1. **自然语言 workflow 是可行方向**
   - 内部 orchestrator 不必一开始训练模型，但可以先定义 workflow schema。

2. **prompt 不是外围配置，而是编排核心**
   - 给哪个 agent、写什么子 prompt、看哪些上下文，是系统质量关键。

3. **复杂任务要允许更多 compute**
   - Conductor 学到“难题多调用，简单题少调用”，这应成为企业系统的预算策略。

4. **从规则 workflow 到学习 workflow**
   - 第一阶段手写：planner -> executor -> verifier。
   - 第二阶段根据评测数据选择 workflow。
   - 第三阶段训练 Conductor-like orchestrator。

---

## 深度版分析：Conductor 为什么是 Fugu-Ultra 的关键基础

### 1. Conductor 的核心创新：让模型输出“组织方案”，而不是输出“答案”

传统 LLM 的输出是答案；Conductor 的输出是一个可执行的协作程序：

```text
workflow = [
  (agent_1, subtask_1, visibility_1),
  (agent_2, subtask_2, visibility_2),
  ...
]
```

这一步非常关键。它把 LLM 从“答题者”升级为“组织者”：

- 答题者直接解决问题。
- 组织者决定谁来解决、按什么顺序解决、每个人看到什么、最终如何收敛。

Fugu-Ultra 的本质就是这种组织者能力的产品化。

### 2. 为什么“自然语言 workflow”比固定模板更重要

固定多 agent 模板通常长这样：

```text
planner -> solver -> critic -> final
```

或者：

```text
model A, B, C 并行回答 -> aggregator 汇总
```

这种模式简单，但有两个问题：

1. 所有任务都被迫套同一种协作结构。
2. 子任务 prompt 是固定的，不能精确适配输入。

Conductor 的自然语言 workflow 允许模型动态写：

- “先让 Gemini 做数学推导。”
- “让 Claude 检查边界条件。”
- “让 Qwen 翻译中文输入并抽取结构。”
- “让 DeepSeek 用前两步计划写代码。”
- “如果上一轮看起来可疑，再调用自己重新规划。”

这不是简单 routing，而是动态 prompt engineering + topology design。

### 3. Conductor 训练任务的本质：训练“可执行计划生成器”

Conductor 的 RL 设置可以理解成：

```text
输入问题 q
Conductor 生成 workflow w
系统执行 workflow w
得到最终答案 a
根据 a 是否正确给 reward
更新 Conductor
```

它不是训练每个 worker 变强，而是训练 Conductor 生成更好的 workflow。worker 可以是闭源强模型，Conductor 只需要学会如何调用它们。

这使 Conductor 成为一种“系统层 scaling”：

- 不扩大 worker 参数。
- 不重新训练 frontier model。
- 通过更好的组织方式提升最终能力。

### 4. Reward 设计的工程含义

论文中 reward 至少包含两个层次：

1. **格式条件**
   - workflow 是否能被解析。
   - agent list、subtask list、visibility 是否符合格式。

2. **正确性条件**
   - workflow 执行后的最终答案是否正确。

这告诉我们一个落地原则：

> 编排器训练不能只看最终质量，还必须惩罚不可执行、不合规、超预算、越权调用等 workflow 级错误。

企业版 reward 应该扩展为：

| Reward 维度 | 示例 |
|---|---|
| 任务质量 | 答案评分、测试是否通过、人工是否接受 |
| 可执行性 | workflow schema 是否合法 |
| 成本 | 是否超出预算 |
| 延迟 | 是否满足 SLA |
| 合规 | 是否调用了 forbidden provider |
| 可审计性 | 是否保留引用和中间证据 |
| 稳定性 | 多次运行结果是否一致 |

### 5. Conductor 的实验结果应如何解读

论文报告 7B Conductor 超过 worker agent 和多种 multi-agent baseline。这个结果有三层含义：

#### 5.1 超过 worker：组织比个体更强

如果 Conductor 本身只是 7B，但能组织更强 worker 获得更好结果，说明系统能力不等于单个模型能力。

#### 5.2 超过 self-reflection：自我反思不如角色分工

很多 agent 系统用同一个模型反复反思。Conductor 的结果提示：让不同模型承担不同子任务，可能比让一个模型自我反思更有效。

#### 5.3 超过 MASRouter/Smoothie：自由 workflow 优于固定候选集合

MASRouter 类方法通常从预设 scaffolds 或 model choices 中选。Conductor 则直接生成自然语言子任务和通信拓扑，表达空间更大。这也是其性能提升的重要来源。

### 6. Conductor 的“任务难度自适应”很重要

论文观察到：

- 简单事实类任务：Conductor 可能只调用少量 agent。
- LiveCodeBench 等复杂任务：Conductor 会分配更多 agent 和步骤。
- 7B 比 3B 的优势不只在 agent selection，更在 prompt engineering。

这对企业系统意味着：

```text
预算不应固定，而应随任务复杂度动态分配。
```

一个实际设计可以是：

| 复杂度 | 策略 |
|---|---|
| low | cheap direct |
| medium | strong direct 或 cheap_then_verify |
| high | planner -> executor -> verifier |
| very high | Fusion / Conductor workflow |

### 7. Recursive Conductor 的意义

Conductor 允许把自己作为 worker，用于递归修正 workflow。这个设计容易被误解为“递归调用更强”。真正意义是：

- 第一次 workflow 执行后，Conductor 可以看到结果。
- 如果结果暴露出问题，它可以重新规划。
- 这相当于给 orchestrator 增加 test-time adaptation。

但生产中必须限制递归：

- 最大递归深度。
- 最大成本。
- 最大 wall-clock time。
- 失败降级。
- 循环检测。

否则很容易出现不可控成本和长尾延迟。

### 8. Conductor 与 MoA 的关键区别

MoA 是固定分层聚合：

```text
layer 1 proposers -> layer 2 aggregators -> final
```

Conductor 是动态 workflow：

```text
task -> custom subtasks + custom agents + custom visibility -> final
```

MoA 的优势是简单、稳定、容易实现；Conductor 的优势是适配性强、能做任务拆解、能让 agent 只看必要上下文。

因此企业落地可按顺序推进：

1. 先做 MoA/Fusion。
2. 再做固定 workflow。
3. 最后做 Conductor-like dynamic workflow。

### 9. Conductor 与 TRINITY 的关键区别

TRINITY 的 action space 是离散小集合，Conductor 的 action space 是自然语言。

这形成两个方向：

| 路线 | 优势 | 风险 |
|---|---|---|
| TRINITY-like | 稳定、低延迟、易训练 | 表达力有限 |
| Conductor-like | 表达力强、能 prompt engineer | 解析、成本、延迟、越权风险更高 |

Fugu 同时保留这两条线，说明产品化时需要分层：日常任务用低延迟选择，高价值任务才启用深度自然语言 workflow。

### 10. 论文局限的深层含义

Conductor 的实验主要依赖可验证任务，如代码、数学、选择题、标准答案 benchmark。这带来一个重要问题：

> 如果任务是战略分析、法律意见、市场研究，reward 怎么定义？

这不是论文缺陷，而是企业落地的最大挑战。

可行做法是：

- 对代码任务用测试/编译/benchmark。
- 对研究任务用 DRACO-like rubric。
- 对客服任务用人工接受率和后续追问率。
- 对业务决策任务用专家评分和风险 checklist。

换句话说，Conductor-like 系统真正的难点不只是训练算法，而是 reward engineering。

### 11. 对我们系统设计的直接推导

如果要借鉴 Conductor，建议先定义一个受控 workflow schema：

```yaml
workflow:
  max_steps: 5
  budget_usd: 1.00
  steps:
    - id: plan
      model: planner_model
      instruction: ...
      sees: [user_request]
    - id: solve
      model: executor_model
      instruction: ...
      sees: [user_request, plan]
    - id: verify
      model: verifier_model
      instruction: ...
      sees: [user_request, plan, solve]
```

先让规则或强模型生成 workflow，但必须通过 validator：

- schema 合法。
- provider 合规。
- 预算没超。
- 工具权限合法。
- 输出需要引用时必须保留来源。

之后再收集 workflow 结果，逐步训练或优化 orchestrator。

### 12. 领导可能追问

**问：Conductor 为什么不是普通 agent？**

答：普通 agent 自己做任务；Conductor 生成一个多模型协作计划，让其他模型执行。它的输出是 workflow，而不是答案。

**问：自然语言 workflow 会不会不可控？**

答：会，所以生产系统必须用 schema、parser、validator、budget guardrail 和权限控制。自然语言提供表达力，结构化执行层提供安全边界。

**问：Conductor 适合降本吗？**

答：它不是第一优先的降本方案。它适合高复杂任务提质。降本应先用 RouteLLM/FrugalGPT。Conductor 的成本收益来自“用更少但更准的 agent 步骤替代盲目多模型并行”。
