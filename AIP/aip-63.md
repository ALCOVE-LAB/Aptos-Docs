---
aip: 63
title: Coin to Fungible Asset Migration
author: lightmark, davidiw, movekevin
Status: Draft
type: Standard
created: <12/05/2023>
---

> [!TIP]
> 
> 译者注： 本文中的可替代资产(fungible asset) 和 `FA` 是一个意思，“硬币” 则代表了 `coin`
> 
> 原文链接：https://github.com/aptos-foundation/AIPs/blob/main/aips/aip-63.md
> 

[TOC]


# AIP-63 - Coin to Fungible Asset Migration (硬币向可替代资产迁移方案)

## 一、概述
[AIP-21](https://github.com/lightmark/AIPs/blob/main/aips/aip-21.md)提出了可替代资产（FA）标准，旨在取代现有的硬币标准，用于链上表达所有形式的可替代资产，包括但不限于传统的硬币。
随着Aptos生态系统的发展，存在从硬币框架向更先进的可替代资产框架过渡的需求。本提案（AIP）的宗旨是在尽量减小对现有生态系统项目和用户的影响的前提下，实现从硬币向可替代资产的过渡。

该提案制定了一套全面的硬币与可替代资产之间的映射机制，允许`coin`模块中的硬币与其对应的可替代资产互换，并且在可行情况下，将`CoinStore<CoinType>`转换为对应的主要可替代资产存储，以支持这一迁移过程。

### 1. 目标

本提案的目标包括：
- 向生态系统公开现有硬币对应的FA，以便开发者能够为其DApps和应用程序适配FA（未来将仅支持FA）。诸如DeFi、钱包及其他应用程序可以通过`coin`模块无缝识别FA与等同硬币，并能够等价透明地进行操作。
    - 若尚未存在，系统自动为硬币产生者地址下的硬币类型创建对应的可替代资产元数据，这包含`AptosCoin`。或者，硬币创建者可以选择手动为硬币类型建立可替代资产联结。
    - 硬币/替代资产产生者如果尚未建立连接，且所有条件均满足，可以选择手动进行硬币和可替代资产的配对。
    - 提供辅助函数，用于在已配对的硬币与可替代资产之间进行转换，同时提供不同级别的可见性。
    - 为了与当前依赖硬币标准API的现存DApps保持兼容，本提案建议调整`coin`模块中的若干函数，以便在需要时将硬币转换为其对应的可替代资产，并维持相同的函数签名。
- 为用户提供灵活性，随时可将其`CoinStore<CoinType>`迁入对应的FA `PrimaryFungibleStore`，这将赋予他们完全依据FA标准构建的dapps的体验。
- 制定一个综合性的迁移策略，以硬币向可替代资产的转化为目标，需符合以下要求：
    - 不对现有链上dapps造成破坏。
    - 尽可能降低dapp开发者和用户的迁移负担，即迁移过程应最小化干扰并尽量透明化。技术层面上，这意味着解决方案应能在用户同时持有同一资产类型的FA和硬币时，实现无缝衔接。

### 2. 不在讨论范围内的内容

- 强制用户将他们的`CoinStore`迁移至`PrimaryFungibleStore`
- 使APT的FA `PrimaryFungibleStore`成为新账户默认的`CoinStore<AptosCoin>`替代选项。



## 二、动机

在Aptos网络中DeFi应用广泛接纳之前，启动迁移过程以简化新dapps的部署至关重要。否则，开发者可能需要在两个不同的标准上执行相同的业务逻辑，而资产流动性可能会因此分散，这对于整个Aptos生态系统来说可能是一个沉重且不健康的负担。此外，CeFi应用也有可能面临类似的挑战。当前混乱的局面，即存在多种资产标准，给开发者和用户带来了显著的困扰，这是一个亟需解决的问题。



## 三、影响

- 硬币创建者：
  - 每种硬币将仅与一种FA类型匹配。原有的权限模型（如`MintCapability`、`FreezeCapability`、`BurnCapability`）将用于生成匹配FA的对应权限引用(`MintRef`、`TransferRef`、`BurnRef`)。
- 用户：
  - 迁移过程设计为平滑无障碍，但要求用户主动参与。用户启动迁移功能后，迁移流程将自动展开。迁移完成后，用户存储中的硬币将无缝转变为对应FA形式，并且所有新的存款将直接进入主要FA存储。
- DApps：
  - 智能合约：现有基于硬币标准的dApps将不会因迁移而中断，由于该迁移策略设计为无破坏性质。与这些协议互动的用户在迁移前后可能持有同类资产的硬币或FA。取款过程亦会遵循此规则。迁移完成之后，所有账户的硬币都将以FA形式表示，使得生态系统得以依照FA API开展开发，并能发行无需匹配硬币的全新FA。
    - 索引：迁移对索引服务可能带来的影响会有所差异，这种变化可能根据不同服务的跟踪余额与供应策略而异。
    - 事件：迁移完成之后，仅会生成与FA相关的事件。
    - WriteSet更改：一旦触发迁移代码，`CoinStore`将被清除，由主要FA存储所取代。在某些情况下，可能会有`CoinStore`和FA存储的暂时共存。
- 钱包：
  - 钱包需更新以正确显示余额，结合索引，反映硬币及其匹配FA的总额。
  - 随着迁移完成以及无配对硬币FA的生成，钱包应转向使用FA相关的Move API，如资产转移操作。



## 四、替代方案

1. 手动匹配方案，要求硬币创建者为每种硬币资产手动生成对应FA（如果未存在），并在映射中进行注册。这一方式面临几个问题：
    - 如何处理权限引用(`MintRef`、`TransferRef`、`BurnRef`)？若FA缺乏这些怎么办？
    - 如硬币创建者未能创设FA，则可能妨碍特定硬币类型的迁移进程。
    - 某些硬币是在无实际所有者控制的资源账户下初始化的。因此无法获得这些账户的`signer`进行FA配对。

该方案相对于方案1的优势包括：
- 无需硬币创建者的手动介入，适用于个人账户和资源账户。
- 保留原始的权限模型，确保与匹配FA的一致。

2. 选择性迁移方案，用户可明确授权通过特定账户标志，表明他们是否将配对的硬币和FA视作同一资产。在此模式下，若用户选择不迁移，则至其账户的APT FA不可作为APT硬币使用，而只能当作独立的FA资产。

相对于方案2的优势包括：
- 对用户和dApps的影响最小化。
    - 普通用户通常只在乎自己的资产总额，而对资产背后的技术细节不太关注。他们可能不完全理解区别于传统硬币的FA概念。因此，如果用户收到了10个APT FA，却无法在自己的余额中直观地看到这些资产，可能会引发不安和困惑。
    - 用户无需进行注册或采取额外步骤，因为迁移程序设计为最终可以主动、自动地进行，无需用户自己手动注册或参与。
    - 由于迁移流程会透明化，DApps在展示用户余额时将遇到更少的问题。这样，即使是不同用户，余额的显示方式也将保持统一。
- 创建一个对用户完全透明、无需用户主动介入的迁移流程。
- 消除了过多的配置资源以及后续的清理工作。
- 直接解决了资产类型碎片化的问题，这一问题阻碍了FA标准的广泛采用和开发。例如，在余额同时由硬币和FA构成时，对于仅支持FA的DApps可能出现无法正常运作的问题。如用户想要通过使用`fungible_asset::transfer`转移20个APT，但仅有10个APT硬币和10个APT FA时，将导致交易无法成功执行。
- 选择方案2并未减轻，反而可能增加了工程负担，无论是对内还是对外部开发者而言。从长期角度来看，团队最终仍需要满足本提案中列出的所有条件和需求。



## 五、规范

`CoinConversionMap`将存储在`@aptos_framework`下：
```rust
    /// 硬币和可替代资产之间的映射。
    struct CoinConversionMap has key {
        coin_to_fungible_asset_map: Table<TypeInfo, address>,
    }
```
通过在可替代资产元数据对象中插入一个 `PairedCoinType` 资源来实现反向映射：
```rust
/// 硬币和可替代资产之间的映射。
    struct PairedCoinType has key {
        type: TypeInfo,
    }
```

这两个辅助函数执行转换：
```rust
// 从硬币转换为可替代资产
public fun coin_to_fungible_asset<CoinType>(coin: Coin<CoinType> ): FungibleAsset;

// 从可替代资产转换为硬币。若不使用 public 则用以推动向FA的迁移。
public fun fungible_asset_to_coin<CoinType>(fungible_asset: FungibleAsset):
```

- 对APT的匹配FA，元数据地址将被设定为`0xA`，而对于其他硬币类型，则可指定为任意地址。

注意事项：本AIP不包含由FA转换回硬币的流程，因此`fungible_asset_to_coin`函数并非设为公开可用。当执行`coin_to_fungible_asset<CoinType>`函数，且尚未存在与之匹配的FA时，硬币模块将自动创建一个新的FA元数据，并将其加入到类型映射中，确保与`CoinType`相匹配。这个新创建的匹配FA将拥有与硬币类型一致的名称、符号和小数精度。

函数`maybe_convert_to_fungible_store`可以实现移除`CoinStore<CoinType>`并自动将所有剩余硬币转换为匹配的FA，继而存储于主要FA存储中。
```rust

fun maybe_convert_to_fungible_store<CoinType>(account: address) acquires CoinStore, CoinConversionMap, CoinInfo {
    if (exists<CoinStore<CoinType>>(account)) {
        let CoinStore<CoinType> {
            coin,
            frozen,
            deposit_events,
            withdraw_events
        } = move_from<CoinStore<CoinType>>(account);
        event::emit(CoinEventHandleDeletion {
        event_handle_creation_address: guid::creator_address(event::guid(&deposit_events)),
        deleted_deposit_event_handle_creation_number: guid::creation_num(event::guid(&deposit_events)),
        deleted_withdraw_event_handle_creation_number: guid::creation_num(event::guid(&withdraw_events))});
        event::destory_handle(deposit_events);
        event::destory_handle(withdraw_events);
        let fungible_asset = coin_to_fungible_asset(coin);
        let metadata = fungible_asset::asset_metadata(&fungible_asset);
        let store = primary_fungible_store::ensure_primary_store_exists(account, metadata);
        fungible_asset::deposit(store, fungible_asset);
        // 注意：
        // 在调用此函数之前，主要的可替代资产存储可能已经存在。
        // 在这种情况下，如果账户拥有一个冻结的CoinStore和一个未冻结的主要可替代资产存储，
        // 这个函数会将剩余的硬币转换并存入主要存储，并冻结它，
        // 以使`frozen`的语义尽可能一致。
        fungible_asset::set_frozen_flag_internal(store, frozen);
    };
}
```

`Supply`和`balance`被相应地修改，以反映硬币和配对的可替代资产的总和。

此外，为了保持能力语义的一致性，本AIP提出了3个辅助函数，从硬币的`MintCapability`、`FreezeCapability`和`BurnCapability`生成配对的可替代资产的`MintRef`、`TransferRef`和`BurnRef`，这样`fungible_asset.move`中的move API也可以在新代码中使用。
```rust
    /// 从`MintCapability`获取硬币类型的配对可替代资产的`MintRef`。
    public fun paired_mint_ref<CoinType>(_: &MintCapability<CoinType>): MintRef;

    /// 从`FreezeCapability`获取硬币类型的配对可替代资产的TransferRef。
    public fun paired_transfer_ref<CoinType>(_: &FreezeCapability<CoinType>): TransferRef;

    /// 从`BurnCapability`获取硬币类型的配对可替代资产的`BurnRef`。
    public fun paired_burn_ref<CoinType>(_: &BurnCapability<CoinType>): BurnRef;
```

### 1. 案例研究
以APT硬币为例。假设APT硬币和FA已经创建并配对。

1. 在用户删除`CoinStore<AptosCoin>`之前：
- `withdraw`/`burn_from`/`collect_into_aggregatable_coin`：首先从`CoinStore<AptosCoin>`中提取APT硬币。如果余额不足，那么从`PrimaryFungibleStore`中提取更多的APT FA，并将其转换回硬币。
- `deposit`：APT硬币将被存入`CoinStore<AptosCoin>`。
注意：即使用户还没有迁移`CoinStore`，他们也可以通过可替代资产模块API从其他人那里获取APT FA，而不是硬币模块API。

2. 在用户删除`CoinStore<AptosCoin>`之后：
- `withdraw`/`burn_from`/`collect_into_aggregatable_coin`：APT FA将全部从APT `PrimaryFungibleStore`中提取出来，并转换回硬币。
- `deposit`：APT硬币将被转换为FA，并存入APT `PrimaryFungibleStore`。

`coin::register`还没有被弃用，所以有可能在用户删除`CoinStore`后，它会再次被创建。所以在罕见的情况下，案例2可以回到案例1。然而，这个AIP的目标不是要消除这种情况，而是要让生态系统的dapps能够在用户的账户下同时存在同一种资产类型的硬币和FA，并且能够完美地工作。
关于长期的计划，请参考[未来计划](#future-potential)



## 六、参考实现
https://github.com/aptos-labs/aptos-core/pull/11224



## 七、测试（可选）

通过单元测试进行了彻底的验证。将部署到devnet并手动测试。



## 八、风险和不足之处

- 所有新的Dapp若想通过可替代资产（FA）的Move API与用户互动，都假设用户账户已完成迁移。尚未迁移的账户可能无法与这些Dapp兼容。
- 那些不在`CoinStore`中的硬币无法通过本AIP自动迁移。人们需要手动完成迁移任务。然而，作为一个框架，我们没法确保了解到这一过程何时彻底结束。



## 九、未来潜力

### 1. 阶段1
本AIP为迁移计划的初步阶段。意味着，如果用户未主动迁移其`CoinStore`，同时没有通过FA API接入FA，那么从用户角度看，这并无任何操作上的变化。

### 2. 阶段2
所有Dapps、交易所、钱包等都需更新其机制，从仅仅监测`CoinStore`余额，转向同时跟踪`CoinStore`与`PrimaryFungibleStore`余额并显示总和。

### 3. 阶段3
将禁止创建新的`CoinStore`实例。随后会执行以下步骤：
- 迁移所有现有账户的`CoinStore<AptosCoin>`至APT的`PrimaryFungibleStore`。
- 在执行任何硬币类型的`Withdraw<CoinType>`和`Deposit<CoinType>`函数时，内部触发迁移流程。这样，在用户与这些资产进行交互时，可以间接帮助实现其他硬币类型的迁移。

### 4. 阶段4
当绝大部分资产均已迁移至FA之后，我们将需要提出新的AIP来禁止新硬币的生成，鼓励整个生态系统基于FA模块API发展，并淘汰旧的硬币模块。



## 十、时间线

### 1. 建议的实施/部署时间线
预计于v1.10版本完成，预期与2024年第一季度的v1.10/v1.11版本一同发布。

### 2. 建议的开发者平台支持时间表
计划在2024年第一季度推出支持。



## 十一、安全性顾虑
代码需经过严格的安全审核。