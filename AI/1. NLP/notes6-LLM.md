# LLM

## Pre-Training

**key-ideas**:

- Dont' use labeled data
- can process large-scale, diverse datasets
- Compute-aware scaling

### Subword Model

原始的token粒度：

- Word-level:
  - 词表巨大
  - OOV（Out-of-Vocabulary）严重
  - 形态变化（run / running / ran）
- Character-level
  - 序列太长
  - 语义粒度太细
  - 训练效率低

**Subword Model**: 用“常见的子串”作为 token 单位。 例如：

```
unhappiness
→ un + happi + ness
```

**BPE（Byte-Pari Encoding)**:

- 从最小单位开始（字符 or byte），逐步合并，直到达到目标词表大小。

- **它是在固定词表大小下合并最常见的子串**
- 最终结果:
  - 一套 subword vocabulary（子词表）
  -  **一套 merge rules（合并规则，有序）**

### Objective of Pre-training (Three structures)

#### Masked LM (Encoder only) 

- 随机 mask 掉句子中部分 token
- 预测被 mask 的 token

> I love $MASK$ learning → deep

- 特点：

  - **双向上下文**

  - 擅长理解类任务（分类、匹配）

**BERT (Bidirectional Encoder Representations from Transformers)**:

- Token = Token Embedding + Position Embedding + Segment Embedding
- Two Tasks:
  - **Masked Language Model（MLM）**: 随机选 15% token, 其中：
    - 80% → `[MASK]`
    - 10% → 随机词
    - 10% → 保持不变
  - **Next Sentence Prediction（NSP）**: 输入句对 (A, B), 判断 B 是否是 A 的下一句
- Extensions:
  - **RoBERTa**: mainly just train BERT for longer and remove next sentence prediction!
  - **SpanBERT**: masking contiguous spans of words makes a harder, more useful pretraining task
- 2018 年，Google 提出（NAACL）; 几乎“统一”了 NLP 理解任务

#### Denoising AutoEncoder (Encoder-Decoder)

- 代表模型： BART / T5

- 做法：

  - 对输入加噪声（mask、打乱、删除）

  - 让模型恢复原始文本

- 这可以看作是：**Seq2Seq + 自监督**

**T5**:

- 模型不再“知道”任务类型 (QA、Translation、classification)，只学“如何把文本变成文本”

- **Span Corruption**：

  - 随机 mask **连续的 token span**
  - 用一个特殊 token 替代整个 span

  ```
  Input : The cat <extra_id_0> on the mat <extra_id_1>
  Target: <extra_id_0> sat <extra_id_1> slept
  ```

  - 比 BERT 的单点 MLM 更接近真实生成

#### Autoregressive LM (Decoder only)

$$
\max \sum_t \log p(x_t \mid x_1, \dots, x_{t-1})
$$

- 代表模型：GPT 系列 / LLaMA / Qwen

- 特点：

  - **只能看左侧上下文**

  - 擅长生成

**GPT**:

- GPT1: 

  - **两阶段流程：**
    - **无监督预训练**: Language Modeling
    - **有监督微调**: 每个任务一个 head

  - 系统性提出：**Pretrain once, fine-tune everywhere**
  - Decoder-only + causal attention

- GPT2: 

  - GPT-2 的核心不是新结构，而是 **Scaling + Data**
  - **Zero-shot 任务能力**: Language Models are Unsupervised Multitask Learners
  - **当语言模型足够大时，任务能力会“隐式”地从语言建模中学出来**

- GPT3: 

  - 规模爆炸
  - **Few-shot Learning**: 模型在 prompt 中“看几个例子”， 就能学会一个新任务
  -  **In-Context Learning (ICL)**: 模型在不更新任何参数的情况下，仅通过输入中的示例，在一次前向推理中学会执行新任务
    - **Transformer 在 forward pass 中， 可以实现类似“学习”的计算过程。**
    - 换句话说：
      - 模型参数：**固定**
      - 学习发生在：**隐状态（hidden states）中**
    - 在 GPT-3 之前：**任务 = 训练一个模型**;  在 GPT-3 之后：**任务 = 写一个 prompt**





## Post-Training

```
Pre-training
   ↓
SFT   ← 基础行为对齐
   ↓
RLHF / DPO ← 偏好优化
   ↓
Safety / Tool / Domain tuning
```

### Fine-tuning

- 在一个已经预训练好的模型基础上， 使用「特定任务 / 特定目标 / 特定数据」， 继续训练模型参数的过程。

#### SFT (Supervised Fine-Tuning)

- 使用**人工或高质量合成的「输入-输出对」数据**， 对预训练语言模型进行 **有监督学习**， 让模型学会“按指令生成理想回答”

给定指令 x 和答案 y：
$$
\mathcal{L}_{SFT}
= - \sum_{t=1}^{|y|} \log p_\theta(y_t \mid x, y_{<t})
$$

- teacher forcing
- cross-entropy loss
- 只在 target tokens 上计算

Fine-tuning不一定需要更新所有参数：

- **Full Fine-tuning**: 
  - 更新全部参数
  - 效果最好，成本最高
- **Parameter-Efficient FT（PEFT）**
  - LoRA: 低秩增量
  - Prefix / Prompt tuning: 只学 prompt
  - Adapter: 插小模块

#### Instruction-Following

Instruction-following 本质是教模型学习一种映射：
$$
(\text{instruction}, \text{optional input}) \;\rightarrow\; \text{desired output}
$$

- 仍然是 **标准的 next-token prediction**

- 但输入被**结构化为“任务指令 + 内容”**

- 模型被迫学会：

  - 区分 **任务类型**

  - 根据指令 **调整生成策略**

### RLHF(Reinforcement learning from human preferences)

- 不直接告诉模型“标准答案”， 而是让人类告诉模型 **“哪个回答更好”**， 再通过强化学习，让模型倾向于生成更受人类偏好的输出

- SFT 的局限：
  - 同一问题往往 **不存在唯一正确答案**
  - SFT 只能模仿标注答案，**难以表达“偏好”**

#### Core components of RLHF

- **Human Preference Data**: 数据形式不是 `(x, y)`，而是：

  ```
  Prompt: 解释什么是 Transformer
  Response A: ...
  Response B: ...
  
  Human label: A > B
  ```

  - 成本比 SFT 低
  - 信号更贴近真实偏好

- **Reward Model（RM）**

  - **目标**：把“人类偏好”变成一个可微的打分函数。

  - 给定 `(x, y⁺, y⁻)`：
    $$
    \mathcal{L}_{RM}
    = - \log \sigma(r_\phi(x, y^+) - r_\phi(x, y^-))
    $$

    - $r_\phi$：奖励模型
    - 输出一个 scalar reward

- **强化学习（PPO)**

  - **Policy**：当前语言模型
  - **Reward**：Reward Model 给出的分数
  - **约束**：不要偏离原模型太远（KL penalty）

  目标函数（简化）：
  $$
  \max_\theta
  \; \mathbb{E}[r_\phi(x, y)]
  - \beta \, \text{KL}(\pi_\theta \,\|\, \pi_{ref})
  $$

- θ=当前语言模型（policy）的所有参数（ RLHF 本质仍然是在“训练一个神经网络”， 只是优化目标不再是 cross-entropy）
- 期望E:
  - 从 prompt 数据集 $\mathcal{D}$ 中采样一个指令 $x$
  - 用当前模型 $\pi_\theta$ 生成一个完整回答 $y$
  - 对这些样本的 reward 和 KL 做平均
- ϕ=Reward Model 的参数, 越大 → 回答越符合人类偏好, PPO 的目标就是 **生成 reward 高的回答**
- $π_{ref}$ : SFT 后冻结的模型

```
Pre-trained LM
      ↓
SFT（得到初始 policy π₀）
      ↓
用 π₀ 生成回答 → 人类偏好 → 训练 RM
      ↓
固定 RM，用 PPO 优化 πθ（RLHF）
```

**Policy and Reference**:

- 可以把 reference model 理解为时间轴上某一时刻冻结下来的模型状态；在此基础上继续被优化、参数可变的模型就是 policy model

### DPO（Direct Preference Optimization）

- **把 RLHF 的“reward + KL 目标”， 转化成一个“纯监督学习的 loss”**
- 本质就是: **用 DPO 的 loss 继续优化同一套模型参数 $\theta$**

- DPO 的 loss（最小必要公式）

$$
\mathcal{L}_{\text{DPO}}
=
- \log \sigma
\Big(
\beta \big[
\log \pi_\theta(y^+|x)
-
\log \pi_\theta(y^-|x)
\big]
\Big)
$$

直觉解释：

- 希望模型 **更偏向 y⁺ 而不是 y⁻**
  - **$x$**：prompt / instruction
  - **$y^+$**：人类偏好的回答（chosen）
  - **$y^-$**：被拒绝的回答（rejected）
- 偏好差距越大，loss 越小

### IPO (Implicit Preference Optimization)

- DPO 的 logistic / sigmoid 形式并非必须，可以用更“直接、更稳定”的目标函数

- 或者说：**IPO不关心概率比例的精确形状，只关心：**
  $$
  \log \pi_\theta(y^+|x)
  -
  \log \pi_\theta(y^-|x)
  \quad\text{足够大}
  $$

- Loss Function:

$$
\mathcal{L}_{\text{IPO}}
=
\big(
\log \pi_\theta(y^+|x)
-
\log \pi_\theta(y^-|x)
-
\frac{1}{\beta}
\big)^2
$$