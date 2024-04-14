---
aip: 22
title: 无代码数字资产（Token 对象）
author: davidiw, movekevin, lightmark, capcap
discussions-to: https://github.com/aptos-foundation/AIPs/issues/31
Status: Accepted
last-call-end-date (*可选):
type: 标准（框架）
created: 2023/3/18
updated:
requires: [AIP-11](Tokens as Objects)
---

[TOC]

# AIP-22 - 无代码数字资产（Token 对象）

## 一、概要

本 AIP 提出了围绕 Token 对象的框架，使开发人员可以在不编写任何 Move 代码的情况下创建 token 和 collection 。它为业务逻辑、数据布局上提供了决策，并且提供了必要的入口功能函数。



## 二、动机

大多数 Aptos 上的创作者都利用了现有的 token 标准，无需额外的功能。由于 Move 允许跨模块进行互操作，许多创作者可能永远不需要配置自己的 token 。这使得创作者可以更多地专注于链下内容，同时具有良好的灵活性来利用链上数据。

该标准采取以下立场：
* 创建者在创建时决定所有功能。这包括允许在创建后对 collection 、royalty 和 token 进行变动。
* 只有创建者才有权限对该对象进行改动。如果用户需要进行调整，可以通过将资源账户指定为创建者，并确保签名能力（signer capability）在相应的业务逻辑（business logic）下进行，从而达到此目的。
* 支持销毁和冻结 token，但必须在 collection 创建时指定。
* token 可以创建为灵魂绑定（soul bound）。
* royalty  和可变字段被定义在 collection 级别。
* 默认 royalty 将创建者设置为收款方。
* 用于读取和变更状态的公共函数（public function）。
* 用于变更状态的入口函数（entry function）。
* 用于读取状态的视图函数（view function）。
* 用于存roy储与 token 相关数据的属性映射，限制为 Move 基本类型、`vector<u8>` 和 `0x1::string::String`。

与原有的 token 标准相比，本标准缺少的特性包括：
* **可互换性：**这个特性在 Aptos 的 token 中很少用到，而它会使代码复杂度大幅提高。
* **半可互换性：**该特性的实施依赖于其可替代性，但目前还未在实际中被应用。
* **属性映射中的显式类型**。现在使用的属性映射不强制限定数据类型，也不对输入数据进行校验。因此，许多机构和组织已经开发出了各自的框架来操作属性映射，这样做最终导致了用户体验上的不一致。

## 三、理念

这是第一次尝试定义无代码（no-code）解决方案。对象模型使得我们能够创建新的无代码解决方案，并且对于大部分用户来说，这些解决方案与既有的使用体验无异，唯有那些寻求新特性的用户才能感受到差别。事实上，这些解决方案能够与现有方案并行运作。举个例子，如果可替代性或者半可替代性成为重要需求时，我们应当提出一个关于可替代性的新的无代码方案，并将其整合进 Aptos 的框架之中。

另一个选择是不在框架中设置它，而是建立一个固定不变的合约。在这个范畴里，最主要的担忧是可能出现的错误，或者是这种特殊无代码代币可能需要的附加功能。

## 四、规范

无代码解决方案由三个组件组成：

1. *AptosCollection* - 集合的封装。
2. *AptosToken* - 代币的封装。
3. *PropertyMap* - 用于代币的通用元数据。

### 1. AptosCollection

在创建时，AptosCollection 被初始化为以下内容：
* 确定 collection 字段（描述和 URI）的可变性。
* 指定整个 collection 的 royalty 。
* 确定所有 token 字段（描述、名称和 URI）的可变性。
* 确定 token 是否可冻结（freezable）或销毁（burnable）。
* 允许在生成后创建或变更代币属性。

AptosCollection 利用了 `FixedSupply` 并限制了可创建的 token 数量。这不够灵活，并且在之后无法调整。

### 2. AptosToken

collection 层主要决策 AptosToken。在代币生成期间，框架会审查 AptosCollection，以适当地从代币获取 `ref` 供以后使用，这包括变更、销毁和转移 `ref`。

collection 层为 AptosToken 做出了大部分决策。在 token 生成过程中，框架会审查 AptosCollection 以适当地获取 token 的 `ref`，以便后续使用，这包括变更（mutation）、销毁（burn）和转移`ref`（transfer `ref`）。

### 3. PropertyMap

属性映射（PropertyMap）是从原始 token 标准的属性映射派生出来的，但有一个关于验证的注意事项。在将任何字段写入属性映射之前，它必须具有有效的类型并符合该类型。用户可以像使用早期的属性映射一样利用这个属性映射，但需要注意的是，类型必须是字符串。

考虑到效率，属性映射将给定值存储为 `u8` 类型。

### 5. 数据结构

#### 5.1 AptosCollection 数据结构

```move
#[resource_group_member(group = aptos_framework::object::ObjectGroup)]
/// 用于管理无代码 collection 的存储状态。
struct AptosCollection has key {
    /// 用于变更 collection 字段
    mutator_ref: Option<collection::MutatorRef>,
    /// 用于变更 royalty
    royalty_mutator_ref: Option<royalty::MutatorRef>,
    /// 确定创建者是否可以变更 collection 的描述
    mutable_description: bool,
    /// 确定创建者是否可以变更 collection 的 URI
    mutable_uri: bool,
    /// 确定创建者是否可以变更 token 描述
    mutable_token_description: bool,
    /// 确定创建者是否可以变更 token 名称
    mutable_token_name: bool,
    /// 确定创建者是否可以变更 token 属性
    mutable_token_properties: bool,
    /// 确定创建者是否可以变更 token 的 URI
    mutable_token_uri: bool,
    /// 确定创建者是否可以销毁 token
    tokens_burnable_by_creator: bool,
    /// 确定创建者是否可以冻结 token
    tokens_freezable_by_creator: bool,
}
```

#### 5.2 AptosToken 数据结构

```move
#[resource_group_member(group = aptos_framework::object::ObjectGroup)]
/// 用于管理无代码 Token 的存储状态。
struct AptosToken has key {
    /// 用于销毁。
    burn_ref: Option<token::BurnRef>,
    /// 用于控制冻结。
    transfer_ref: Option<object::TransferRef>,
    /// 用于变更字段
    mutator_ref: Option<token::MutatorRef>,
    /// 用于变更属性
    property_mutator_ref: Option<property_map::MutatorRef>,
}
```

#### 5.3 PropertyMap 数据结构

```move
/// PropertyMap 为 AptosToken 提供通用的元数据支持。它是 SimpleMap 的一个特殊化版本，
/// 通过使用常量 u64 表示类型并以 bcs 格式存储值，从而强制执行严格的类型检查，同时使用最小的存储空间。
#[resource_group_member(group = aptos_framework::object::ObjectGroup)]
struct PropertyMap has key {
    inner: SimpleMap<String, PropertyValue>,
}

struct PropertyValue has drop, store {
    type: u8,
    value: vector<u8>,
}

struct MutatorRef has drop, store {
    self: address,
}

// PropertyValue::type
const BOOL: u8 = 0;
const U8: u8 = 1;
const U16: u8 = 2;
const U32: u8 = 3;
const U64: u8 = 4;
const U128: u8 = 5;
const U256: u8 = 6;
const ADDRESS: u8 = 7;
const BYTE_VECTOR: u8 = 8;
const STRING: u8 = 9;
```
PropertyMap 强制设置了最大项数为 1000，属性名长度最多为 128 个字符。在实践中，应尽可能保持 PropertyMap 的大小为最小。由于存储成本决定了写入成本，因此在索引和 API 查询期间加载大型数据存储可能会导致不良的延迟。一个非经验性的建议是将数据保持在10KB以下。 

### 6. API

由于 API 的数量庞大，内容留在参考实现中。

## 五、参考实现

在 https://github.com/aptos-labs/aptos-core/pull/7277 的第二次提交中。

## 七、风险和缺陷

PropertyMap 未来可能具有实用性，因为它是独立于 AptosCollection 和 AptosToken 设计的，以便在其他应用程序和标准中使用。

开发人员和应用程序不需要关注 AptosCollection 和 AptosToken 中的数据，因为所有核心数据已经存储在底层的 token 和 collection 中。如果社区提出新的程序或要求，新的标准可以与这些标准无缝共存。

除了与无代码解决方案相关的支持之外，没有明显的风险。



## 八、未来潜力

确定新特性是属于与标准相关的现有工作，还是属于并行标准，这一点非常重要。



## 九、建议的实施时间表

- 探索性版本自 2023 年 1 月起存储在 `move-examples` 中。
- 到 2023 年 3 月中旬达成对框架的普遍认可。
- 2023 年 5 月中旬的 Mainnet -- 1.4 版本发布。

