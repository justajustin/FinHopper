# FinHopper — A Controllable Multi-Hop QA Benchmark for Finance

> [中文版 README](README.md)

**FinHopper** is a multi-hop question answering (Multi-Hop QA) benchmark for financial agents, introduced in the paper *FinHopper: A Progressive Multi-Hop Benchmark for Complex Financial Agents with Step-wise Evaluation*.

This repository provides **Chinese and English versions** of the benchmark. Each sample contains the full question, the final reference answer, step-level gold intermediate answers, and judge prompts, enabling evaluation of large language models on long-chain financial reasoning tasks.

---

## Background

Financial analysis is inherently multi-hop: an analyst first looks up a list of companies → filters by financial indicators → traces back to specific time points → and finally synthesizes a judgment, where each step is conditioned on the intermediate result of the previous one. Unlike multi-hop QA in general domains, every intermediate answer in financial scenarios is a **precise, typed constraint** (an entity, a date, an event, or a numerical indicator). Once any single hop goes wrong, the entire search trajectory may look plausible while being completely detached from the facts.

However, most existing benchmarks focus on single-hop retrieval or closed-form computation (e.g., FinQA, TAT-QA, ConvFinQA), and lack a comprehensive design for open-domain financial search, strict dependency preservation, and step-wise evaluation. FinHopper builds a diagnosable evaluation framework for financial agents around three core ideas: **state-transition-driven controllable synthesis, two-layer verification during and after assembly, and step-wise scoring**.

---

## Files

| File | Description |
|------|-------------|
| `benchmark_en.json` | English version of the benchmark |
| `benchmark_zh.json` | Chinese version of the benchmark |

Both files share identical reasoning chains, final answers, and step-level intermediate answers; only the language of the questions, reference answers, and judge prompts differs. The main experiments in the paper use the Chinese version, while the English version is used for comparative validation with representative models.

---

## Dataset Overview

### Scale and Difficulty Tiers

| Tier | Hop range | #Samples | Description |
|------|-----------|----------|-------------|
| **L1** | 1–2 hops | 100 | Basic retrieval and simple filtering tasks |
| **L2** | 3–4 hops | 60 | Medium-length chained reasoning tasks |
| **L3** | 5–6 hops | 40 | High-complexity long-chain reasoning tasks |

Detailed hop distribution:

| Hops | 1-hop | 2-hop | 3-hop | 4-hop | 5-hop | 6-hop |
|------|-------|-------|-------|-------|-------|-------|
| Count | 50 | 50 | 39 | 21 | 19 | 21 |

Each K-hop question comes with K step-level gold answers, yielding **572 step-level checkpoints** in total.

### Domain Coverage

The dataset covers **14** financial sub-domains: intraday data, cross-period analysis, individual stock information, fund indicators, financial statements, index funds, stock events, active funds, market data, fixed income, capital flows, macro events, industry events, and money market funds.

---

## Field Description

Each sample (JSON object) contains the following 8 fields:

| Field | Type | Description |
|-------|------|-------------|
| `prompt_id` | `string` | Unique question identifier in the format `{tier}_{hop}_{index}`, e.g., `L1_1hop_1`, `L2_3hop_7`, `L3_6hop_9` |
| `prompt` | `string` | The full multi-hop financial question. In the released version, the answers to intermediate hops are hidden; models must perform step-by-step retrieval and reasoning on their own |
| `reference_response` | `string` | **Full reference answer** with detailed reasoning, intermediate data values, logical derivation, and the final conclusion. Serves as the basis for both process scoring (s_process) and final-answer scoring (s_final) |
| `response_final_reference` | `string` | **Concise final answer** containing only the core result (e.g., company name, date, number), used for exact-match scoring of the final answer. Surface differences in units, date formats, and percent signs are normalized during scoring |
| `judge_prompt_template` | `string` | Judge prompt template with three placeholders: `{prompt}`, `{response_reference}`, and `{response}`. Replace `{response}` with the model output to construct the judging request |
| `label` | `string` | Difficulty label: `L1_1hop` / `L1_2hop` / `L2_3hop` / `L2_4hop` / `L3_5hop` / `L3_6hop` |
| `final_judge_system_prompt` | `string` | **System prompt for final-answer scoring**. Two hard filters are applied first: ① grounding check — whether the student answer is based on searched data rather than the model's own inference; ② content existence check — the answer must not be empty or garbled. Failing either filter yields 0 directly. Otherwise, the student's final conclusion is compared with the reference: consistent (1) or inconsistent (0); intermediate scores are forbidden |
| `process_judge_system_prompt` | `string` | **System prompt for process scoring**. Defines a two-step method: ① extract N key process nodes (excluding the final conclusion) from the reference answer; ② check the student answer's coverage of each step-level gold answer node by node. Process score = 0.5 × hits / N. Enabled only when the final answer is wrong; skipped when it is correct |

### Scoring Logic

The overall score uses a two-stage hierarchical scheme (Hierarchical Score):

```
s_process = 0.5 × (1/N) × Σ I(k_i ∈ A)      // α = 0.5
S(A) = s_final + (1 − s_final) × s_process
```

- If the final answer is correct → `s_final = 1`, total score = 1.0 (full marks); the process score is not computed;
- If the final answer is wrong → `s_final = 0`, total score = `s_process` (ranging 0.0–0.5), awarded proportionally to the hit ratio of intermediate reasoning steps.

This "final-answer-first, process-supplementary" design guarantees correctness comes first while retaining fine-grained diagnostic signals for partially correct reasoning. The paper aggregates per-question scores with both micro-average (weighted by data distribution) and macro-average (equal weight across the six hop categories), reflecting overall performance and long-chain stability, respectively.

---

## Usage

### 1. Load the data

```python
import json

with open("benchmark_en.json", "r", encoding="utf-8") as f:
    data = json.load(f)

# Split by difficulty tier
l1 = [item for item in data if item["label"].startswith("L1")]
l2 = [item for item in data if item["label"].startswith("L2")]
l3 = [item for item in data if item["label"].startswith("L3")]

# Count step-level checkpoints
total_steps = sum(int(item["label"].split("hop")[0][-1]) for item in data)
print(f"Total: {len(data)} questions, {total_steps} step-level checkpoints")
```

### 2. Construct the judge request

```python
item = data[0]

# Replace placeholders in the template with actual content
judge_prompt = (
    item["judge_prompt_template"]
    .replace("{prompt}", item["prompt"])
    .replace("{response_reference}", item["reference_response"])
    .replace("{response}", student_answer)  # replace with model output
)

# Final-answer scoring: use final_judge_system_prompt as the system prompt
# Process scoring (only when s_final = 0): use process_judge_system_prompt
```

## Citation

If you use this benchmark, please cite the following paper:

```bibtex
@inproceedings{finhopper2026,
  title     = {FinHopper: A Progressive Multi-Hop Benchmark for Complex
               Financial Agents with Step-wise Evaluation},
  author    = {Wu, Yuxuan and Lin, Miaopei and Ying, Qiufang and Dan, Lin
               and Zhao, Hao and Liu, Qirui and Shen, Yuteng and Wu, Shaohui
               and Luo, Weihong and Du, Xiku and Tang, Xing and Chen, Hao},
  booktitle = {Findings of the 2026 Conference on Empirical Methods in
               Natural Language Processing (EMNLP Findings)},
  year      = {2026},
  url       = {https://github.com/YOUR-USERNAME/FinHopper}
}
```
