---
aip: 43
title: 并行化数字资产（Token V2）的铸造/销毁
author: igor-aptos (https://github.com/igor-aptos), vusirikala (https://github.com/vusirikala)
discussions-to (*optional): https://github.com/aptos-foundation/AIPs/issues/209
Status: 已接受
last-call-end-date (*optional): <mm/dd/yyyy 最后留下反馈和审查的日期>
type: 标准（框架）
created: 07/20/2023
updated (*optional): <mm/dd/yyyy>
requires (*optional): AIP-47, AIP-44
---

[TOC]

# AIP-43 - 并行化数字资产（Token V2）的铸造/销毁

## 一、概述

本 AIP 提出了一种解决方案，通过并行化（从单线程到多线程）过程来加速 Token v2 的铸造和销毁。目前，Token v2 在单个集合或资产的铸造时是单线程和顺序执行的，通过并行化，我们可以期望获得更高的吞吐量 / 峰值。

请注意，本AIP引入了一个破坏性的变化！请阅读AIP的其余部分了解详情。

## 二、动机

目前，Token V2 在单个集合内的铸造和销毁是单线程的，因为这些操作对多个每个集合字段进行读取-修改-写入操作 -
current_supply，total_supply和事件处理中的序列号。
这意味着从单个集合铸造的吞吐量比同时从多个集合铸造要低5-8倍。

目标是消除/优化使Token V2操作成为顺序的操作



目前，Token V2 在单个集合（collection）中的铸造和销毁操作是依次进行的，原因是这些操作需要在集合级别的多个字段上执行读取、修改和写入 —— 这些字段包括当前供给量（current_supply）、总供给量（total_supply）以及事件处理中的序列号。由此产生的结果是，同一集合中铸币的效率比在多个集合中同时铸币的效率要低5到8倍。 我们的目标是移除或改进那些导致 Token V2 操作按序执行的环节。

## 三、影响

这将使 Token V2 NFT 单个集合的铸造/销毁的吞吐量更高，在单个集合需求量高时提供更好的体验。

对于直接获取底层数据的用户，比如使用索引服务或者通过 RestAPI，将会有**破坏性的变化**的变化。



数字资产 `Token` 结构体（来自 [token.move](https://github.com/aptos-labs/aptos-core/blob/main/aptos-move/framework/aptos-token-objects/sources/token.move)）内的两个字段将被弃用 - name 和 index，而是 `TokenIdentifiers` 将包含它们。 `Token.name` 将被替换为 `TokenIdentifiers.name.value`，类似地，对于 `index` 字段也是如此。

此外，将向以下内容添加新的变体：
- 数字资产（Token）集合（[collection.move](https://github.com/aptos-labs/aptos-core/blob/main/aptos-move/framework/aptos-token-objects/sources/collection.move)），即 `ConcurrentSupply`（除了当前的 `FixedSupply` 和 `UnlimitedSupply`），它将现在存储当前、总和最大供应量

新集合将发出新事件：新集合将发出 `Mint` 和 `Burn`（而不是 `MintEvent` 和 `BurnEvent`）事件。

将提供索引器更改以返回 Token 名称的正确值（即 `COALESCE(TokenIdentifiers.name.value, Token.name)`），以及对供应相关字段的处理。它还将读取新事件，并将它们索引为常规事件。

## 四、规范

AIP 36 删除了一个序列化点。我们可以通过以下方式删除其余的：
- 在 collection.move 中使用聚合器来处理 total_supply 和 current_supply 计数器
- 并行化事件创建，使用模块事件完全删除序列号

此外，我们将添加新的 API 来铸造 token（mint token）：
```
  public fun create_numbered_token(
        creator: &signer,
        collection_name: String,
        description: String,
        name_with_index_preffix: String,
        name_with_index_suffix: String,
        royalty: Option<Royalty>,
        uri: String,
    ): ConstructorRef {
```
它允许铸造 token，其中索引是名称的一部分，同时仍然允许并发发生。

## 五、基本原理

替代方案是停止跟踪供应量 / 提供索引，但这对于需要有限供应的 NFT 集合是行不通的。
我们可以使用分片计数器（即将供应分成 10 个单独的计数器），来强制执行有限供应，但是 NFT 将不会获得单调递增的索引（即后来的铸造可能会获得较早的索引）。（[原型实现](https://github.com/aptos-labs/aptos-core/compare/main...igor-aptos:aptos-core:igor/bucketed_counter)）

或者，我们可以调整 NFT 铸造流程，首先需要获得一个排队号（排队位置的索引），然后再用这个排队号作为凭证，在另一个交易中进行铸造，通过这种方式可以减小依次执行步骤的范围。但这样的做法在智能合约中的应用会变得更加复杂，并且要铸造一个 NFT，需要提交两次交易。

## 六、参考实现

- [PR](https://github.com/aptos-labs/aptos-core/pull/9971) 到 token v2 以使用聚合器和模块事件，包括新的 create_numbered_token 方法。
- 还有一些跟进 PR，包含了更改和澄清，所有代码都通过 CONCURRENT_TOKEN_V2 在 [collection.move](https://github.com/aptos-labs/aptos-core/blob/main/aptos-move/framework/aptos-token-objects/sources/collection.move) 和 [token.move](https://github.com/aptos-labs/aptos-core/blob/main/aptos-move/framework/aptos-token-objects/sources/token.move) 中进行了门控。

## 七、风险和缺点

除了需要注意部署，因为上述的破坏性更改之外，唯一的其他细微之处是确保新流程的 Gas 费与先前的 Gas 费相当。

## 八、未来潜力

我们将进一步研究框架中的任何顺序瓶颈，并查看它们是否可以通过某种方式解决。

## 九、时间表

### 1. 建议的实施时间表

实施这项改进提案（AIP）的过程将与草案文档大体相符，操作起来非常直接。复杂的部分主要是因为它需要与其他提案相互协调。

### 2. 建议的开发者平台支持时间表

Indexer 的更改正在开发中。

### 3. 建议的部署时间表

计划的部署时间表如下：
- 使用 v1.10 框架和功能标志升级，新的 `TokenIdentifiers`（带有 `name` 和 `index`）将开始填充（除了当前字段之外）
- 几周后，将启用 CONCURRENT_TOKEN_V2 功能标志，随之而来的：
  - `Token` 结构中的 `name` 和 `index` 字段将被弃用，并且对于任何新的 token 发行，它们将为空（分别为 "" 和 0）
  - 任何新的数字资产收藏品都将被创建为“并行” - 使用新的 ConcurrentSupply 变体，并提供性能/吞吐量优势
    - 新的 collectoin 将发出 Mint/Burn 事件。
  - 任何旧的数字资产收藏品都可以调用 upgrade_to_concurrent(&ExtendRef) 函数，并切换到“并行”模式，从而实现性能/吞吐量优势

## 十、安全考虑

设计已在团队内部进行了审查，并且任何 PR 都将被仔细审查。这个 PR 提供了今天存在的等效功能，以一种可并行化的方式实现，大多数安全考虑将来自相关 AIP 的实现。

## 十一、测试（可选）

一旦实施，并且一旦依赖项被实现，测试将包括 move 单元测试以确保正确性，同时也将进行性能基准测试，以评估其对系统性能的潜在影响。