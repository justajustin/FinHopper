# FinHopper — 金融可控多跳 QA 评测基准

**FinHopper** 是一个面向金融智能体的多跳问答（Multi-Hop QA）评测基准，源自论文 *FinHopper: A Progressive Multi-Hop Benchmark for Complex Financial Agents with Step-wise Evaluation*。

本仓库提供该基准的 **中/英双语版本**，每条数据均包含完整问题、最终参考答案、过程级标准中间答案与评分提示词，可用于评估大语言模型在金融长链推理任务上的表现。
---

## 论文背景

金融分析天然是多跳推理：分析师需要先查公司名单 → 再筛财务指标 → 再回溯时间节点 → 最后综合得出判断，每一步都以「上一步的中间结果」为前提。与通用领域的多跳问答不同，金融场景中的每一跳中间答案都是一个**精确的类型化约束**（实体、日期、事件或数值指标），一旦某个环节出错，整个搜索轨迹会看似合理却完全偏离事实。

然而，现有评测基准大多侧重单跳检索或封闭计算（如 FinQA、TAT-QA、ConvFinQA），缺乏面向开放域金融搜索、严格依赖保持和过程级评价的综合性设计。FinHopper 围绕**「状态转移驱动的可控合成 + 组装中/组装后双层验证 + 过程级评分」**三大核心思路，提出了一套面向金融智能体的可诊断评测框架。

---

## 文件说明

| 文件 | 说明 |
|------|------|
| `benchmark_en.json` | 英文版基准，200 条金融多跳问题（含完整字段） |
| `benchmark_zh.json` | 中文版基准（待上传） |

两个文件共享相同的推理链、最终答案和步骤级中间答案，仅问题 / 参考答案 / 评分提示词的语言不同。论文中主力评测使用中文版，英文版进行了代表性模型的对比验证。

---

## 数据集概览

### 规模与难度分层

| 难度层级 | 跳数范围 | 样本数 | 说明 |
|---------|---------|-------|------|
| **L1** | 1~2 跳 | 100 条 | 基础检索与简单筛选任务 |
| **L2** | 3~4 跳 | 60 条 | 中等链式推理任务 |
| **L3** | 5~6 跳 | 40 条 | 高复杂度长链推理任务 |

详细跳数分布：

| 跳数 | 1-hop | 2-hop | 3-hop | 4-hop | 5-hop | 6-hop |
|------|-------|-------|-------|-------|-------|-------|
| 数量 | 50 | 50 | 39 | 21 | 19 | 21 |

每条 K-hop 问题包含 K 个步骤级标准答案，整个数据集共 **572 个步骤级检查点**。

### 领域覆盖

数据集涉及 **14 个**金融细分方向：日内数据、跨周期分析、个股信息、基金指标、财务数据、指数基金、股票事件、主动基金、市场数据、固定收益、资金流向、宏观事件、行业事件、货币基金。

---

## 字段说明

每条数据（JSON 对象）包含以下 8 个字段：

| 字段 | 类型 | 说明 |
|------|------|------|
| `prompt_id` | `string` | 问题唯一标识符，格式为 `{层级}_{跳数描述}_{序号}`，如 `L1_1hop_1`、`L2_3hop_7`、`L3_6hop_9` |
| `prompt` | `string` | 完整的多跳金融问题文本。对外发布时中间跳的答案被隐藏，模型需自主完成逐跳检索与推理 |
| `reference_response` | `string` | **完整参考答案**，包含详细的推理过程、中间步骤的数据值、逻辑推演以及最终结论，同时作为过程评分（s_process）和最终答案评分（s_final）的依据 |
| `response_final_reference` | `string` | **精简版最终答案**，仅包含问题所问的核心结果（如公司名、日期、数值等），用于最终答案的精确匹配评分。评分时会对单位、日期格式和百分号等表面差异做归一化处理 |
| `judge_prompt_template` | `string` | 评分提示模板，包含 `{prompt}`、`{response_reference}`、`{response}` 三个占位符。使用时将模型输出替换 `{response}` 即可构造评分请求 |
| `label` | `string` | 难度标签，取值：`L1_1hop` / `L1_2hop` / `L2_3hop` / `L2_4hop` / `L3_5hop` / `L3_6hop` |
| `final_judge_system_prompt` | `string` | **最终答案评分的系统提示词**。先通过硬过滤器检查：① 实证检查——学生答案是否基于搜索数据而非模型自身推断；② 内容存在性检查——学生答案不能为空或含乱码。不通过直接判 0 分。通过后比较学生答案与参考答案的最终结论是否一致（1 分）或不一致（0 分），禁止给中间分值 |
| `process_judge_system_prompt` | `string` | **过程评分的系统提示词**。定义两步评分方法：① 从参考答案中提取 N 个关键过程节点（不含最终结论）；② 逐节点检查学生答案对步骤级标准答案的覆盖情况。过程得分 = 0.5 × 命中数 / N。仅在最终答案错误时启用，最终答案正确时跳过 |

### 评分逻辑

总分采用两阶段层次化评分（Hierarchical Score）：

```
s_process = 0.5 × (1/N) × Σ I(k_i ∈ A)      // α = 0.5
S(A) = s_final + (1 − s_final) × s_process
```

- 如果最终答案正确 → `s_final = 1`，总分 = 1.0（满分），不计算过程分；
- 如果最终答案错误 → `s_final = 0`，总分 = `s_process`（取值 0.0~0.5），按中间推理步骤的命中比例给分。

这种"结果优先 + 过程补充"的设计，既能保证正确性第一，又能保留对「部分正确推理」的细粒度诊断信号。论文实验以微平均（按数据分布加权）和宏平均（六类跳数等权）两种方式汇总单题得分，分别反映模型的整体表现和长链稳定性。

---

## 使用方式

### 1. 加载数据

```python
import json

with open("benchmark_en.json", "r", encoding="utf-8") as f:
    data = json.load(f)

# 按难度分层
l1 = [item for item in data if item["label"].startswith("L1")]
l2 = [item for item in data if item["label"].startswith("L2")]
l3 = [item for item in data if item["label"].startswith("L3")]

# 统计步骤级检查点
total_steps = sum(int(item["label"].split("hop")[0][-1]) for item in data)
print(f"Total: {len(data)} questions, {total_steps} step-level checkpoints")
```

### 2. 构造评分请求

```python
item = data[0]

# 将模板中的占位符替换为实际内容
judge_prompt = (
    item["judge_prompt_template"]
    .replace("{prompt}", item["prompt"])
    .replace("{response_reference}", item["reference_response"])
    .replace("{response}", student_answer)  # 替换为模型输出
)

# 最终答案评分：使用 final_judge_system_prompt 作为 system prompt
# 过程评分（仅在 s_final = 0 时）：使用 process_judge_system_prompt
```

## 引用

如果您使用了本基准，请引用以下论文：

```
@inproceedings{finhopper2026,
  title     = {FinHopper: A Progressive Multi-Hop Benchmark for Complex
               Financial Agents with Step-wise Evaluation},
  author    = {Anonymous},
  booktitle = {Anonymous ACL Submission},
  year      = {2026},
  url       = {https://anonymous.4open.science/r/FinHopper-aabb/}
}
```


