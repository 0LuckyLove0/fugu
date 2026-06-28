# RouterBench 深度解析

## 基本信息

- 论文：RouterBench: A Benchmark for Multi-LLM Routing System
- arXiv：2403.12031
- 本地 PDF：`papers/pdfs/routerbench_2403.12031.pdf`
- 本地文本：`papers/text/routerbench_2403.12031.clean.txt`
- 页数：16
- 定位：多 LLM 路由系统评测框架。

## 一句话结论

RouterBench 的价值不在某个 router 算法，而在它把多模型路由系统的评测标准化：同一批样本、多个模型结果、成本、质量和 router 策略都放到一个成本-性能平面上比较。

## 研究问题

多模型路由系统越来越多，但缺乏统一评测：

- 不同论文用不同数据集。
- 不同系统用不同模型价格。
- 有些只看质量，不看成本。
- 有些只看平均成本，不看质量曲线。

RouterBench 试图提供一个 benchmark 和理论框架，让各种 routing strategies 可比较。

## 数据集设计

RouterBench 包含：

- 405,467 个 inference outcomes。
- 11 个代表性模型。
- 8 个数据集。
- 64 个任务。

数据包含每个模型在样本上的：

- 输出。
- 质量/性能分数。
- 估算成本。

这允许离线模拟 router，而不必每次真实调用所有模型。

## 理论框架

RouterBench 把每个模型或 router 表示为成本-质量平面上的一个点：

```text
(expected cost, expected quality)
```

然后通过：

- interpolation。
- non-decreasing convex hull。
- zero router baseline。
- AIQ 指标。

来评估 router 是否真的比简单策略有价值。

核心思想是：

> 一个 router 只有在成本-质量曲线上超过简单插值基线时，才真的有系统价值。

## Router 类型

论文比较了：

1. **Predictive routers**
   - KNN router。
   - MLP router。
   - 根据输入预测各模型表现，再选成本收益最优模型。

2. **Cascading routers**
   - 先调用一个模型。
   - 用 judge/score 判断是否继续调用更强模型。

3. **Oracle router**
   - 理想上知道哪个模型会答得最好。
   - 用来衡量上限。

## 实验结论

论文发现：

- 简单 KNN/MLP router 在某些任务上能以更低或类似成本达到最佳单模型性能。
- 许多 router 并没有显著超过 zero/interpolation baseline，说明路由评测必须严谨。
- Oracle router 通常显著更强，说明还有很大提升空间。
- Cascading router 的表现高度依赖 judge/error rate。
- RAG 任务中 router 也能带来明显收益。

## 核心贡献

1. **标准化 router 评测**
   - 成本和质量一起看。

2. **提供大规模离线路由数据**
   - 降低研究和企业实验成本。

3. **提出 router 应优于 convex hull baseline**
   - 防止把随机/插值策略误认为智能路由。

4. **揭示路由算法仍有很大改进空间**
   - 简单 router 有用，但离 oracle 远。

## 局限

1. **主要关注性能和经济成本**
   - 不充分覆盖延迟、稳定性、安全、合规、用户满意度。

2. **数据集有限**
   - 虽然覆盖多个任务，但仍不等于企业真实流量。

3. **成本估算会变化**
   - API 价格和模型能力变化很快。

4. **cascading 依赖 judge 准确率**
   - judge 错误会直接影响级联收益。

## 与 Fugu/Fusion 的关系

RouterBench 更接近 Fugu 低延迟路由的评测基础，而不是 Fusion。

它能帮助我们回答：

- router 是否真的比固定模型好？
- 降本来自哪里？
- 与 oracle 差距多大？
- 什么任务值得路由？

## 对我们的启示

1. **必须建立内部 RouterBench**
   - 记录多个模型在同一任务集上的输出、成本和评分。

2. **评估 router 不要只看平均准确率**
   - 要看成本-质量曲线。

3. **加入 zero/convex hull baseline**
   - 防止自建 router 只是看起来智能。

4. **oracle gap 是路线图**
   - 如果 oracle 大幅好于现有 router，说明值得继续训练路由器。

---

## 深度版分析：RouterBench 为什么是多模型系统的“审计工具”

### 1. RouterBench 的核心价值：防止把“随机省钱”误认为“智能路由”

很多 routing demo 会展示：

- 平均成本下降。
- 平均质量差不多。

但如果没有强 baseline，很容易误判。例如，只要把一部分请求随机发给便宜模型，成本也会下降；如果任务本来简单，质量也可能不明显下降。这不代表 router 有智能。

RouterBench 的重要贡献是把所有策略放到成本-质量平面，并要求 router 超过简单 baseline。

### 2. 成本-质量平面是企业必须使用的评估方式

每个模型或 router 都可以表示成：

```text
(expected cost, expected quality)
```

这比单点指标更真实，因为业务要的是 trade-off：

- 预算有限时用哪个策略？
- 质量目标提高时成本增加多少？
- router 是否比固定模型组合更优？

RouterBench 的 non-decreasing convex hull 思想可以直接用于内部评估：

```text
如果一个 router 点落在简单插值曲线下方，它没有价值。
```

### 3. Zero Router 的意义

Zero router 表示一个非常基础的成本-质量参考点。论文用它帮助判断 router 是否真的带来额外收益。

企业内部也应设置简单 baseline：

- 全部 cheap。
- 全部 strong。
- 随机按比例分配。
- 按任务类型规则分配。
- cheap-first 固定阈值。

如果训练 router 不能超过这些 baseline，不应上线。

### 4. RouterBench 数据结构对我们的启发

RouterBench 保存每个样本在多个模型上的输出、成本和质量。这是训练/评估 router 的理想数据结构。

企业内部应建立类似表：

| 字段 | 含义 |
|---|---|
| task_id | 样本 ID |
| input | 用户请求 |
| task_type | 任务分类 |
| risk_level | 风险等级 |
| model_id | 模型 |
| output | 模型输出 |
| token_cost | token 成本 |
| latency_ms | 延迟 |
| quality_score | 人工或 judge 分 |
| accepted | 是否可接受 |
| failure_type | 错误类型 |

这张表是所有 router、cascade、fusion 决策的基础资产。

### 5. Predictive router 与 cascading router 的差异

RouterBench 同时研究 predictive 和 cascading：

| 类型 | 决策时机 | 优点 | 缺点 |
|---|---|---|---|
| Predictive | 生成前 | 延迟低、成本可控 | 不能看到候选答案 |
| Cascading | 生成后 | 可根据答案质量升级 | 需要 judge，延迟更高 |

论文显示 cascading router 的效果受 judge error rate 影响极大。这对落地很关键：如果 verifier 不可靠，级联策略可能比 pre-router 更危险。

### 6. Oracle gap 的战略意义

RouterBench 中 oracle router 常常远强于实际 router。这说明：

```text
模型池里存在可被利用的互补性，但当前路由器还没有充分捕捉。
```

企业可以用 oracle gap 判断是否值得继续投资：

- 如果 oracle 比 best single model 高很多，说明模型互补性强，值得做 router。
- 如果 oracle 和 best single model 差不多，说明路由空间有限，直接用 best model 更简单。

### 7. 为什么很多 router 没有显著超过 baseline

论文指出一些简单 router 并不显著超过 Zero/NDCH baseline。原因可能包括：

1. 特征不足。
2. 训练数据少。
3. 模型能力差距不够互补。
4. 成本估算不准确。
5. 任务本身不适合路由。

这提醒我们：多模型路由不是必然有效，必须先证明任务分布里存在可利用结构。

### 8. RouterBench 与 DRACO 的互补

RouterBench 更偏 routing cost-quality 评估，DRACO 更偏 deep research answer quality 评估。

企业需要两套评测：

| 评测 | 解决问题 |
|---|---|
| RouterBench-like | 哪个模型/策略最省钱且质量够 |
| DRACO-like | 复杂研究报告是否事实准确、完整、有引用 |

如果只做 DRACO，无法系统降本；如果只做 RouterBench，无法评估复杂报告质量。

### 9. 对内部评测平台的要求

一个合格的内部 RouterBench 应支持：

- 离线 replay：同一输入跑多个模型。
- 价格版本化：模型价格会变。
- 模型版本化：模型能力会变。
- 评分版本化：judge/rubric 会变。
- 策略模拟：不真实调用也能模拟 router 结果。
- curve report：输出成本-质量曲线，而非单点。

### 10. 领导可能追问

**问：为什么要先做评测平台，而不是直接做 router？**

答：没有评测平台就不知道 router 是否真的优于简单规则。RouterBench 说明 routing 的价值必须在成本-质量曲线上证明。

**问：我们需要 405k 样本吗？**

答：不需要一开始这么大。PoC 可以从 200-1000 条高质量真实任务开始，但字段结构要按 RouterBench 思路设计，便于后续扩展。

**问：如果 oracle gap 很小怎么办？**

答：说明当前任务/模型池中路由收益有限，应优先做 prompt 优化、缓存或直接选择一个稳定模型，不要强推复杂 router。
