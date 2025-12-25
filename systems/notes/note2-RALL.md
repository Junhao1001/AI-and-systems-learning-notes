# RALL （Resource Acquisition Is Intialization）

**核心**：资源的生命周期 = 对象的生命周期

- **构造函数**：获取资源
- **析构函数**：释放资源
- 离开作用域时 → 析构函数**自动调用**

## Resources

RALL 不仅释放“内存”，还释放**其他一切资源**

- 内存：new / delete
- 文件：fopen / fclose
- 锁：lock / unlock
- socket：connect / close
- GPU / 设备：init / shutdown
- 事务：begin / commit / rollback

## RALL 的工作机制

- 栈对象作用域结束时必然析构

```
void foo() {
    Resource r;   // 构造：获取资源
    // ...
}              
```

- RAII **通常以类对象的形式存在**，因为 C++ 保证；当对象的生命周期结束（尤其是作用域结束）时，**其析构函数一定会被调用**
- 换句话说：**利用 C++ 对对象生命周期（构造 / 析构）的语言级保证， 将资源管理问题转化为对象生命周期管理问题**

```cpp
FILE* f = fopen("a.txt", "r");
if (!f) return;

// ... 使用 f

fclose(f);
```

- 对于上述例子，我虽然在编码上保证了资源的释放，但是仍然不满足RALL的思想

## RALL的经典标准库实现

- 智能指针

  - `std::unique_ptr`

  - `std::shared_ptr`

```
std::unique_ptr<Foo> p = std::make_unique<Foo>();
```

- 容器

  - `std::vector`

  - `std::string`

  - `std::map`

- 锁

  - `std::lock_guard`

  - `std::unique_lock`

- 文件流

  - `std::ifstream`

  - `std::ofstream`

## 栈对象

**栈对象**

```
void foo() {
    Foo a;
}
```

- 析构时机：**离开作用域立即析构**
- RAII 最理想的载体

**堆对象**：

```
Foo* p = new Foo();
delete p;
```

- 对象本身在堆上
- 生命周期由 `new / delete` 决定
- **不会自动析构**；极易泄漏

**静态对象**：

- 函数内静态

```
void foo() {
    static Foo s;
}
```

- 全局 / 命名空间静态

```
Foo g;
```

- 生命周期：程序开始 → 程序结束
- 析构：**程序退出时**

**成员对象**

```
struct A {
    Foo f;
};
```

- 析构时机： **随所属对象一起析构**

**临时对象（Temporary object)**

```cpp
Foo f = Foo();
```

- 生命周期通常是：表达式结束

- 受优化（RVO / NRVO）影响

- 析构一定发生，但时机不总是直观

**智能指针管理的对象**

```
std::unique_ptr<Foo> p = std::make_unique<Foo>();
```

- `Foo` 在堆上
- `unique_ptr` 是栈对象（或成员 / 静态）
- **真正 RAII 的是 `unique_ptr`**

## 构造函数和析构函数

- 构造函数是在对象“被创建时”自动调用的特殊成员函数， 用来完成对象的初始化与资源获取

- 构造函数需要**成员初始化列表**：

  - `const` 成员
  - 引用成员
  - 没有默认构造函数的成员
  - **性能更好（直接构造，而非先默认构造再赋值）**

- 析构函数是在对象“生命周期结束时”自动调用的特殊成员函数，用来完成资源释放和清理工作

  - 没有参数
  - 没有返回值
  - **每个对象只调用一次**
  - 析构函数 **不应抛异常**

- 构造 / 析构的调用顺序：**后构造的，先析构**

  ```
  {
      A a;
      B b;
  }
  ```

  顺序：

  ```
  A 构造
  B 构造
  B 析构
  A 析构
  ```

- 在继承关系中：

  ```
  Base 构造 → Derived 构造
  Derived 析构 → Base 析构
  ```



## 移动构造（Move Constructor）

**拷贝构造**：

- 用一个“已有对象”来构造一个“新的对象”，并复制其状态
  - 之前常用的赋值、传参、函数返回对象等
- 本质是复制资源或复制所有权

**移动构造：从一个“即将被销毁的对象”中，转移资源的所有权**

基本形式：

```cpp
T(T&& other);
```

- `other` 绑定到 **右值 / 临时对象**
- 表示：这个对象 **马上不用了**; 我可以“安全地掠夺资源”

**典型实现**:

```cpp
Buffer(Buffer&& other) noexcept
    : size(other.size), data(other.data) {
    other.data = nullptr;
    other.size = 0;
}
```

关键点：

- **不分配新内存**
- 只是转移指针
- 把源对象置为“可析构的安全状态”

### 右值绑定

简单的例子：

```
int&& r = 10;
```

这里：

- `10` 是一个**右值 / 临时对象**
- `r` 是一个**引用**
- `r` **绑定到** 这个临时对象

**没有发生赋值**；

可以理解为：给这个临时对象起了一个名字 `r`

### 生命周期的延长

```cpp
const Buffer& ref = Buffer(10);
```

- 在原来的理解里，临时对象在语句结束时就会结束; 这里的ref似乎也就失去了意义
- 然而有个规则：**绑定到 const 引用的临时对象，其生命周期延长到引用的作用域结束**
  - `Buffer(10)` 不会在语句结束时析构
  - 会在 `ref` 作用域结束时析构

对于普通的非const 定义

```cpp
Buffer&& r1 = Buffer(10);
```

**在这条语句结束时**：

- `Buffer(10)` 这个临时对象 **被析构**
- `r1` **仍然存在**, 但 `r1` 现在 **引用的是一个已经被析构的对象**
- **`r1` 变成了悬垂引用（dangling reference）**
- 这种写法不被推荐

### 右值绑定和普通引用的区别

- `const T&` 表达的是“只读观察”

- `T&&`获得的语义是：

  - ✅ 可以修改
  - ✅ 可以 move
  - ✅ 可以“掏空”资源
  - **`T&&` 表达的是“我即将消费你”**

- **`std::move(x)` 并不是一个新对象**

  ```
  std::move(x)
  ```

  - **不会创建新对象**
  - **不会产生独立实体**
  - 只是把 `x` **转换成一个右值表达式**

  等价于（本质）：

  ```
  static_cast<T&&>(x);
  ```

- **`std::move(x)` 不是“另一个东西”，它还是 `x`**

### move和赋值的差异 （内存分配角度）

#### 拷贝构造：

```cpp
Buffer a(10);
Buffer b = a;   // 拷贝
```

**内存步骤（必然）**：

1. 为 `b.data` **新分配一块堆内存**
2. 把 `a.data` 指向的数据：
   - `memcpy` / 逐元素拷贝
3. `b.data` 指向新的堆地址
4. `a` **完全不变**

```cpp
a.data ---> [ heap A ]
b.data ---> [ heap B ]  
```

**成本**：

- 一次 heap allocation

- 一次数据复制（O(n)）

#### 移动构造

```cpp
Buffer a(10);
Buffer b = std::move(a);   // 移动
```

**内存步骤（典型实现）**：

1. `b.data = a.data`
2. `b.size = a.size`
3. `a.data = nullptr`
4. `a.size = 0`

**内存示意图**：

```
a.data ---> nullptr
b.data ---> [ heap A ]   // 原来属于 a
```

成本：

- 几次指针 / 整数赋值
- **无分配**
- **无拷贝**