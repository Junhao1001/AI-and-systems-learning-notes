# MoE/GQA/Long Context

## MoE

核心思想：**不让所有参数都对每个 token 生效，而是让每个 token 只激活少量“专家”**

### Experts

- 每个 expert 本质上是一个 **子网络**
- 在 Transformer 中，通常是 **FFN（MLP）** 被替换为 MoE-FFN

```
Dense FFN:
x → W1 → activation → W2

MoE FFN:
x → Router → 选择 1~k 个 expert
             → Expert_i(x)
             → 聚合输出
```

- 一个 expert 的参数规模 **和 dense FFN 一样**

**Expert内部**：

一个 expert 通常是：
$$
\text{Expert}(x) = W_2 \, \sigma(W_1 x)
$$
其中：

- $W_1, W_2$ 都是 **稠密矩阵**
- 没有大量 0
- 没有掩码

**稀疏激活**：

- E 个 expert
- 每个 token 只用 k 个（通常 k=1 或 2）
- MoE 是 **扩展模型容量** 的方法

### Router / Gating Network

Router 是一个**小网络**，负责决定：

- 当前 token 该送给哪些 expert
- 每个 expert 的权重

形式上常见：
$$
p = \text{softmax}(W_r x)
$$
然后选：

- **Top-1** 或 **Top-2** expert

### Training

- **主任务损失**
  - 语言建模 loss / SFT loss
  - 反向传播时：只有被选中的 expert 会更新
- **负载均衡损失（MoE特有）**
  - 避免专家塌缩：**防止 Router 把大多数 token 都送到少数几个 expert**
  - 让 token 在 expert 之间分布得尽量均匀
  - 引入辅助损失：

$$
\mathcal{L}_{\text{balance}}
= E \sum_{i=1}^E f_i \cdot p_i
$$

- $p_i(x)$：Router 对 token x 分给第 i 个 expert 的概率

- $f_i$：第 i 个 expert 实际接收到的 token 比例

- 高概率 expert 的概率下降；低使用 expert 的概率上升

### 目的

- **MoE 不是为减少计算，而是为在“算力可承受范围内继续扩大模型能力”**

-  Dense Transformer 中：想要更强能力 → 只能：
  - 增大 d_model
  - 增加层数
- MoE 引入了一个新轴：**参数容量（#experts）**
- 增加 experts：
  - 参数量 ↑↑
  - **每 token 计算量 ≈ 不变（Top-k）**



## MQA/GQA

### KV cache

生成第 `t` 个 token 时：

- 输入：`x₁, x₂, …, x_t`
- Attention 本质上需要：
  - 当前 token 的 **Q**
  - 所有历史 token 的 **K / V**

**历史 token 的 K/V 在未来每一步都会反复用到**

因此，KV cache就是：**历史 token 的 Key / Value 一次算好，存起来，后续直接复用**

**具体步骤（计算第t个token）**:

1. **只计算当前 token 的 Qₜ, Kₜ, Vₜ**
2. **把 Kₜ, Vₜ 追加到 KV cache**
3. **Attention 计算时：**
   - `Qₜ` × `[K₁…Kₜ]ᵀ`
   - 加权 `[V₁…Vₜ]`

```
KV cache:
K = [K1 | K2 | ... | Kt]
V = [V1 | V2 | ... | Vt]

当前：
Q_t  ×  K_all
```

- 另外KV cache 几乎只用在**推理部分**（训练阶段能一次性看到完整的序列，所有的QKV是同时算的，无需KV cache)

### MHA(回忆)

标准 **Multi-Head Attention (MHA)**：

- 有 `H` 个 head
- 每个 head 都有 **独立的 Q / K / V**
- 计算：

$$
\text{Attn}_h = \text{softmax}\left(\frac{Q_h K_h^\top}{\sqrt{d}}\right)V_h
$$

- **KV cache 随 head 数线性增长**
- KV cache ≈ `O(H × seq_len × d_head)`
- GQA / MQA 的目标：**减少 KV 的数量，但尽量保留多 Query head 的表达能力**

### MQA(Multi-Query Attention)

- **所有 Query head 共享同一组 K / V**
- Query：仍然是 `H` 个 head（不同）
- Key / Value：**只有 1 组**

```
Q1  Q2  Q3 ... QH
 |   |   |     |
 +---+---+-----+
        ↓
      shared K, V
```


$$
\text{Attn}_h = \text{softmax}\left(\frac{Q_h K^\top}{\sqrt{d}}\right)V
$$

- 特点
  - **KV cache 减少 H 倍**
  - 推理速度显著提升（memory-bound 场景）
  - 非常适合 decoder-only LLM 推理

### GQA（Grouped-Query Attention）

- **Query head 分组，每一组共享一套 K / V**；这是 **MHA 和 MQA 的折中方案**

例如：

- Query heads：`H = 8`
- KV heads：`G = 2`
- 每 `4` 个 Q head 共享一组 KV

```
Q1 Q2 Q3 Q4 | Q5 Q6 Q7 Q8
  ↓           ↓
 KV1         KV2
```

对第 `g` 组：
$$
\text{Attn}_{h \in g} = \text{softmax}\left(\frac{Q_h K_g^\top}{\sqrt{d}}\right)V_g
$$

- 缺点在于 
  - KV 表达能力下降
  - 在复杂任务上可能损失一点性能



## Long context

- **Long Context**：模型能处理 / 生成很长的序列
- 其主要消耗在于：
  - **读 KV cache（显存带宽）**
  - 而不是 QKᵀ / softmax / matmul 的算力

Long context下的**KV cache 的规模**（以 MHA 为例）
$$
\text{KV size} =
2 \times L \times H \times T \times d_{head} \times \text{bytes}
$$

- `L`: layers（如 32）
- `H`: heads（如 32）
- `T`: context length（如 128k）
- `d_head`: 128
- fp16：2 bytes
- 非常容易到几十GB

解决思路：

减小KV cache的量：

- 前面提到的GQA/MQA
- **KV Cache Quantization**:
  - fp16 → int8 / int4
  - 减少显存 + 带宽

### Sliding Window Attention

- **当前 token 只关注最近 W 个 token，而不是全部历史**

```
|-------------------------------|  历史
                |------W------|  窗口
                         ↑
                       当前 token
```

------

Attention 计算变成：
$$
Q_t \times K_{t-W:t}
$$
而不是：
$$
Q_t \times K_{1:t}
$$

- 另外在Sliding Window 下，仍然会顺序存储KV
  - 只 *访问* 最近 W个KV 
    - 仍然全存，显存仍然线性增长
  - 或 *物理淘汰* 旧 KV （现存）
    - 显存始终为O(W)
- Sliding Window **会导致模型失去远程信息**
  - 但很多生成任务的局部依赖远大于全局依赖，所以模型仍能工作

### Hybrid Long Context

显示中常见的一些组合：

- **Full + Sliding Window**

  - 前 N tokens：全注意力（prompt）

  - 后续生成：sliding window

- **Global Tokens**

  - 少数 token（system / summary）永远可见

  - 其他 token 用 sliding window

- **Summary / Compression**

  - 旧 token → 压缩成 summary token

  - summary 进入 KV cache