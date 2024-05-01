---
aip: 31
title: 允许委托池进行白名单管理
author: junkil-park, movekevin, michelle-aptos, wintertoro, alexfilip2
discussions-to: https://github.com/aptos-foundation/AIPs/issues/121
Status: 审核中
last-call-end-date (*optional): <mm/dd/yyyy 最后留下反馈和评论的日期>
type: 框架
created: 5/3/2023
updated: 04/16/2024
---

[TOC]

# AIP-31 - 允许质押池白名单

## 一、概述

这个 AIP 引入了在委托池中实行白名单的概念，使得委托池的所有者能够控制谁有资格在其委托池中质押资产。这一功能提供了一套方法，用于增加或删除委托方至白名单，以及将不在白名单上的委托方清除出池。默认情况下，委托池是开放型的，并且没有白名单，除非所有者通过此功能显式地设立了白名单。



## 二、高级概览

目前，所有的委托池都是无需许可的，这意味着任何质押者都可以质押到任何委托池。然而，出于各种原因，如合规性，一些委托池可能更喜欢基于许可的方式运行，限制质押只能来自白名单中的钱包地址。此功能将使池所有者能够建立一个白名单，控制谁可以在其委托池中进行质押。



## 三、影响

此功能影响几个方面：

- 池所有者：池所有者可以为其委托池设置白名单。他们可以向白名单中添加或移除地址，并清除不在白名单上的委托者。
- 委托者：如果委托池启用了白名单功能，不在白名单上的委托者将无法在池中添加质押或重新激活质押。池所有者还可以将委托者从池中清除，使其质押解锁。然后，被清除的委托者可以提取其解锁的资金。
- 质押用户界面：质押用户界面（UI）需要更新以显示委托池的白名单管理状态以及委托者的相应状态。





## 四、规范和实现细节

### 1. 规范
以下是规范：
1. 默认情况下，委托池是无需许可的，并且没有白名单。池所有者可以设置白名单，将池转换为许可状态。
2. 当在委托池中首次建立白名单时，它开始为空。这个初始的空白名单不影响池中现有的质押。然而，由于最初没有人在白名单上，该池不会接受任何来自质押者的新质押或待激活的质押的重新激活。
3. 池所有者可以将质押者的钱包地址添加到白名单中。一旦添加了地址，质押者就可以在池中添加新的质押或重新激活现有的质押。请注意，此功能不提供质押者请求加入白名单的机制；这类请求必须在链下管理（例如，通过操作者与质押者之间的私下通信，或基于用户界面的解决方案）。
4. 如果从白名单中移除了一个委托者，则该委托者将无法添加或重新激活质押。然而，他们仍然可以解锁并提取他们现有的质押。
5. 池所有者可以清除不在白名单上的委托者。此操作将解锁委托者的整个质押，将其所有活跃的质押转换为待激活状态。由于被清除的委托者不在白名单上，他们无法重新激活他们的质押。请注意，这些代币将被锁定直到锁定期结束，而现有的质押也将在此期间继续获得奖励。锁定期结束时，资金将被解除质押（处于非活跃状态），但将保留在池中直到委托者发起提取。
6. 如果一个委托者的质押因被驱逐而进入待激活状态，池所有者随后可以将该委托者重新添加到白名单中。然而，此操作不会自动重新激活质押。不提供自动重新激活以防止恶意池所有者的潜在滥用，后者可能会反复驱逐和重新添加委托者以阻止其离开池。一旦委托者重新加入白名单，委托者必须手动调用`reactivate_stake`函数来重新激活他们的质押。
7. 启用了白名单功能的委托池可以禁用它。当禁用时，委托池变为无需许可，允许任何质押者对其进行质押。
8. 本AIP不包括黑名单功能，即允许所有质押，除了黑名单上的质押。其理由是，位于黑名单地址背后的个人可以轻易地通过创建新的账户地址来规避这一限制。

### 2. 实现细节

白名单被实现为一个智能表（Smart table），其包含资源将被发布在池地址下（pool address）。智能表将存储地址的白名单和一个布尔值，指示地址是否在白名单上。智能表将定义如下：
```
struct DelegationPoolAllowlisting has key {
    allowlist: SmartTable<address, bool>,
}
```

池所有者通过与`aptos_framework::delegation_pool`模块交互来为他们的委托池构建白名单。一个账户只能拥有一个委托池，因此，其签名者唯一地标识了它拥有的池。以下是添加到`aptos_framework::delegation_pool`模块的入口函数：
- `enable_delegators_allowlisting(owner: &signer)`：启用白名单并创建一个空白名单
- `disable_delegators_allowlisting(owner: &signer)`：禁用白名单并删除现有的白名单，池变为无需许可
- `allowlist_delegator(owner: &signer, address)`：将一个地址添加到白名单中，如果未启用白名单，则失败
- `remove_delegator_from_allowlist(owner: &signer, address)`：从白名单中删除一个地址，如果未启用白名单，则失败
- `evict_delegator(owner: &signer, address)`：通过 **解锁** 他们的整个质押，从池中清除一个委托者，如果未启用白名单或委托者在白名单上，则失败

这些是添加到`aptos_framework::delegation_pool`模块的视图函数：
- `allowlisting_enabled(pool: address)`：返回`pool`是否已启用白名单
- `delegator_allowlisted(pool: address, delegator: address)`：返回`delegator`是否在`pool`的白名单上
- `get_delegators_allowlist(pool: address)`：返回在`pool`上定义的白名单，如果未启用白名单，则失败

定义了几种事件类型以通知白名单中的更改：
```
#[event]
struct EnableDelegatorsAllowlisting has drop, store {
    pool_address: address,
}

#[event]
struct DisableDelegatorsAllowlisting has drop, store {
    pool_address: address,
}

#[event]
struct AllowlistDelegator has drop, store {
    pool_address: address,
    delegator_address: address,
}

#[event]
struct RemoveDelegatorFromAllowlist has drop, store {
    pool_address: address,
    delegator_address: address,
}

#[event]
struct EvictDelegator has drop, store {
    pool_address: address,
    delegator_address: address,
}
```



## 五、参考实现

[https://github.com/aptos-labs/aptos-core/pull/12213](https://github.com/aptos-labs/aptos-core/pull/12213)



## 六、测试

参考实现包括涵盖各种情况的多个单元测试案例。这些情况将在测试网络上进行测试，使用在那里建立的活跃委托池。



## 七、风险和缺陷

由于每个委托者的活跃和待激活质押的最低质押要求为 10 个 APT，因此池所有者无法清除确实持有 10 个 APT 作为活跃质押的委托者。当尝试解锁时，由于“活跃到待激活”的转换引起的舍入误差可能会导致质押金额略低于 10 个 APT ，从而导致解锁函数失败。然而，除非验证者处于非活跃状态，否则奖励将继续累积在委托者的活跃质押上，使其超过 10 个 APT。一旦活跃余额超过 10 个 APT，池所有者就可以继续清除委托者。



## 八、未来潜力

对于任何想要使用许可委托池运行验证器的交易所、金融机构或其他实体来说，此功能可能是必须的。



## 九、时间表

### 1. 建议的实施时间表
实现已经落地在 `main` 分支和 v1.11 的分支上。

### 2. 建议的开发者平台支持时间表
在此功能在主网上启用之前，质押用户界面的更改将在开发者平台上实现。

### 3. 建议的部署时间表
- 在开发网络：随着版本1.11的发布
- 在测试网络和主网：取决于开发者平台支持的时间表，特别是质押用户界面的更新。





