+++
title = "trait对象的静态分发与动态分发"
description = "在开发个人项目roaring-bloom-filter-rs时发现之前学习rust时略过的traits大有文章，虽然在编译器老师的指导下完成了代码，但是还是需要系统地补充理论知识。"
date = 2022-03-09T08:51:43+08:00

[taxonomies]
categories = ["Rust"]
tags = ["rust", "programming language"]

[extra]
toc = true
comments = true
+++

[rust by example]是这么定义trait的：

> A trait is a collection of methods defined for an unknown type: Self. They can access other methods declared in the same trait.

自然，我们就会需要传递“实现了某个trait”的struct这种范型功能。在rust中，提供了*两种方式*来实现这种能力，先引入一个trait和两个struct用于讲解后面的内容。

```rust
trait Run {
    fn run(&self);
}

struct Duck;

impl Run for Duck {
    fn run(&self) {
        println!("Duck is running");
    }
}

struct Dog;

impl Run for Dog {
    fn run(&self) {
        println!("Dog is running");
    }
}
```

## 静态分发和动态分发

首先引入分发(dispatch)：当代码涉及多态时，编译器需要某种机制去决定实际的调用关系。rust提供了两种分发机制，分别是静态分发(static dispatch)和动态分发(dynamic dispatch)。

### 静态分发

静态分发其实就是编译期范型，所有静态分发在编译期间确定实际类型，Rustc会通过单态化(Monomorphization)将泛型函数展开。

而静态分发有两种形式：

```rust
fn get_runnable<T>(runnable: T) where T: Run {
    runnable.run();
}

fn get_runnable(runnable: impl Run) {
    runnable.run();
}
```

两者在调用时都能通过

```rust
get_runnable(Dog {});
```

方式调用，区别在于前者可以使用`turbo-fish`语法（也就是`::<>`操作符）：

```rust
get_runnable::<Dog>(Dog {});
```

### 动态分发

首先引入trait对象(trait object)的概念：trait对象是指实现了某组traits的非具体类型值，这组trait一定包含一个[对象安全（object safe）](https://doc.rust-lang.org/reference/items/traits.html#object-safety)的基trait，和一些[自动trait（auto trait）](https://doc.rust-lang.org/reference/special-types-and-traits.html#auto-traits)。

在2021版本后，要求trait对象一定需要`dyn`关键字标识，以和trait本身区分开来。对于某个trait`Trait`，以下东西都是trait对象：

* dyn Trait
* dyn Trait + Send
* dyn Trait + Send + Sync
* dyn Trait + 'static
* dyn Trait + Send + 'static
* dyn Trait +
* dyn 'static + Trait.
* dyn (Trait)

动态分发也就是运行时范型，虽然trait对象是[Dynamically Sized Types(DST, 也叫unsized types)](https://doc.rust-lang.org/reference/dynamically-sized-types.html)，意味着它的大小只有运行时可以确定，意味着rustc不会允许这样的代码通过编译：

```rust
fn get_runnable(runnable: dyn Run) {
    runnable.run();
}
```

但是指向实现trait的struct的指针大小是一定的，因此可以把trait对象隐藏在指针后（如`&dyn Trait`，`Box<dyn Trait>`，`Rc<dyn Trait>`等），编译器编译时会默认对象实现了trait，并在运行时动态加载调用的对应函数。

```rust
fn get_runnable(runnable: &dyn Run) {
    runnable.run();
}
```

动态分发靠的就是指向trait对象的指针。

## 实现原理

### 静态分发

静态分发的实现原理比较简单，每多一种调用struct，rustc就会生成多一个函数：

```rust
fn get_runnable<T>(runnable: T) where T: Run {
    runnable.run();
}

fn main() {
    get_runnable::<Dog>(Dog {});
    get_runnable::<Duck>(Duck {});
}
```

通过编译后，`get_runnable`函数会生成两种：

```rust
fn get_runnable_for_dog(runnable: Dog) {
    runnable.run()
}

fn get_runnable_for_duck(runnable: Duck) {
    runnable.run()
}
```

rustc会自动将类型与调用函数匹配。

显而易见的，通过静态分发实现的多态无运行时性能损耗，但是编译出的二进制文件大小增加。

### 动态分发

动态分发就略复杂了，实现的关键在指针，每个指向trait对象的指针包含：

* 指向实现某个trait实例的指针
* 虚拟函数列表(virtual method table, 一般直接叫vtable)，包含
    - 某个trait和它父trait的所有函数
    - 指向这个实例对函数列表内函数的实现的指针

使用trait对象的目的是对方法的“延迟绑定（late binding）”，调用trait对象的某个方法最终在运行时才分发，也就是说调用时先从vtable中载入函数指针，再间接调用这个函数。对于vtable中每一个函数的实现，每个trait对象都可以不一样。

其实rust中`str`字符串类型和`[T]`数组类型都是trait对象。

## 对象安全

trait对象一定要基于[对象安全](https://doc.rust-lang.org/reference/items/traits.html#object-safety)的trait，这里不大谈特谈，只简单提及两个有趣的地方。

* `std::Sized`
    - 当不希望trait被用为trait对象时，可以加上`Self: Sized`的约束
    - 当不希望某个函数出现在trait对象的vtable中，可以加上`where Self: Sized`的约束
* trait对象的可分发函数不能有类型（范型）参数，所以如果trait中存在范型参数，只能静态分发了
   ```rust
   trait Run {
    fn run<T>(&self, t: T);
   }
   ```
* `Self`只能出现在方法的接受者（receiver）中，也就是方法的第一个参数，`&self`、`&mut self`...
