# 深度探索 CPP 对象模型

# 对象 *

## 使用对象封装带来的额外负担

使用对象封装并不会带来多大的成本：

+ data members 直接包含在 object 之中，就像 C struct 一样
+ 虽然 member function 会出现在 class 声明之中，但并非每一个 object 都有一个函数实体。如果 member function 是 non-inline member function (.h 和 .cpp 分离式)，则全局都只会有一个函数实体；如果其是 inline member function (即函数定义直接写在 class 内部)，则每一个模块中会含有一个函数实体（并不多啦）
+ 额外负担主要由 virtual 引起
  + virtual function 机制，即**执行期绑定开销**
  + virtual base class **虚基类的特殊处理开销**，使继承体系中出现多个 base class 时只有一个单一被共享的实体

## c++ 对象模型

cpp有 static 和 non-static 两种成员数据； static、non-static 和 virtual 三种成员函数，该如何用 object 存储他们呢：

1. non-static data members 直接在每一个 object 内部
2. static data members 是整个类共用的数据，放在 object 之外
3. 所有的 member function 都放置于 object 之外，由 类/虚函数表 寻找。

虚函数采取如下策略支持：

1. 每个 class 都会有一个虚函数表(vtbl)，存放了一堆指向 virtual function 的指针。虚函数表中还包含了 type_info object，用于运行时的类型识别(runtime type identification, RTTI)。
2. 每一个 object 都有一个指针(vptr)指向相应的虚函数表。这个 vptr 的赋值由类的构造、析构、assignment运算符完成。

![](/home/penguincat/Documents/SynologyDrive/NAS/notes/docs/cpp/深度探索cpp对象模型/1_3.png)

当涉及到继承的时候，该怎么办呢？

+ cpp 最初采用的策略是，没有任何间接性，base class object 的 data members 直接放置于 derived class object 中，这样对其的访问很高效，空间也很紧凑，但是如果 base class data members 有了任何改变，则所有与之相关的 derived class 都要重新编译。
+ cpp 11 开始，有了 virtual base class  概念，需要一些间接的 base class 的表现方法。在 object 中为每一个相关联的 virtual base class 加上一个指针。其也有一些变种，详见 3.4 节。

## 与 C 的一些差异

1. 以后自己用的时候，不用刻意去区分 class 关键字 或是 struct 关键字，没必要。
2. C struct 中，其 data 保证以声明顺序出现在内存布局当中。但是在 CPP 中，其仅保证同一个 access section 中的数据，必定以声明顺序出现在内存布局当中，但是不同 access section 中的各数据，排列顺序就不一定了。例如，protected data members 内部，是确定的，但是 protected data members 和 private data members 这两块之间的顺序，谁前谁后就不一定了。

## 对象、指针类型与多态

+ 一个 object 需要多少内存空间？一般而言要有：
  + non-static data members 的总和
  + 由于对齐(alignment) 所需填补的空间
  + 为了支持 virtual 产生的额外负担（如 vptr等）（或许cpp11开始，有 virtual base class 的相关指针？详见3.4？）
+ 不同类型的指针内容没什么不同，而是其所寻址出来的 object 类型不同，即指针类型会教导编译器，如何解释某个特定地址的**内存内容**及其**大小（指针所能涵盖的地址空间）**。
+ 加上多态之后：
  + 为什么指针和引用能够正确调用虚函数咧？（因为他们能通过 vtbl 找到对的函数）（引用通常以一个指针来实现）
  + 为什么直接将 derived object 赋值给 base object 对象时， base object 的 vptr 不指向 derived class 的 vtbl 呢？（因为编译器要保证，在初始化或assignment时，某 object 的 vptr 的内容不会因为 base class 的初始化而改变，这个 object 的类型应该是固定的定死的。这和下面提到的支持多态的思想是相符的）
  + 为什么直接将 derived object 赋值给 base object 对象时，不能调用子类实现的虚函数咧？（vtbl 找不到了啊！背后的原因是 --> 支持多态的思想是：改变指针（或引用）指向的内存的大小及其解释方式，而不是改变内存内容。基于这个思想，derived object 赋值给 base object 的时候，内存会被切割裁剪，以便能塞进 base type 的内存。此时 derived type 不会留下任何痕迹，编译器会回避 virtual 机制）

# 构造函数 语义学 

# Data 语义学 *

# Function 语义学 *

# 构造、解析、拷贝 语义学

# 执行器语义学

# 紫禁之巅 （错误处理、泛型、类型推断）

