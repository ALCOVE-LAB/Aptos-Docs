---
aip: 44
title: 模块事件
author: lightmark
Status: Accepted
type: Standard (Core, Interface, Framework)
created: 07/20/2023
---

[TOC]

# AIP-44 - 模块事件

## 一、概述

本改进提案（AIP）定义了一个模块级（Module-level）事件框架，其目标是替换掉目前使用的实例（Instance）事件框架。新的事件框架不再使用实例事件的 `EventHandle` 来操作，而是为每个事件流匹配一个固定的结构类型。



## 二、动机

事件已经在各种智能合约中被广泛使用，用于记录发生重大动作时的情况，而不是解析每个事务的输出以了解执行后的语义。但实例事件方案设计回到了 Libra/Diem 时代，这带来了许多与`EventHandle`相关的问题：

- 在使用之前需要创建
- 删除包含结构会删除句柄，缺乏该句柄使得SDK中的事件历史不可访问。
- `EventHandle` 在深层嵌入的数据结构内部是不可得的，尤其是涉及到了表格的场合。
- 事件序列号降低了系统的并行处理能力，并且对最终用户几乎没有益处。
- `EventHandle` 在某些情形下只部分依赖于地址来识别，这在语义上可能不够明确。
- 创建和删除涉及`signer`，这使得模块合约设计变得复杂。
- 在存储层面，唯一不能定制的就是次级索引。
- 由于事件句柄和事件随处存在于所有账户/对象中，导致数据碎片化。

模块事件的目标是解决所有上述问题。



## 三、影响

所有的 Aptos Move 开发者将受益于模块事件，并应该开始采用模块事件并废弃实例事件。



## 四、理由

要解决实例事件序列号影响并行性能的问题，一个可选方法是使用未更新的聚合器版本。然而，这样的改变对用户而言同样不明显。同时，这一方法也无法解决其他存在的问题。



## 五、规范

在Move智能合约级别，模块事件将被标识为具有`#[event]`属性的`struct`类型，该属性将由扩展类型检查器评估。

模块事件示例:

```rust
/// 一个示例模块事件结构，表示一次 coin 转账。
#[event]
struct TransferEvent<Coin> has store, drop {
  sender: address,
  receiver: address,
  amount: u64
}
```

要发出事件，将在事件模块中引入一个新的本机函数`emit`（或任何合适的名称）：

```rust
/// 在由T标识的事件流中以载荷`msg`发出事件。T必须具有`#[event]`属性。
public fun emit<T: store + drop>(event: T) {
    write_to_module_event_store < T > (event);
}
```

在存储级别上，不会添加新的表，因为我们将重用主事件表。

```rust
//! table_event_data
//! |<------key------>|<------------value--------------->|
//! | version | index | event_type_tag | bcs_event_bytes |
```

在API级别，将为模块事件引入新的API端点。在索引器支持之后，我们将为模块事件添加新的API。



## 六、风险和缺点

实例事件和模块事件必须长时间共存。需要进行大量工作来推广模块事件而不是实例事件。



## 七、未来潜力

Aptos官方索引器可以支持灵活的索引配置，以满足用户的需求。



## 八、致谢

感谢在本AIP的[讨论](https://github.com/aptos-foundation/AIPs/issues/200)中做出贡献的个人。



## 九、时间线

### 1. 建议的实施时间表

到第三季度末



### 2. 建议的开发者平台支持时间表

到第三季度末



### 3. 建议的部署时间表

v1.7



## 十、安全考虑

无