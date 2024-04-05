---
aip: 11
title: 数字资产：作为对象的 Token 
author: davidiw, movekevin, lightmark, capcap, kslee8224, neoul
discussions-to: https://github.com/aptos-foundation/AIPs/issues/31
Status: Accepted
type: Standard (Framework)
created: 2023/1/9
updated: 2023/3/18
requires: "AIP-10: Move Objects"
---

[TOC]

# AIP - 11 - 数字资产：作为对象的 Token 

## 一、概述

本AIP提议使用 *Move Objects* 为 Aptos 提供一个新的 Token 标准。在这个模型中，每一个 Token 代表着庞大资产（assets）集合（collection）中的独一无二的某项资产。不论是代表资产集合，还是资产本身的 Move 对象，它们都利用了对象模型的可扩展性。此特性支持构建丰富的应用类别，例如非同质化代币（NFTs）、游戏资产和“无代码”解决方案等，同时确保这些应用在接口上保持兼容。这种设计使得基于 Token 的应用，不仅可以针对 Token 的高级功能进行专门化设计，例如某个具体的游戏出版商可能会有自己的 Token 商店，还可以实现对所有 Token 的泛化处理（ generalize across），如 Token 市场。



## 二、动机

对象提供了一种自然的方式来表示具有以下特点的 Token ：

- **全球寻址**：早期的 Aptos Token 标准允许 Token 放置在任何地方，如果用户没有仔细跟踪他们的 Token ，这种方式可能导致混乱的用户体验。
- **共享状态**：每个 Token 共享相同的核心数据模型和业务逻辑，使它们能广泛适用于不同的通用 Token 应用。
- **可扩展**： Token 公开了可以修改核心业务的逻辑。
- **存储效率**：每个 Token 将其资源存储在资源组内（resource group），可以被其他层共享。
- **可组合**：Token 能够拥有其他 Token ，使创建者可以构建丰富的链上应用程序。
- **简化**：最后， Token 具有明确定义的属性，如绑定灵魂 (Soul bound)、销毁 (Burning) 和冻结 (Freezing)，并且他们之间可以完美协作（work seamlessly）。

许多相同的属性同样适用于 Collection 。

## 三、基本原理

现有的 Token 模型存在两个主要问题：过于全面且在 Move 编程环境中过分依赖 `store` 功能。为了解决 Move 之前对象的局限性，Token 标准不得不试图应对 Token 领域可预见的挑战，如果这个问题处理不当，将难以避免产生问题。这个过程中衍生出了许多难以理解的功能，如 `property_versions`，同时也缺少了如冻结（freeze）或灵魂绑定（soul bound）等重要特性，以及一些其他的问题。另外，由于该模型过于依赖于 `store` 功能，Token 数据可能会被放置在链上任何地方，从而可能带来许多用户体验方面的问题。这个基于对象的新模型解决了许多已知的问题，但这样的改进可能需要现有的基础设施过渡到新模型，虽然这一变动对使用数据检索工具（indexers）和开发工具包（SDKs）的用户来说可能几乎无感。

同样重要的是，预计将继续维护和支持现行的 Token 标准，并为现有 Token 提供向新标准过渡的方案。社区也应意识到定期更新 Token 标准是必要的，我们需要适应这样的变革。

同样重要的是，它将期望继续管理和支持现有的 Token 标准。并为现有的 Token 提供迁移到新标准的路径。对于 Move 这样一个发展中的新兴语言，它的许多功能和设计模式仍在不断探索和完善中，社区也认识到有必要在 Token 标准中进行定期迭代。



## 四、规范

###  1. Token 和 Collection 的对象标识符

因为 Token 建立在对象之上，它们必须遵循对象标识符（Object IDs）的生成方式。对象标识符使用以下方式生成：`sha3_256(address_of(creator) | seed | 0xFE)` 或 `sha3_256(GUID | 0xFD)`。 Token 可以使用 `seed` 字段为每个 Token 生成唯一的地址。当生成种子（seed）时， Token 使用以下方式：`bcs::to_bytes(collection) | b"::" | bcs::to_bytes(name)`，也支持通过GUID生成对象标识符。类似地，Collection在生成种子时使用以下方式：`bcs::to_bytes(collection)`。这确保了所有Collection和 Token 的全局唯一性，除非 Token 或 Collection 已被销毁。请注意，选择`0xFE`是为了提供域分离（domain separation），并确保地址生成是唯一的，以防止对象和账户之间的重复地址生成。

由于现有的 Token 迭代不利用基于 `GUID` 的对象标识符生成，因为存在发现性（discoverability）挑战。具体来说，一旦 Aptos 有足够的节点内（on-node）索引来识别基于`GUID`的对象，我们才可能推出更新的标准。

### 2. 核心逻辑

Token 标准包括 Token 本身、Token 集合（Collections） 以及版税（Royalties）。

####  2.1 Tokens

在创建 Token 之前，必须首先存在一个 Collection  。

 Collection 的创建者是唯一能够向该 Collection 添加 Token 的实体（entity）。

 Token 公开了允许修改 Token 字段和销毁 Token 的 `ref`。

它不会改变 Move 对象的基础可扩展性。

####  2.2 Collections

 Collection 能够监控其供给量。目前， Collection 可以使用`FixedSupply`跟踪器或`UnlimitedSupply`。`FixedSupply`和`UnlimitedSupply`都跟踪所有铸造以及燃烧和铸造事件。此外，`FixedSupply` 还会与 Collection 关联的 Token 的最大发行数量。不使用跟踪器允许并行处理，但需代价是不能指定 Token 最大数量或为每个 Token 分配唯一索引到 Collection 中。

 Collection 公开了允许修改 Collection 字段的`ref`。

它不会改变Move对象的基础可扩展性。

#### 2.3 Royalties 

 Royalties  指定了一种方式来指定在销售中接收部分付款的实体（entity ）。

使用可扩展模型， Royalties 可以存在于 Collection 级别或 Token 级别。如果在两者都定义了，接口将优先使用 Token 级别的  Royalties 。

 Royalties  设置了一个可变引用 `ref` ，它允许创建和更新与 Collection 或 Token 相关的  Royalties  。

它不会改变Move对象的基础可扩展性。

### 3. 数据结构

本标准指定了 Token 、 Collection 和  Royaltie 的数据结构。

###  4. Token 数据结构

```move
#[resource_group_member(group = aptos_framework::object::ObjectGroup)]
/// 表示所有 Token 的通用字段。
struct Token has key {
    /// 该 Token 所属的 Collection 。
    collection: Object<Collection>,
    ///  Collection 内的唯一标识符，可选，0表示未分配
    collection_id: u64,
    ///  Token 的简要描述。
    description: String,
    ///  Token 的名称，在 Collection 内应该是唯一的；名称的长度应小于128个字符，例如："Aptos Animal #1234"
    name: String,
    ///  Token 的创建者名称。由于 Token 是以名称作为统一资源标识符（URI）的一部分创建的，该URI指向存储在链下的JSON文件。
    uri: String,
    /// 在 Token 发生任何变化时发出。
    mutation_events: event::EventHandle<MutationEvent>,
}

/// 这使得可以销毁NFT，如果可能，还将删除对象。注意，内部和 self 的数据每个占据 32 字节，而不是都有，这个数据结构对支持任意一种的小优化，同时占用固定的 34 字节。
struct BurnRef has drop, store {
    inner: Option<object::DeleteRef>,
    self: Option<address>,
}

/// 这使得更高级别的服务可以通过 MutatorRef 来修改描述和 URI 。
struct MutatorRef has drop, store {
    self: address,
}

/// 它记录了哪些字段发生了变化。这样的设计大大方便了索引器（indexers）的工作，让它们能够直观地了解写集（writeset）中发生的变动。
struct MutationEvent has drop, store {
    mutated_field_name: String,
}
```

####  4.1 Collection 数据结构

```move
#[resource_group_member(group = aptos_framework::object::ObjectGroup)]
/// 表示 Collection 的通用字段。
struct Collection has key {
    /// 此 Collection 的创建者。
    creator: address,
    ///  Collection 的简要描述。
    description: String,
    /// 类似于 Token 的可选分类。
    name: String,
    /// URI 用来指向存储在链外的 JSON 文件；但是这个 URL 应该有一个长度上限，我们尚未设置，您有什么建议吗？
    uri: String,
    /// 在 Collection 发生任何变化时发出。
    mutation_events: event::EventHandle<MutationEvent>,
}

/// 这使得更高级别的服务可以通过 MutatorRef 来修改描述和 URI 。
struct MutatorRef has drop, store {
    self: address,
}

/// 包含变更的字段名称。这使得索引器（indexers）更容易理解写入集（writeset）中的行为。
struct MutationEvent has drop, store {
    mutated_field_name: String,
}

#[resource_group_member(group = aptos_framework::object::ObjectGroup)]
/// 固定供应量追踪器，它能确保铸造代币的数量有限。此外，它还能将事件记录和供应量追踪功能添加到一个 Colection 里。
struct FixedSupply has key {
    /// 当前供应量 - 总燃烧量
    current_supply: u64,
    max_supply: u64,
    total_minted: u64,
    /// 在烧毁 Token 时发出。
    burn_events: event::EventHandle<BurnEvent>,
    /// 在铸造 Token 时发出。
    mint_events: event::EventHandle<MintEvent>,
}

#[resource_group_member(group = aptos_framework::object::ObjectGroup)]
/// 无限供应跟踪器，这对于为 Collection 添加事件和供应量跟踪非常有用。
struct UnlimitedSupply has key {
    current_supply: u64,
    total_minted: u64,
    /// 在烧毁 Token 时发出。
    burn_events: event::EventHandle<BurnEvent>,
    /// 在铸造 Token 时发出。
    mint_events: event::EventHandle<MintEvent>,
}
```

### 5. 数据限制

为了确保可以访问存储在链上的数据，例如在索引或查询API期间，这个标准对能够动态调整长度的字段设置了几个限制：

- 描述（Description）字段最多可以包含2048个字符（characters）。
- 名称（Name）字段最多可以包含128个字符（characters）。
- URI字段最多可以包含512个字符（characters）。

#### 5.1 版税数据结构

```move
#[resource_group_member(group = aptos_framework::object::ObjectGroup)]
/// 此 Collection 中 Token 的 Royalty --这是可选的
struct Royalty has copy, drop, key {
    numerator: u64,
    denominator: u64,
    /// Royalty 的收款方。参见`shared_account`，了解如何管理多个创作者的 Royalty。
    payee_address: address,
}

/// 这使得可以创建或覆盖 MutatorRef。
struct MutatorRef has drop, store {
    inner: ExtendRef,
}
```

### 6. API

此标准涵盖了进行 Token、集合（collection）和版税（Royalty）操作的所有的有效 API，包括存取（accessing）、修改（mutating）和删除（deleting）。它并未定义入口函数，因为 Token 标准推荐使用更具体的实施方式来处理。

####  6.1 Token API

```move
/// 从 Token 名称创建一个新的 Token 对象，并返回构造器引用（ConstructorRef），以便进行后续的特定处理。
public fun create_named_token(
    creator: &signer,
    collection_name: String,
    description: String,
    name: String,
    royalty: Option<Royalty>,
    uri: String,
): ConstructorRef;

/// 从帐户 GUID 创建一个新的 Token 对象，并返回用于进一步特殊处理的 ConstructorRef。
public fun create_from_account(
    creator: &signer,
    collection_name: String,
    description: String,
    name: String,
    royalty: Option<Royalty>,
    uri: String,
): ConstructorRef;

/// 根据创建者地址和 Collection 名称生成 Collection 地址
public fun create_token_address(creator: &address, collection: &String, name: &String): address;

/// 命名对象来源于一个种子（seed）， Collection 的种子（seed）是其名称。
public fun create_token_seed(collection: &String, name: &String): vector<u8>;

/// 创建一种变更引用（MutatorRef），它控管着对所有可被修改字段的变更能力（ability）。
public fun generate_mutator_ref(ref: &ConstructorRef): MutatorRef;

/// 创建 BurnRef，用于控制烧毁给定 Token 的能力（ability）。
public fun generate_burn_ref(ref: &ConstructorRef): BurnRef;

/// 从 BurnRef 中提取 Token 地址。
public fun address_from_burn_ref(ref: &BurnRef): address;

public fun creator<T: key>(token: Object<T>): address;

public fun collection<T: key>(token: Object<T>): String;

public fun collection_object<T: key>(token: Object<T>): Object<Collection>;

public fun description<T: key>(token: Object<T>): String;

public fun name<T: key>(token: Object<T>): String;

public fun uri<T: key>(token: Object<T>): String;

public fun royalty<T: key>(token: Object<T>): Option<Royalty>;

public fun burn(burn_ref: BurnRef);

public fun set_description(mutator_ref: &MutatorRef, description: String);

public fun set_name(mutator_ref: &MutatorRef, name: String);

public fun set_uri(mutator_ref: &MutatorRef, uri: String);
```

####  6.2  Collection API

```
/// 创建一个固定大小的 Collection ，或支持固定数量 Token 的 Collection 。这对于在链上创建有保证的、有限的供应数字资产非常有用。例如，一个包含1111个恶毒的毒蛇的 Collection 。请注意，创建诸如上限等限制会导致数据结构，阻止Aptos对此类型的 Collection 的铸造进行并行化。

/// 创建一个固定大小的 Collection ，或支持限定数量 Token（Token）的 Collection 。
/// 这种方式对于创造一个确定供应量且不可增加的区块链数字资产非常有用。
/// 举个例子，就如同一个只有 1111 条凶猛蝰蛇的特定集合。
/// 请注意，设置如最大数量这样的限制可能会形成一种数据结构，这种结构会阻止 Aptos 系统并行地铸造（mint）这类集合的 Token。
public fun create_fixed_collection(
    creator: &signer,
    description: String,
    max_supply: u64,
    name: String,
    royalty: Option<Royalty>,
    uri: String,
): ConstructorRef;

/// 创建一个无限的 Collection 。这支持供应量跟踪，但能不限制 Token 的供应。
public fun create_unlimited_collection(
    creator: &signer,
    description: String,
    name: String,
    royalty: Option<Royalty>,
    uri: String,
): ConstructorRef;

public fun create_collection_address(creator: &address, name: &String): address;

/// 命名对象来源于一个种子（seed）， Collection 的种子（seed）是其名称。
public fun create_collection_seed(name: &String): vector<u8>;

///创建了一个 MutatorRef（变更引用），它能够控制对所有可变字段的修改权限。
public fun generate_mutator_ref(ref: &ConstructorRef): MutatorRef;

public fun count<T: key>(collection: Object<T>): Option<u64> acquires FixedSupply;

public fun creator<T: key>(collection: Object<T>): address;

public fun description<T: key>(collection: Object<T>): String;

public fun name<T: key>(collection: Object<T>): String;

public fun uri<T: key>(collection: Object<T>): String;

public fun set_description(mutator_ref: &MutatorRef, description: String);

public fun set_uri(mutator_ref: &MutatorRef, uri: String);
```

### 7. 版税API

```
/// 添加 royalty，给定 ConstructorRef。
public fun init(ref: &ConstructorRef, royalty: Royalty);

/// 如果 royalty 不存在，则设置 royalty ，否则替换它。
public fun update(mutator_ref: &MutatorRef, royalty: Royalty);

public fun create(numerator: u64, denominator: u64, payee_address: address): Royalty;

public fun generate_mutator_ref(ref: ExtendRef): MutatorRef;

public fun exists_at(addr: address): bool;

public fun get<T: key>(maybe_royalty: Object<T>): Option<Royalty>;

public fun denominator(royalty: &Royalty): u64;

public fun numerator(royalty: &Royalty): u64;

public fun payee_address(royalty: &Royalty): address;
```

## 五、参考实现

https://github.com/aptos-labs/aptos-core/pull/7277 中的第一个提交

## 六、风险和缺陷

在 Token 领域（space）快速迭代可能会给寻求稳定的开发者带来负面影响。识别迭代的合适频率和动机是极其关键的。同时，我们也不希望因为稳定性与创新之间的张力而限制了潜在的创造可能性。

## 七、未来潜力

把 Token 看作对象既可以增强表达力，又可以实现核心 Token 代码与其应用之间的解耦（decouples）。在当前这个阶段之后，核心 Token 代码可能只会有很少的更改，新应用会在核心 Token 模块之上开发自己的代码。



## 八、建议的实施时间表

- 探索版本自2023年1月起存在于`move-examples`中。
- 到2023年3月中旬，对框架的一般性认识已经达成。
- 到2023年5月中旬，主网（Mainne）发布 -- 版本1.4。

