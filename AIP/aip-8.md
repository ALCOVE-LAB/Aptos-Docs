---
aip: 8
title: 集合类型的高阶内联函数
author: wrwg
discussions-to: https://github.com/aptos-foundation/AIPs/issues/33
Status: 已接受
last-call-end-date: 待定
type: 标准（框架）
created: 2023年1月9日
updated: 2023年1月9日
---

[TOC]

# AIP-8 - 集合类型的高阶内联函数

## 一、概述

Move 语言在目前的版本里不支持诸如高阶函数（可以作为参数或结果的函数）和动态分发（根据对象类型动态调用对应方法）的特性，因为这些概念在 Move 字节码和虚拟机层面都不受支持。不过，最近 Aptos Move 引入了一个新概念，即*内联函数*，这类函数在编译期就进行展开，因此不在 Move 的字节码中存在。这让它们能实现一项特性，即接受函数作为参数，这些函数可由 lambda 表达式（匿名函数）给出。基于此，我们可以为 Move 的集合类型定义一系列像 'for_each'、'filter'、'map' 和 'fold' 这样的高阶函数。在本次改进提案（AIP）中，我们建议为这些函数制定一系列标准化的约定。



## 二、动机

高阶函数因其能使操作集合的代码变得更简洁准确而广为人知。这类函数在许多主流编程语言中都非常流行，如 Rust、TypeScript、Java 和 C++ 等。



## 三、原理

目前，Move 缺少特性（trait）机制，因此无法定义一个 `Iterator` 类型，该类型旨在融会贯通不同集合类型所支持的功能函数。我们的目标是至少为其中一些最普遍的功能函数建立统一的命名和语义标准。这将使得框架的构建者明确需要提供哪些函数，同时让开发人员记住可用的函数，以及帮助审计人员清晰理解代码内的相关语义。



## 四、规范

### 1. Foreach （对于每个）

每个可迭代的集合应该提供以下三个函数（以 `vector<T>` 类型为例）：

```move
public inline fun for_each<T>(v: vector<T>, f: |T|);
public inline fun for_each_ref<T>(v: &vector<T>, f: |&T|);
public inline fun for_each_mut<T>(v: &mut vector<T>, f: |&mut T|);
```

这些函数将会按该集合的特定顺序迭代集合。第一个函数执行后集合就不再可用，第二个函数允许你引用（查看元素内容而不做改动），而第三个则可用于修改元素的值。下面是一个 `for_each_ref` 的使用示例：

```move
fun sum(v: &vector<u64>): u64 {
  let r = 0;
  for_each_ref(v, |x| r = r + *x);
  r
}
```

### 2. Fold、Map 和 Filter

每个可迭代的集合 *应该* 提供以下三个函数，这些函数将集合转换为相同类型或不同类型的新集合：

```move
public inline fun fold<T, R>(v: vector<T>, init: R, f: |R,T|R ): R;
public inline fun map<T, S>(v: vector<T>, f: |T|S ): vector<S>;
public inline fun filter<T:drop>(v: vector<T>, p: |&T|bool) ): vector<T>;
```

为了解释 `fold` 和 `map` 函数的意义，我们将通过展示后者（`map`）是怎样利用前者（`fold`）定义出来的，以阐述它们之间的联系：

```move
public inline fun map<T, S>(v: vector<T>, f: |T|S): vector<S> {
    let result = vector<S>[];
    for_each(v, |elem| push_back(&mut result, f(elem)));
    result
}
```

### 3. 在 Aptos 中受影响的数据类型

Aptos 框架中的这些数据类型应该获取高阶函数（待完成）：

- Move stdlib （标准库）
    - vector
    - option
- Aptos stdlib（标准库）
    - simple map
    - ？
- Aptos framework（框架）
    - ？
- Aptos  tokens
    - property  map
    - ？


### 4. 注意事项

- 建议在 Aptos 框架之外的集合类型使用相同的约定。
- 只有在单个交易中可迭代的集合类型应该提供这些函数。例如，表格不是这样的集合。
- 只有那些在单个交易中可以被迭代的集合类型才应该提供这些函数。例如，表（tables）就不属于这类可迭代的集合。
- 一些集合类型可能会通过添加更少或更多的这些函数而发生分歧，这取决于集合的类型。



## 五、参考实现

TODO：一旦 PR 合并，就在框架的 `main` 分支中链接到代码。



## 六、风险和缺陷

目前没有明显的风险或缺陷。



## 七、未来潜力

- Aptos 框架的部分可以进行重写，使其更易于审计，方法是通过删除低级别的循环并将其替换为对高阶函数的调用。
- Move Prover 可以从这些函数中受益，以避免使用低级别的循环，这是使用证明器时最具挑战性的部分之一。
- 未来，Move VM 也可能支持函数参数。那么，此提案将简单地泛化为集合上的高阶函数，便不一定需要是内联函数了。



## 八、建议的实施时间表

Move 编译器的实现已经在 [PR 822](https://github.com/move-language/move/pull/822) 中完善了功能。剩下的工作量很小，所以我们希望能够在下一个版本中完成这项工作。