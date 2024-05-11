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
> 译者注： 本文中的可替代资产(fungible asset) 和 `FA` 是一个意思，“ Coin ” 则代表了 `coin`
> 
> 原文链接：https://github.com/aptos-foundation/AIPs/blob/main/aips/aip-63.md
> 

[TOC]


# AIP-63 - Coin to Fungible Asset Migration ( Coin 向可替代资产迁移方案)

## 一、概述
[AIP-21](https://github.com/lightmark/AIPs/blob/main/aips/aip-21.md)提出了可替代资产（FA）标准，旨在取代现有的 Coin 标准，用于链上表达所有形式的可替代资产，包括但不限于传统的 Coin。

随着Aptos生态系统的发展，存在从 Coin 框架向更先进的可替代资产框架过渡的需求。本 AIP 的宗旨是在尽量减小对现有生态系统项目和用户的影响的前提下，实现从 Coin 向可替代资产的过渡。

本 AIP 提议在Coin 与可交换资产之间建立一种全局映射关系，从而允许 `coin` 模块把成对的 Coin 和可交换资产等同对待。这样做便于通过将 `CoinStore<CoinType>` 转换为相应可交换资产的主要存储，以实现平滑迁移。



### 1. 目标

本提案的目标包括：

- 我们开始向整个生态系统公开现有 Coin 的可替代资产（FA），以便开发者们可以开发和准备能支持可替代资产（FA）的 dapps 和应用程序（未来可能只有 FA）。这样，DeFi、电子钱包以及其他应用程序都可以通过 Coin 模块，自然地处理 FA 和相等的 Coin ，并透明地使用它们。
  - 若尚未存在，系统自动为 Coin 产生者地址下的 Coin 类型创建对应的可替代资产元数据，这包含`AptosCoin`。或者， Coin 创建者可以选择手动为 Coin 类型建立可替代资产的连接。

- Coin / 可替代资产的创建者如果尚未建立连接，且所有条件均满足，可以选择手动进行 Coin 和可替代资产的配对。
- 提供辅助函数，用于在已配对的 Coin 与可替代资产之间进行转换，同时提供不同级别的可见性。
- 为了与当前依赖 Coin 标准 API 的现存DApps保持兼容，本提案建议调整`coin`模块中的若干函数，以便在需要时将 Coin 转换为其对应的可替代资产，并维持相同的函数签名。

- 为用户提供灵活性，随时可将其`CoinStore<CoinType>`迁入对应的FA `PrimaryFungibleStore`，这将赋予他们完全依据 FA 标准构建的 DApps 的体验。
- 制定一个综合性的迁移策略，以 Coin 向可替代资产的转化为目标，需符合以下要求：
    - 不会对现有链上 DApps 造成破坏。
    - 尽可能降低 DApp 开发者和用户的迁移负担，即迁移过程应最小化干扰并尽量透明化。技术层面上，这意味着解决方案应能在用户同时持有同一资产类型的 FA 和 Coin 时，实现无缝衔接。

### 2. 不在讨论范围内的内容

- 强制用户将他们的`CoinStore`迁移至`PrimaryFungibleStore`
- 使 APT 的 FA `PrimaryFungibleStore` 成为新账户默认的`CoinStore<AptosCoin>`替代选项。



## 二、动机

在 Aptos 网络的 DeFi 应用得到广泛推广之前，发起一个迁移过程以简化新 dapps 的开发极为关键。否则，开发者可能需要在两套不同的标准上重复实现相同的业务逻辑，导致流动性分散，这不仅繁琐还可能对整个 Aptos 生态系统造成不利影响。同样，CeFi（中心化金融）应用也可能面临同样的挑战。多种资产标准并存的当前状态为开发者和用户带来了显著的困扰，亟需采取措施解决。



## 三、影响

-  Coin 创建者：
  - 每种 Coin 将仅与一种 FA 类型匹配。原有的权限模型（如`MintCapability`、`FreezeCapability`、`BurnCapability`）将用于生成匹配 FA 的对应权限引用(`MintRef`、`TransferRef`、`BurnRef`)。
- 用户：
  - 迁移过程设计为平滑无障碍，但要求用户主动参与。用户启动迁移功能后，迁移流程将自动展开。迁移完成后，用户存储中的 Coin 将无缝转变为对应 FA 形式，并且所有新的存款将直接进入主要的 FA 存储。
- DApps：
  - 智能合约：现有基于 Coin 标准的 dApps 将不会因迁移而中断，由于该迁移策略设计为无破坏性质。与这些协议互动的用户在迁移前后可能持有同类资产的 Coin 或 FA。取款过程亦会遵循此规则。迁移完成之后，所有账户的 Coin 都将以 FA 形式表示，使得生态系统得以依照 FA API 开展开发，并能发行无需匹配 Coin 的全新 FA。
    - 索引：迁移对索引服务可能带来的影响会有所差异，这种变化可能根据不同服务的跟踪余额与供应策略而异。
    - 事件：迁移完成之后，仅会生成与 FA 相关的事件。
    - WriteSet 更改：一旦触发迁移代码，`CoinStore`将被清除，由主要FA存储所取代。在某些情况下，可能会有`CoinStore`和 FA 存储的暂时共存。
- 钱包：
  - 钱包需更新以正确显示余额，结合索引，反映 Coin 及其匹配FA的总额。
  - 随着迁移完成以及无配对 Coin FA 的生成，钱包应转向使用 FA 相关的 Move API，如资产转移操作。



## 四、替代方案

1. 手动匹配方案，要求 Coin 创建者为每种 Coin 资产手动生成对应FA（如果未存在），并在映射中进行注册。这一方式面临几个问题：
    - 如何处理权限引用(`MintRef`、`TransferRef`、`BurnRef`)？若FA缺乏这些怎么办？
    - 如 Coin 创建者未能创设FA，则可能妨碍特定 Coin 类型的迁移进程。
    - 某些 Coin 是在无实际所有者控制的资源账户下初始化的。因此无法获得这些账户的`signer`进行FA配对。

建议的方案相对于方案1的优势包括：
- 无需 Coin 创建者的手动介入，适用于个人账户和资源账户。
- 保留原始的权限模型，确保与匹配 FA 的一致。

2. 选择性迁移方案，用户可明确授权通过特定账户标志，表明他们是否将配对的 Coin 和 FA 视作同一资产。在此模式下，若用户选择不迁移，则其账户的 APT FA 不可作为 APT Coin 使用，而只能当作独立的 FA 资产。

建议的方案相对于方案2的优势包括：
- 对用户和 dApps 的影响最小。
    - 普通用户通常只在乎自己的资产总额，而对资产背后的技术细节不太关注。他们可能不完全理解区别于传统 Coin 的 FA 概念。因此，如果用户收到了 10 个 APT FA，却无法在自己的余额中直观地看到这些资产，可能会引发不安和困惑。
    - 用户无需进行注册或采取额外步骤，因为迁移程序设计为最终可以主动、自动地进行，无需用户自己手动注册或参与。
    - 由于迁移流程会透明化，DApps 在展示用户余额时将遇到更少的问题。这样，即使是不同用户，余额的显示方式也将保持统一。
- 创建一个对用户完全透明、无需用户主动介入的迁移流程。
- 消除了过多的配置资源以及后续的清理工作。
- 直接解决了资产类型碎片化的问题，这一问题阻碍了 FA 标准的广泛采用和开发。例如，在余额同时由 Coin 和FA构成时，对于仅支持 FA 的 DApps 可能出现无法正常运作的问题。如用户想要通过使用`fungible_asset::transfer`转移 20 个 APT ，但仅有 10 个 APT  Coin 和 10 个 APT FA 时，将导致交易无法成功执行。
- 选择方案 2 并未减轻工作负担，反而可能增加了负担，无论是对内部还是对外部开发者而言。从长期角度来看，团队最终仍需要满足本提案中列出的所有条件和需求。



## 五、规范

`CoinConversionMap`将存储在`@aptos_framework`下：
```rust
///  Coin 和可替代资产之间的映射。
struct CoinConversionMap has key {
    coin_to_fungible_asset_map: Table<TypeInfo, address>,
}
```
通过在可替代资产元数据对象中插入一个 `PairedCoinType` 资源来实现反向映射：
```rust
///  存储在可替代资产元数据对象中的配对 Coin 类型信息。
struct PairedCoinType has key {
    type: TypeInfo,
}
```

这两个辅助函数执行转换：
```rust
// 从 Coin 转换为可替代资产
public fun coin_to_fungible_asset<CoinType>(coin: Coin<CoinType> ): FungibleAsset;

// 从可替代资产转换为 Coin 。若不使用 public 则用以推动向 FA 的迁移。
public fun fungible_asset_to_coin<CoinType>(fungible_asset: FungibleAsset):
```

- 对 APT 的匹配 FA，元数据地址将被设定为`0xA`，而对于其他 Coin 类型，则可指定为任意地址。

注意事项：本 AIP 不包含由 FA 转换回 Coin 的流程，因此`fungible_asset_to_coin`函数并非设为公开可用。当执行`coin_to_fungible_asset<CoinType>`函数，且尚未存在与之匹配的FA时， Coin 模块将自动创建一个新的FA元数据，并将其加入到类型映射中，确保与`CoinType`相匹配。这个新创建的匹配FA将拥有与 Coin 类型一致的名称、符号和小数精度。

函数`maybe_convert_to_fungible_store`可以实现移除`CoinStore<CoinType>`并自动将所有剩余 Coin 转换为匹配的FA，继而存储于主要 FA 存储中。
```rust
fun maybe_convert_to_fungible_store<CoinType>(account: address) acquires CoinStore, CoinConversionMap, CoinInfo {
    if (!features::coin_to_fungible_asset_migration_feature_enabled()) {
        abort error::unavailable(ECOIN_TO_FUNGIBLE_ASSET_FEATURE_NOT_ENABLED)
    };
    let metadata = ensure_paired_metadata<CoinType>();
    let store = primary_fungible_store::ensure_primary_store_exists(account, metadata);
    let store_address = object::object_address(&store);
    if (exists<CoinStore<CoinType>>(account)) {
        let CoinStore<CoinType> { coin, frozen, deposit_events, withdraw_events } = move_from<CoinStore<CoinType>>(
            account
        );
        event::emit(
            CoinEventHandleDeletion {
                event_handle_creation_address: guid::creator_address(
                    event::guid(&deposit_events)
                ),
                deleted_deposit_event_handle_creation_number: guid::creation_num(event::guid(&deposit_events)),
                deleted_withdraw_event_handle_creation_number: guid::creation_num(event::guid(&withdraw_events))
            }
        );
        event::destory_handle(deposit_events);
        event::destory_handle(withdraw_events);
        fungible_asset::deposit(store, coin_to_fungible_asset(coin));
        // 注意：
		// 在调用此函数之前，主要的可替代存储可能已经存在。
		// 在这种情况下，如果账户拥有一个已冻结的 CoinStore 和一个未冻结的主要可替代存储，此函数将会转换并存入剩余的货币到主要存储中，并将其冻结，以尽可能地保持“冻结”语义的一致性。
        fungible_asset::set_frozen_flag_internal(store, frozen);
    };
    if (!exists<MigrationFlag>(store_address)) {
        move_to(&create_signer::create_signer(store_address), MigrationFlag {});
    }
}
```

当一个 `CoinStore<CoinType>` 被停用，并且其中剩余的货币被转移到相应的可替代资产时，将触发一个 `CoinEventHandleDeletion` 事件。这作为一个通知给观察者，表明事件句柄标识的即将删除。 

同时，如果调用了 `maybe_convert_to_fungible_store`，框架将在主要的可替代存储对象中创建一个 `MigrationFlag` 资源，表示用户已经迁移。根据这个标志，在 `coin` 模块中的 API 行为将有所不同，详细情况见提供的案例研究。

`Supply`和`balance`被相应地修改，以反映 Coin 和配对的可替代资产的总和。

此外，为了保持能力语义的一致性，本AIP提出了3个辅助函数，从 Coin 的`MintCapability`、`FreezeCapability`和`BurnCapability`生成配对的可替代资产的`MintRef`、`TransferRef`和`BurnRef`，这样`fungible_asset.move`中的move API也可以在新代码中使用。

例如，对于 `MintRef`，辅助函数集合如下：

```rust
#[view]
/// 检查是否未使用 `MintRef`。
public fun paired_mint_ref_exists<CoinType>(): bool;

/// 使用热土豆收据从 `MintCapability` 获取配对可替代资产的 `MintRef`。
public fun get_paired_mint_ref<CoinType>(_: &MintCapability<CoinType>): (MintRef, MintRefReceipt);

/// 使用热土豆收据在使用后返回 `MintRef`。
public fun return_paired_mint_ref(mint_ref: MintRef, receipt: MintRefReceipt);
```

这里采用了"热土豆"移交机制，以确保只能使用 `&MintCapability` "借用"一次 `MintRef`，但必须在同一交易结束时归还。

`MintRefReceipt` 的定义如下：

```rust
// 用于闪电借用 MintRef 的热土豆收据。
struct MintRefReceipt {
    metadata: Object<Metadata>,
}
```

### 1. 案例分析

让我们以 APT 币为例进行说明。假设已经创建了 APT 币和 FA，并将它们配对。

#### 情况1

对于选择参与此迁移的用户 A：

- 如果迁移被触发，用户 A 的该 `CoinType` 的所有 coin 将从 `CoinStore` 转换为带有 `MigrationFlag` 的 `PrimaryFungibleStore`。`CoinStore` 将被删除。
- 如果发送更多该类型的币给用户 A，则这些币将自动通过 `coin::deposit` 转换为 FA。
- 对于用户 A，`coin::balance` 将考虑用户的 FA 余额。
- 如果在用户 A 的账户上调用 `coin::withdraw`，它将把指定数量的 FA 转换回币。
- 如果 `CoinType` 是 APT，为了支付 gas 费用，`coin::burn_from` 将从用户 A 的 APT 的 `PrimaryFungibleStore` 中 burn。

#### 情况2

对于另一个没有选择参与此迁移的用户 B：

- 有人通过 `primary_fungible_store::deposit` 向他们发送相应的 FA。
- 现在用户 B 既有 `CoinStore`，也有没有此资产类型的 `MigrationFlag` 的 `PrimaryFungibleStore`。
- 如果向用户 B 发送一些 coin ，则这些 coin 将存入 `CoinStore`。
- `coin::balance` 会考虑来自 `CoinStore` 和 `PrimaryFungibleStore` 的余额。
- 如果调用 `coin::withdraw`，则首先从 `CoinStore` 中提取，然后再从 `PrimaryFungibleStore` 中提取，如果有不足的话。

#### 情况3

对于新用户 C，特别是 APT：

- APT 被转移到 C 的地址，如果他们的账户在链上不存在，则调用 `account::create_account`。`CoinStore<AptosCoin>` 仍然被创建，现在用户 C 可以选择用户 A 或 B 的路径。

#### 情况4

对于尚未在链上创建账户的用户 D：

- 用户 D 发送了一个由他人支付的代付交易。
- gas 费支付者流程发现用户 D 的账户尚不存在（没有序列号）。它通过调用 [account::create_account_if_does_not_exist](https://github.com/aptos-labs/aptos-core/blob/main/aptos-move/aptos-vm/src/aptos_vm.rs#L538) 自动创建账户。
  用户 D 的账户现在在链上创建了，但没有关联的 CoinStore。
- 有人把 APT 作为 FA 发送给用户 D。现在用户 D 有了一个带有 `MigrationFlag` 的 APT 的 `PrimaryFungibleStore`。
- 用户 D 可以从他们的 FungibleStore 中支付 gas 费，并且可以自行发送交易。
- 用户 D 选择参与 APT 的 FA 迁移（仍然是代付）。没有要删除的 `CoinStore`。APT 的 `PrimaryFungibleStore` 已经创建。
- 从现在开始，用户 D 类似于用户 A。

#### 情况5

对于用户 E：
用户 E 选择参与 FA 迁移，现在有了带有 `MigrationFlag` 的 APT 的 `PrimaryFungibleStore`，没有 `CoinStore<AptosCoin>`：

- 有人为用户 E 调用 `coin::register<AptosCoin>`。这会报错，对于选择使用 `MigrationFlag` 的用户，不允许这样操作。在他们的流程的第 6 步之前，这不适用于用户 D。

`coin::is_account_registered` 和 `coin::register` 尚未被弃用，但已修改，以便如果用户的配对 FA 的主要可交换存储 `PrimaryFungibleStore<T>` 已经创建，则不能创建 `CoinStore<T>`。因此，情况 2 的用户永远不能回到情况 1 。
对于长期计划，请参考[未来计划](#九、未来潜力)。



## 六、参考实现
https://github.com/aptos-labs/aptos-core/pull/11224



## 七、测试（可选）

通过单元测试进行了彻底的验证。将部署到devnet并手动测试。



## 八、风险和不足之处

- 所有新的Dapp若想通过可替代资产（FA）的Move API与用户互动，都假设用户账户已完成迁移。尚未迁移的账户可能无法与这些Dapp兼容。
- 那些不在`CoinStore`中的 Coin 无法通过本AIP自动迁移。人们需要手动完成迁移任务。然而，作为一个框架，我们没法确保了解到这一过程何时彻底结束。



## 九、未来潜力

### 1. 阶段1
本 AIP 为迁移计划的初步阶段。意味着，如果用户未主动迁移其`CoinStore`，同时没有通过 FA API 接入 FA，那么从用户角度看，这并无任何操作上的变化。

### 2. 阶段2
所有 Dapps、交易所、钱包等都需更新其机制，从仅仅监测`CoinStore`余额，转向同时跟踪`CoinStore`与`PrimaryFungibleStore`余额并显示总和。

### 3. 阶段3
将禁止创建新的`CoinStore`实例。随后会执行以下步骤：
- 迁移所有现有账户的`CoinStore<AptosCoin>`至APT的`PrimaryFungibleStore`。
- 在执行任何 Coin 类型的`Withdraw<CoinType>`和`Deposit<CoinType>`函数时，内部触发迁移流程。这样，在用户与这些资产进行交互时，可以间接帮助实现其他 Coin 类型的迁移。

### 4. 阶段4
当绝大部分资产均已迁移至FA之后，我们将需要提出新的 AIP 来禁止新 Coin 的生成，鼓励整个生态系统基于 FA 模块 API 发展，并淘汰旧的 Coin 模块。



## 十、时间线

### 1. 建议的实施/部署时间线
预计于v1.10版本完成，预期与2024年第一季度的v1.10/v1.11版本一同发布。

### 2. 建议的开发者平台支持时间表
计划在2024年第一季度推出支持。



## 十一、安全性顾虑
代码需经过严格的安全审核。