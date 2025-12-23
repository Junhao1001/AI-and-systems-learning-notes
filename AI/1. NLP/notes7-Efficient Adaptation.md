# Efficient Adaptation

## Prompting

- Prompting 是通过“精心设计或学习输入提示”，在不（或几乎不）更新模型参数的情况下，引导大模型完成特定任务的方法
- 个人理解：**一种在推理阶段，通过设计输入文本来引导模型完成特定任务的方式；它本质上是一种特定的提问 / 指令表达方法**

```
预训练 → 表示 &语言建模
          ↓
Instruction Following（SFT）
          ↓
模型能把 prompt 当任务
          ↓
Zero-shot 能力显现
          ↓
In-context learning
          ↓
Few-shot 能力显现
```

**Discrete Prompting**：

- **zero-shot**
- **few-shot**

**Soft Prompting**：

Prompt 不再是文本，而是**可学习向量**：
$$
[x_{\text{prompt}}, x_1, x_2, \dots]
$$

- Prompt 向量维度 = embedding 维度
- 只训练 prompt

**Prefix Tuning（Attention 级别 Prompt）**：

- 在 **每一层 attention** 的 K/V 前加前缀
- 比输入层 prompt 表达力更强

## PEFT (Parameter-Efficient Fine-Tuning)

核心思想：在微调大模型时，只训练“极少量新增或选定参数”，而**冻结绝大多数原模型参数**

```
PEFT
├── Additive（加参数）
│   ├── Adapter
│   ├── LoRA
│   └── Prefix / Prompt Tuning
├── Selective（选参数）
│   ├── BitFit
│   └── Partial Fine-tuning
└── Re-parameterization（重参数化）
    └── LoRA（也属于）
```

### Adapters

- 核心思想：在冻结大模型参数的前提下，**在每一层网络中插入一个小型可训练模块，用极少参数实现对新任务的适配**
  - 插入（insert）
  - 小模块（bottleneck）
  - 主干冻结（freeze backbone）

- 在Transformer中的位置
  - **FFN 后（最经典）**
  - Attention 后
  - 两者都插（表达能力更强，但参数稍多）

```
Self-Attn → Add&Norm → Adapter
FFN       → Add&Norm → Adapter
```

- **Basic Architecture**:

  - 输入维度：`d`（模型隐藏维度）
  - Adapter bottleneck 维度：`r`（r ≪ d）
  - 数学形式：

$$
\text{Adapter}(h) = W_{\text{up}} \, \sigma(W_{\text{down}} h)
$$

  - 并通过 **残差连接** 加回原表示。

<img src="./images/notes7-Efficient Adaptation/image-20251223171857701.png" alt="image-20251223171857701" style="zoom:80%;" />

### Sparse Subnetworks （Pruning）

- **Core Idea**: 指在一个大模型中，只使用部分神经元或权重来执行任务，从而在保留大部分参数的前提下，实现高效训练或推理

- 假设原始模型权重矩阵为 $W \in \mathbb{R}^{d_{\text{out}} \times d_{\text{in}}}$

- **定义稀疏掩码**：

$$
M \in \{0,1\}^{d_{\text{out}} \times d_{\text{in}}}
$$

  - 1 表示激活权重
  - 0 表示冻结权重（不更新、不计算）

- **稀疏子网络计算**：

$$
\tilde{W} = W \odot M
$$

  - 只有 $\tilde{W}$ 的非零部分参与前向和反向传播
  - 梯度只流向激活部分

#### 不同实现方式：

- **Static Sparsity**：

  - 在训练前确定哪些权重激活（例如 LTH）

  - 优点：训练中稳定，推理效率高

  - 缺点：需要预先找到合适子网络

- **Dynamic Sparsity**：

  - 每轮训练/每步迭代选择活跃子网络

  - 典型算法：SET、RigL

  - 优点：灵活，可能找到更好子网络

  - 缺点：实现复杂，需要动态 mask

- **Mixture of Experts (MoE)** 

  - 每次只激活部分专家网络

  - 这也是 Sparse Subnetwork 的一种“任务级稀疏”

Pruning 和 Sparse Subnetwork的关系：

- Sparse Subnetwork = Task-aware Pruning + 子网络选择

#### Pruning的分类

**Magnitude-based Pruning**：

- 核心思想：权重越小，对输出影响越小 → 可以剪掉
- 具体步骤：
  1. 训练模型
  2. 对权重按绝对值排序
  3. 保留 top-k %，剪掉剩余
- 特点：简单、高效
- 典型应用：LTH（Lottery Ticket Hypothesis）

**Gradient / Sensitivity-based Pruning**：

- 根据权重对任务 loss 的敏感性选择保留/剪掉
- 方法：

$$
\text{score}(w_i) = \left|\frac{\partial L}{\partial w_i} \cdot w_i \right|
$$

- 优点：更任务相关
- 缺点：计算梯度信息较昂贵

**Structured Pruning**：

- 不剪单个权重，而剪**整个神经元、通道或注意力头**
- 优点：推理加速明显
- 缺点：对性能影响较大，需要精细调节
