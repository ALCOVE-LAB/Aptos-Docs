---
aip: 83
title: 框架级不可转让对象
author: 
  - name: davidiw
discussions-to: <指向官方讨论线程的URL>
Status: 草案
type: 框架
created: 04/27/2024
updated: <mm/dd/yyyy>
requires: 21
---
[TOC]


# AIP-83 - 框架级不可转让的可替代资产存储

## 一、摘要

Aptos对象模型提供了一个可扩展的数据模型，允许一个对象扮演许多不同类型。例如，一个对象可以既是可替代资产，也是数字资产。一些资产类别需要更严格的控制，因此现有的框架与它们的目标相冲突。具体来说，许多 API 通过 `ConstructorRef` 公开添加新对象，但 `ConstructorRef` 也可以用来启用新的转让政策。结果，当前没有方法在现有对象模型中强制执行某些转让政策。本AIP通过引入一种称为 `object::set_untransferable` 的新方法来构造对象，确保无论在对象创建期间或之后执行的任何操作，对象所有者都被永久设置。

Aptos 的对象模型推出了一个可拓展的数据模型，允许一个对象同时担当多种不同的角色。例如，一个对象可以即是可替代资产，也是数字资产。一些资产类型需要更严格的控制，而现有框架无法满足其目标。具体而言，许多 API 通过一个叫做 `ConstructorRef` 的接口来实现增加新对象的功能，但这个接口也能够引入新的转移规则。这就导致了在当前的对象模型中无法强制实施特定的转移规则。AIP 提出了朝向加强控制方向迈出的第一步，它引入了一种新的对象构建方法`object::set_untransferable`，确保对象所有者一旦设置，无论在其创建期间或之后进行了何种操作，都将永久保持。

具体想到的应用是可替代资产，其中可替代资产存储可以通过`fungible_asset::set_frozen_flag`被冻结，然而，这并不阻止资产拥有者发送和接收新的可替代存储并继续访问资产。

### 1. 不讨论的部分

在未来，理想的发展方向应探索让单个对象能够更清晰地规定哪些转移规则是被允许的。可能这意味着需要展示一种可操作的对象转移模型，不过，这一点超出了当前 AIP 所讨论的范围。



## 二、宏观概述

在创建对象期间，任何可以访问 `ConstructorRef` 的代码都可以调用 `object::set_untransferable`，这将阻止任何对 `object::transfer` 和 `object::transfer_with_ref` 的调用。将向可替代资产元数据添加类似功能，以指示为特定可替代资产创建的存储应该是不可转让的：`fungible_asset::set_untransferable`。然后，对于该元数据的所有对 `fungible_asset::create_store` 的调用将通过代理调用 `object::set_untransferable`。

因此，如果账户的创建者希望将某个账户冻结，他们只需要冻结主要的可替代账户即可。若主账户并不存在，就需要先创建它，然后再进行冻结。

冻结在取出和存入期间强制执行。首先，资产必须通过冻结可替代资产存储并使用可替代资产元数据的 `TransferRef` 来进行转让，或通过使用 [AIP-73](https://github.com/aptos-foundation/AIPs/blob/main/aips/aip-73.md) 中讨论的可调度来配置为通过可替代转让函数 (alternative transfer functions)。当用户尝试转让时，可替代函数应首先获取每个提供存储的最初的拥有者，并验证这些拥有者的主账户没有被冻结。只有满足这些所有的条件时，才能在提供的存储上执行取出或存入操作。

本 AIP 还引入了一种方法来获取 Aptos 中对象的最终拥有者，由于对象的嵌套性质。即，可替代资产存储可能被冻结的账户间接拥有。正在引入 `object::root_owner` 以确定对象的最高或最终所有者。然后，这可以用来评估与该身份相关联的属性。



## 三、影响

本文提议的变化将简化构建应用程序的流程，这些应用能够统一处理各种类型的对象和流通资产。如果不采纳这些建议，每个项目都需要独立开发它们自己的特定分派逻辑来处理需要此类操作的对象。这样会大大延缓对于需要精细管理的对象的接纳速度，由于每种新资产的引入都需要更新调度表。除了维护的成本，随着时间的积累，因加载过多的外部模块，调度表有可能逐渐变得效率低下。



## 四、替代解决方案

* 要求每个提供商构建自己的专业功能，然后让每个提供商构建自己的调度功能。如上所述，这很快变得不可行且不可扩展。
* 利用存储创建的动态调度。这是可行的，但我们更愿意在此时收到关于此要求的反馈，而不是向框架公开更多的动态调度。这可能会导致不安全的临时实现。
* 开发专门针对流通资产存储的逻辑。尽管实现这一目标是可行的，但将这类逻辑集成到核心系统中，不仅能简化系统的设计，还能确保`create_store`功能为那些不可转让资产的开发提供一个统一、连贯的体验。

这个解决方案统一了这个问题的解决方法并最小化了代码。



## 五、规范和实现细节

本 AIP 引入了以下新功能：

在 `object.move` 中：

```
#[resource_group_member(group = aptos_framework::object::ObjectGroup)]
struct Immovable has key {}

public fun set_untransferable(ref: &ConstructorRef) acquires ObjectCore {
    let object = borrow_global_mut<ObjectCore>(ref.self);
    object.allow_ungated_transfer = false;
    let object_signer = generate_signer(ref);
    move_to(&object_signer, Untransferable {});
}

public fun root_owner<T: key>(object: Object<T>): address acquires ObjectCore {
    let obj_owner = owner(object);
    while (is_object(obj_owner)) {
        obj_owner = owner(object::address_to_object<ObjectCore>(obj_owner));
    };
    obj_owner
}

public fun enable_ungated_transfer(ref: &TransferRef) acquires ObjectCore {
    assert!(!exists<Immovable>(ref.self), error::permission_denied(ENOT_MOVABLE));
    ...

public fun generate_linear_transfer_ref(ref: &TransferRef): LinearTransferRef acquires ObjectCore {
    assert!(!exists<Immovable>(ref.self), error::permission_denied(ENOT_MOVABLE));
    ...


public fun transfer_with_ref(ref: LinearTransferRef, to: address) acquires ObjectCore {
    assert!(!exists<Immovable>(ref.self), error::permission_denied(ENOT_MOVABLE));
    ...
```

在 `fungible_asset.move` 中：

```
#[resource_group_member(group = aptos_framework::object::ObjectGroup)]
struct Untransferable has key {}

/// Set that only untransferable stores can be create for this fungible asset.
public fun set_untransferable(constructor_ref: &ConstructorRef) {
    let metadata_addr = object::address_from_constructor_ref(constructor_ref);
    assert!(exists<Metadata>(metadata_addr), error::not_found(EFUNGIBLE_METADATA_EXISTENCE));
    let metadata_signer = &object::generate_signer(constructor_ref);
    move_to(metadata_signer, Untransferable {});
}

public fun is_untransferable<T: key>(metadata: Object<T>): bool {
    exists<Untransferable>(object::object_address(&metadata))
}

public fun create_store<T: key>(
    constructor_ref: &ConstructorRef, 
    metadata: Object<T>,
): Object<FungibleStore> {
    if (is_immovable(metadata)) {
        object::set_immovable(constructor_ref);
    };
    ...
```

## 六、参考实现

https://github.com/aptos-labs/aptos-core/pull/13175 



## 七、测试 (可选)

所有功能通过单元测试进行了验证。我们还设计了一个利用这一概念的真实可替代资产示例，并验证了冻结的账户不能存入或取出资产。



## 八、风险和缺点

可能存在通过 `ConstructorRef` 创建对象的模块引入了这个新功能，从而使依赖模块无效。当然，那些创建或操作对象的模块也可以任意破坏自己。



## 九、安全考虑

如果不正确实施，冻结的账户理论上可以获得他们本不应拥有的可替代资产的访问权。



## 十、未来潜力

我们期待随着这个特性的成熟，后期的反馈将决定我们是否需要为存储创建公开动态调度。



## 十一、时间线

这个特性应该在 2024 年 5 月初进入主线分支。从那时起，预计将在 1.13 分支中可用。
