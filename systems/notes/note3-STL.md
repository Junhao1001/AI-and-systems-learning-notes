# STL (Standard Template Library)

**STL 是一套基于模板的、可复用的通用算法 + 数据结构 + 迭代器框架**

## 核心构成

**容器（Containers）**：负责存储容器

- 顺序容器：`vector`, `deque`, `list`, `array`
- 关联容器（有序）：`map`, `set`, `multimap`, `multiset`
- 无序容器（哈希）：`unordered_map`, `unordered_set`
- 适配器：`stack`, `queue`, `priority_queue`

**算法（Algorithms）**：负责操作数据

- 在 `<algorithm>` 中，几乎全部是 **模板 + 迭代器**

  ```cpp
  std::sort(v.begin(), v.end());
  std::find(v.begin(), v.end(), 10);
  std::count(v.begin(), v.end(), 5);
  ```

  - **不关心容器类型**
  - 只关心：
    - 迭代器区间
    - 操作规则

**迭代器（Iterators）**

- 容器和算法之间的“接口”

```
auto it = v.begin();
++it;
std::cout << *it;
```

- 可以理解为：迭代器是“泛化版指针”

**函数对象 & Lambda**

- STL 中大量使用 **可调用对象（Callable）**

```cpp
struct Greater {
    bool operator()(int a, int b) const {
        return a > b;
    }
};

std::sort(v.begin(), v.end(), Greater{});
```

## Vector

- `std::vector` 是一个“连续内存 + 自动扩容 + RAII 管理”的动态数组
- 其有如下特点：
  - 连续内存
  - 随机访问 O(1)
  - 自动管理内存（构造 / 析构）
  - 扩容时涉及 **move / copy**

### 内存模型

```
data() ────────────────► [ T | T | T | T |   |   |   ]
                         ↑             ↑
                       begin()        end()
                                     capacity()
```

- **size()**：已使用元素个数
- **capacity()**：已分配空间容量
- **data()**：首元素地址（连续内存

### 初始化

```
std::vector<int> v1;              // 空
std::vector<int> v2(5);           // 5 个 0
std::vector<int> v3(5, 42);       // 5 个 42
std::vector<int> v4 = {1,2,3};    // 初始化列表
```

### 常用操作

- 访问元素

```cpp
v[0];        // 不检查越界
v.at(0);     // 检查越界，抛异常
v.front();
v.back();
```

- 插入元素

  - `push_back`:

    ```cpp
    T x;
    v.push_back(x);        // 拷贝
    v.push_back(std::move(x)); 
    ```

  - `emplace_back`

    ```cpp
    v.emplace_back(10, 20);
    ```

    - 原地构造，避免临时对象
    - 对复杂类型优先使用`emplace_back`

- 扩容：

  - 分配 **更大的新内存**
  - *move / copy** 旧元素到新内存
  - 释放旧内存
  - 通常是旧容量的1.5~2倍

- `reversze`和`resize`

  - `reserve(n)`
    - 只分配内存
    - **不构造元素**
    - size 不变
  - `resize(n)`
    - 改变 size
    - 会构造 / 析构元素

- 迭代器&失效

  ```cpp
  auto it = v.begin();
  v.push_back(x);
  ```

  - 迭代器会失效：
    - 扩容时：**全部失效**
    - `erase`：被删元素之后的迭代器失效

## Array

- `std::array<T, N>` 是“具有对象语义的、固定大小的连续数组”
  - 固定大小（编译期确定）
  - 连续内存
  - 是对象（不是退化成指针）
  - 零运行时开销

- 内存位置：

  ```cpp
  std::array<int, 4> a;
  ```

  - 通常在 **栈上**
  - 连续内存
  - 生命周期 = 所在作用域

- 基本操作：和vector基本一样
- 使用场景：
  - 元素个数 **固定**
  - 小数组
  - 性能敏感
  - 作为类成员



## Unique_ptr

- `std::unique_ptr<T>` 是“独占所有权”的智能指针
  - 同一时间 **只有一个** `unique_ptr` 拥有这块资源
  - 析构时自动释放
  - 不可拷贝，只能move

### Ownership

- **独占**

  ```cpp
  std::unique_ptr<int> p1 = std::make_unique<int>(10);
  // std::unique_ptr<int> p2 = p1;  //  编译错误
  ```

  - **不能拷贝**

- **只能move**

  ```
  std::unique_ptr<int> p2 = std::move(p1);
  ```

  - `p2` 拥有资源
  - `p1 == nullptr`

### 常用接口

```cpp
auto p = std::make_unique<Foo>(args...); // 创建指针
p.get();      // 返回裸指针（不转移所有权）
p.release();  // 放弃所有权（你负责 delete）
p.reset();    // 删除当前对象，接管新对象
p.reset(new Foo());

*p;           // 解引用
p->method();  // 成员访问
```

### 常见用法

- 作为参数：表达“转移所有权”

  ```cpp
  void take(std::unique_ptr<Foo> p);
  
  take(std::move(p));  // 明确转移
  ```

  - **调用后你不再拥有这个对象**

- 作为返回值

  ```cpp
  std::unique_ptr<Foo> create() {
      return std::make_unique<Foo>();
  }
  ```

  - 自动 move
  - 没有性能损失


unique_ptr **可以认为是一个「拥有（own）指针资源的对象」**， 而不是一个“更安全的指针”

- 在指针本身析构时，会释放其所拥有的指针资源，不会存在**悬空指针问题**



# 微基准与性能直觉

## Chrono/Clock

- `std::chrono` 是一个**用“类型系统”来严格表示时间点和时间长度的标准库**

### duration:

本质：

```
duration<Rep, Period>
```

- `Rep`：数值类型（int / long / double）
- `Period`：单位（1ms、1s、1ns）

例子：

```
std::chrono::milliseconds ms(100);
```

### time_point

- 表示 **“某一刻”**

```cpp
auto t = std::chrono::steady_clock::now();
```

- 直觉理解：**当前这一刻**

- **不能直接打印**，但可以：

```cpp
auto elapsed = t2 - t1; 
```

### clock

c++标准提供三种主要clock:

| clock                   | 特点                 | 用途                    |
| ----------------------- | -------------------- | ----------------------- |
| `system_clock`          | 系统时间             | 显示当前时间            |
| `steady_clock`          | 单调递增，不会回拨   | **性能计时首选**        |
| `high_resolution_clock` | 最高精度（实现相关） | 有时等于 `steady_clock` |

### duration_cast

计算时间长度时需要 **显式转换单位**：

```cpp
auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(d);
```

否则：

- 精度不明确
- 容易出现隐式截断



## Debug/Release

核心区别

| 维度          | Debug         | Release         |
| ------------- | ------------- | --------------- |
| 优化          | ❌ 基本不开    | ✅ 大量开启      |
| 断点 / 单步   | ✅ 精确        | ❌ 不可靠        |
| 变量是否保留  | ✅             | ❌ 可能被优化掉  |
| 边界检查      | ✅（STL 常有） | ❌               |
| 断言 `assert` | ✅             | ❌（通常被移除） |
| 性能          | 非常慢        | 非常快          |

- 总结来说Debug 方便调试、断点、检查；Release 让代码尽可能跑的快

### Release优化

- 函数内联：函数调用语句可能直接被展开

- Dead Code Elimination: 

  ```
  int x = 10;
  x += 5;
  ```

  如果 `x` 没被用：就会删掉争端代码

- 循环优化：

  ```
  for (int i = 0; i < 1'000'000; ++i);
  ```

  如上代码，并没有具体执行语句；**此处整个循环会被优化为什么都不做**

- 常量折叠 / 表达式预计算：

  ```cpp
  int x = 3 * 5; -> int x = 15; //release优化
  ```

- STL的Debug检查被移除：
  - Debug下`vector` 有：
    - 越界检查
    - iterator 合法性检查
  - Release下都会被移除

### `-O0 / -O2 / -O3` 的区别

`-O0`（No Optimization）：

- 编译器会有如下行为

  - 不内联

  - 不重排代码

  - 不删除“看似无用”的代码
  - 不合并表达式

  - 变量、行号基本保持

- 使用于：

  - 单步调试/看变量变化

  - **但不能用于性能测试**

`-O2`（Strong Optimization）

- 编译器开启大类优化：
  - inlining
  - 循环优化
  - 死代码消除
  - CSE
- **生产环境较为常用**

`-O3`（Aggressive Optimization）：

- 更为激进的一些优化：
  - 激进的循环展开
  - 更积极的向量化
  - 可能导致过度内联
- 风险点：不一定更好
  - 编译时间更长
  - 可执行文件更大
  - **有时反而更慢**



# 内存模型

- c++程序的经典布局（之前已经学习过）

  ```
  高地址
  ┌──────────────────┐
  │      Stack       │ ← 局部变量、函数调用
  ├──────────────────┤
  │      Heap        │ ← new / malloc
  ├──────────────────┤
  │   BSS Segment    │ ← 未初始化的全局/静态变量
  ├──────────────────┤
  │  Data Segment    │ ← 已初始化的全局/静态变量
  ├──────────────────┤
  │   Text Segment   │ ← 代码
  └──────────────────┘
  低地址
  ```

- 对于其中堆和栈的区别，主要区别如下：

| 对比点       | Stack（栈）            | Heap（堆）             |
| ------------ | ---------------------- | ---------------------- |
| 分配方式     | 编译器自动分配         | 程序员/运行时显式分配  |
| 释放方式     | **自动**（作用域结束） | **手动 / 智能指针**    |
| 生命周期     | 作用域级别             | 可跨作用域             |
| 访问速度     | 快                     | 相对慢                 |
| 是否容易出错 | 很安全                 | 容易泄漏/悬垂          |
| 常见用法     | 普通变量、局部对象     | 大对象、共享对象、多态 |

### 栈

- **栈是由编译器自动管理的内存区域**，函数调用时创建，函数返回时整体销毁

- 栈创立的对象的生命周期 = 作用域

  ```cpp
  {
      int a = 5;
  } // a 在这里必然被销毁
  ```

- **分配/释放极快**： 栈本质是 **移动栈指针**

- 大小有限：

  - 每个线程的栈大小有限（通常几 MB）
  - 递归太深 会导致 **stack overflow**

- 通常适用于

  - 局部变量
  - 临时对象
  - 不需要跨作用域的对象
  - 生命周期非常清晰的资源

### 堆

- **堆是运行时动态分配的内存区域**，生命周期由程序控制
  - 如果没有手动`delete`,可能会导致**内存泄漏**
- 生命周期灵活:
  - 对象可以跨函数、跨作用域
- 分配、释放较慢：
  - 需要内存管理器（malloc/free 或 new/delete）
  - 可能产生 **内存碎片**
- 现代C++的堆：**RALL化**
  - 尝试使用智能指针

### 从汇编角度看栈和堆（后续补充）