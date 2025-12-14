# Systems Roadmap

# 🧠 总目标（先写清楚）

> **能独立实现并优化：低比特（int8 / ternary / binary）矩阵乘与算子，并通过 PyTorch 调用**

------

# 第一阶段：C++ & 工程基础（必须扎实）

**目标：写出“不会成为性能瓶颈”的 C++**

## ✅ 1. C++ 核心（必须 100% 掌握）

-  指针 / 引用 / const 正确用法
-  RAII（构造 / 析构管理资源）
-  拷贝构造 vs 移动构造
-  `std::move` 的真实语义
-  `constexpr` vs `inline`
-  `noexcept` 对性能的影响

📌 **验收标准**

> 你能解释：为什么 kernel 代码里几乎不用 `new/delete`

------

## ✅ 2. STL（只学对 kernel 有用的）

-  `std::vector`（容量 / 对齐 / 连续内存）
-  `std::array`
-  `std::unique_ptr`
-  避免 `std::map` / `unordered_map` 在 hot path

📌 **验收标准**

> 你知道 `vector.reserve()` 为什么在 kernel 前很重要

------

## ✅ 3. 编译与构建（非常关键）

-  g++ / clang++ 常用参数
  - `-O0 / -O2 / -O3`
  - `-march=native`
-  CMake 基础（add_library / target_link_libraries）
-  Debug vs Release 区别

📌 **验收标准**

> 你能写一个最小 CMake 项目并切换 Debug / Release

------

# 第二阶段：内存 & 性能认知（分水岭）

**目标：你知道“慢”是怎么产生的**

## ✅ 4. 内存模型

-  stack vs heap
-  cache line（64B）
-  alignment（`alignas(64)`）
-  false sharing

📌 **验收标准**

> 你能解释为什么两个线程写不同变量也可能变慢

------

## ✅ 5. Profiling（不会 profile = 不会优化）

-  `perf stat`
-  `perf record / report`
-  基本 timing（但不迷信 chrono）

📌 **验收标准**

> 你能找出程序是 compute bound 还是 memory bound

------

# 第三阶段：CPU 架构 & SIMD（BitNet 核心）

**目标：开始写“像论文作者一样的代码”**

## ✅ 6. CPU 执行原理

-  pipeline
-  out-of-order execution
-  branch misprediction

📌 **验收标准**

> 你知道为什么 `if` 会让 kernel 变慢

------

## ✅ 7. SIMD（重点）

-  AVX2 基础
-  packed int8 / int16
-  `_mm256_load_si256`
-  `_mm256_maddubs_epi16`
-  bitwise ops（AND / XOR / SHIFT）

📌 **验收标准**

> 你能用 AVX2 写一个向量点积（int8）

------

# 第四阶段：CPU Kernel 实战（核心能力）

**目标：你已经能“造轮子”了**

## ✅ 8. GEMM 基础

-  naive GEMM（for-loop）
-  cache blocking
-  loop reorder
-  SIMD GEMM

📌 **验收标准**

> 你能解释为什么 GEMM 是一切的核心

------

## ✅ 9. 低比特计算

-  int8 GEMM
-  scale + accumulate
-  saturation / overflow

📌 **验收标准**

> 你能解释 int8 比 fp16 快在哪里、慢在哪里

------

## ✅ 10. Bit-level 计算

-  binary weight
-  ternary weight
-  bitwise accumulation

📌 **验收标准**

> 你能用 bitwise 操作模拟乘法

------

# 第五阶段：GPU（CUDA）

**目标：为后续 BitNet GPU kernel 做准备**

## ✅ 11. CUDA 基础

-  thread / block / grid
-  shared memory
-  warp 概念

📌 **验收标准**

> 你知道 global memory 为什么慢

------

## ✅ 12. CUDA 性能

-  memory coalescing
-  occupancy
-  bank conflict

📌 **验收标准**

> 你能解释一个 CUDA kernel 为什么只有 30% occupancy

------

## ✅ 13. CUDA 低比特

-  int8 Tensor Core（概念）
-  bitwise CUDA kernel

------

# 第六阶段：深度学习框架底层（非常重要）

**目标：你能把 kernel 接到模型里**

## ✅ 14. PyTorch C++ Extension

-  ATen Tensor
-  C++ binding
-  CPU / CUDA 双实现

📌 **验收标准**

> 你能在 Python 中 `import your_op`

------

## ✅ 15. 算子设计

-  forward kernel
-  backward kernel（理解即可）
-  memory layout 设计

📌 **验收标准**

> 你知道算子接口和 kernel 之间的边界

------

# 第七阶段：BitNet / Efficient AI 专项

**目标：对齐论文作者的思路**

## ✅ 16. 量化理论

-  per-tensor vs per-channel
-  symmetric quantization
-  scale / zero-point

------

## ✅ 17. BitNet 关键点

-  ternary weight 的统计分布
-  bitwise accumulation
-  memory footprint 分析

📌 **验收标准**

> 你能解释 BitNet 为什么在推理阶段省能耗

------

## ✅ 18. 真实项目阅读

推荐顺序：

-  int8 GEMM 实现
-  binary neural network
-  BitNet kernel 代码

📌 **验收标准**

> 你能改 kernel 并验证性能变化

------

# 🎯 最终毕业项目（强烈建议）

> **实现一个简化版 BitNet Linear Layer**

要求：

- ternary weight
- 自定义 C++ kernel
- PyTorch 调用
- benchmark 对比 fp16 / int8

------

## 给你一句实话（很重要）

> **做 Efficient AI 的人，最终拼的是：
>  硬件理解 × kernel 能力 × 模型直觉**

你现在这条路线是**极少数真正“吃得久”的方向**。