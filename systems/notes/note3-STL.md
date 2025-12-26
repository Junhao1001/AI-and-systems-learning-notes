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



