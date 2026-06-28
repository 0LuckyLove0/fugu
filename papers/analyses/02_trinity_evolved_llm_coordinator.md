# TRINITY: An Evolved LLM Coordinator 深度解析

## 基本信息

- 论文：TRINITY: An Evolved LLM Coordinator
- arXiv：2512.04695
- 本地 PDF：`papers/pdfs/trinity_evolved_llm_coordinator_2512.04695.pdf`
- 本地文本：`papers/text/trinity_evolved_llm_coordinator_2512.04695.clean.txt`
- 页数：30
- 定位：Fugu 低延迟路由/选择能力的重要基础。

## 一句话结论

TRINITY 证明了一个极轻量的小模型 coordinator，通过演化策略学习“选哪个 agent、让它扮演什么角色”，可以稳定超过单模型和多种手工多 agent baseline。

## 研究问题

多模型系统常见问题是：

- 静态 MoA 或 debate workflow 太固定。
- 手写 scaffold 依赖人类经验。
- 让大模型临时充当 coordinator 不稳定。
- RL 或 imitation learning 训练多步 agent selector 成本高且容易不稳定。

TRINITY 的问题定义是：

> 能否用一个很小的 coordinator，在每一步根据上下文选择 agent 和 role，从而以较低训练参数和较高效率获得多模型协作收益？

## 方法设计

### Coordinator 形态

TRINITY 使用 Qwen3-0.6B 作为 coordinator 的 SLM，并在其 hidden state 上加一个轻量 head。论文提到 learnable parameters 少于 20K。

每一轮 coordinator 输出的不是自由文本，而是有限 action：

```text
action = agent selection + role selection
```

也就是：

- 选择哪个 LLM agent。
- 让它以什么角色行动。

### 三种角色

TRINITY 定义三类角色：

| 角色 | 作用 |
|---|---|
| Thinker | 规划、分解任务、设定策略 |
| Worker | 执行具体推理、代码、计算、回答 |
| Verifier | 检查、验证、纠错、给后续轮次提供反馈 |

这个设计很重要，因为它说明“模型选择”不够。即使是同一个模型，作为 planner、coder、verifier 的提示和职责不同，效果也不同。

### 训练方法

TRINITY 的核心是使用 sep-CMA-ES，即 separable Covariance Matrix Adaptation Evolution Strategy。

论文强调：

- 多轮 coordinator 的 reward 来自最终任务结果。
- 每一步都要调用 agent，训练代价高。
- 传统 RL、imitation learning 或随机搜索不一定高效。
- sep-CMA-ES 在低维、结构化、block-diagonal 类搜索空间上更适合。

这使 TRINITY 成为一种“演化出来的 coordinator”，而不是靠人工写规则。

## 实验设置

论文在多个任务上训练和评估：

- Math500
- MMLU
- RLPR
- LiveCodeBench
- 以及若干 held-out benchmark

agent pool 包含 GPT、Gemini、Claude、Qwen 等不同类型模型。

baseline 包括：

- 单个 strongest agent。
- 随机 agent selection。
- self-reflection。
- MoA。
- Smoothie。
- prompted LLM as coordinator。
- 其他多 agent routing/scaffolding 方法。

## 实验结论

论文报告：

- TRINITY 在四个 in-distribution benchmarks 上稳定超过单模型和多模型 baseline。
- 在 LiveCodeBench Jan-Apr 2025 上达到 state-of-the-art，pass@1 86.2 +/- 0.5%。
- 在 held-out benchmarks 上也有泛化能力。
- ablation 显示 agent selection、role selection、上下文 hidden state 和训练算法都重要。
- prompted LLM as coordinator 明显弱于训练过的 TRINITY。

特别重要的是：

> TRINITY 不只是“选择最强模型”，而是学习何时让不同模型担任不同角色。

## 核心贡献

1. **把多 agent 编排压缩成小 coordinator**
   - 不是每次都让大模型思考怎么协调。
   - 小模型做选择，强模型做任务。

2. **role selection 证明了“职责分配”的价值**
   - 多模型系统不是模型列表问题，而是组织结构问题。

3. **演化策略适合训练黑盒多模型系统**
   - 许多 agent 调用不可微。
   - reward 稀疏。
   - 演化策略能直接优化终局表现。

4. **训练过的 coordinator 优于 prompting coordinator**
   - 这点对企业很关键：不要以为 prompt 一个强模型“请选择最佳模型”就足够。

## 局限

1. **action space 有限**
   - TRINITY 选择固定 agent 和固定 role。
   - 不像 Conductor 那样自由生成自然语言 workflow。

2. **训练仍然昂贵**
   - 每次评估需要真实调用多个 agent。
   - agent pool 越大，训练成本越高。

3. **主要面向可验证 benchmark**
   - 对开放研究、长任务、工具调用复杂场景，还需要进一步扩展。

4. **角色模板仍是人工设计**
   - Thinker/Worker/Verifier 是强 inductive bias，但不是完全自动发现。

## 与 Fugu 的关系

Fugu 低延迟版直接继承 TRINITY 的思想，但做了生产简化：

- Fugu 使用 selection head 选择 worker。
- Fugu 不分配三种 role，降低 latency 和复杂度。
- Fugu 更强调生产路由和交互延迟。

可以说：

> TRINITY 是 Fugu 低延迟 orchestrator 的研究原型。

## 对我们的启示

1. **第一代内部 orchestrator 可以不大**
   - 一个小模型或轻量 classifier 就能做很多路由工作。

2. **仅按任务类型路由不够**
   - 应该记录“模型在不同角色中的表现”：planner、executor、verifier、summarizer。

3. **演化/黑盒优化值得关注**
   - 当 API 模型不可微、reward 来自业务指标时，传统监督学习不一定足够。

4. **MVP 可先实现有限 action**
   - 从固定策略集合开始：direct、cheap、strong、verify、fusion。
   - 等数据足够后再学习更自由的 workflow。

---

## 深度版分析：TRINITY 真正解决了什么

### 1. 这篇论文的核心不是“多模型”，而是“可训练的组织决策”

很多多模型系统停留在“把几个模型并排调用，再做投票或汇总”。TRINITY 的研究问题更尖锐：如果多模型系统的价值来自“谁在什么时候做什么”，那么这个组织决策本身能不能被训练？

TRINITY 给出的答案是可以，但它选择了一个非常克制的表达空间：

```text
每一轮只做两个选择：
1. 选择哪个 agent
2. 选择让它扮演 Thinker / Worker / Verifier 中的哪个角色
```

这和 Conductor 的自然语言 workflow 形成鲜明对比。TRINITY 不让 coordinator 自由写复杂子任务，而是把编排压缩成一个有限动作空间。这个取舍非常重要：

- 好处：可优化、可评估、延迟低、行为更稳定。
- 代价：表达力弱，不能像 Conductor 那样自由设计通信拓扑和子 prompt。

所以 TRINITY 是一条“工程上更容易产品化”的路线：先不要让 coordinator 写一切，而是让它学会少数关键选择。

### 2. 三角色设计的真实意义

Thinker / Worker / Verifier 看似是简单 prompt role，实际是把多 agent 协作拆成三个抽象能力：

| 抽象能力 | 角色 | 系统意义 |
|---|---|---|
| 问题表征与计划 | Thinker | 决定问题怎么拆、后续该看什么 |
| 任务执行 | Worker | 产出候选解、代码、推理步骤 |
| 质量控制 | Verifier | 捕捉错误、纠偏、给后续轮次提供信号 |

这解释了为什么“模型选择”不够。一个模型在 Worker 位置强，不代表在 Verifier 位置强。比如代码任务里，某模型可能很会写代码，但不擅长发现自己或别人代码里的边界条件；另一个模型可能生成能力一般，但审查能力更稳。

对企业系统的启发是：模型能力标签不应只写“coding/math/reasoning”，还要写“planner/executor/verifier/summarizer/retriever”等协作角色能力。

### 3. 为什么使用演化策略，而不是普通监督学习

论文强调 sep-CMA-ES 的原因，不只是“用了一个高级优化算法”。多 agent 编排有几个训练难点：

1. **目标不可微**
   - 底层 agent 可能是闭源 API。
   - 最终 reward 来自 benchmark 是否答对。
   - 中间调用和输出无法反向传播。

2. **reward 稀疏**
   - 一个五轮协作最后答错，中间哪一步错很难归因。
   - 普通 policy gradient 方差可能很大。

3. **真实调用昂贵**
   - 每评估一个 coordinator 参数，都要调用多个 LLM。
   - 训练算法必须样本效率高。

4. **搜索空间结构化**
   - head 参数很少。
   - agent/role logits 近似有块结构。
   - sep-CMA-ES 的 diagonal/separable 假设适合这种低维结构化搜索。

因此，TRINITY 的工程含义是：当系统由闭源 LLM API 组成时，不要只盯着神经网络反向传播；黑盒优化、演化策略和 bandit 类方法可能更适合训练 orchestrator。

### 4. 论文实验该怎么读

TRINITY 报告了几个重要结果，但解读时要区分三层含义：

#### 4.1 超过单模型：证明“组合有价值”

论文显示 TRINITY 在 Math500、MMLU、RLPR、LiveCodeBench 等任务上超过单个 agent。这个结果说明 agent pool 里确实存在互补性。

但不能简单理解成“多模型必胜”。更准确是：

```text
当不同模型在不同任务/阶段有互补能力时，训练过的 coordinator 能把这种互补性转化为最终质量。
```

#### 4.2 超过手工多 agent baseline：证明“学习组织”有价值

如果 TRINITY 只超过单模型，不足以说明训练 coordinator 必要。关键是它还超过了 MoA、self-reflection、Smoothie 等多 agent baseline。这说明固定 workflow 无法充分适配不同任务。

#### 4.3 超过 prompted LLM coordinator：证明“强模型临时当调度员”不够

论文中 prompted Gemini 2.5 Pro as Coordinator 明显弱于 TRINITY。这一点对落地很关键。很多工程团队第一反应是：

> 让一个强模型看一下任务，然后选择模型，不就行了？

TRINITY 的结果说明，这种 prompt-only coordinator 可能无法稳定掌握各 agent 的真实能力边界。真正可靠的 coordinator 需要通过任务结果训练或长期反馈学习。

### 5. Ablation 的关键含义

TRINITY 的 ablation 不是简单证明每个组件都有用，而是在回答“收益来自哪里”：

| Ablation 方向 | 说明的问题 |
|---|---|
| 去掉 agent selection | 如果固定模型，只换 role，收益下降，说明模型互补性重要 |
| 去掉 role selection | 如果只选模型不分工，收益下降，说明职责分配重要 |
| 换 token hidden state | hidden representation 里确实包含可路由信息 |
| 换训练算法 | sep-CMA-ES 对该结构更合适 |
| prompted LLM coordinator | prompt-only 调度不够稳定 |

这些结果共同支持一个判断：

> 多模型系统的质量来自 agent choice、role choice、context representation、training algorithm 的共同作用，而不是单点技巧。

### 6. 和 RouteLLM 的本质区别

RouteLLM 是 query-level binary router：

```text
这个请求给强模型还是弱模型？
```

TRINITY 是 turn-level multi-agent role router：

```text
这一轮让哪个 agent 以什么角色参与协作？
```

二者区别很大：

- RouteLLM 优先解决成本。
- TRINITY 优先解决质量和协作。
- RouteLLM 每次通常只调用一个模型。
- TRINITY 允许多轮调用多个模型。

企业落地时，二者可以组合：先用 RouteLLM 式 router 判断是否需要复杂编排；只有复杂任务才进入 TRINITY/Fugu-like coordinator。

### 7. 和 Conductor 的本质区别

TRINITY 与 Conductor 的差异可以理解为“有限动作 vs 自然语言程序”：

| 维度 | TRINITY | Conductor |
|---|---|---|
| 编排表达 | agent + role logits | 自然语言 workflow |
| 优化方法 | 演化策略 | 强化学习 |
| 表达力 | 低到中 | 高 |
| 稳定性 | 更高 | 依赖格式/解析/执行 |
| 延迟 | 更低 | 更高 |
| 可产品化难度 | 相对低 | 相对高 |

Fugu 的两个版本正好映射这两个方向：Fugu 更像 TRINITY，Fugu-Ultra 更像 Conductor。

### 8. 论文对我们系统设计的直接推导

如果我们要借鉴 TRINITY，不应该一开始训练 0.6B coordinator。更现实的路线是：

1. **定义有限动作空间**
   - `cheap_direct`
   - `strong_direct`
   - `planner_then_executor`
   - `executor_then_verifier`
   - `fusion_panel`
   - `human_review`

2. **给每个动作记录结果**
   - 成本。
   - 延迟。
   - 用户是否接受。
   - 自动评测分。
   - 人工评分。

3. **先用规则或轻量模型路由**
   - 不急着训练大模型。
   - 用 task type、长度、风险、历史失败率作为 features。

4. **再用黑盒优化调策略**
   - 可以从 contextual bandit 或 Bayesian optimization 开始。
   - 当 action space 扩大后再考虑 evolution strategy。

5. **最后才训练 coordinator**
   - 前提是已有足够任务-动作-结果日志。

### 9. 领导可能追问

**问：TRINITY 这么强，为什么不直接复现？**

答：论文环境中有固定 agent pool、固定 benchmark、可验证 reward。企业场景更复杂：任务分布变、模型价格变、合规限制变、反馈延迟长。我们应先复现思想，即“学习组织决策”，而不是直接复现算法。

**问：为什么不用强模型直接当 coordinator？**

答：论文显示 prompted LLM coordinator 弱于训练过的 TRINITY。原因是强模型并不天然知道每个 agent 在每个任务和角色上的真实表现；这些知识需要从结果数据中学习。

**问：TRINITY 对降本有帮助吗？**

答：它主要是提质和提高协作效率，不是纯降本。降本应先看 RouteLLM/FrugalGPT。TRINITY 的降本价值在于用小 coordinator 做决策，避免每一步都让最贵模型思考调度。
