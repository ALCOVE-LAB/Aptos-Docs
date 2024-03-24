---
aip: 36
标题: 全局唯一标识符
作者: satya@aptoslabs.com, ikabiljo@aptoslabs.com
讨论区 (*可选): https://github.com/aptos-foundation/AIPs/issues/154
状态: 草案
类型: 标准
创建日期: 06/01/2023
---

# AIP 36 - 全局唯一标识符

## 摘要

这个AIP提议在Aptos框架中添加两个新的原生函数generate_unique_address_internal和get_txn_hash。generate_unique_address_internal为每个函数调用生成并输出一个唯一的256位标识符（地址类型）。get_txn_hash函数输出当前交易的哈希。generate_unique_address_internal函数调用可以高效并行运行。换句话说，当两个交易运行generate_unique_address_internal方法时，它们可以并行执行，没有任何冲突。最初，我们将这些唯一标识符作为新创建的Move对象的地址内部使用，特别是在Token V2铸币中。 我们还提议添加一个名为generate_auid的包装函数，该函数将generate_unique_address_internal生成的唯一地址包装在不可复制的AUID结构类型中，保证每两个AUID值实例都是不同的。

## 动机

有一个通用的需求，就是能够创建唯一的标识符或地址。Move今天没有这样的实用程序，因此已经使用了各种替代方案，这带来了性能影响。我们想为所有需要它的用例提供这样的实用程序。

具体来说，当创建一个新对象时，我们需要将其与一个唯一的地址关联起来。对于命名对象，我们从名称中确定性地派生出它。但对于所有其他对象，我们当前是从我们即时创建的GUID（全局唯一标识符）中派生出来的。GUID由一个元组(address, creation_num)组成。我们通过让address成为对象或资源的创建者的账户地址，creation_num是该账户创建的对象或资源的guid序列号，来创建GUID。由于序列号creation_num必须为每个对象/资源创建递增，GUID生成在同一地址内本质上是顺序的。例如，在Token V2中，每当使用token::create_from_account铸造新的token时，都会创建一个对象来支持它，该对象使用基于集合地址生成的GUID，因此来自同一集合的所有铸币都是本质上的顺序。

因此，这个AIP创建了一种新的标识符类型，称为UUID（通用唯一标识符）。这是一个256位的标识符，它在所有账户为所有目的生成的所有标识符中是全球唯一的（并且与使用Scheme中的域分离生成标识符的其他方法不同，在authenticator.rs中）。
我们提议在Aptos框架中添加两个原生函数create_unique_address和get_txn_hash。每次MOVE代码调用create_unique_address时，该函数都会输出一个全球唯一的256位值（地址类型）。该函数通过使用当前交易的哈希来创建一个唯一的地址。 由于暴露交易哈希可能在未来有更多的应用，我们也提议添加get_txn_hash函数，该函数返回当前交易的哈希。


我们还创建了object::create_object和token::create_token函数，利用新的实用函数，提供更高效的未跟踪的token铸币。这些函数废弃object::create_object_from_account、object::create_object_from_object和token::create_from_account函数。

我们还提议添加create_uuid Move函数，该函数调用create_unique_address函数并输出一个包含唯一地址的UUID结构。UUID结构的地址字段不能被其他Move模块修改，也不能复制该结构。由于创建UUID结构的唯一方式是调用create_uuid方法，所以任何接收到UUID结构作为输入的代码都可以确信，底层地址是唯一生成的，没有两个UUID实例会匹配。

## 规范

Aptos框架包含一个transaction_context模块，MOVE模块可以使用它来检索与当前正在执行的交易相关的信息。当执行一个交易时，会话首先创建一个上下文。上下文包含了NativeTransactionContext等内容。transaction_context模块提供了一个API接口，用于回答关于NativeTransactionContext的问题。 

对于这个AIP，当创建一个会话时，我们将在NativeTransactionContext中存储交易哈希。我们还会在上下文中存储一个uuid_counter，并将计数器初始化为0。我们将在transaction_context模块中添加一个名为create_unique_address的原生函数。当调用时，create_unique_address函数首先会增加uuid_counter，并输出以下格式的值

```
output identifier = SHA3-256(txn_hash || uuid_counter || 0xFB)
```

为了进行域分离，我们在authenticator.rs的Scheme中添加了一个新条目（0xFB），它被追加到SHA3-256输入的末尾进行域分离，确保生成的标识符与那里定义的机制和目的不同。公式等同于如何从Guid(txn_hash, uuid_counter)在0xFB命名空间中创建输出标识符。

SessionId的哈希被用作txn_hash。每个用户交易都包含发送者的账户信息和每个交易递增的序列号。对于BlockMetadata交易，使用BlockData的哈希（其中包含纪元和轮次号）。并且应该只有一个创世交易。 这意味着，每个被Aptos共识接受的交易都有唯一的字节集，因此产生唯一的交易哈希（假设没有SHA3-256碰撞）。

## 代替方案

进行域分离的替代位置可能是让每个调用站点进行自己的域分离。object.move为现有的使用情况进行域分离，从它的角度看，保持UUID域分离可能更清晰。但这将需要所有其他调用者处理域分离，因此我们在object.move中添加了一个注释，明确指出相同的域分离正在原生代码中发生。 如果有人需要从一个与authenticator.rs不对齐的方法进行域分离，他们可以在返回的uuid之上进行额外的域分离。

上述create_unique_address函数的另一种替代方案，是暴露get_txn_hash和get_txn_unique_seq_num原生函数，并在move中进行哈希。

## 参考示例

此功能目前在PR https://github.com/aptos-labs/aptos-core/pull/8401 中实现.

## 注释

上述函数生成的唯一标识符在它们自身（假设没有SHA3-256碰撞）或在authenticator.rs中捕获的生成唯一地址的其他方式（如所有当前的GUID流程）之间没有冲突。但它不是“防攻击者的”（当前的GUID流程也不是），即如果有人知道交易哈希，它可以预先运行并创建一个知道该对象地址的交易。因此，为了保证没有冲突，可以通过限制谁可以创建资源并不允许输入地址来提供这样的保证。（即，object.move不提供一个函数在给定的输入地址上创建一个对象，它总是内部计算它）。

生成的标识符看起来是随机的，稍后检索它们的唯一方式是存储您需要访问它们的地址（或在索引期间使用事件进行链下分析）。相比之下，从名称创建uuids是一个可重复的过程，稍后可以查找，而无需索引句柄。出于这个原因，此时，我们正在替换object.move中的所有函数，除了create_named_object以使用上述实用程序。
