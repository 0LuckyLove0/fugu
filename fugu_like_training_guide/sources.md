# Sources and Evidence Boundaries

更新时间：2026-06-30

## 1. Fugu 官方与论文

- Sakana Fugu release  
  https://sakana.ai/fugu-release/  
  用途：确认 Fugu/Fugu Ultra 的产品定位、single model API、multi-agent orchestration、agent pool、recursive/self-call 等公开叙述。

- Sakana Fugu product page  
  https://sakana.ai/fugu/  
  用途：确认公开产品入口和调用形态。产品细节可能随时间变化。

- Sakana Fugu Technical Report, arXiv:2606.21228  
  https://arxiv.org/abs/2606.21228  
  本地副本：`papers/pdfs/fugu_technical_report_2606.21228.pdf`  
  用途：本文档最主要依据。支持以下结论：
  - Fugu 是 learned LLM orchestrator。
  - Fugu 低延迟版使用 lightweight selection head 选择 worker。
  - Fugu 训练包含单步任务 SFT 和端到端任务上的 evolutionary strategies。
  - Fugu Ultra 建立在 Conductor 思路上，输出自然语言 workflow。
  - Fugu Ultra 使用 GRPO 类训练、function-calling workflow state、agent isolation、shared memory。

## 2. 直接相关研究

- TRINITY: An Evolved LLM Coordinator, arXiv:2512.04695  
  https://arxiv.org/abs/2512.04695  
  本地副本：`papers/pdfs/trinity_evolved_llm_coordinator_2512.04695.pdf`  
  用途：Fugu 低延迟 selection/coordinator 路线的重要参考。支持 small coordinator、hidden-state head、agent/role action space、sep-CMA-ES、终端 reward 等设计。

- Learning to Orchestrate Agents in Natural Language with the Conductor, arXiv:2512.04388  
  https://arxiv.org/abs/2512.04388  
  本地副本：`papers/pdfs/conductor_natural_language_orchestration_2512.04388.pdf`  
  用途：Fugu Ultra workflow 路线的重要参考。支持自然语言 workflow、subtask/worker/access-list schema、GRPO、format reward、correctness reward、randomized agent pools、recursive topologies。

- RouteLLM: Learning to Route LLMs, arXiv:2406.18665  
  https://arxiv.org/abs/2406.18665  
  本地副本：`papers/pdfs/routellm_2406.18665.pdf`  
  用途：训练 cheap/strong router 和成本-质量阈值的参考。

- FrugalGPT, arXiv:2305.05176  
  https://arxiv.org/abs/2305.05176  
  本地副本：`papers/pdfs/frugalgpt_2305.05176.pdf`  
  用途：cheap-first cascade 和降本路线参考。

- RouterBench, arXiv:2403.12031  
  https://arxiv.org/abs/2403.12031  
  本地副本：`papers/pdfs/routerbench_2403.12031.pdf`  
  用途：成本-质量评测方法参考。

- DRACO, arXiv:2602.11685  
  https://arxiv.org/abs/2602.11685  
  本地副本：`papers/pdfs/draco_deep_research_benchmark_2602.11685.pdf`  
  用途：deep research rubric、事实准确性、引用质量、成本/延迟评测参考。

## 3. 工具文档

- Hugging Face TRL GRPOTrainer  
  https://huggingface.co/docs/trl/main/en/grpo_trainer  
  用途：说明 GRPOTrainer 支持用 reward functions 训练模型。本文档没有假设它能直接复现 Sakana 训练栈，只把它作为可用开源工具参考。

- Hugging Face TRL PPOTrainer  
  https://huggingface.co/docs/trl/main/en/ppo_trainer  
  用途：作为 RLHF/RL 训练替代工具背景。

- vLLM structured outputs  
  https://docs.vllm.ai/en/latest/features/structured_outputs/  
  用途：说明可以用 structured outputs/constrained decoding 提高 workflow JSON 合法率。

- pycma / CMA-ES  
  https://pypi.org/project/cma/  
  https://github.com/CMA-ES/pycma  
  用途：作为 sep-CMA-ES/CMA-ES 黑盒优化的 Python 工具参考。Fugu/TRINITY 报告使用的是相关思想，具体实现未必是 pycma。

- LiteLLM documentation  
  https://docs.litellm.ai/  
  用途：统一 provider 调用、model routing、fallback 的工程参考。

## 4. 明确没有公开证据的部分

以下内容不是公开 Fugu 细节，不应被当作 Sakana 内部实现：

- 具体训练数据集构成和样本量。
- 真实线上 worker pool 的完整列表。
- 精确 reward 权重。
- 具体模型 checkpoint 和超参数。
- 线上执行器、TraceStore、权限系统实现。
- 是否使用某个具体开源库，如 LiteLLM、TRL、vLLM、pycma。

本文档中关于这些部分的写法是工程建议，目的是指导自建 Fugu-like 系统。

## 5. 可信度分级

| 类型 | 可信度 | 用法 |
|---|---|---|
| Fugu 技术报告 | 高 | Fugu 公开机制的主要依据 |
| Fugu 产品/发布页 | 中高 | 产品定位和公开能力描述 |
| TRINITY/Conductor 论文 | 高 | 训练路线的研究依据 |
| 工具官方文档 | 高 | 说明可用工程工具 |
| 本文档工程推导 | 中 | 可落地路线，需要内部实验验证 |

