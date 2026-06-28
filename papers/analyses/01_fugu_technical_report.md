# Sakana Fugu Technical Report 深度解析

## 基本信息

- 论文：Sakana Fugu Technical Report
- arXiv：2606.21228
- 本地 PDF：`papers/pdfs/fugu_technical_report_2606.21228.pdf`
- 本地文本：`papers/text/fugu_technical_report_2606.21228.clean.txt`
- 页数：31
- 定位：把 TRINITY 和 Conductor 两条研究线产品化为一个可通过单一模型接口调用的多 agent 编排系统。

## 一句话结论

Fugu 的核心不是“再训练一个更大模型”，而是训练一个能够动态选择、组织、隔离、验证和合成多个强模型的 orchestrator，把多模型系统包装成单模型 API。

## 研究问题

论文试图回答的问题是：

1. 当不同 frontier LLM 在代码、科学、长上下文、工具使用等领域出现明显专长分化时，能否通过编排获得超过任一单模型的表现？
2. 能否把多 agent 系统的复杂性隐藏在一个 OpenAI-compatible model interface 后面？
3. 能否在低延迟和高质量之间提供两种不同产品形态？

这与企业落地高度相关，因为企业真正需要的是“每个任务用合适能力解决”，而不是所有任务都调用同一个最贵模型。

## 系统设计

Fugu 提供两个变体：

| 模型 | 核心目标 | 技术形态 |
|---|---|---|
| Fugu | 低延迟和日常任务 | 快速决策模块，通常每次选择一个 worker model |
| Fugu-Ultra | 高质量和复杂任务 | 基于 Conductor 的多步骤自然语言 workflow 编排 |

### Fugu：低延迟路由器

Fugu 的设计接近 TRINITY 的简化生产版本：

- 使用 orchestrator backbone 的 hidden state。
- 用轻量 selection head 输出 worker model logits。
- 只选择模型，不分配 Thinker/Worker/Verifier 角色。
- 不让 orchestrator 自己生成长文本，而是把任务交给选中的 worker。

这使它延迟更低，也让演化优化更可行。

### Fugu-Ultra：高质量编排器

Fugu-Ultra 更接近 Conductor：

- 编排器会生成多步骤 workflow。
- 每一步包含选用模型、子任务、通信/可见性关系。
- 适合 agentic coding、ML research、论文复现、安全分析等长链路任务。
- 论文讨论了 workflow state、agent isolation、persistent shared memory 等生产问题，目标是避免多个 agent 在工具调用和环境交互中互相污染。

## 训练范式

论文把 Fugu 的训练分成几类：

1. **大规模 SFT**
   - 基于每个 worker 在任务上的表现构建监督信号。
   - 不只学习“哪个模型最好”，而是学习选择分布。

2. **演化算法**
   - 对 Fugu 低延迟选择器进行 end-to-end 优化。
   - 目标从静态选择扩展到实际 agentic workflow 轨迹。

3. **强化学习**
   - 用于 Fugu-Ultra 的自然语言编排能力。
   - 继承 Conductor 思路，让 orchestrator 学会拆解、分配、沟通、验证。

## 实验与结果

论文报告 Fugu/Fugu-Ultra 在多类 benchmark 上表现强：

- SWE-Bench Pro
- Terminal Bench 2.1
- LiveCodeBench
- LiveCodeBench Pro
- GPQA-Diamond
- Humanity's Last Exam
- CharXiv Reasoning

论文的关键主张不是某个单项分数，而是：

> Fugu 能根据任务类型把不同 worker 的强项组合起来，在若干任务上达到或超过公开可用 frontier model baseline。

例如论文描述：

- Terminal Bench 中，Fugu 能在 GPT-5.5 和 Claude Opus 4.8 之间切换，利用一个模型构建、另一个模型 debug。
- GPQA-Diamond 中，Fugu 偏向使用 Gemini 的科学知识能力。
- Humanity's Last Exam 中，Fugu-Ultra 的分布更均衡，说明其在多学科任务中需要更广泛的 agent 组合。

## 核心贡献

1. **把编排本身模型化**
   - 不是手写 workflow，也不是简单 router。
   - 编排器通过训练学习任务到 agent workflow 的映射。

2. **提供低延迟和高质量双形态**
   - Fugu 对应“路由式编排”。
   - Fugu-Ultra 对应“深度自然语言 workflow 编排”。

3. **将多模型系统产品化为单模型接口**
   - 用户侧仍像调用一个模型。
   - 内部可以 route、delegate、coordinate。

4. **关注生产约束**
   - latency frontier。
   - workflow state。
   - agent isolation。
   - shared memory。
   - tool/function call 环境。

## 局限与风险

1. **黑盒性强**
   - 用户无法看到每次具体选择了哪些 worker 和为什么选择。
   - 对企业审计、问题定位和成本归因有挑战。

2. **评测外推要谨慎**
   - 论文报告了大量强结果，但企业任务分布可能不同。
   - 部分 baseline 依赖公开报告或特定评测设置，不能直接当作业务 ROI。

3. **供应链复杂**
   - Fugu 依赖多个底层强模型。
   - 数据合规、区域可用性、供应商变动都会影响系统。

4. **训练和运营门槛高**
   - 需要模型池、任务轨迹、奖励设计、评测基础设施。
   - 不是简单接入多个 API 就能复现。

## 与 TRINITY/Conductor 的关系

Fugu 把两条基础研究产品化：

- TRINITY 提供轻量 coordinator、agent selection、role selection、演化优化的思路。
- Conductor 提供自然语言 workflow、RL 训练、通信拓扑、递归调用的思路。

Fugu 的低延迟版更像 TRINITY 的生产化简化版本；Fugu-Ultra 更像 Conductor 的扩展生产版本。

## 对我们的启示

1. **不要一开始就做 Fugu**
   - Fugu 是目标形态，不是 MVP。
   - 我们应先做可观测 gateway、router、cascade、fusion，再积累训练数据。

2. **区分低延迟与高质量两条产品线**
   - 日常任务需要 Fugu-like router。
   - 高价值任务需要 Fugu-Ultra-like orchestrator。

3. **编排日志是未来资产**
   - 哪些任务调用了哪些模型。
   - 哪些模型在哪类任务上赢。
   - 哪些 workflow 成功。
   - 这些数据比“接入更多模型”更有护城河。

4. **生产工程是关键**
   - agent isolation、tool state、memory、budget、fallback 是系统成败的核心。

---

## 深度版分析：Fugu 是如何把研究路线产品化的

### 1. Fugu 的真正定位：不是“一个模型”，而是“模型组织层”

Fugu Technical Report 反复强调一个观点：能力提升不只来自训练更大的基础模型，也可以来自把现有模型按任务组织起来。

这意味着 Fugu 的产品定位应理解为：

```text
单模型 API 外壳 + 多模型/多 agent 编排内核
```

用户看到的是一个 model name；内部可能发生：

- 任务识别。
- worker 选择。
- 多步 workflow。
- 工具调用。
- verifier 检查。
- 结果合成。

这和传统模型网关不同。网关只是转发请求；Fugu 是训练过的策略系统。

### 2. Fugu 和 Fugu-Ultra 代表两个不同业务产品

论文把 Fugu 分成两个版本，不只是规格差异，而是两类产品逻辑：

| 版本 | 目标 | 类比 | 用户期望 |
|---|---|---|---|
| Fugu | latency-quality balance | 智能路由器 | 快、稳、比单模型更会选 |
| Fugu-Ultra | maximum answer quality | 深度项目经理 | 慢一些但更全面、更会推进复杂任务 |

这对我们很有启发：企业内部也不应做一个统一策略覆盖所有任务，而应做策略分层：

- 交互式产品：低延迟路由。
- 后台研究/分析：深度编排。
- 高风险场景：强验证 + 人工复核。

### 3. Fugu 低延迟版的关键取舍

Fugu 没有完全照搬 TRINITY 的三角色设计，而是简化为选择 worker model。论文指出 Fugu 不 assign roles，目的是降低 orchestration latency。

这个取舍说明：

```text
生产系统中，最优研究结构不一定是最优产品结构。
```

TRINITY 的 role selection 能提高质量，但会增加：

- action space。
- prompt复杂度。
- latency。
- 调试成本。

Fugu 对日常场景选择了更简单的 selection head。这是值得学习的产品判断：先把高频路径做到快和稳。

### 4. Fugu-Ultra 的关键工程问题：多 agent 环境状态

Fugu-Ultra 继承 Conductor 的自然语言 workflow，但论文进一步讨论了生产中很现实的问题：

1. **workflow state**
   - 哪些模型被选中。
   - 当前执行到哪一步。
   - 每个 agent 的输出是什么。
   - 哪些工具调用已经发生。

2. **intra-workflow isolation**
   - agent 不能随意看到全部中间状态。
   - 否则容易出现信息污染、重复工具调用、错误传播。

3. **persistent shared memory**
   - 任务需要跨步骤保留关键信息。
   - 但共享记忆必须受控，不能让所有 agent 无差别读写。

4. **orchestration collapse**
   - 如果第一个 agent 直接控制环境或污染上下文，后续 agent 失去独立判断空间。

这部分往往比 benchmark 更重要。真正做多 agent 系统时，失败常常不是模型不够聪明，而是状态管理混乱。

### 5. Fugu 的训练路径说明了“从路由到编排”的演进

Fugu 的训练大致可以看成两条路线：

#### 5.1 Fugu：选择器训练

- 先通过 worker 在任务上的表现构造监督信号。
- 再通过演化优化面向 end-to-end 任务表现。
- 目标是学会“这个 query 或这个阶段该找谁”。

#### 5.2 Fugu-Ultra：workflow 训练

- 使用 Conductor-like RL。
- 学会生成多步骤 workflow。
- 学会 prompt engineer、分配 agent、管理可见性。

这说明企业内部可以分阶段推进：

```text
规则路由 -> 监督路由 -> 黑盒优化路由 -> 固定 workflow -> 学习型 workflow
```

不要直接跳到最后一步。

### 6. Benchmark 结果的正确读法

论文列出 Fugu 在 SWE-Bench Pro、Terminal Bench、LiveCodeBench、GPQA-Diamond、HLE、CharXiv 等任务上的强结果。读这些结果时要注意三点：

#### 6.1 Fugu 的价值来自跨模型专长匹配

论文中多次解释：Terminal Bench 更偏某模型，GPQA 更偏另一个模型，HLE 多学科需要均衡分布。Fugu 的收益来自识别这些差异。

#### 6.2 Fugu-Ultra 的价值在长链路任务更明显

AutoResearch、代码修复、CAD、古籍阅读顺序、盲棋等案例共同说明：多模型编排在“需要持续推进、观察结果、修正策略”的任务上更有优势。

#### 6.3 结果需要本地复验

论文中的 baseline、工具配置、评测版本和 provider-reported scores 需要谨慎对比。企业不能直接引用论文分数作为采购或自研 ROI，应建立自己的 benchmark。

### 7. Fugu 与 Fusion 的本质区别

Fugu 和 OpenRouter Fusion 都是“统一入口 + 多模型”，但差异很大：

| 维度 | Fugu | Fusion |
|---|---|---|
| 策略来源 | 训练出的 orchestrator | 运行时 panel + judge pipeline |
| 主要动作 | route / delegate / workflow | parallel answer / compare / synthesize |
| 透明度 | 较低 | panel 可配置，流程相对清楚 |
| 成本模式 | 打包式产品定价 | 多次调用相加 |
| 最适合 | 长任务、agentic workflow | 单轮复杂研究和多视角分析 |

我们应把 Fusion 当作可控的中间层，把 Fugu 当作高阶目标形态或外部能力 benchmark。

### 8. Fugu 对“AI 主权/供应商依赖”的含义

论文提到 orchestration 可以作为对单一 provider 集中的对冲。这个观点有两面：

积极面：

- 底层模型可替换。
- 某个 provider 限制访问时可以绕开。
- 新模型出现后可以纳入 agent pool。

风险面：

- 如果 Fugu 自身是黑盒 provider，用户只是从依赖 A 变成依赖 orchestrator provider。
- 如果不暴露底层路由，企业难以审计和合规。

所以对企业而言，最佳路径是：

```text
使用 Fugu 作为外部能力 + 自建可观测 gateway 和评测闭环
```

而不是把所有策略完全交给外部黑盒。

### 9. 企业落地的 Fugu-like 分层架构

建议参考 Fugu 做三层：

```text
Layer 1: Gateway
  - provider adapter
  - model registry
  - cost/latency logging
  - compliance policy

Layer 2: Strategy Router
  - cheap direct
  - strong direct
  - cascade
  - fusion
  - workflow agent

Layer 3: Learned Orchestration
  - supervised router
  - bandit/evolution optimization
  - workflow generator
  - verifier feedback loop
```

Fugu 的经验说明：如果没有 Layer 1 和 Layer 2 的数据，直接做 Layer 3 很难。

### 10. 论文对我们的关键启示

1. **高频路径要简单**
   - 类似 Fugu 低延迟版，先解决“该选哪个模型”。

2. **复杂路径要允许深度编排**
   - 类似 Fugu-Ultra，只给高价值任务使用。

3. **状态管理是核心工程壁垒**
   - workflow state、tool state、memory isolation 要早设计。

4. **评测闭环是训练前提**
   - 没有任务结果和偏好数据，就没有 learned orchestration。

5. **不要把供应商黑盒当成长期护城河**
   - 长期护城河应是内部任务数据、路由策略、评测体系和工具集成。

### 11. 领导可能追问

**问：Fugu 是不是证明我们应该直接买外部 Fugu？**

答：不一定。Fugu 证明 learned orchestration 有价值，但外部 Fugu 是黑盒。我们可以把它作为高难任务 benchmark 和临时能力补充，同时自建 gateway、日志和评测，避免长期策略被外部锁定。

**问：Fugu 为什么能比单模型强？**

答：因为它不是靠一个模型学会所有能力，而是根据任务把不同模型的专长组合起来。比如一个模型做构建，另一个模型 debug；一个模型做科学推理，另一个模型做代码执行。

**问：我们是否能一年内做出 Fugu？**

答：可以做 Fugu-like 的第一阶段，也就是统一入口 + 路由 + 级联 + 部分 workflow。真正 Fugu 级 learned orchestrator 需要大量任务轨迹、评测集、偏好数据和训练基础设施，不应作为第一阶段承诺。
