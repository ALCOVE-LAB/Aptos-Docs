---
aip: 21
title: 可互换资产
author: lightmark, movekevin, davidiw
Status: 已接受
type: 标准（框架）
created: 2022年4月11日
---

[TOC]

# AIP-21 - 可互换资产

## 一、概述

本 AIP 提出了一个标准，针对可互换资产（Fungible Assets，简称 FA），应用于 [Move Object](/aips/aip-10.md)。根据该标准，链上资产若以对象形式存在，就可以转化为可互换资产，这意味着一个对象可以分解为多个独立的、可以相互替换的所有权单元。

## 二、动机

将可互换资产派生为对象可以实现无缝的开发人员体验，同时简化应用程序开发时间。这个标准支持潜在的应用，比如：

- 证券（securitie）和商品（commodities）的 token 化提供了部分所有权（fractional ownership）。
- 不动产（real estate）的所有权，使部分所有权成为可能，并为传统上流动性差的市场提供流动性。
- 游戏内的资产，如虚拟货币和角色，可以进行 token 化，让玩家能够拥有和交易他们的资产，为游戏开发者和玩家创造新的收入来源。

除了上述功能外，可互换资产还提供了现有的 Aptos Coin 标准概念的超集，以及确保能够围绕这些资产形成健康生态系统的对象常见属性。



## 三、原因

原因有两个方面：

自 Mainnet 启动以来，现有的 Coin 模块被认为由于 Move 语言的结构特性以及固有的扩展性不足，无法满足当前和未来的需求。举个例子，没有机制能够确保转账按指定路径进行，或者限定只有特定实体能够持有相关资产。总的来说，现行的授权管理模式缺乏必要的灵活性，不利于可互换资产政策的创新发展。

这些问题的根源来自两方面：

* 现有的 `Coin` 结构利用了 `store` 能力，使得链上的资产变得难以追踪。这给链下可观察性和链上管理，例如冻结或销毁，带来了挑战。
* 缺乏访问控制修饰符，使得`Coin`的管理者无法制定转账（transfer）的具体条件和要求。

可互换资产解决了这些问题。



## 四、规格

`fungible_asset::Metadata` 是与某类可兑换资产相关的元数据或信息。具有此种元数据的对象成为了一种可互换资源，该资源的所有权可以通过 `FungibleAsset` 的数量来表示。此外，一个对象还可以附带更多资源，以增添更多上下文。举例而言，元数据可以详细描述一颗宝石的类型、颜色、品质和稀有程度，并且通过所有权来表示某一类型宝石的具体数量或总质量。

```rust
#[resource_group_member(group = aptos_framework::object::ObjectGroup)]
/// 定义可互换元数据所需的元数据。
struct Metadata has key {
    /// 可互换资产的当前供应量。
    supply: u64,
    /// 最大供应限制，`option::none()` 表示没有限制。
    maximum: Option<u64>,
    /// 可互换元数据的名称，例如 "USDT"。
    name: String,
    /// 可互换元数据的符号，通常是名称的缩写，例如新加坡元是 SGD。
    symbol: String,
    /// 显示时使用的小数位数。
    /// 例如，如果 `decimals` 等于 `2`，那么 `505` 个硬币的余额应该显示为 `5.05`（`505 / 10 ** 2`）。
    decimals: u8,
}
```

`FungibleStore` 仅作为对象中特定可互换资产余额的容器/持有者存在。

`FungibleAsset` 是可互换资产的一个实例，就像是[热土豆](https://medium.com/@borispovod/move-hot-potato-pattern-bbc48a48d93c)一样。

```rust
#[resource_group_member(group = aptos_framework::object::ObjectGroup)]
/// 持有与账户关联的特定类型可互换资产余额的存储对象。
struct FungibleStore has key {
    /// 基本元数据对象的地址。
    metadata: Object<Metadata>,
    /// 可互换元数据的余额。
    balance: u64,
    /// 如果为 true，则禁用所有者转移，只有 `TransferRef` 可以从此存储中移入/移出。
    frozen: bool,
}

/// FungibleAsset 可以传递给函数，这样不仅可以确保数据类型的正确性，还能保证处理的是确切的数量。
/// FungibleAsset 具有临时性质，它不能直接持久化储存，最终必须归还到某个储存系统中。
struct FungibleAsset {
    metadata: Object<Metadata>,
    amount: u64,
}
```



### 1. 主存储和辅助存储

每个账户可以拥有多个 `FungibleStore`，但只有一个主存储，其余的被称为辅助存储。主存储地址是确定的，`hash(owner_address | metadata_address | 0xFC)`。辅助存储根据需要创建，通常由基于GUID的对象派生。

主存储的主要特点包括：

1. 主存储对象地址是确定的，因此交易可以在知道所有者账户地址的情况下无缝访问。
2. 如果主存储不存在，将在资产转移时创建主存储。
3. 主存储不可被删除。

## 五、参考实现

[https://github.com/aptos-labs/aptos-core/pull/7183](https://github.com/aptos-labs/aptos-core/pull/7183)

[https://github.com/aptos-labs/aptos-core/pull/7379](https://github.com/aptos-labs/aptos-core/pull/7379)

[https://github.com/aptos-labs/aptos-core/pull/7608](https://github.com/aptos-labs/aptos-core/pull/7608)

### 1. 可互换资产主要 API

```rust
public entry fun transfer<T: key>(sender: & signer, from: Object<T>, to: Object<T>, amount: u64)
public fun withdraw<T: key>(owner: & signer, store: Object<T>, amount: u64): FungibleAsset
public fun deposit<T: key>(store: Object<T>, fa: FungibleAsset)
public fun mint( ref: & MintRef, amount: u64): FungibleAsset
public fun mint_to<T: key>( ref: & MintRef, store: Object<T>, amount: u64)
public fun set_frozen_flag<T: key>( ref: & TransferRef, store: Object<T>, frozen: bool)
public fun burn( ref: & BurnRef, fa: FungibleAsset)
public fun burn_from<T: key>( ref: & BurnRef, store: Object<T>, amount: u64)
public fun withdraw_with_ref<T: key>( ref: & TransferRef, store: Object<T>, amount: u64)
public fun deposit_with_ref<T: key>( ref: & TransferRef, store: Object<T>, fa: FungibleAsset)
public fun transfer_with_ref<T: key>(transfer_ref: & TransferRef, from: Object<T>, to: Object<T>, amount: u64)
```

### 2. 可互换存储主要 API

```rust
# [view]
public fun primary_store_address<T: key>(owner: address, metadata: Object<T>): address

# [view]
/// 获取 `account` 的主存储余额。
public fun balance<T: key>(account: address, metadata: Object<T>): u64

# [view]
/// 返回给定账户的主存储是否被冻结。
public fun is_frozen<T: key>(account: address, metadata: Object<T>): bool

/// 通过所有者从 `store` 提取 `amount` 的可互换资产。
public fun withdraw<T: key>(owner: & signer, metadata: Object<T>, amount: u64): FungibleAsset

/// 将 `amount` 的可互换资产存入给定账户的主存储。
public fun deposit(owner: address, fa: FungibleAsset)

/// 将发送者主存储中的 `amount` 的可互换资产转移到接收者的主存储。
public entry fun transfer<T: key>(sender: & signer, metadata: Object<T>, recipient: address, amount: u64)
```

### 3. 可互换存储主要 API

```rust
# [view]
public fun primary_store_address<T: key>(owner: address, metadata: Object<T>): address

# [view]
/// 获取 `account` 的主存储余额。
public fun balance<T: key>(account: address, metadata: Object<T>): u64

# [view]
/// 返回给定账户的主存储是否被冻结。
public fun is_frozen<T: key>(account: address, metadata: Object<T>): bool

/// 通过所有者从 `store` 提取 `amount` 的可互换资产。
public fun withdraw<T: key>(owner: & signer, metadata: Object<T>, amount: u64): FungibleAsset

/// 将 `amount` 的可互换资产存入给定账户的主存储。
public fun deposit(owner: address, fa: FungibleAsset)

/// 将发送者主存储中的 `amount` 的可互换资产转移到接收者的主存储。
public entry fun transfer<T: key>(sender: & signer, metadata: Object<T>, recipient: address, amount: u64)
```

## 六、风险与缺陷

- 使资产可互换并不是一个不可逆的操作，如果不再需要，目前还没有办法清除可互换资产数据。不过可以销毁所有该可互换资产的表示形式（representations）。
- 使用主存储对性能有一定影响，因为应用程序必须执行 SHA-256 哈希算法来访问每个交易（transaction）中的每个资产。与此同时，对象索引器（object indexers）仍处于初级阶段，辅助存储也不容易获得。

## 七、未来潜力

随着这一标准的稳定，我们预计 SDK 和索引解决方案将消除对主存储的需求，从而实现更清晰的交易记录，并为链上交易更好地并行化铺平道路。

## 八、建议的部署时间表

4 月初在开发网络上部署，4 月中旬在测试网络上部署，5 月初在主网络上部署。
