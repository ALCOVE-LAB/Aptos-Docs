---
aip: 18
title: 将 SmartVector 和 SmartTable 引入 aptos_std
author: lightmark
Status: Accepted
type: Standard (Framework)
created: 03/07/2022
---

[TOC]

# AIP-18 - 将 SmartVector 和 SmartTable 引入 aptos_std

## 一、概述

本 AIP 提议将两种存储效率高的数据结构引入 Aptos 框架。总体而言，这两个结构可以通过将多个元素打包到一个存储槽中，而不是每个槽（slot）一个元素，来降低存储占用。



## 二、动机

Move 并不难学习。但是 Move 和基础设施之间的复杂性并不那么直观，比如 Gas 调度的工作，包括存储和执行。具体来说，Move 中的数据结构如何存储在存储中，数据如何表示，布局是什么样的，这些都不是很清楚。鉴于在 Aptos 的各种生态系统项目中见证了许多对 `vector` 和 `table`，即我们的序列和关联容器类型的误用，我们非常清楚由于缺乏对包括执行和存储在内的 Gas 调度的理解，大多数在 Aptos 上的 Move 开发人员无法编写最有效的智能合约代码以进行 Gas 优化。这导致了：

1. 一些项目抱怨  Gas 收费比预期的更昂贵。
2. 人们滥用 `Table`，这是我们长期内试图避免的，尤其是对于小状态存储。

因此，我们计划提供一个一劳永逸的解决方案，同时适用于 `vector` 和 `table` 数据结构，以更优化的方式处理数据扩展问题，考虑到存储模型和 Gas 调度。因此，大多数开发人员不必过多关注不同容器类型之间的 Gas 成本。相反，他们可以更多地关注产品逻辑方面。



## 三、基本原理

设计原则是将更多的数据放入一个槽中，而不会引起显著的写放大。

- SmartVector 应尽可能少地使用槽。每个槽可以包含多个元素。当达到预定义的槽大小时，它将必须打开一个新槽以平衡写入的字节和项目创建的成本。
- SmartTable 也应尽可能多地将键值对打包到一个槽中。当槽超过阈值时，应该能够逐个桶地增长。与此同时，每个槽中的键值对数量不应过于倾斜。



## 四、规范

### 1. SmartVector

#### 1.1 数据结构规范

```move
struct SmartVector<T> has store {
    inline_vec: vector<T>,
    big_vec: Option<BigVector<T>>,
}
```

简而言之，`SmartVector` 包含了一个 `Option<vector<T>>` 和一个 `Option<BigVector<T>>`，后者是具有元数据的 `TableWithLength<T>`。需要注意的是，我们在这里使用 `vector` 替换了 `Option`，以避免 `T` 的 `drop` 能力限制。其基本思想是：

1. 当 SmartVector 中的数据总量相对较小时，只有 `inline_vec` 将包含数据，并且它将所有数据存储为普通 Vector。此时，SmartVector 只是普通 Vector 的一个包装器。
2. 当 `inline_vec` 中的元素数量达到阈值(M) 时，它将在 `big_vec` 中创建一个新的 `BigVector<T>`，其桶大小(K) 根据估计的 `T` 的平均序列化大小计算而得。然后，所有后续要推送的元素将放入这个 `BigVector<T>` 中。

#### 1.2 接口

SmartVector 实现了 `std::vector` 的大多数基本功能。

需要注意的是，`remove`、`reverse` 和 `append` 在存储费用方面代价很高，因为它们都涉及对一些表项进行修改。

#### 1.3 确定默认配置

当前的解决方案是使用当前元素的 `size_of_val`(T) 乘以 `len(inline_vec) + 1`，如果大于一个硬编码值 *150*，则这个新元素将成为 `big_vec` 中的第一个元素，其桶大小 `K` 由一个硬编码值 *1024* 除以 `inline_vec` 中所有元素的平均序列化大小和要推送的元素来计算。

### 2. SmartTable

#### 2.1 数据结构规范

```move
/// SmartTable 条目包含键和值。
struct Entry<K, V> has copy, drop, store {
hash: u64,
key: K,
value: V,
}

struct SmartTable<K, V> has store {
buckets: TableWithLength<u64, vector<Entry<K, V>>>,
num_buckets: u64,
// 用于表示 num_buckets 的位数
level: u8,
// 总项目数
size: u64,
// 当添加新条目时触发分裂的目标负载阈值。以百分比表示。
split_load_threshold: u8,
// 每个桶的目标大小，这不是强制性的，因此可能存在超大的桶。
target_bucket_size: u64,
}
```

`SmartTable` 基本上是一个 `TableWithLength`，其中键是用户键的 `u64` 哈希模 h(hash)，值是一个桶（bucket），由具有相同哈希用户键的所有用户键值（kv）对的向量表示。相较于传统的 `Table`，`SmartTable` 通过在同一个存储槽里打包多个键值对来提升存储密度，而不是每个槽只存一个键值对。

SmartTable 在内部采用了[线性哈希](https://en.wikipedia.org/wiki/Linear_hashing)(LH) 算法，该算法实现了 [哈希表](https://en.wikipedia.org/wiki/Hash_table) 并逐个桶地增长。在我们的提案中，每个桶占据一个槽，由 `TableWithLength` 中值类型的 `vector<Entry<K, V>>` 表示。LH 对于我们的动机很有效，因为目标是在保持类似表格的结构的同时最大程度地减少槽的数量。

有两个参数决定了 SmartTable 的行为。

- split_load_threshold：当插入新的 kv 对时，当前负载因子将被计算为 `load_factor = 100% * size / (target_bucket_size * num_buckets)`。
  - 如果 load_factor ≥ split_load_threshold，则表示当前表格有点臃肿，需要进行分裂。
  - 否则，不需要采取任何行动，因为当前的桶数足以容纳所有数据。
- target_bucket_size：每个桶中理想的 kv 对数。值得注意的是，这不是强制执行的，只是用作计算负载因子的输入。实际上，有时单个桶的大小可能会超过这个值。

#### 2.2 接口

SmartTable 实现了所有 `std::table` 的函数。

#### 2.3 确定默认配置

- split_load_threshold: 75%
- target_bucket_size: `max(1, 1024 / max(1, size_of_val(first_entry_inserted)))

当前的自动计算默认目标桶大小（target_bucket_size）的方法是，如果用户没有明确设定，就将一项免费配额——1024个单位，除以插入表中第一个数据项所占的大小。

#### 2.4 SmartTable 中的线性哈希(LH)

- LH 将 kv 对存储到桶中。每个桶存储所有具有相同键的哈希的 kv 对。在 SmartTable 中，每个桶表示为一个 `vector<Entry<K, V>>` （向量，一种容纳键值对的数据结构）。一个可能的改进方向是将其替换为本机有序映射。
- LH 需要一组哈希函数。在任何时候，该族中会使用两个函数。SmartTable 使用 `h(key)=hash(key) mod 2^{level}` 和 `H(key)=hash(key) mod 2^{level + 1}` 作为哈希函数，这样计算的结果始终为整数。
- level 是一个从 0 开始的内部变量。当创建了 $2^{level}$  个桶时，level 递增，因此 `h(key)` 和 `H(key)` 的模基数一起加倍。例如，之前 `h(key) = hash(key) % 2`，而 `H(key) = hash (key) % 4`，在 `level` 增加之后，就变成了 `h(key) = hash(key) % 4`，`H(key) = hash(key) % 8`。

##### 2.4.1 分裂（split）

1. SmartTable 从 1 个桶和 level = 0 开始。`h(key) = hash(key)%1`，`H(key) = hash(key)%2`。对于每一轮的分裂，我们从桶 0 开始。
2. 如果发生分裂，则下一个要分裂的桶是递增的，直到达到此级别回合的最后一个桶，即 $2^level - 1$。当最后一个桶被分裂时，实际上在此轮中，我们已经分裂了 $2^level $ 个桶，导致额外增加了 $2^level$  个桶，总共的桶数翻倍。然后我们递增级别，并从 0 开始另一个分裂回合。相应地，`h(key)` 和 `H(key)` 一起改变其模基数加倍。
3. 要分裂的桶的索引始终为 `num_buckets ^ (1 << level)`，而不是我们刚刚插入 kv 对的索引。`num_buckets % (1 << level)`
4. 当发生分裂时，分裂桶中的所有条目将使用 `H(key)` 在它和新桶之间重新分配。

##### 2.4.2 查找

查找有些棘手，因为我们必须同时使用 `h(key)` 和 `H(key)`。首先我们计算 `bucket_index = H(key)`，如果结果是现有桶的索引，那么意味着 `H(key)` 实际上有效，所以我们只需使用 `bucket_index` 找到正确的桶。然而，如果结果对于现有桶无效，那么意味着相应的桶尚未被分裂。因此，我们必须转而使用 `h(key)` 来找到正确的桶。



## 五、参考实现

[smart_vector](https://github.com/aptos-labs/aptos-core/pull/5690) 和 [smart_table](https://github.com/aptos-labs/aptos-core/pull/6339)



## 六、风险和缺点

这两种数据结构的潜在缺点包括：

1. 由于每个数据结构将多个条目打包到一个槽/桶（slot/bucket）中，因此索引不够方便。
2. 对于 SmartTable，由于它进行线性搜索以查找和添加条目可能会触发桶分裂和重排，因此当前某些操作的 gas 费可能不太理想。
3. 由于它涉及具有不透明内部的表，智能数据（smart data）结构在索引器方面的支持不够好。
4. 根据当前的 gas 计划，gas 成本可能会更高，因为我们每次重新收取存储费用。但是我们预计很快会发布一个不同的 gas 计划，届时我们将对智能数据结构的 gas 成本进行基准测试。

针对第 2 点，可以通过使用本机有序映射实现作为桶来缓解。



## 七、Gas 节省

在减少了 100 倍执行 gas 后，我们对创建并向 vector/SmartVector/Table/SmartTable 添加 1 个元素的 gas 成本进行了基准测试。

| gas 单位 | 创建包含 10000 个 u64 元素的结构 | 添加新元素 | 读取现有元素 |
| --- | --- | --- | --- |
| vector | 4080900 | 3995700 | 2600 |
| smart vector | 5084900 | 2100 | 400 |

| gas 单位 | 创建包含 1000 个 u64 键值对的结构 | 添加新键值对 | 读取现有键值对 |
| --- | --- | --- | --- |
| table | 50594900 | 50800 | 300 |
| smart table | 2043500 | 700 | 300 |

如上表所示，智能数据结构在创建和更新方面远远优于 vector 和 table，特别是对于大型数据集。

简而言之，我们建议在涉及大型数据集（例如白名单）的用例中使用智能数据结构。如果内部元素具有 `drop`，它们也可以很容易地被销毁。



## 八、未来潜力

- 目前，我们使用 `size_of_val` 来自动确定这两种数据结构的配置。如果 Move 可以原生支持序列化大小估算，这些操作的成本可能会大大降低。
- 如前所述，当使用 `vector` 作为桶时，桶分裂可能会导致重新分配和线性扫描，这是昂贵的。如果有本机的 `map` 结构，gas 成本将大幅降低。



## 九、建议的实现时间表

代码完成：2023 年 3 月



## 十、建议的部署时间表

测试网发布：2023 年 3 月