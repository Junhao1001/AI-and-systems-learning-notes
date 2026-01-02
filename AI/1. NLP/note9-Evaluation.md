# Evaluation (Benchmark)

## Key concepts

- F1: 是**Precision（精确率）和 Recall（召回率）的调和平均（harmonic mean）**，用来同时衡量：
  - **预测得准不准**（Precision）
  - **该找的有没有找全**（Recall）
  - 数学定义：

$$
\text{F1} = \frac{2 \cdot P \cdot R}{P + R}
$$

- **Exact Match**: 完全匹配， 预测答案 **与标准答案完全一致** → 1， 否则 → 0
- 在 **SQuAD** 里：
  - **EM**：答案 span 是否完全一样
  - **F1**：token 级别重叠程度
- **NLU**: General Natural Language Understanding
  - 通用 NLU 指模型在不针对单一任务专门设计的情况下， 能够在多种语言理解任务上表现稳定、可迁移、鲁棒
  - Semantic Understanding: 语义理解
  - Syntactic Awareness: 句法于结构理解
  - Pragmatics: 语用与上下文
  - Reasoning: 推理能力
  - Knowledge-aware: 知识使用
- **negation bias**：模型在处理“否定表达（not, no, never, without …）”时， 倾向于忽略、弱化或错误理解否定语义，从而做出系统性错误判断
  - 例子：The movie is not good.”
    - 正确：负面
    - 但模型容易预测：正面，  因为 **“good” 的情感信号压过了 “not”**

## Close-ended evaluations: 

**模型的输出被限制在一个明确的候选集合中**，或者可以被**严格判定对/错**

### Classification:

- 固定label space
- 强监督
- 指标：Accuracy / F1

| 任务         | 代表 Benchmark       | 能力侧重 |
| ------------ | -------------------- | -------- |
| 情感分析     | SST-2                | 句级语义 |
| 语义相似     | MRPC / QQP           | 语义等价 |
| 自然语言推理 | MNLI / RTE           | 逻辑推理 |
| 综合评测     | **GLUE / SuperGLUE** | 通用 NLU |

### 多项选择（Multiple Choice）

- 输出是 A/B/C/D
- 非生成式

- **指标**：Accuracy

| Benchmark            | 领域       |
| -------------------- | ---------- |
| ARC (Easy/Challenge) | 科学推理   |
| HellaSwag            | 常识推理   |
| PIQA                 | 物理直觉   |
| **MMLU**             | 多学科知识 |
| BIG-bench (MC 子集)  | 涌现能力   |

### 抽取式 QA（Extractive QA）

- 答案必须是原文 span

- 指标：EM / F1

| Benchmark | 说明         |
| --------- | ------------ |
| SQuAD v1  | 基础阅读理解 |
| SQuAD v2  | + 无答案判断 |
| NewsQA    | 新闻理解     |

### 数值 / 程序正确性（Exact Match）

- 数学题、程序输出
- 评测时往往只看final answer
- 指标：Exact Match

数学/逻辑

| Benchmark | 特点     |
| --------- | -------- |
| GSM8K     | 小学数学 |
| MATH      | 竞赛数学 |
| DROP      | 数值推理 |

代码

| Benchmark | 评测方式   |
| --------- | ---------- |
| HumanEval | 单元测试   |
| MBPP      | 程序通过率 |

## Open-ended Evaluation

**模型可以自由生成文本，答案空间不受限**，通常不存在唯一“正确答案”。

### Generative QA

- 自由文本
- 无唯一答案
- 指标：依赖人或LLM-judge

| Benchmark                | 说明     |
| ------------------------ | -------- |
| Natural Questions（gen） | 知识问答 |
| TruthfulQA（generation） | 真实性   |
| ELI5                     | 长解释   |

### Instruction & Chat

| Benchmark     | 评测方式  |
| ------------- | --------- |
| **MT-Bench**  | 多轮对话  |
| AlpacaEval    | 指令跟随  |
| Chatbot Arena | 人类偏好  |
| Vicuna Bench  | Chat 能力 |

### Summarization / Writing

- 评测：人类评价 、 ROUGE / BERTScore（辅助）

| 任务 | Benchmark               |
| ---- | ----------------------- |
| 摘要 | CNN/DailyMail, SummEval |
| 改写 | ParaBank                |
| 写作 | WritingPrompts          |

### Reasoning / CoT

- 更关注 reasoning quality
- 过程本身是评测对象

| Benchmark             | 特点     |
| --------------------- | -------- |
| BIG-bench Hard        | 高阶推理 |
| StrategyQA            | 隐式推理 |
| GSM8K（CoT）          | 推理过程 |
| Tree-of-Thought Tasks | 搜索能力 |

### Alignment

| Benchmark           | 维度               |
| ------------------- | ------------------ |
| HELM                | 多维评测           |
| Anthropic HH        | Helpful / Harmless |
| SafetyBench         | 安全               |
| RealToxicityPrompts | 毒性               |

## Types of evaluation methods for text generation

### Content Overlap Metrics

**通过比较模型输出与参考答案在“词、n-gram、表示空间”上的重叠程度来评分**

- **BLEU**：主要用于机器翻译
  - n-gram precision
  - 有 brevity penalty
  - 偏向 **短、保守的输出**
- **ROUGE（摘要）**: 以Recall为中心
  - ROUGE-1 / 2 / L
  - 基于 recall
  - 偏向 **覆盖率**： 覆盖 reference 中的关键词
- **METEOR**
  - 同义词匹配
  - 词干化
- **BERTScore**
  - 用 BERT embedding
  - **计算 token 相似度**

### Model-based Metrics

**使用一个模型（通常是 LLM）来判断生成结果的质量、正确性或偏好**

- 打分式（Scoring）

  - 给 1–10 分

  - 多维度评分

- 偏好式（Pairwise Preference）

  - A vs B

  - 哪个更好？
  - **稳定性更强**

- 裁判式（LLM-as-a-Judge）

  - GPT-4 / Claude

  - 用自然语言标准判断