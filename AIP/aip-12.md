---
aip: 12
title: 多签账户
author: movekevin
discussions-to: https://github.com/aptos-foundation/AIPs/issues/50
Status: Accepted
last-call-end-date:
type: Standard (Framework)
created: 2023/01/24
updated: 2023/01/26 
---

[TOC]

# AIP - 12 - 多签账户

## 一、概述

这份 AIP 推出了一个全新的多签名账户标准。这个标准主要通过在智能合约（multisig_account）中定义明确的数据结构和功能实现管理，与目前采用多重 Ed25519 授权密钥（multied25519-auth-key）的账户相比，使用起来更为便捷，功能也更加全面。此外，它还强调了整合为 Aptos 中更广义的账户系统一部分的发展方向，为用户提供更多样化的账户类型和账户管理功能。 

这不是为了直接提供一个功能齐全的多签名钱包产品，而是作为一个基础架构和可能提供的 SDK 支持，目的是让社区有能力开发出更高级的多签解决方案。



## 二、动机

多签账户在加密货币中很重要，应用如下：

- 作为 DAO 或开发者组的一部分，用于升级、操作和管理智能合约。
- 管理链上资金。
- 保护个人资产，以防一个密钥丢失导致的资金损失。

目前，Aptos 支持 multied25519 认证密钥，允许多签交易：

- 这与多代理交易（ multi-agent transactions ）形成鲜明对比，在多签的情况下，多个签名者各自对交易进行签名，导致在交易被执行的过程中需要验证多个签名者的签名。
- 可以通过调用 `create_account` 并传入正确的地址来创建多签账户，该地址是所有者公钥列表的哈希值，阈值k（k-of-n multisig）和 multied25519 方案标识符（1）串联而成。多签强制执行将通过多签账户的认证密钥完成。
- 要创建多签交易，首先需传递交易数据负载（ the tx payload needs to passed around），并且多签账户的 k 个私钥需要用正确的[身份验证设置](https://aptos.dev/guides/creating-a-signed-transaction/#multisignature-transactions) (https://aptos.dev/guides/creating-a-signed-transaction/#multisignature-transactions)。
- 要添加或删除所有者或更改阈值（ threshold），所有者需要发送具有足够签名的交易，以更改认证密钥以反映新的所有者公钥列表和新的阈值。

当前的多签名机制存在几个问题，这些问题使得使用起来比较困难：

- 很难确定多签账户的当前所有者是谁，以及所需的签名阈值是多少。这些信息需要从认证密钥中手动解析。
- 要创建多签账户的认证密钥，用户需要串联所有者的公钥，并在末尾添加签名阈值。大多数人甚至不知道如何获取自己的公钥，也不知道它们与地址不同。
- 用户必须手动传递交易的数据负载（ tx payload ）以收集足够的签名。即使SDK简化了签名部分，存储和传递此交易仍然需要某个地方的数据库和一些协调工作以在有足够的签名时执行。
- 在多签名交易中的 nonce 必须是是多签名账户自己的 nonce，而不能用账户拥有者各自的 nonce。通常，如果在此多签名交易之前有其他交易被执行，导致 nonce 发生变化，那么这个多签名交易就可能被认定无效。对于多个同时进行的交易来说，管理各自的 nonce 是一项相当复杂的任务。
- 添加或删除所有者不容易，因为涉及更改认证密钥。这种交易的数据负载（ payload）不容易理解，并且需要一些特殊逻辑来进行解析/差异化。

## 三、提议

我们可以创建一个对用户更加友好的多签账户标准，技术生态系统能在这个新标准的基础上扩展自己的功能。这包括两个主要组成部分：	

1. <mark>一个多签名账户模块主要负责以下几项内容：创建和管理多签名账户，以及生成、审批、拒绝和执行多签名账户的交易。默认情况下，执行功能（function）是私有的（private），仅允许特定人员操作：</mark>

   > A multisig account module that governs creating/managing multisig accounts and creating/approving/rejecting/executing multisig account transactions. Execution function will be private by default and only executed by:

2. 一种新的交易类型，它允许执行人（必需是所有者之一）代替多签名账户执行交易数据负载（transaction payload）。这种执行是通过调用多签名账户模块的私有执行函数来进行验证的。此类交易也可以扩展应用到其他模拟 / 委托场景（impersonation / delegation），比如支付 Gas 费来执行另一个账户的交易。

### 1. 数据结构和多签账户模块

- 一个多签账户（multisig_account）模块，允许更轻松地创建和操作多签账户
    - 多签账户将作为一个独立的资源账户创建，并且它拥有自己的地址
    - 多签账户将存储多签配置（所有者列表，阈值）和要执行的交易列表。交易必须按顺序执行（或拒绝），这增加了确定性。
    - 此模块还使得所有者能够通过标准的用户交易过程创建并审批或驳回多签交易（这些操作会成为标准的公共接口）。只有在执行这些多签账户的交易时，才需使用到新的交易类型。

```rust
struct MultisigAccount has key {
  // 所有者地址列表。
  owners: vector<address>,
  // 要通过交易所需的签名数量（k中的k-of-n）。
  signatures_required: u64,
  //  这是一个映射，关联了每个递增编号的交易ID和该多签名账户需要执行的交易。
  // 为了节省存储空间，已经执行的交易会被删除，但这些交易信息仍然可以通过相关事件来查询。
  transactions: Table<u64, MultisigTransaction>,
  // 上次执行或拒绝的交易id。用于强制执行提案的有序性。
  last_transaction_id: u64,
  // 要分配给下一个交易的交易id。
  next_transaction_id: u64,
  // 这是用于控制多签名账户（资源账户）的签名权限。它可以与签名者互换。
  // 当前尚未启用，因为 MultisigTransaction 能够在虚拟机（VM）中直接对签名者进行验证和创建，
  // 但未来在链上进行组合操作时，这一机制可发挥重要作用。
  signer_cap: Option<SignerCapability>,
}

/// 要在多签账户中执行的交易。
/// 这必须包含完整的交易数据负载或其哈希值（存储为字节）。
struct MultisigTransaction has copy, drop, store {
  payload: Option<vector<u8>>,
  payload_hash: Option<vector<u8>>,
  // 已批准的所有者。使用简单映射去重。
  approvals: SimpleMap<address, bool>,
  // 已拒绝的所有者。使用简单映射去重。
  rejections: SimpleMap<address, bool>,
  // 创建此交易的所有者。
  creator: address,
  // 关于交易的元数据，如描述等。
  // 这也可以在未来重复使用，以添加多签交易的新属性，例如到期时间。
  metadata: SimpleMap<String, vector<u8>>,
}
```


### 2. 新的交易类型以执行多签账户交易

```rust
// 当前采用的结构体，用于定义入口函数的数据载荷。
#[derive(Clone, Debug, Hash, Eq, PartialEq, Serialize, Deserialize)]
pub struct EntryFunction {
    pub module: ModuleId,
    pub function: Identifier,
    pub ty_args: Vec<TypeTag>,
    #[serde(with = "vec_bytes")]
    pub args: Vec<Vec<u8>>,
}
// 当前用于定义入口函数数据载荷的结构体，比如用于实现“coin::transfer”这类函数的调用。
#[derive(Clone, Debug, Hash, Eq, PartialEq, Serialize, Deserialize)]
pub struct EntryFunction {
    pub module: ModuleId,
    pub function: Identifier,
    pub ty_args: Vec<TypeTag>,
    #[serde(with = "vec_bytes")]
    pub args: Vec<Vec<u8>>,
}

// 此处我们采用了枚举类型，目的是为了未来的功能扩展，比如说将来我们可能支持脚本类型的负载。
pub enum MultisigTransactionPayload {
    EntryFunction(EntryFunction),
}

/// 允许多签账户的所有者以多签账户身份执行预先批准的交易的多签交易。
#[derive(Clone, Debug, Hash, Eq, PartialEq, Serialize, Deserialize)]
pub struct Multitsig {
    pub multisig_address: AccountAddress,

    // 如果已经在链上存储，则交易数据载荷是可选的。
    pub transaction_payload: Option<MultisigTransactionPayload>,
}
```

### 3. 端到端流程

1. 拥有者可以通过调用 `multisig_account::create` 来创建一个新的多签账户。
    1. 这可以作为普通用户的交易（入口函数）完成，或者在链上通过另一个在其基础上构建的模块来实现。
2. 拥有者可以随时通过调用 `multisig_account::add_owners` 或  `multisig_account::remove_owners` 来添加 / 移除拥有者。进行此类交易仍需遵循多签账户指定的 `k-of-n` 方案。
3. 要创建一个新的交易，拥有者可以调用 `multisig_account::create_transaction` 并提供交易数据载荷：指定的模块（地址 + 名称）、要调用的函数名称和参数值。
    1. 负载数据结构仍在实验阶段。我们希望使链下系统（off-chain systems）能够正确构建此负载（或负载哈希），并且在出现问题时能够进行调试。
    2. 这将在链上（ on chain）存储完整的交易负载，这增加了去中心化（无法进行审查），并且使得获取所有等待执行的交易变得更容易。
    3. 如果需要进行 Gas 优化，拥有者可以选择调用 `multisig_account::create_transaction_with_hash`，其中仅存储负载哈希（模块 + 函数 + 参数）。以后的执行将使用哈希进行验证。
    4. 只有拥有者可以创建交易，并且将分配交易ID（递增ID）。
4. 交易必须按顺序执行。但是，拥有者可以提前创建多个交易并对其进行批准/拒绝。
5. 要批准或拒绝交易，其他拥有者可以调用 `multisig_account::approve()` 或 `reject()` 并提供交易ID。
6. 如果有足够的拒绝（≥ 签名门槛），任何拥有者都可以通过调用 `multisig_account::remove()` 来移除交易。
7. 如果有足够的批准（≥ 签名门槛），任何拥有者都可以使用特殊的 MultisigTransaction 类型创建下一个交易，如果仅在链上存储了哈希，则交易负载是可选的。如果在创建时存储了完整的负载，则多重签名交易不需要指定除多签账户地址之外的任何参数。VM中的详细流程如下：
    1. 交易序言（prologue）：VM首先调用一个私有函数（`multisig_account::validate_multisig_transaction`）来验证提供的多重签名账户中待执行的下一个交易是否存在，并且是否有足够的批准（approvals）来进行执行。
    2. 交易执行：
        1. VM 首先获取多签交易中底层调用的数据负载。如果交易序言（验证）成功，这个步骤不应该失败。
        2. 然后 VM 尝试执行此函数并记录结果。
        3. 如果成功，VM调用 `multisig_account::successful_transaction_execution_cleanup` 来跟踪和发出成功执行的事件。
        4. 如果失败，VM 抛出（throws）执行负载的结果（通过重置VM会话），同时保留到目前为止已花费的 Gas 。然后它调用 `multisig_account::failed_transaction_execution_cleanup` 来跟踪失败。
        5. 最后，从发送者账户中扣除 Gas，并且所有待定的Move模块发布事宜也将得到处理（例如，如果多签交易涉及到模块的发布）。

### 4. 参考实现

[https://github.com/aptos-labs/aptos-core/pull/5894](https://github.com/aptos-labs/aptos-core/pull/5894)

## 四、风险和缺陷

主要风险是智能合约风险，即智能合约代码（multisig_account 模块）或 API 和 VM 执行中可能存在错误或漏洞。可以通过彻底的安全审计和测试来减轻此风险。

## 五、未来潜力

此提案的一个即时扩展是为多重签名交易添加脚本支持。这将允许定义更复杂的原子多重签名交易。

长期来看：

目前的提案不允许在链上执行多重签名交易——其他模块只能创建交易并允许拥有者批准/拒绝。执行将需要发送一个专门的多重签名交易类型。然而，在未来，随着 Move 中动态调度支持（dynamic dispatch support ）的引入，使其变得更简单。这将允许在链上执行，也可以通过标准用户交易类型（而不是特殊的多重签名交易类型）在链下执行。动态调度还允许向多重签名账户模型添加更多模块化组件，以实现自定义交易认证等。

多重签名账户还可以实现另一种更通用的账户抽象模型，其中链上认证可以定制，以允许账户 A 根据账户 B 定义的模块/功能执行交易。这将使得更强大的链下系统（如游戏）能够抽象化交易和认证流程，而无需用户对它们的工作原理有深入的了解。

## 六、建议的实施时间表

目标完成代码（包括安全审计）和测试网发布：2023年2月。