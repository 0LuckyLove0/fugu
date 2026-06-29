# 06. 评测与安全：怎么证明 Fugu-like 系统真的有效

## 1. 评测目标

Fugu-like 系统要同时证明四件事：

1. **质量提升**：复杂任务比 strong_direct 更好。
2. **成本可控**：简单任务比 always-strong 更便宜。
3. **延迟可控**：低延迟路径接近直接调用 worker。
4. **安全可审计**：不会越权调用模型、工具或泄漏数据。

如果只证明第一点，系统可能太贵。  
如果只证明第二点，系统可能牺牲质量。  
如果没有第四点，系统不能进入生产。

## 2. RouterBench-like 评测

### 2.1 数据格式

```json
{
  "task_id": "task_001",
  "task_type": "extract",
  "outputs": {
    "cheap": {"reward": 0.95, "cost": 0.005, "latency_ms": 800},
    "strong": {"reward": 0.98, "cost": 0.080, "latency_ms": 2500},
    "workflow": {"reward": 0.99, "cost": 0.320, "latency_ms": 18000}
  }
}
```

### 2.2 比较对象

- always cheap
- always strong
- hand-written rules
- router classifier
- selection head
- oracle best worker

### 2.3 关键图

```text
x-axis: average cost per task
y-axis: average reward / pass rate
```

一个 router 有价值，必须在成本-质量平面上优于固定策略。

### 2.4 不能只看平均值

还要看：

- p95 cost
- p95 latency
- high-risk task failure rate
- cheap false-accept rate
- escalation rate
- user rejection rate

## 3. DRACO-like 深度研究评测

复杂研究类任务至少拆成：

| 维度 | 说明 |
|---|---|
| factual accuracy | 事实、数字、日期、实体是否正确 |
| breadth/depth | 是否覆盖关键角度、权衡、反例 |
| citation quality | 引用是否真实，是否支撑 claim |
| objectivity | 是否区分事实、推断和观点 |
| actionability | 是否给出可执行建议 |

### 3.1 Claim-level 引用验证

不要只数引用数量。要验证每条 claim：

```text
claim: "Fugu Ultra uses GRPO for workflow training."
source: Fugu Technical Report section 3.2.1/3.2.3
support: supported
```

分类：

- supported
- contradicted
- insufficient
- source_not_found

### 3.2 Judge 设计

建议三层：

1. 自动规则检查：链接、格式、引用数量、schema。
2. LLM judge：按 rubric 打分。
3. 人工抽检：高风险或 judge 分歧样本。

LLM judge 的分数只能作为证据之一，不应成为唯一真相。

## 4. Agentic coding 评测

Fugu 报告强调 coding 和 terminal tasks。自建评测应包含：

- 能跑单元测试的 bugfix 任务。
- 能执行命令的 terminal tasks。
- 需要多步探索的 repo tasks。
- 失败可归因的 patch tasks。

最低字段：

```yaml
task_id: bugfix_001
repo_snapshot: git_sha
instructions: "Fix failing parser test"
test_command: "pytest tests/test_parser.py -q"
success_condition: "exit code 0"
timeout_seconds: 600
allowed_tools:
  - read_file
  - edit_file
  - run_tests
```

评分：

```text
1.0 = tests pass and patch minimal
0.7 = tests pass but patch risky/large
0.3 = partial progress, diagnosis correct
0.0 = no useful progress
```

## 5. Orchestration-specific metrics

普通模型评测不够。Fugu-like 系统还要看编排行为。

| 指标 | 意义 |
|---|---|
| worker entropy | 是否所有任务都退化成同一个 worker |
| domain-route alignment | 数学/代码/研究是否路由到合理 worker |
| workflow valid rate | workflow 是否可解析可执行 |
| step efficiency | 是否用过多 step 换分 |
| tool-call precision | 工具调用是否必要 |
| isolation violation | agent 是否看到不该看的上下文 |
| fallback success | worker 失败时是否能恢复 |

## 6. 安全边界

### 6.1 Model allowlist

所有 worker 必须来自 allowlist：

```yaml
allowed_models:
  normal_data:
    - openai/gpt-5.5
    - anthropic/claude-opus-4.8
    - google/gemini-3.1-pro
  restricted_data:
    - local/qwen
    - private/vllm-model
```

Orchestrator 不能绕过 allowlist。

### 6.2 Tool permissions

Workflow step 必须显式声明工具权限：

```json
{
  "worker": "coding",
  "subtask": "Run tests and inspect failures.",
  "tool_policy": ["read_files", "run_tests"]
}
```

禁止默认开放：

- shell 全权限。
- 网络访问。
- 凭据读取。
- 删除文件。
- 写生产数据库。

### 6.3 Budget guard

服务端强制预算：

```python
if workflow.estimated_cost > request.cost_budget:
    reject_or_downgrade(workflow)
if workflow.max_parallel_calls > policy.max_parallel_calls:
    reject_or_downgrade(workflow)
if workflow.max_steps > policy.max_steps:
    reject_or_downgrade(workflow)
```

预算不能交给模型自觉遵守。

### 6.4 Data policy

每个请求要先分级：

| 等级 | 策略 |
|---|---|
| public | 可用外部 API |
| internal | 只用签约 provider |
| restricted | 只用本地/私有模型 |
| regulated | human-in-loop 或禁用 |

ModelRegistry 必须记录 provider 的数据政策。

## 7. 上线准入标准

建议：

```text
Router path:
  cost reduction >= 30%
  quality drop <= 2%
  high-risk false-cheap rate <= 0.5%

Workflow path:
  valid workflow rate >= 99%
  complex task win rate vs strong_direct >= 55%
  p95 latency within product budget
  no critical policy violation in red-team set

Overall:
  every response trace replayable
  every model/tool call auditable
  rollback available within one deploy
```

## 8. 失败时怎么排查

按层排：

```text
1. TaskProfile 错了吗？
2. Strategy 选错了吗？
3. Orchestrator 输出合法但不合理吗？
4. Worker 本身失败了吗？
5. Tool/environment 失败了吗？
6. Verifier 错了吗？
7. Synthesizer 把正确中间结果写错了吗？
8. Reward 没捕捉真实用户偏好吗？
```

每个失败必须能映射到一层。否则系统不可维护。

