# DRACO 深度解析

## 基本信息

- 论文：DRACO: a Cross-Domain Benchmark for Deep Research Accuracy, Completeness, and Objectivity
- arXiv：2602.11685
- 本地 PDF：`papers/pdfs/draco_deep_research_benchmark_2602.11685.pdf`
- 本地文本：`papers/text/draco_deep_research_benchmark_2602.11685.clean.txt`
- 页数：38
- 定位：深度研究系统评测 benchmark，也是 OpenRouter Fusion 公开评测使用的重要 benchmark。

## 一句话结论

DRACO 的贡献是把“深度研究质量”拆成事实准确性、分析完整性、表达客观性和引用质量，并用真实生产查询和专家 rubric 构建 100 个跨领域复杂任务。

## 研究问题

传统 benchmark 很难评估 deep research agent，因为它们通常：

- 问题太短或太封闭。
- 不需要多来源综合。
- 不评价引用质量。
- 不贴近真实用户研究需求。

DRACO 要解决的是：

> 如何评估一个 deep research system 是否真的能做复杂、开放、跨来源、可引用的研究？

## 数据构造

DRACO 包含：

- 100 个复杂 deep research tasks。
- 覆盖 10 个 general/specialized domains。
- 涉及 40 个国家的信息来源。
- 任务来源于 Perplexity Deep Research 生产请求，经匿名化、过滤、改写和增强。

每个任务都配有专家设计 rubric。

## 评价维度

DRACO 主要评估四个方面：

| 维度 | 含义 |
|---|---|
| Factual Accuracy | 事实是否正确 |
| Breadth and Depth of Analysis | 分析是否完整、有深度 |
| Presentation Quality / Objectivity | 表达是否清晰、客观 |
| Citation Quality | 引用是否正确支撑结论 |

论文指出 factual accuracy 在 rubric 中占比最大，反映深度研究任务中事实错误的高风险。

## 评分方式

DRACO 使用 LLM-as-a-judge：

- judge 按每条 rubric criterion 判断是否满足。
- 加权聚合为 normalized score。
- 同时计算 pass rate。
- 论文使用 Gemini-3-Pro 作为主 judge，并报告 GPT-5.2、Sonnet-4.5 等 judge 下的稳定性。

论文强调：

- 相对排名较稳定。
- 绝对分数会随 judge 变化。

## 实验结论

论文评估了多个 deep research system：

- Perplexity Deep Research。
- OpenAI Deep Research。
- Gemini Deep Research。
- Claude Opus 等。

报告中 Perplexity Deep Research 在整体分数和 pass rate 上领先。更重要的是，论文展示了不同系统在：

- token usage。
- latency。
- citation quality。
- domain performance。
- rubric axis performance。

上的差异。

## 核心贡献

1. **真实生产查询来源**
   - 比人工玩具题更贴近用户需求。

2. **专家 rubric**
   - 比单纯参考答案或 LLM 自动评分更细。

3. **多维度评估**
   - 不只看答案对错，也看完整性、客观性、引用。

4. **暴露 deep research 的成本/延迟差异**
   - 高质量系统通常消耗大量输入 tokens 和等待时间。

## 局限

1. **单轮评测**
   - 不覆盖多轮澄清、任务追踪和长期研究。

2. **静态任务集**
   - 可能被污染，也可能不代表未来任务。

3. **文本到文本**
   - 不覆盖多模态深度研究。

4. **英文评测**
   - 对中文和跨语言任务泛化有限。

5. **依赖 LLM judge**
   - 相对排名稳定不等于绝对分数准确。

## 与 Fusion/Fugu 的关系

DRACO 是 Fusion 类系统最相关的评测之一，因为它要求：

- 多来源检索。
- 事实核查。
- 多角度分析。
- 引用质量。
- 可读报告。

这正是多模型 panel 和 deep research orchestrator 希望提升的任务类型。

## 对我们的启示

1. **内部必须有 DRACO-like benchmark**
   - 真实业务任务。
   - 专家或业务 owner rubric。
   - 事实、完整性、引用分开评估。

2. **不要只评估“答案看起来好不好”**
   - 引用是否支撑结论很关键。

3. **要记录 token 和 latency**
   - deep research 成本可能很高。

4. **中文场景需要自建评测**
   - DRACO 是英文，不能直接代表中文业务深度研究。

## 深度版分析：DRACO 是判断 Fusion/Fugu 是否真实有效的标尺

### 1. Deep research 的难点不是“回答长”，而是“答案没有单一标准答案”

很多 benchmark 可以用一个 reference answer 或单个正确选项评估。但 deep research 任务通常不是这样：

- 用户问题开放，答案需要范围界定。
- 事实来自多个来源，且来源可靠性不同。
- 好答案必须取舍，不能无限罗列。
- 需要识别反例、限制条件和不确定性。
- 引用不仅要存在，还必须支撑具体论点。

DRACO 的贡献是把这种开放任务拆成可审计 rubric，而不是只让 judge 给一个整体印象分。它对每个任务设计具体 criteria，再按 factual accuracy、breadth/depth、presentation/objectivity、citation quality 等维度聚合。

这对评估 Fusion/Fugu 特别重要。多模型系统很容易写出“看起来更完整”的长答案，但如果事实错误更多、引用不支撑结论、遗漏关键约束，那么质量并没有真实提升。DRACO 迫使系统在可检查的维度上比较。

### 2. Factual Accuracy 是权重中心，不是普通维度之一

论文数据显示，DRACO 每个任务平均约 39.3 条 rubric criteria，其中 factual accuracy 平均 20.5 条，占比超过一半。负向 criteria 也很关键：错误事实、无根据断言、有害建议会被显式扣分，医学等高风险领域有更重惩罚。

这个设计传递了一个重要信号：

> 深度研究系统的第一性风险不是“不够会写”，而是“用漂亮结构包装错误事实”。

这正好是 Fusion/MoA/Fugu 需要特别警惕的问题。多模型聚合会增加表述完整性，但也可能增加事实入口：每个 proposer 都可能引入错误，aggregator 还可能把错误融合成更难识别的结论。没有 factual verifier 和 citation verifier，多模型系统可能在主观偏好上赢，在事实准确性上输。

### 3. Citation Quality 是多模型研究系统的硬约束

DRACO 把 citation quality 单独列为轴，而不是把引用当装饰。这一点非常重要。

在 deep research 中，引用有四层要求：

1. 引用 URL 或文献必须真实存在。
2. 引用来源必须足够权威或至少适合该论点。
3. 引用内容必须实际支持被引用的句子。
4. 引用不能被滥用，例如用泛泛来源支撑具体数字。

普通 LLM judge 很容易被“有很多引用”的外观欺骗。DRACO 的 rubric 化思路提醒我们，内部评测不能只统计 citation count，而要抽查 support relation：

```text
claim -> cited source -> source passage -> 是否支持该 claim
```

如果我们建设 Fusion 或 Fugu-like deep research 入口，必须保存检索证据、引用位置和最终 claim 的对应关系。否则无法判断答案为什么得分，也无法定位错误来源。

### 4. DRACO 也在评估成本和延迟，不只是分数

论文不仅报告 normalized score 和 pass rate，还报告 token usage 与 latency。这个细节对商业落地很关键，因为 deep research 的质量提升往往依赖大量检索、长上下文、工具调用和多轮推理。

一个系统如果分数高 3%，但成本高 5 倍、延迟高 10 倍，是否值得取决于任务价值。相反，一个预算 panel 如果接近 frontier deep research 的得分且成本显著更低，就可能对企业很有吸引力。

因此，对 Fusion/Fugu 的评估必须同时看：

| 指标 | 为什么重要 |
|---|---|
| normalized score | 加权质量，反映任务重要维度 |
| pass rate | criteria 层面的满足比例，更容易诊断 |
| factual accuracy | 防止漂亮但错误 |
| citation quality | 防止引用装饰化 |
| input/output tokens | 直接影响成本 |
| latency | 决定是否可用于交互 |
| variance | 判断系统是否稳定 |

单看平均分会掩盖很多问题。特别是企业场景中，稳定性和尾部失败往往比平均质量更重要。

### 5. 为什么 DRACO 对 OpenRouter Fusion 特别相关

OpenRouter Fusion 的官方叙述重点就是在 DRACO-like deep research 任务上，panel 可以超过单模型，甚至预算模型 panel 也能接近或超过部分 frontier 模型。这个结论的可信度取决于三个前提：

- DRACO 是否代表用户关心的任务分布。
- Fusion 的 panel 是否能真实使用检索、工具和独立推理，而不是互相泄露或被提示污染。
- Judge 是否能稳定区分事实、引用和分析质量，而不仅是偏好长答案。

因此，对 Fusion 的正确态度不是简单接受“panel beats frontier”，而是把它当作一个需要复现的假设：

```text
在我们的任务分布上：
  budget panel 是否超过 strong direct？
  frontier panel 是否超过 best frontier direct？
  提升来自事实准确、完整性、引用，还是只来自表达更完整？
  成本和延迟是否仍然可接受？
```

如果没有这组问题，DRACO 分数很容易被误用于营销，而不是工程决策。

### 6. DRACO 对 Fugu/Fugu Ultra 的评估意义

Fugu 的卖点是把 learned orchestration 封装成单一 API；Fugu Ultra 则偏向复杂多步骤任务的高质量输出。DRACO 这类 benchmark 可以评估它是否真的比强单体模型和普通 panel 更有效。

但 Fugu 的黑盒属性也带来额外评估要求：

- 它内部调用了哪些 agent？用户不一定知道。
- 它是否在不同任务上动态选择了不同工作流？外部难以验证。
- 它的失败来自路由、检索、工具、聚合还是最终生成？黑盒输出难以定位。
- 它的成本是否可预测？Ultra 质量更高，但预算和延迟可能更难控。

因此，如果企业试用 Fugu/Fugu Ultra，应把 DRACO-like 评测当作准入测试，而不是只看厂商公开 benchmark。必须用自己的任务、自己的风险权重、自己的语言和引用要求评估。

### 7. 内部 DRACO-like benchmark 应该怎么建

建议分三层，而不是一开始追求 100 个复杂任务：

**第一层：20 个高价值真实任务**

- 来自真实用户或业务 owner。
- 覆盖最常见的 3-5 类任务。
- 每个任务有人工写的 pass/fail criteria。
- 先用来比较 strong direct、cheap direct、Fusion、Fugu。

**第二层：50-100 个分层样本**

- 按任务类型、风险等级、语言、资料可得性分层。
- 加入中文、双语、领域术语和本地来源。
- 每题记录标准检索来源或至少参考资料集合。

**第三层：持续刷新集**

- 每月从失败样本、用户差评、人工返工案例中抽样。
- 防止系统只对静态 benchmark 过拟合。
- 保留一部分 hidden set，只用于版本准入。

每个样本应包含：

```text
task_id
user_query
domain
risk_level
expected_sources_or_source_policy
rubric_criteria[]
criterion_weight
negative_criteria[]
judge_prompt_version
human_review_notes
```

### 8. 评测设计中的几个陷阱

1. **只看总分**
   - 总分提升可能来自 presentation，而事实准确性下降。

2. **只用 LLM judge**
   - 相对排名可能稳定，但绝对分数会随 judge 变化；关键样本需要人工复核。

3. **引用不落到 claim**
   - 引用列表存在不代表引用支撑结论。

4. **忽略成本和延迟**
   - Deep research 质量提升如果不可承受，就不能作为默认路径。

5. **英文 benchmark 直接外推到中文**
   - 中文资料生态、搜索质量、引用形式、领域术语都不同。

6. **没有失败标签**
   - 只知道系统错了不够，还要标注是检索失败、事实错误、推理错误、引用错误还是表达遗漏。

### 9. 领导层问答

**问：DRACO 能不能直接作为我们的内部评测？**  
不能直接替代。它可以作为方法模板，但我们的业务语言、领域、风险和资料来源不同，需要自建 DRACO-like 集合。

**问：Fusion/Fugu 在 DRACO 上强，是否就说明我们应该采用？**  
只能说明它们值得进入评测。是否采用取决于我们任务上的质量、成本、延迟、合规和可审计性。

**问：为什么要花时间写 rubric？**  
因为没有 rubric，就无法区分“答案看起来更专业”和“答案事实上更可靠”。多模型系统尤其需要这种约束。

**问：最小可行评测集多大？**  
第一版 20 个真实高价值任务就有意义，但必须有清晰 criteria、成本/延迟记录和至少部分人工复核。规模可以之后扩展。
