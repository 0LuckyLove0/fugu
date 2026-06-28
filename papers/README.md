# 多模型编排论文包

生成日期：2026-06-28

本目录包含 9 篇与 Fugu、多模型路由、级联、融合、评测相关的论文 PDF、文本抽取和中文深度解析。

## 目录结构

- `pdfs/`：下载到本地的论文 PDF。
- `text/`：从 PDF 抽取的正文文本，`.clean.txt` 为去除 NUL 字节后的检索版本。
- `analyses/`：逐篇中文深度解析。
- `manifest.md` / `manifest.json`：论文清单、来源、页数、本地路径。
- `synthesis.md`：横向总结、方向差异和对我们的启示。

## 覆盖论文

1. Sakana Fugu Technical Report
2. TRINITY: An Evolved LLM Coordinator
3. Learning to Orchestrate Agents in Natural Language with the Conductor
4. FrugalGPT: How to Use Large Language Models While Reducing Cost and Improving Performance
5. RouteLLM: Learning to Route LLMs with Preference Data
6. RouterBench: A Benchmark for Multi-LLM Routing System
7. LLM-Blender: Ensembling Large Language Models with Pairwise Ranking and Generative Fusion
8. Mixture-of-Agents Enhances Large Language Model Capabilities
9. DRACO: a Cross-Domain Benchmark for Deep Research Accuracy, Completeness, and Objectivity

## 阅读顺序建议

如果目标是理解 Fugu：

1. `analyses/02_trinity_evolved_llm_coordinator.md`
2. `analyses/03_conductor_natural_language_orchestration.md`
3. `analyses/01_fugu_technical_report.md`

如果目标是做企业落地：

1. `analyses/04_frugalgpt.md`
2. `analyses/05_routellm.md`
3. `analyses/06_routerbench.md`
4. `analyses/07_llm_blender.md`
5. `analyses/08_mixture_of_agents.md`
6. `analyses/09_draco_deep_research_benchmark.md`
7. `synthesis.md`
