---
aip: 19
title: 启用更新委托奖金比例
author: michelle-aptos, gerben-stavenga
discussions-to（*可选）： <a url pointing to the official discussion thread>
状态：已接受
last-call-end-date（*可选）： 
type: 标准
created: 2023年3月8日
updated（*可选）： 2023年3月8日
---

[toc]

# AIP-19 - 在staking_contract模块中启用更新commission_percentage

## 一、概述

本 AIP 提议对`staking_contract.move`进行更新，允许质押池所有者更改`commission_percentage`。

## 二、动机

当前，`commission_percentage`无法更改。更新委托奖金比例功能将使得质押池能够更好地适应不断变化的市场条件。

## 三、基本原理

**考虑因素：**

1. 质押合约跟踪需要支付给操作员的佣金金额。更新`commission_percentage`是添加到`staking_contract.move`的便利功能，允许质押池所有者更新支付给操作员的佣金比例。
2. `Commission_percentage`（佣金比例）可以由质押池所有者在任何时候进行更新。佣金的收取是根据周期来计算的。一旦调用更新功能，这项更改会立即影响未来所有的佣金收入，但它不会对已经赚取的佣金产生追溯效果。
3. 当调用 `update_comission `函数时，将发出 `UpdateCommissionEvent`。

**备选方案：**

为了更改`commission_percentage`，必须终止质押合约并创建一个新的质押合约。这是一个不太理想的解决方案，因为它会增加更多的运营开销，并导致错过质押奖励。

## 四、参考实现

[https://github.com/aptos-labs/aptos-core/blob/main/aptos-move/framework/aptos-framework/sources/staking_contract.move](https://github.com/aptos-labs/aptos-core/blob/main/aptos-move/framework/aptos-framework/sources/staking_contract.move)

[https://github.com/aptos-labs/aptos-core/pull/6623/](https://github.com/aptos-labs/aptos-core/pull/6623/files)

## 五、风险和缺陷

更改`commission_percentage`可能会引入不确定性，因为在解锁周期内，可能会以不同的佣金比例支付给操作员。然而，对于操作员而言，不需要额外的操作，因为更改会立即生效。

我们可以在将来的迭代中通过实施每周期最大佣金变更来减轻这一问题。这对于当前的所有者 - 操作员结构并不是一个问题。

## 六、未来潜力

该功能将使质押池的所有者对`commission_percentage`具有更大的灵活性，以反映不断变化的市场条件。

## 七、建议的实施时间表

目标是在第一季度末完成。

## 八、建议的部署时间表

该功能当前在devnet和testnet上作为v1.3的一部分。