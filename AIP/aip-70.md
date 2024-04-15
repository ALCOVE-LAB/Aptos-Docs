---
aip: 70
title: 并行化可互换资产
author: igor-aptos (https://github.com/igor-aptos), vusirikala (https://github.com/vusirikala)
discussions-to (*optional): https://github.com/aptos-foundation/AIPs/issues/209
状态: 审查中
最后征求意见截止日期 (*可选): <mm/dd/yyyy 最后留下反馈和评论的日期>
类型: 标准（框架）
创建日期: 07/20/2023
更新 (*可选): <mm/dd/yyyy>
需要 (*可选): AIP-47, AIP-44
---

[TOC]

# AIP-70 - 并行化可互换资产

## 一、概述

本 AIP 提出了一种解决方案，通过并行化过程（从单线程到多线程）来加速可互换资产操作。目前，可互换资产在从单个集合或资产铸造（minting）时是单线程和顺序的，通过并行操作，我们可以期望获得更高的吞吐量/峰值。

请注意，本 AIP 引入了一个破坏性变更！请阅读 AIP 的其余部分了解详情。



## 二、动机

目前，可互换资产在可互换资产内的铸币和销毁是单线程的，因为这些操作需要读取-修改-写入事件处理中的供应字段（supply field）和序列号（ sequence number）。这意味着从单个资产铸造的吞吐量比同时从多个资产铸造的吞吐量低5-8倍。

此外，存款和提款操作也对余额字段（balance field）进行读取-修改-写入，使得在同一个可互换资产存储上的操作也是顺序的。

目标是消除或优化使可互换资产操作变为顺序的操作。



## 三、影响

这将为单个可互换资产的铸造或销毁，以及单个可互换资产存储的存入（deposit）和取出（withdraw ）提供更高的吞吐量，当对可互换资产 / 可互换资产存储的需求很高时，提供更好的体验。

对于访问原始资源的任何人 —— 例如索引器或直接通过 RestAPI 访问的用户 —— 这是一个**破坏性的变更**。

将为以下内容添加新的变体：
- 可互换资产（[fungible_asset.move](https://github.com/aptos-labs/aptos-core/blob/main/aptos-move/framework/aptos-framework/sources/fungible_asset.move)），即 `ConcurrentSupply`（除了当前的 `Supply`外），它将存储当前供应量和总供应量。还将添加 `ConcurrentFungibleBalance`，用于存储余额。

新的集合将发出新的事件：新集合将发出 `Mint` 和 `Burn` 事件（而不是 `MintEvent` 和 `BurnEvent`）。

将提供索引器更改，以读取新的事件，并将它们索引为常规事件。





## 四、规范

> [!TIP]
>
> 译者注：原文未提供。

## 五、理由

> [!TIP]
>
> 译者注：原文未提供。

## 六、参考实现

- [PR](https://github.com/aptos-labs/aptos-core/pull/9972) 对可互换资产供应量使用聚合器
- [PR](https://github.com/aptos-labs/aptos-core/pull/11183) 对可互换资产余额使用聚合器



## 七、风险和缺点

除了需要注意部署，由于上述的破坏性变更，其他需要注意的细节是确保新流程的 Gas 费用与之前的 Gas 费用相当。



## 八、未来潜力

我们将进一步研究框架中的任何顺序瓶颈，并看看是否可以通过某种方式解决它们。



## 九、时间表

### 1. 建议的实施时间表

实现这个 AIP 将接近于链接的草案，并且是简单的。大部分复杂性来自于依赖的 AIP。

### 2. 建议的开发者平台支持时间表

索引器的更改正在开发中

### 3. 建议的部署时间表

计划的部署时间表：
- 使用 v1.11 框架和功能标志升级，我们计划启用 `CONCURRENT_FUNGIBLE_ASSETS` 功能标志，该标志将：
  - 任何新的可互换资产集合将被创建为“并行（concurrent）”的 —— 使用新的 ConcurrentSupply 变体，并提供性能 / 吞吐量的优势
    - 新集合将发出 ConcurrentMintEvent / ConcurrentBurnEvent 。
  - 任何旧的可互换资产集合都可以调用 `upgrade_to_concurrent(&ExtendRef)` 函数，并切换到“并行”模式，从而实现性能/吞吐量的优势
  - 并行集合上的余额字段（balance field）将移动到 `ConcurrentFungibleBalance.balance.value`

## 十、安全注意事项

此 PR 以可并行化的方式提供了当前已存在功能的等效功能，大多数安全考虑将来自依赖的 AIP 的实现

此 PR 影响了所有可互换资产 —— 包括 APT，因此任何依赖的AIP（即AggregatorsV2）中的问题都可能导致资金损失或凭空铸造，因此需要仔细评估这个特性。



## 十一、测试（可选）

一旦实施，并且一旦依赖项得到实施，测试将包括 Move 单元测试以验证正确性，以及基准评估以评估性能影响。