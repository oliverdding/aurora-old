+++
title = "通过例子快速入门C++"
description = "[原文链接](http://www.tfd.chalmers.se/~hani/kurser/OS_CFD_2007/c++.pdf)"
date = 2022-03-07T14:37:06+08:00

[taxonomies]
categories = ["C Plus Plus"]
tags = ["c plus plus", "programming language"]

[extra]
toc = true
comments = true
+++

:warning: 本文为阅读笔记，[原文](http://www.tfd.chalmers.se/~hani/kurser/OS_CFD_2007/c++.pdf)为Tom Anderson对C++的**子集**的介绍。

在学习前需要解释的一点是，C++的内容可以划分为：

1. 基础部分，包含类、成员函数、构造器等等
2. 进阶部分，包含单例继承、模版等等
3. 垃圾部分，包含多重继承、异常、超载、引用等等

1是本文重点，每个C++程序员都应该掌握；2则是看到再查，但避免在自己的程序中使用；3则永远不要碰。

---

## C++包含C

很大程度上，C++是C的超集，大部分符合ANSI的C程序都可以被C++编译器编译。不过有以下几个地方需要注意：

1. 所有函数都要先声明后使用。（在C中使用未声明的函数默认返回值是int）
2. 所有函数的声明和定义头必须使用new-style声明，如

   ```c++
   extern int foo(int a, char* b);
   ```

   （在C中可以`extern int foo();`表示函数foo未知的参数数量和类型）
3. 需要链接C的对象文件时，需要这样声明C函数：

   ```c++
   extern "C" int foo(int a, char* b);
   ```
4. C++包含需要新关键词，不能被用作标识符（identifiers）。它们是：`new`，`delete`，`const`和`class`

## 基础部分

在开始讲解之前，下面三组面对对象语言的通用概念必须了解：

* **class** and **object**
* **member functions**
    + **constructor**
    + **destructor**
* **private** vs. **public**

接下来用例子描述C++中的各类基础特性。

### 类

为了直观展示C++中的类使用，将以栈类的代码实现为例子。

在C++中，类的定义和实现是分离的（有种特殊情况，后文提到）。通常，在`.h`头文件中存放类的定义，在`.cc`文件中存放类的成员函数实现。

```c++
// stack.h
class Stack {
        int* stack;            // 1.1
    public:                    // 1.2
        Stack(int sz);         // 1.3
        ~Stack();              // 1.4
        void Push(int value);  // 1.5
        bool Full() const { return (top == size); };     // 1.6
    private:                   // 7
    private:                   // 7
        int top;
        int size;
}
```

#### 成员函数

在stack.h头文件中*1.1*就是`Stack`类的普通成员函数。我们给出它的实现：

```c++
// stack.cc
void
Stack::Push(int value) {         // 2.1
    ASSERT(!this->Full());       // 2.2
    stack[this->top++] = value;  // 2.3
}
```

首先看到*2.1*，*class::function*的标识符表明`Push`是类的成员函数。

在*2.2*中，成员函数体内可以直接使用对象的成员，这里访问了`Full()`函数。C++可以通过`->`操作符访问对象的成员，同时，在成员函数内可以通过`this`变量指代当前对象，通常`this`是隐式的，因此可以改写成`ASSERT(!Full());`，*2.3*即是隐式调用对象成员。

#### 私有成员

面对对象编程自然必须涉及封装。C++提供了私有成员的方式实现封装。默认情况下C++类成员为私有成员，如*1.1*就是私有的，成员函数内可以随意使用，但外部不可见。

可以显示使用`public`关键词表明后续成员为公有成员，如*1.2*所示。

对之对应，可以使用`private`关键词表明后续成员为私有成员。

:heart: 所有包含数据的成员变量都应当私有。

#### 构造函数和new操作符

```c++
// stack.cc
Stack::Stack(int sz) {
    size = sz;
    top = 0;
    stack = new int[size];  // 3.1
}
```

*1.3*即是Stack类的构造函数，构造函数用于给对象成员赋予默认值，并进行一些初始化操作。在C++中，使用`new`关键词进行对象创建，它会自动调用类的构造函数。这个流程同样适用于自动变量。

对于类来说构造函数是必须的，若没有显示定义，编译器会给你加上一个空构造函数。

有两种方式传递函数参数给构造函数：

```c++
void
test() {
    // 通过new关键词（堆上变量）
    Stack* s1 = new Stack(128);  // 4.1

    // 自动变量或全局变量（栈上变量）
    Stack s2(128);               // 4.2
}
```

函数内的变量在函数返回时会自动释放，比如*4.2*定义的s2就会在test()函数返回时释放。但是通过new关键词创建的变量（如*4.1*)存储在堆（heap）上，即是函数返回仍然存在，必须手动[delete](#析构函数和delete操作符)。

new关键词也可以用来分配数组，如*3.1*所示。同样需要手动delete。

#### 析构函数和delete操作符

```c++
Stack::~Stack() {
    delete [] stack;  // 5.1
}
```

有构造函数就有析构函数，就如有new就有delete。

对于new出来的对象，使用delete手动释放，如*4.1*的s1：`delete d1;`。

对于new出来的数组，也使用delete手动释放，如*5.1*所示。注意中间的`[]`表明释放数组，而不是数组中单个变量。

### 其他C++的基础特性

1. 类可以作为数据类型使用。
2. 事实上可以在类是声明时（头文件）内同时声明实现函数，如*1.6*所示。但通常要求函数体简单。这么做有两个好处：方便和性能。在类声明时实现的函数会被当作`inline`函数。
3. const关键词除了像C一样声明变量时使用，表示变量不可变，还可以用于函数声明，表明函数对数据只读而不会修改，如*1.6*。
4. `cin`和`cout`对象是C++中的标准输入输出。使用`>>`和`<<`分别输入、输出。

## 进阶部分

这部分内容容易误用，但如果用得恰当可以大幅简化逻辑。

后面内容由于博主没有使用场景，暂时不学习。:stuck_out_tongue:

### （单例）继承

### 模版

## 垃圾部分
