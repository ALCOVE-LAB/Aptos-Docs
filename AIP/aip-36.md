---
aip: 36
标题: 全局唯一标识符
作者: satya@aptoslabs.com, ikabiljo@aptoslabs.com
讨论区 (*可选): https://github.com/aptos-foundation/AIPs/issues/154
状态: 草案
类型: 标准
创建日期: 06/01/2023
---

[TOC]

# AIP 36 - 全局唯一标识符

## 一、概述

这个AIP提议在 Aptos 框架中添加两个新的原生函数 `generate_unique_address_internal` 和 `get_txn_hash`。`generate_unique_address_internal` 为每个函数调用生成并输出一个唯一的 256 位标识符（地址类型）。`get_txn_hash` 函数输出当前交易的哈希。`generate_unique_address_internal` 函数调用可以高效并行运行。换句话说，当两个交易运行 `generate_unique_address_internal` 方法时，它们可以并行执行，没有任何冲突。最初，我们将这些唯一标识符作为新创建的 Move 对象的地址内部使用，特别是在 Token V2 铸币中。 我们还提议添加一个名为 `generate_auid` 的包装函数，该函数将 `generate_unique_address_internal` 生成的唯一地址包装在不可复制的 AUID 结构类型中，保证每两个 AUID 值实例都是不同的。



## 二、动机

有一个通用的需求，就是能够创建唯一的标识符或地址。Move 现在没有这样的实用程序，因此已经使用了各种替代方案，这带来了性能影响。我们想为所有需要它的用例提供这样的实用程序。

具体来说，当创建一个新对象时，我们需要将其与一个唯一的地址关联起来。对于命名对象，我们从名称中确定性地派生出它。但对于所有其他对象，我们当前是从我们即时创建的 GUID（全局唯一标识符）中派生出来的。GUID 由一个元组 (address, creation_num) 组成。我们通过让 address 成为对象或资源的创建者的账户地址，creation_num 是该账户创建的对象或资源的 guid 序列号，来创建 GUID。由于序列号 `creation_num` 必须为每个对象/资源创建递增，GUID 生成在同一地址内本质上是顺序的。例如，在 Token V2 中，每当使用 `token::create_from_account` 铸造新的 token 时，都会创建一个对象来支持它，该对象使用基于集合地址生成的 GUID ，因此来自同一集合的所有铸币都是本质上的顺序。

因此，这个 AIP 创建了一种新的标识符类型，称为 UUID（通用唯一标识符）。这是一个256位的标识符，它在所有账户为所有目的生成的所有标识符中是全球唯一的（并且与使用 Scheme 中的域分离生成标识符的其他方法不同，在 `authenticator.rs` 中）。
我们提议在 Aptos 框架中添加两个原生函数 `create_unique_address` 和 `get_txn_hash`。每次 MOVE 代码调用 `create_unique_address` 时，该函数都会输出一个全球唯一的 256 位值（地址类型）。该函数通过使用当前交易的哈希来创建一个唯一的地址。 由于暴露交易哈希可能在未来有更多的应用，我们也提议添加 `get_txn_hash` 函数，该函数返回当前交易的哈希。


我们还创建了 `object::create_object` 和 `token::create_token` 函数，利用新的实用函数，提供更高效的未跟踪的 token 铸币。这些函数废弃 `object::create_object_from_account`、`object::create_object_from_object` 和 `token::create_from_account` 函数。

我们还提议添加 `create_uuid` Move 函数，该函数调用 `create_unique_address` 函数并输出一个包含唯一地址的 UUID 结构。UUID 结构的地址字段不能被其他 Move 模块修改，也不能复制该结构。由于创建 UUID 结构的唯一方式是调用 `create_uuid` 方法，所以任何接收到 UUID 结构作为输入的代码都可以确信，底层地址是唯一生成的，没有两个 UUID 实例会匹配。

## 三、规范

Aptos 框架包含一个 `transaction_context` 模块，MOVE 模块可以使用它来检索与当前正在执行的交易相关的信息。当执行一个交易时，会话首先创建一个上下文。上下文包含了 `NativeTransactionContext` 等内容。`transaction_context` 模块提供了一个API接口，用于回答关于 `NativeTransactionContext` 的问题。 

对于这个AIP，当创建一个会话时，我们将在 `NativeTransactionContext` 中存储交易哈希。我们还会在上下文中存储一个 `uuid_counter`，并将计数器初始化为 0 。我们将在 `transaction_context` 模块中添加一个名为 `create_unique_address` 的原生函数。当调用时，`create_unique_address` 函数首先会增加 `uuid_counter`，并输出以下格式的值

```
output identifier = SHA3-256(txn_hash || uuid_counter || 0xFB)
```

为了进行域分离，我们在 `authenticator.rs` 的 `Scheme` 中添加了一个新条目 （`0xFB`），它被追加到 SHA3-256 输入的末尾进行域分离，确保生成的标识符与那里定义的机制和目的不同。公式等同于如何从 `Guid(txn_hash, uuid_counter)` 在`0xFB`命名空间中创建输出标识符。

SessionId 的哈希被用作 `txn_hash`。每个用户交易都包含发送者的账户信息和每个交易递增的序列号。对于 BlockMetadata 交易，使用 BlockData 的哈希（其中包含纪元和轮次号）。并且应该只有一个创世交易。 这意味着，每个被 Aptos 共识接受的交易都有唯一的字节集，因此产生唯一的交易哈希（假设没有 SHA3-256 碰撞）。

## 四、代替方案

进行域分离的替代位置可能是让每个调用站点进行自己的域分离。`object.move` 为现有的使用情况进行域分离，从它的角度看，保持 UUID 域分离可能更清晰。但这将需要所有其他调用者处理域分离，因此我们在 `object.move` 中添加了一个注释，明确指出相同的域分离正在原生代码中发生。 如果有人需要从一个与 `authenticator.rs` 不对齐的方法进行域分离，他们可以在返回的 uuid 之上进行额外的域分离。

上述 `create_unique_address` 函数的另一种替代方案，是暴露 `get_txn_hash` 和 `get_txn_unique_seq_num` 原生函数，并在 move 中进行哈希运算。

## 五、参考示例

此功能目前在 PR https://github.com/aptos-labs/aptos-core/pull/8401  中实现.

## 六、注释

上述函数生成的唯一标识符在它们自身（假设没有SHA3-256碰撞）或在 `authenticator.rs` 中捕获的生成唯一地址的其他方式（如所有当前的 GUID 流程）之间没有冲突。但它不是“防攻击者的”（当前的 GUID 流程也不是），即如果有人知道交易哈希，它可以预先运行并创建一个知道该对象地址的交易。因此，为了保证没有冲突，可以通过限制谁可以创建资源并不允许输入地址来提供这样的保证。（即，`object.move` 不提供一个函数在给定的输入地址上创建一个对象，它总是内部计算它）。

生成的标识符看起来是随机的，稍后检索它们的唯一方式是存储您需要访问它们的地址（或在索引期间使用事件进行链下分析）。相比之下，从名称创建 uuids 是一个可重复的过程，稍后可以查找，而无需索引句柄。出于这个原因，此时，我们正在替换 `object.move` 中的所有函数，除了 `create_named_object` 以使用上述实用程序。
