---
aip: 4
title: 更新SimpleMap以节省Gas费用
author: areshand
discussions-to: https://github.com/aptos-foundation/AIPs/issues/15
Status: Accepted
last-call-end-date (*optional):
type: Standard (framework)
created: 12/8/2022
---

[TOC]

# AIP-4 - 更新 SimpleMap 以节省 Gas 费

## 一、概述

更改 SimpleMap 的内部实现，以降低 Gas 费，对 Move 和 Public API 的影响最小。



## 二、动机

当前SimpleMap的实现使用基于BCS的对数比较器来识别在Vector中存储数据的位置。不幸的是，这比一个简单的线性实现要昂贵得多，因为每次比较都需要进行BCS序列化，然后进行比较。BCS转换无法像传统比较器那样快速解决，并且对Gas价格有重大影响。

当前SimpleMap的实现采用了基于BCS（Blockchain Compare Serialized）的对数比较器，用于确定数据在Vector结构中的存储位置。然而，这种方法比起简单直接的线性实现要更加耗费资源。因为每一次数据的比较，都需要先执行 BCS 序列化，接着才是比较过程。与传统比较器相比，BCS 转换的处理速度远不及传统方法，这大大增加了执行操作所需的 Gas 费。



## 三、提议

- 将当前对数实现的`find`内部定义替换为对向量的线性搜索。
- 将`add`内的功能替换为调用 `vector::push_back` 并追加新值，而不是将它们插入到它们的排序位置中。
- 修改`add`函数的功能，将其更改为调用`vector::push_back`，以便直接追加新元素，而非将新元素插入到它们排序的（sorted）位置。

## 四、原理

这将导致以下性能差异：

| 操作 | 更新前Gas单位 | 更新后Gas单位 | 变化 |
| --- | --- | --- | --- |
| 创建集合 | 174200 | 174200 |  |
| 第一次创建  token | 384800 | 384800 |  |
| 铸造  token | 117100 | 117100 |  |
| 修改  token | 249200 | 249200 |  |
| 修改  token 添加10个新属性 | 1148700 | 390700 | 64% |
| 修改  token 修改10个现有属性 | 1698300 | 411200 | 75% |
| 修改  token 添加90个新属性 | 20791800 | 10031700 | 51% |
| 修改  token 修改100个现有属性 | 27184500 | 10215200 | 62% |
| 修改  token 添加300个新属性（100个现有，300个新） | 126269000 | 135417900 | -7% |
| 修改  token 修改400个现有属性 | 143254200 | 136036800 | 5% |

当 Token 在链上仅有 1 个属性时，我们可以看到修改 Token 的成本并不会改变。然而，如果用户想要添加 10 个新属性或更新现有属性，Gas 成本减少了**64% 和 75%**；如果用户想要在链上存储 100个 属性，Gas 成本减少了 51% 和62% ；我们还对600条属性的变更进行了测试，但是因为超出了交易所能承受的最大计算资源（gas）上限，这些测试都未通过。

## 五、实施

草案基准PR链接 https://github.com/aptos-labs/aptos-core/pull/5765/files

```rust
// 提议更改之前
public fun add<Key: store, Value: store>(
        map: &mut SimpleMap<Key, Value>,
        key: Key,
        value: Value,
  ) {
      let (maybe_idx, maybe_placement) = find(map, &key);
      assert!(option::is_none(&maybe_idx), error::invalid_argument(EKEY_ALREADY_EXISTS));

      // 追加到末尾，然后交换元素直到列表再次排序
      vector::push_back(&mut map.data, Element { key, value });

      let placement = option::extract(&mut maybe_placement);
      let end = vector::length(&map.data) - 1;
      while (placement < end) {
          vector::swap(&mut map.data, placement, end);
          placement = placement + 1;
      };
 }
// 提议更改之后
public fun add<Key: store, Value: store>(
    map: &mut SimpleMap<Key, Value>,
    key: Key,
    value: Value,
) {
    let maybe_idx = find_element(map, &key);
    assert!(option::is_none(&maybe_idx), error::invalid_argument(EKEY_ALREADY_EXISTS));

    vector::push_back(&mut map.data, Element { key, value });
}
```

```rust
// 提议更改之前
fun find<Key: store, Value: store>(
    map: &SimpleMap<Key, Value>,
    key: &Key,
): (option::Option<u64>, option::Option<u64>) {
    let length = vector::length(&map.data);

    if (length == 0) {
        return (option::none(), option::some(0))
    };

    let left = 0;
    let right = length;

    while (left != right) {
        let mid = left + (right - left) / 2;
        let potential_key = &vector::borrow(&map.data, mid).key;
        if (comparator::is_smaller_than(&comparator::compare(potential_key, key))) {
            left = mid + 1;
        } else {
            right = mid;
        };
    };

    if (left != length && key == &vector::borrow(&map.data, left).key) {
        (option::some(left), option::none())
    } else {
        (option::none(), option::some(left))
    }
}

// 提议更改之后
fun find_element<Key: store, Value: store>(
    map: &SimpleMap<Key, Value>,
    key: &Key,
): option::Option<u64>{
    let leng = vector::length(&map.data);
    let i = 0;
    while (i < leng) {
        let element = vector::borrow(&map.data, i);
        if (&element.key == key){
            return option::some(i)
        };
      i = i + 1;
    };
    option::none<u64>()
}
```

## 六、风险和缺点

在修改前后生成的两个 SimpleMaps 的内部表现形式将会有所不同。比如说，假设一个集合`{c: 1, b: 2, a: 3}`。在更改前，系统会按照排序创建一个内部向量：`{a: 3, b: 2, c: 1}`。但根据这次修改，数据将会按照输入的顺序存储，即`{c: 1, b: 2, a: 3}`。这导致了两种兼容性破坏：

- 在 Move 中，SimpleMap 的等价性判断规则发生变化（ equality property changes）。
- 作为客户端，SimpleMap 的内部布局发生变化。

据我们所知，这些风险的影响相对有限。在 Move 中使用 SimpleMap 进行相等比较是一种罕见的应用，团队尚未在实践中见到。SimpleMap的布局在API层或任何AptosLabs SDK中都不被考虑。

根据我们的了解，这些风险的影响相对较小。利用 SimpleMap 进行等价性比较是一种非常专业且鲜为人知的用途，我们的团队在真实的使用场景中尚未遇到。SimpleMap 的内部结构在 API 层面或者 AptosLabs 的任何 SDK 中都不是一个需要考虑的因素。

我们还注意到，当一个巨大的 simple_map 在区块链上的数据项超过400条时，其性能会有所下降。

**时间表**

待定