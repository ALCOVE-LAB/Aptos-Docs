---
aip: 78
title: Aptos Token Objects 框架更新
author: johnchanguk
discussions-to: ""
status: 审查中
last-call-end-date: "<最后留下反馈和评论的日期 (mm/dd/yyyy)>"
type: 框架
created: 2024-03-23
updated: "<更新日期 (mm/dd/yyyy)>"
---

[TOC]

# AIP-78 - Aptos Token对象框架更新

## 一、概述

本AIP旨在扩展`aptos-token-objects`框架中的`collection`、`token`和`property_map`模块。其目的是为 collection 和 token 对象引入可扩展性和自定义性。



## 二、动机

目前没有办法更改 collection 的名称或最大供应量，也无法使用提供的 token 和自定义种子（seed）创建 token 。用户可能希望这样做的原因包括：

1. 有些用户可能不希望链上数据公开可见，例如在创建 collection 时的 collection 名称。能够在稍后设置这一点，让用户能够自信地部署 collection ，而不会泄露任何敏感信息。
2. 用户可能希望调整 collection 的最大供应量。如果需要调整总供应量，或者仅仅是为了纠正错误，这种情况就会发生。
3. 当前的 API 只允许在确定性地址上创建一个具有单一名称的 token 。本 AIP 提供了地址由用户提供的种子派生的功能，允许在具有相同名称的确定性地址上创建多个 token 。
     1. Solidity通过 **[CREATE](https://docs.openzeppelin.com/cli/2.8/deploying-with-create2#create)** 和 [**CREATE2**](https://docs.openzeppelin.com/cli/2.8/deploying-with-create2) 操作码提供了这种机制。这允许用户在智能合约部署到地址之前向该地址发送资金。

此外，当前没有选项可以在 token 对象创建后向对象添加属性映射。目前唯一的选项是在对象创建期间通过`ConstructorRef`初始化属性映射。用户可能希望在以后的某个阶段允许在链上添加属性。



## 三、规格

### 1.  **collection 模块**

1. 在 `collection.move` 中添加一个新的方法，用于更改指定 collection 的名称。此方法将接受 `&MutatorRef`，与 collection 模块中的所有可变（mutator）函数一样。

```rust
public fun set_name(mutator_ref: &MutatorRef, name: String) acquires Collection {
    assert!(string::length(&name) <= MAX_COLLECTION_NAME_LENGTH, error::out_of_range(ECOLLECTION_NAME_TOO_LONG));
    let collection = borrow_mut(mutator_ref);
    collection.name = name;
    event::emit(
        Mutation { mutated_field_name: string::utf8(b"name") },
    );
}
```

2. 在 `collection.move` 中添加一个新的方法，用于更改 collection 的最大供应量。此方法将接受 `&MutatorRef`，与 collection 模块中的所有可变（mutator ）函数一样。此方法会断言新的 `max_supply` 大于当前供应量。

```rust
public fun set_max_supply(mutator_ref: &MutatorRef, max_supply: u64) acquires ConcurrentSupply, FixedSupply {
    let collection = object::address_to_object<Collection>(mutator_ref.self);
    let collection_address = object::object_address(&collection);

    if (exists<ConcurrentSupply>(collection_address)) {
        let supply = borrow_global_mut<ConcurrentSupply>(collection_address);
        let current_supply = aggregator_v2::read(&supply.current_supply);
        assert!(
            max_supply > current_supply,
            error::out_of_range(EINVALID_MAX_SUPPLY),
        );
        supply.current_supply = aggregator_v2::create_aggregator(max_supply);
        aggregator_v2::add(&mut supply.current_supply, current_supply);
    } else if (exists<FixedSupply>(collection_address)) {
        let supply = borrow_global_mut<FixedSupply>(collection_address);
        assert!(
            max_supply > supply.current_supply,
            error::out_of_range(EINVALID_MAX_SUPPLY),
        );
        supply.max_supply = max_supply;
    } else {
        abort error::invalid_argument(ENO_MAX_SUPPLY_IN_COLLECTION)
    };

    event::emit(SetMaxSupply { collection, max_supply });
}
```

### 2. **Token模块**

当前，所有 token 创建函数都接受 `collection_name` 参数，如果更改此参数可能会导致问题，因为将无法正确计算 collection 地址。为了解决这个问题，将添加四个新的函数来使用 `Object<Collection>` 创建 token ，将集合对象传递进来，而不是指定 `collection_name`。

1. 在 `collection.move` 中添加一个新方法，使用传入的 `Object<Collection>` 创建具有唯一地址的 token ，而不是使用 `collection_name`。

```rust
public fun create_token(
     creator: &signer,
     collection: Object<Collection>,
     description: String,
     name: String,
     royalty: Option<Royalty>,
     uri: String,
 ): ConstructorRef {
     create(creator, collection::name(collection), description, name, royalty, uri)
 }
```

2. 在 `collection.move` 中添加一个新方法，使用传入的 `Object<Collection>` 创建具有编号的 token 对象，而不是使用 `collection_name`。

```rust
public fun create_numbered_token_object(
   creator: &signer,
   collection: Object<Collection>,
   description: String,
   name_with_index_prefix: String,
   name_with_index_suffix: String,
   royalty: Option<Royalty>,
   uri: String,
): ConstructorRef {
   create_numbered_token(creator, collection::name(collection), description, name_with_index_prefix, name_with_index_suffix, royalty, uri)
}
```

3. 在 `collection.move` 中添加一个新方法，使用传入的 `Object<Collection>` 创建具有可预测地址的命名 token 对象，而不是使用 `collection_name`。

```rust
public fun create_named_token_object(
   creator: &signer,
   collection: Object<Collection>,
   description: String,
   name: String,
   royalty: Option<Royalty>,
   uri: String,
): ConstructorRef {
   create_named_token(creator, collection::name(collection), description, name, royalty, uri)
}
```

4. 在 `token.move` 模块中添加一个新方法，允许创建具有可预测地址的 token 。
    - 此方法接受 `collection`、`name` 和 `seed` 参数。 token 种子是通过连接 `name` 和 `seed` 生成的，而对象地址是通过使用 collection 创建者地址和 token 种子生成的。
    - 通过传入 collection 创建者、名称和种子，可以推导出 token 地址。`token_address_from_seed` 返回 token 地址。

```rust
/// 根据 token 名称和种子创建一个新的 token 对象。
/// 返回用于进一步特殊化的 ConstructorRef。
public fun create_named_token_from_seed(
    creator: &signer,
    collection: Object<Collection>,
    description: String,
    name: String,
    seed: String,
    royalty: Option<Royalty>,
    uri: String,
): ConstructorRef {
    let creator_address = signer::address_of(creator);
    let seed = create_token_seed(&name, &seed);

    let constructor_ref = object::create_named_object(creator, seed);
    create_common(&constructor_ref, creator_address, collection::name(collection), description, name, option::none(), royalty, uri);
    constructor_ref
}

#[view]
public fun token_address_from_seed<T: key>(collection: Object<T>, name: String, seed: String): address {
   let creator = collection::creator(collection);
   let seed = create_token_name_with_seed(&collection::name(collection), &name, &seed);
   object::create_object_address(&creator, seed)
}

public fun create_token_name_with_seed(collection: &String, name: &String, seed: &String): vector<u8> {
   assert!(string::length(name) <= MAX_TOKEN_NAME_LENGTH, error::out_of_range(ETOKEN_NAME_TOO_LONG));
   assert!(string::length(seed) <= MAX_TOKEN_SEED_LENGTH, error::out_of_range(ESEED_TOO_LONG));
   let seeds = *string::bytes(collection);
   vector::append(&mut seeds, b"::");
   vector::append(&mut seeds, *string::bytes(name));
   vector::append(&mut seeds, *string::bytes(seed));
   seeds
}
```

### 3. **Property map 模块**

在 `property_map.move` 模块中添加一个新方法，允许向对象添加 `PropertyMap`。此方法将接受 `&ExtendRef`，需要使用它生成对象的 `signer`。

```rust
public fun extend(ref: &ExtendRef, container: PropertyMap) {
    let signer = object::generate_signer_for_extending(ref);
    move_to(&signer, container);
}
```

## 四、风险和缺陷

**collection 模块**

如果在区块链合约或链下系统中，collection 名称作为了关键标识符，更改这个名称可能会破坏依赖于该名称的集成（integrations）或追踪系统。这样的变动可能会使维护可靠的审计路径带来额外难度。历史记录中的旧名称还可能引起在进行数据检索时的混乱或错误。

此外，更改最大供应量可能会影响依赖于这些信息为静态的服务。

**缓解措施**

目前， token 地址的计算方式与以前相同，但 `token.move` 允许更改 token 的名称。因此，我们将引导消费者以与检索 token 地址相同的方式来获取 collection 地址。

- 我们将通知依赖于通过 collection 名称计算 collection 地址的服务，因为它们将受到影响。需要存储 collection 地址以供参考，并需要将其直接传播给消费者（索引、市场等）。
- 我们将发出变更事件，我们将向依赖于集合名称计算集合地址的服务发出信号，表明集合的`name`字段已更改。因此，必须更新集合地址，使其不再通过集合名称进行计算。

只有在传递 `MutatorRef` 的情况下，才能更改 collection 的最大供应量。当 collection 的最大供应量发生变化时，将发出 `SetMaxSupply` 事件。这是一种通知下游何时发生此事件的方式。

**token 模块**

虽然可预测的地址对于某些用例可能是优势，但它们也可能会使恶意行为者更容易干扰创建过程或进行前置交易。

要生成可预测的地址，必须保证没有创建相同的地址。如果发生这种情况，则无法创建 token ，因为全局存储中已存在该对象。

**Property map模块**

在创建 token 对象后添加属性映射意味着可以在之后修改 token 。如果 token 最初设计为不可变的，则应丢弃 `ExtendRef`，因为这将阻止对当前 token 对象的修改。

## 五、时间表

目标是在 mainnet v1.11 中发布。

## 六、未来潜力

我们计划扩展这些模块，为当前的开发引入灵活性，并在我们认为合适时进行扩展。随着生态系统的成熟，我们可能会看到对生态系统有益的其他要求。

