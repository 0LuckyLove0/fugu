# Fugu-like Orchestrator Training Guide

更新时间：2026-06-30  
目标：基于公开资料，解释 Fugu/Fugu Ultra 的核心机制，并给出一条可落地的 Fugu-like 模型训练路线。

## 先说边界

这套文档不是 Sakana Fugu 的内部复刻手册。Sakana 公开了 Fugu 的高层架构、训练范式、部分公式、评测和若干生产设计原则，但没有公开：

- 真实训练数据集。
- 完整 worker agent pool 配置。
- 训练脚本。
- 模型权重。
- 线上执行器代码。
- 评测 harness 的全部实现细节。

因此，这里采用两层写法：

1. **公开证据层**：只陈述 Sakana Fugu 技术报告、Fugu 发布页、TRINITY、Conductor、RouterBench、DRACO、TRL/vLLM/pycma 等公开资料能支持的内容。
2. **工程复现层**：在证据之上，给出一个可自己训练的 Fugu-like orchestrator 路线。它追求类似效果：单一入口、动态选择模型、按任务构建 workflow、基于任务结果持续优化。

## 阅读顺序

1. [01_fugu_deep_dive.md](01_fugu_deep_dive.md)  
   深入解释 Fugu/Fugu Ultra 到底是什么，不是什么。

2. [02_architecture_blueprint.md](02_architecture_blueprint.md)  
   把 Fugu-like 系统拆成控制面、执行面、训练面、评测面。

3. [03_training_data_and_rewards.md](03_training_data_and_rewards.md)  
   说明训练数据、轨迹日志、reward 和失败标签怎么设计。

4. [04_training_playbook.md](04_training_playbook.md)  
   手把手路线：从规则网关，到 router，到 TRINITY-like coordinator，到 Conductor-like workflow model。

5. [05_minimal_reproducible_stack.md](05_minimal_reproducible_stack.md)  
   给一个最小技术栈和代码骨架。

6. [06_evaluation_and_safety.md](06_evaluation_and_safety.md)  
   如何证明系统真的强、真的省钱，并且不会失控。

7. [07_build_plan.md](07_build_plan.md)  
   2 周、6 周、3 个月的落地路线。

8. [sources.md](sources.md)  
   所有来源和可信度说明。

## 一句话总览

Fugu-like 系统不是“训练一个更大的单体模型”，而是训练一个 **orchestrator**：

```text
用户请求
  -> 任务理解和风险分级
  -> 选择 direct / router / cascade / workflow / fusion
  -> 调用一个或多个 worker LLM
  -> 执行工具、验证、合成
  -> 记录轨迹、成本、延迟、reward
  -> 用这些结果继续训练 orchestrator
```

Fugu 低延迟路线更接近 TRINITY/selection-head：快速选择合适 worker。  
Fugu Ultra 高质量路线更接近 Conductor：生成自然语言 workflow，让多个 worker 分工协作。

