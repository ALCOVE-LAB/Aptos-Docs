---
aip: 13
title: Coin 标准改进
author: movekevin
discussions-to: https://github.com/aptos-foundation/AIPs/issues/24
Status: Accepted
last-call-end-date:
type: Standard (Framework)
created: 2023/01/05
updated: 2023/01/2
---

[TOC]

# AIP - 13 - Coin 标准改进

## 一、概述

自从 Aptos 主网上线以来，社区提出了一些建议性改进，这将有助于加速 Coin 标准的应用。具体包括：

1. 隐式 Coin 注册：目前，新币种的接收方需要显式注册才能在第一次接收到它。这在用户体验上与交易所（包括CEX和DEX）以及钱包之间造成了较差的体验。转换为隐式模型，改为无需注册从而能够显著改善用户体验。
2. 增加对批量转账的支持：这是一个次要的便利性改进，允许账户在单个交易中发送 Coin，包括网络 Coin（APT）和其他 Coin。

由于（2）是一个次要的改进，本提案的其余部分将讨论（1）的细节。

## 二、动机和原理

目前，在许多情况下，Coin（包括 APT 在内的 ERC-20 token）不能直接发送到一个账户，如果：
- 该账户尚未创建。需要由另一个可以支付 Gas 费用的账户调用`Aptos_account::transfer` 或 `account::create_account` 。这通常是一个轻微的烦恼，但不是太大的问题。
- 该账户尚未注册以接收该 Coin 种类。对于每个自定义 coin 都必须完成此操作（创建账户时默认注册了APT）。这是产生埋怨和苦恼（ complaints/pain）的主要原因。

这种设计主要是出于对历史原因的考虑，目的是让账户能够主动添加他们想要的 token / coin，避免接收到不需要的随机 token / coin。但这却给用户和开发人员带来了麻烦，因为他们需要牢记注册 coin 类型，特别是在只有接收账户才能注册 Coin 类型时，这会让开发者和用户增加额外的不便之处，因为他们需要更多的步骤来处理特定类型的 coin 。例如，中心化交易所 (CEX) 进行涉及特定 coin 的转账就是一个遭遇此问题的重要场景。

## 三、提议

我们可以切换到一种模式，在特定的 CoinType 没有 CoinStore 存在时，如果转移，则隐式创建 CoinStore（通过注册）。这可以作为 aptos_coin 中的一个单独流程添加，类似于`aptos_coin::transfer`，如果一个 APT 转移不存在则隐式创建一个账户。此外，账户可以选择退出此行为（例如，为了避免接收垃圾 Coin）。详细流程如下：
1. `aptos_coin::transfer_coins<CoinType>(from: &signer, to: address, amount: u64)`  默认情况下，如果 CoinType 不存在，则将注册接收方地址（创建 CoinStore ）。
2. 账户可以选择退出（例如，为了避免接收垃圾 Coin ），方法是调用 `aptos_account::set_allow_direct_coin_transfers(false)` 。他们也可以稍后使用 `set_allow_direct_coin_transfers(true)` 来恢复。默认情况下，在此提案实施之前的所有现有账户和此后的新账户都将被隐式选择接收所有 Coin 。

## 四、实施
参考实现：
- 隐式币种注册：https://github.com/aptos-labs/aptos-core/pull/5871
- 批量转账支持：https://github.com/aptos-labs/aptos-core/pull/5895

## 五、风险和缺点
由于这是一种新的替代流程，而不是修改现有的 `coin::transfer` 函数，因此对 `coin::register` 和 `coin::transfer` 的现有依赖不应受到影响。只有一个已知的潜在风险：由于账户默认选择接收任意 Coin ，因此它们可能会受到恶意用户发送的数百或数千个垃圾 Coin 的骚扰。可以采取以下措施进行缓解：
- 钱包可以维护一个已知的信誉币种（reputable coin）列表，并使用它来过滤用户的币种列表。这是其他链/钱包中常见的标准用户体验（如： Metamask）。
- Resources API 现支持分页功能，这在账户拥有大量资源时特别有用。这样可以降低因为制造大量资源（为每个随机的 coin 创建一个 CoinStore）而导致账户遭受分布式拒绝服务攻击（DDOS）的风险。
- 索引系统（Indexing systems ）可以进一步帮助过滤垃圾 Coin 。这可以在其他链上的热门的区块链浏览器（如 Etherscan）中看到。

## 六、时间表
此更改可以在2月1日或2月8日（PST时间）的一周内推出到测试网进行测试。