---
aip: 79
title: 实现即时链上随机性
author: 
  - name: Alin Tomescu
    email: alin@aptoslabs.com
  - name: Zhuolun "Daniel" Xiang
    email: daniel@aptoslabs.com
  - name: Zhoujun Ma
    email: zhoujun@aptoslabs.com
discussions-to: "<指向官方讨论主题的 URL>"
status: 起草中
last-call-end-date: "<最后留下反馈和评论的日期 (mm/dd/yyyy)>"
type: 标准 (核心, 网络, 框架)
created: 2024-02-12
updated: "<更新日期 (mm/dd/yyyy)>"
requires: "<AIP 号码>"
---

[TOC]
# AIP-79 - 实现即时链上随机性

## 一、摘要

本 AIP 展示了在 Aptos 区块链上根据 AIP-41[^aip-41] 实施的随机性 API，其目标是为 Move 智能合约提供接入 (1) **立即生成的**，(2) **无偏差的** 以及 (3) **不可预知的** 随机性。这种随机性基于保障区块链自身安全的股权证明（PoS, Proof-of-Stake）的假设。实现同时需要高效，即对区块链系统的吞吐量或者延迟的影响极小。AIP 主要聚焦于链上随机性协议的描述与实施，及其与高效区块链系统的集成细节。具体地，我们介绍了在 PoS（权益证明）环境下，如何实施一种加权的分布式密钥生成（DKG, Distributed Key Generation）协议，使得在验证节点间建立阈值密钥，并且让验证节点利用它们的密钥以及一个加权的可验证不可预测函数（VUF, Verifiable Unpredictable Function）在每个区块生成随机数。我们也将描述如重构变更和 Aptos 虚拟机（VM）变动等其他主要的系统更新。 

### 1. 不在讨论范围内的内容

密码学方案的正式描述及其安全性论证不在此范围内，因为它们已在我们的论文[^DPTX24e]中描述了。

随机性 API 规范及其设计原理不在此范围内，因为它们已在AIP-41[^aip-41]中详细介绍了。

本 AIP 侧重于如何将密码学原语集成到 Aptos 区块链系统中，构建端到端的即时链上随机性系统。



## 二、动机

正如在AIP-41[^aip-41]中已经描述的那样，链上随机性的需求在许多应用程序中都存在，例如去中心化游戏、抽奖、随机NFT、随机空投等。此外，许多区块链应用程序都能从随机性中获益，以确保公平性、安全性和功能性。



## 三、影响

**验证者网络性能**。
链上随机性与由验证节点运行的区块链协议之间的互动会影响后者。因此，这两者的设计需精密配合，以确保系统的性能表现，特别是在减少延迟方面。通过在 previewnet 进行的性能测试表明，我们的实现在各种压力测试下仅对整体交易处理时间增加了微小的延迟（数十毫秒）。

**生态系统影响**。
下游应用程序，如索引器和 SDK，需要支持此实现添加的新事务类型，如[新交易类型](#2. 新交易类型)中所述。



## 四、替代解决方案

**外部信任和非PoS**

使用如 [Drand](https://drand.love/docs/overview/#how-drand-works) 这类外部信标，应用程序需要对这些信标的安全性和可用性给予外部的信任。此外，随机数无法即刻获取，而需通过一个提交-揭示的流程，这不仅让开发过程变得更复杂，还会导致获取随机数的过程出现延迟。

[DFINITY](https://internetcomputer.org/whitepaper.pdf) 依靠一种简单的非加权阈值密码系统来生成链上的随机数，而不是采用加权方式。这是由于 DFINITY 采用的是一个非权益证明（PoS）安全模型，在该模型下，只要不超过三分之一的验证节点遭到破坏，区块链就能保持安全。这种阈值设置比 Aptos 的加权机制要简单得多，因为你可以直接将份额总数定为验证节点的数量（比如说，数百个）。与之相对的是，在 Aptos 的加权机制中，份额总数与总权益呈正比关系，这意味着，即便是进行了适当取整，份额总数也可能会更大（例如，从数百增加到数千）。

**VDF** 当前基于可验证延迟函数 (VDF) 的技术，比如 [Bicorn](https://eprint.iacr.org/2023/221)，在最不利的情况下会导致较高的延迟，这对于追求高性能的区块链如 Aptos 而言并不理想。 

**交互式 VSS 和 VUF** 采用不同设计的区块链会通过基于交互式的 VSS DKG 过程，在验证节点间共享加密。遗憾的是，这种方法需要新周期的验证者在生成用于随机数产生的阈值密钥时必须在线，这不符合 Aptos 现有的系统架构设计。 

**其他不安全的方法。** 正如我们在 [博客文章](https://aptoslabs.medium.com/roll-with-move-secure-instant-randomness-on-aptos-c0e219df3fb1) 中所讨论的，一些现存的区块链采用了不安全的链上随机性机制。这些机制存在可被操纵或预测的风险，导致开发者和用户无法安全地利用链上随机数。



## 五、规范

链上随机性给当前的 Aptos 验证节点带来了多项变动，包括每个时期更换时都会执行一次 DKG，改动原有的重配置过程，并为每个区块生成一个**随机性种子**。本部分将概述介绍上述系统的改变。具体的实施细节，请参考[参考实现](#六、参考实现)章节。

### 1. Aptos 区块链背景

Aptos 是一款通过**权益证明（PoS）**机制运行的区块链，其共识算法以每两小时为一个周期运行，这个周期被称为**纪元**。在每个纪元内，验证节点及其权益分布保持不变，但会在纪元更换时发生改变。下一个纪元的验证节点只有在新纪元开始时才会开始运作。 

此外，该区块链将**共识**过程（当前采用名为 [Jolteon](https://arxiv.org/abs/2106.10362) 的 BFT 共识协议）和**执行**过程（采用名为 [BlockSTM](https://medium.com/aptoslabs/block-stm-how-we-execute-over-160k-transactions-per-second-on-the-aptos-blockchain-3b003657e4ba) 的乐观并发控制执行引擎）进行了分离。在这种机制中，每个区块会先经过共识过程最后确认，然后通过执行过程更新区块链的状态。这种共识与执行的分离设计对于链上随机性来说极为重要，因为它允许网络在计算并最终公布随机数之前，先达成对交易的排序，从而确保所生成的随机数既不可预知也无法被操控。



### 2. 设计概述

这一部分将向你宏观地介绍设计的实现，涉及的内容包括密文原始算法，加权分配式密钥生成机制以及随机数生成过程。 

Aptos 的链上随机性机制首先会将各验证节点的权益简化为更小的 *权重*，以确保在权益证明机制的安全框架内提高效率。关于具体的权益简化细节，请参阅["权益简化"](#1. 权益简化)。为了本节的讨论，我们假设验证器已经按照权益简化处理得到了相应的权重分配。



#### 2.1 工作原理

简单来说，验证节点之间建立了一个共享密钥，并利用各自持有的密钥份额来计算每个区块的随机性种子。这些种子用于在对应区块的每个 API 调用中生成伪随机数。

这个过程分为两部分：第一部分是分布式密钥生成（DKG），第二部分是随机数生成。

每个纪元结束时，会进行执行 DKG，以准备下一纪元随机数生成所需的密钥。 

由至少半数权益的验证折共同重建的这个共享密钥，可以安全用于计算该纪元内每个区块的随机性种子。

这个种子是通过对一个与区块相关的无法被操纵的消息（例如，纪元号和轮数）应用加权验证随机函数（wVRF）得出的。



#### 2.2 使用 wPVSS 的加权分布式密钥生成（DKG）

DKG 的目标是使验证者共同生成一个**共享的密钥**，该密钥将用于为每个区块计算随机性种子。
这里是为 Aptos 链上随机性设计和实现的非交互式 DKG 的描述。

- 在当前的纪元达到预设的时间限制（2小时）或需要根据治理提案进行重配置时，每个验证节点将扮演 **dealer** 角色，基于权重分布和阈值计算一份**加权公开可验证的秘密共享记录（wPVSS）**，并通过一个可靠的组播技术与所有验证节点互相交换。这份记录将包含每个验证节点处理的随机值的密钥份额的加密信息。
- 一旦每个验证者从66%的权益中收到 **有效传输** 的安全子集，它就将有效传输聚合到单个最终 **聚合传输** 中，并将其传递给共识。一旦共识最终确定了第一个有效的聚合 wPVSS 传输，验证者完成 DKG 并开始新纪元。
- 最后，当新纪元开始并且新的验证者在线时，它们可以从聚合传输中解密出自己的密钥份额。此时，新的验证者将准备好通过在秘密共享的基础上以密钥共享方式评估 VUF 来为每个区块生成随机性，如我们稍后解释的那样。

如描述的那样，Aptos 链上随机性在**每个**时期更改之前运行一个 **非交互式** DKG。
设计理念如下：

- 为什么使用加权 DKG 而不是非加权 DKG？

  如前所述，链上随机性的安全性应基于 PoS 假设。因此，DKG 需要在生成用于随机性生成的共享密钥时考虑验证者的权益。具体来说，共享的密钥只能由大多数权益重建。

- 为什么在纪元更改之前运行 DKG 而不是在之后运行？

  如果DKG在纪元更改后开始，那就太晚了：验证者在时期开始时就需要建立共享的密钥。否则，没有这样的共享密钥，验证者则无法处理任何需要随机数的交易。因此，我们在纪元更改之前运行 DKG，以便验证者可以在进入新纪元后立即生成随机数并处理交易。

- 为什么使用基于 wPVSS 的非交互式 DKG？

  下一纪元的新验证节点可能在新纪元开始之前一直处于离线状态，因此需要新节点与现有节点相互协作的交互式 DKG 在这种情况下便不实用了。相比之下，非交互式 DKG 不要求新验证节点必须在线，而且能够在纪元变更时利用区块链作为旧验证节点和新验证节点之间的广播渠道。

  

#### 2.3 使用VUFs生成随机性

在每一个纪元，验证者们将协同工作，利用分布密钥生成（DKG）过程建立的共享密钥，对 **每一个** 经过共识流程确认的区块，计算一个 **加权可验证不可预测函数（VUF）** 以生成随机性。

- 当一个区块被共识确认后，每个验证者都将利用其从加权公开可验证的秘密共享（wPVSS）汇总记录中解密得到的部分，结合区块的特定不可篡改信息（比如纪元号和轮数），计算出它对该区块的 VUF **份额**，然后通过组播技术可靠地转发出去。
- 当收集到的 VUF 份额超过了重建的权重阈值后，每个验证者将把这些份额合并起来，得到一个一致的最终 VUF **结果**。这一结果相当于在共享的密钥之下，对于区块特定的不可篡改信息进行 VUF 算法的单一计算。验证者最终将这个 VUF **结果** 作为区块的种子连接上，并将这个区块提交去执行。

重要的是，只有50%或更多的权益才能计算VUF 评估，这确保了**不可预测性**。此外，VUF的唯一性属性以及wPVSS方案的保密性确保了针对控制少于50%权益的对手的**无偏见性**。

关键在于，只有当参与者控制的权益达到 50% 或以上时，才能计算出 VUF 的结果，这保障了其**不可预测性**。此外，VUF 所固有的唯一性质和 wPVSS 方案的安全性共同确保了即使面对持有不到 50% 份额的对手，也能保证**公正无私**。

**用户交易中使用的随机性**。每个区块的随机性种子会在该区块中任何用户交易之前，以新的区块元数据交易的形式被公布在区块链上。用户的交易会将此区块级随机种子与其交易的唯一元数据结合起来，以此生成专属于该笔交易的随机性种子。



## 六、参考实现

本节详细描述了每个组件的实现，包括权益舍入、新的交易类型、重新配置相关的更改、DKG 相关的更改和随机性生成相关的更改。

### 1. 权益简化

Aptos 对链上随机性的处理是通过调整每个验证者资产的算力 **权重**，以求在实际操作中提高性能，但这种调整对于保密性和系统的稳定运行（可用性）有一定的影响。具体而言，任何调整权重的规则都可能造成**精度损失**：比如，有些参与者可能会得到比应有的更多的秘密数据片段，而有些则得到更少。这样一来，原本的阈值秘密共享方式就变成了一个所谓的 **斜坡秘密共享方式**，这意味着保密和重构的阈值发生了变化。通常情况下，重构的阈值会更高。因此，为了确保网络的顺畅运行，这项机制必须保证掌握超过 66% 权益的任何一方都能进行数据重组，同时对于掌握小于或等于33%权益的一方，保证其不能解密秘密信息。为了最大限度减少 MEV（矿工提取价值）攻击的影响，我们决定提高保密的阈值，将其设为 50% ，同时保持数据能在掌握权益低于 66% 的情况下重组，从而对抗持有 33% 权益的不利因素。

以下简要介绍了舍入接口及其保证。舍入算法的更多技术细节可以在我们的论文[^DPTX24e]中找到。

**舍入接口**

- **输入**：
  - `validator_stakes`：验证者的权益分配。
  - `secrecy_threshold_in_stake_ratio`：在验证者的任何一个子集中，如果他们控制的总权益比例不超过阈值，那么他们就无法获取秘密的随机数数据。Aptos 在其实际运用中将这个阈值设置为了50%
  - `reconstruct_threshold_in_stake_ratio`：在验证者的任何一个子集中，如果他们控制的总权益比例超过了阈值，那么他们就始终可以获取秘密的随机数数据。Aptos 在其实际运用中将这个阈值设置为了 66%
- **输出**：
  - `validator_weights`：舍入后分配给验证者的权重分布。
  - `reconstruct_threshold_in_weights`：其权重总和 ≥ 此值的验证者子集始终可以揭示秘密（随机性）。

**确保机制**：根据 `validator_weights` 中定义的权重分配，任何权重和  $>=$ ` reconstruct_threshold_in_weights` 的验证者群组，其控制的权益比例必须在 `(secrecy_threshold_in_stake_ratio, reconstruct_threshold_in_stake_ratio]` 这个区间。 

**原因解释**：由于在实际运行环境中，秘密的保持阈值 `secrecy_threshold_in_stake_ratio` 定为 `0.5`，而秘密重构阈值 `reconstruct_threshold_in_stake_ratio` 定为 `0.66`，所以能够获取秘密随机数的验证者群组，其控制的权益比例必须在 `(0.5, 0.66]` 之间。根据权益证明（Proof-of-Stake）的假设，任何敌对方最多只能掌握 `1/3` 的股份，这意味着他们无法单独获取秘密数据（因为 `1/3 < 0.5`），而诚实的验证者可以单独揭露秘密数据，因为他们掌握的股份比例 `2/3 > 0.66`。

### 2. 新交易类型

#### 2.1 新的内部交易类型：`BlockMetadataExt`

VUF 的评估结果需要作为 `BlockMetadata` 事务中的区块随机种子被记录在区块链上（该事务永远是一个区块中的第一个事务）。 

为了确保与旧版本的兼容，我们需要引入一种新的事务类型 `BlockMetadataExt`（直接更新现有的 `BlockMetadata` 事务是行不通的）。

```rust
pub enum Transaction { // 内部交易类型
    UserTransaction(SignedTransaction),
    GenesisTransaction(WriteSetPayload),
    BlockMetadata(BlockMetadata),
    StateCheckpoint(HashValue),
    ValidatorTransaction(ValidatorTransaction),
    BlockMetadataExt(BlockMetadataExt), // 新的变体
}

pub enum BlockMetadataExt {
    V0(BlockMetadata),
    V1(BlockMetadataWithRandomness),
}

pub struct BlockMetadataWithRandomness {
    pub id: HashValue,
    pub epoch: u64,
    pub round: u64,
    pub proposer: AccountAddress,
    pub previous_block_votes_bitvec: Vec<u8>,
    pub failed_proposer_indices: Vec<u32>,
    pub timestamp_usecs: u64,
    pub randomness: Option<Randomness>, // 与 `BlockMetadata`相比唯一的新字段
}
```

交易API考虑

虽然可行，但添加相应的外部交易类型可能会破坏下游生态系统，并且也不能带来多的好处。

为了解决这个问题，可以通过将 `BlockMetadataWithRandomness ` 变体导出为现有的 `BlockMetadata` 变体，并忽略 `randomness` 字段。



#### 2.2 新的验证者交易变体：`DKGResult`

需要一个新的验证者交易类型 `DKGResult` 来将 DKG 传输记录到链上。

```rust
pub enum ValidatorTransaction {
    ObservedJWKUpdate(jwks::QuorumCertifiedUpdate),
    DKGResult(DKGTranscript), // 新的变体
}

pub struct DKGTranscript {
    pub metadata: DKGTranscriptMetadata,
    pub transcript_bytes: Vec<u8>,
}

pub struct DKGTranscriptMetadata {
    pub epoch: u64,
    pub author: AccountAddress,
}
```



### 3. 重新配置相关的更改

#### 3.1 在链上配置重构

链上配置是特殊的链上资源，用于控制验证器组件的行为。例如，共识组件在新的纪元开始时重新加载 `ConsensusConfig` ；块执行器为每个块重新加载 `GasSchedule` 和 `Features`。

目前，链上配置的更新和初次设置是通过直接修改链上资源，随即执行即时重配置的方式来实施的（具体定义请见[这里](#how-to-land-new-reconfiguration-mode-async-reconfiguration)）。这么做是为了防止出现这样的情况：某个验证者重启后使用新的配置进行操作，但其他验证者还在使用旧配置。



为了同时支持即时重新配置（当随机性关闭时）和异步重新配置（当随机性开启时），需要进行以下更改：

- 对于 `config_x`，应该有一个 `config_x::set_for_next_epoch()` 治理函数来缓冲更新，以及一个 `config_x::on_new_epoch()` 函数在纪元时间被调用来应用更新。
  - 建议以一种既支持初始化又支持更新的方式实现 `set_for_next_epoch()` 和 `on_new_epoch()`。
- 除了在创世区块之外，应该禁用现有的应用并重新配置API（例如，`consensus_config::set()`）。
- `aptos_governance::reconfigure()` 函数在随机性关闭的情况下应当执行即时重配置，若随机性开启，则应触发异步重配置。

这些更改允许以下配置初始化 / 更新模式在随机性开 / 关时都能工作。

```
config_x::set_for_next_epoch(&framework_signer, new_config);
aptos_governance::reconfigure(&framework_signer);
```

一些参考实现点

- 更新的 `ConsensusConfig` [在这里](https://github.com/aptos-labs/aptos-core/blob/be0ef975cee078cd7215b3aea346b2dhttps://github.com/aptos-labs/aptos-core/blob/f1d583760848c118afe88dda329105d67eea35a2/aptos-move/framework/aptos-framework/sources/configs/consensus_config.move#L52-L69)。



#### 3.2 重新配置期间的验证者集锁定

尽管大多数链上配置可以在 DKG 过程中继续更新其缓冲区数据，但验证者集合变化的缓冲区在重配置过程中则必须锁定。 如果不这么做，验证者的投票权分配和 VUF 权重分配可能会出现不一致，导致随机性的安全性受到威胁。

应该通过以下方式进行：

- 引入一个全局的链上指示器，指示重新配置是否正在进行中；
- 在重新配置操作中更新该指示器。
- 如果有任何重新配置正在进行，则中止可能触及下一个验证者集的任何用户交易。

一些参考实现：

- 在链上指示器 [在这里](https://github.com/aptos-labs/aptos-core/blob/main/aptos-move/framework/aptos-framework/sources/reconfiguration_state.move)。
- 更新的公共框架函数 `stake::leave_validator_set()` [在这里](https://github.com/aptos-labs/aptos-core/blob/59586fee4ebb88d659f0f74afa094d728cf32b5d/aptos-move/framework/aptos-framework/sources/stake.move#L1109)。



#### 3.3 新的重新配置模式：异步重新配置

重配置是一个关键过程，旨在： 

- 增加链上的纪元（Epoch）计数器； 
- 更新验证者集（`ValidatorSet`）；
- 发出新纪元事件（`NewEpochEvent`）以激发验证者执行纪元切换相关的操作。



当前的重新配置在一次 [Move函数调用](https://github.com/aptos-labs/aptos-core/blob/main/aptos-move/framework/aptos-framework/sources/reconfiguration.move#L96) 中执行上述步骤
（因此，在此AIP中被称为**即时重新配置**）。

链上随机性设计要求以下**异步重新配置**，具有2个链上步骤和1个链下过程。

- 链上步骤一
  - 计算下一个验证者集，就好像纪元正在立即切换一样。
  - 锁定用于变更验证者集的缓冲区。 
  - 触发一个 `DKGStartEvent` 事件。 

- 在链下过程中，验证者执行 DKG。在此过程中，一个验证者会获取 DKG 的记录，然后将这个记录作为交易提案，来启动链上步骤二。
  - 这个过程是在 `DKGManager` 中完成的，具体细节可参见[这个部分](#how-to-land-dkgmanager-new-validator-component-to-execute-DKG)。

- 链上步骤二，在获得 DKG 传输记录后触发。
  - 在链上发布 DKG 传输记录。
  - 更新 `ValidatorSet`。
  - 解锁验证者集更改缓冲区。
  - 应用所有在缓冲中的链上配置更改。
  - 增加链上纪元的计数器。
  - 发出 `NewEpochEvent` 事件，以触发验证器中的纪元切换操作。

参见参考实现[此处](https://github.com/aptos-labs/aptos-core/blob/main/aptos-move/framework/aptos-framework/sources/reconfiguration_with_dkg.move)。

### 4. DKG 相关的更改

#### 4.1 新组件：`DKGManager`

- 一旦在当前纪元（Epoch）触发 `DKGStartEvent` 事件，节点需与其他节点协作运行 DKG 。  

  - 其具体操作包括为下一轮的验证者集合创建一个转录，并与其他节点交换这些信息，目的是获取一个得到超过三分之一投票权支持的综合转录。 

- 获得这个综合转录后，需将其封装成 `DKGResult` 验证者交易，并提交到验证者的交易池中提议。  
  - 要有效执行此操作，必须启用[验证者交易](https://github.com/aptos-foundation/AIPs/blob/main/aips/aip-64.md)功能。 

- 当 `NewEpochEvent` 事件发生时，将撤回任何已提出的 `DKGResult` 交易。 
- 在系统重启后故障恢复时，检查链上的 DKG 状态。如果当前纪元的 DKG 流程尚未完成，节点应继续参与进程。



实施说明：关键在于最终认可的 DKG 转录能够通过验证。即使验证者对自己的转录或者对汇总转录的本地视图有不一样的理解也没关系。基于这个特性，我们可以构建一个 `DKGManager`，而不必保存任何状态信息。

参考实现：https://github.com/aptos-labs/aptos-core/blob/df715afc2ca6646bcdee63e44e30c549ebe14bf3/dkg/src/dkg_manager/mod.rs#L51

#### 4.2 共识更改

- 在提交新区块提案时，验证者会包括 `DKGResult` 验证人交易作为区块的一部分。

  - 该操作的[示例代码实现](https://github.com/aptos-labs/aptos-core/blob/1de391c3589cf2d07cb423a73c4a2a6caa299ebf/consensus/src/liveness/proposal_generator.rs#L384-L403)可供参考。但只要启用了[验证者交易功能](https://github.com/aptos-foundation/AIPs/blob/main/aips/aip-64.md)，这一步骤就无需额外操作。

- 在对区块提案进行审核时，如果发现区块中包含了 `DKGResult` 验证者交易，但区块链的随机性特性被禁用了，验证者应该驳回该提案。    
  - 这一做法符合[AIP-64 中讨论的安全措施要求](https://github.com/aptos-foundation/AIPs/blob/main/aips/aip-64.md#security-considerations)。  
  - 关于该做法的[代码实现示例](https://github.com/aptos-labs/aptos-core/blob/1de391c3589cf2d07cb423a73c4a2a6caa299ebf/consensus/src/round_manager.rs#L687-L691)也可供参考。

### 5. 随机性生成相关的更改

#### 5.1 新组件：`RandManager`

- 纪元开始时需要执行的任务：
  - 解密链上 DKG 转录中的 VUF 份额（shares）。
  - 根据实际的 VUF 方案的需求，可能需要进行额外工作： 
    - 利用 VUF 密钥分片生成增强型密钥对。
    - 与其他节点交换增强型公钥，并为自己的增强型公钥获取确认函。  
    - 交换已获得确认的增强型公钥，并将之持久化存储以备系统崩溃时恢复使用。
- 接收经过共识机制排序后的区块流。
- 确保每个排序后的区块都包含随机种子（也就是对该区块进行的 VUF 计算）。 
  - VUF 计算结果是通过与其他节点交换 VUF 分片得到的，使用节点自身的增强型密钥对进行签名，以及用其他节点的增强型公钥进行验证。 

- 将携带随机数的区块流发送至执行管道。

参见参考实现[此处](https://github.com/aptos-labs/aptos-core/blob/1de391c3589cf2d07cb423a73c4a2a6caa299ebf/consensus/src/rand/rand_gen/rand_manager.rs#L50)。

#### 5.2 执行管道（pipeline）更改

在执行管道中，将随机性准备就绪的块处理为交易列表，其中第一个交易需要是一个 `BlockMetadataExt` 交易，用于携带块的随机性种子。

参见参考实现[此处](https://github.com/aptos-labs/aptos-core/blob/1de391c3589cf2d07cb423a73c4a2a6caa299ebf/consensus/src/state_computer.rs#L203)。

#### 5.3 Move VM 的更改

Move VM 需要能够执行新的交易变体 `BlockMetadataExt` 和 `ValidatorTransaction::DKGResult`，这是异步重新配置的 2 个步骤。

`BlockMetadataExt `事务应该执行与 `BlockMetadata` 事务相同的操作，以及以下差异。

- 在当前纪元超时的时候，触发异步重新配置而不是即时重新配置。
  请记住，触发异步重新配置意味着：
  - 完成下一个验证者集，
  - 锁定验证者集资源，
  - 发出一个包含下一个验证者集的 `DKGStartEvent`。
- 如果可用，则将块随机性种子写入链上。

与`BlockMetadata`事务一样，`BlockMetadataExt`事务不应中止。
参见参考实现[此处](https://github.com/aptos-labs/aptos-core/blob/1de391c3589cf2d07cb423a73c4a2a6caa299ebf/aptos-move/aptos-vm/src/aptos_vm.rs#L1935)

`ValidatorTransaction::DKGResult` 交易应执行以下步骤： 

- (a) 检验包含的聚合型 DKG 传输记录。

- 释放验证者集合所占用的相关资源。

- 实施所有待处理的链上配置更改。

- 将 DKG 证据发布到区块链上。

- 若检测到纪元超时，执行 `BlockMetadata` 交易中的相应操作。

 如果遭到恶意验证者的干扰导致步骤（a）无法通过，那么交易应被丢弃。 

其余步骤绝不应该中断。 相关的[代码实现可以在这里查看](https://github.com/aptos-labs/aptos-core/blob/1de391c3589cf2d07cb423a73c4a2a6caa299ebf/aptos-move/aptos-vm/src/validator_txns/dkg.rs#L70)。





### 6. VM 中的攻击预防

应从 VM 端防止一些已知的针对随机性事务的攻击。

#### 6.1 测试中止攻击
在[test-and-abort攻击](https://github.com/aptos-foundation/AIPs/blob/main/aips/aip-41.md#test-and-abort-attacks)中，
一个 dApp 为用户定义了一个公共函数，用于：

1. 抛硬币，
2. 如果硬币 $=$ 1，则获得奖励，否则获得惩罚。
恶意用户可以编写另一个合约来调用此公共函数，如果没有奖励，则中止，因此它永远不会收到惩罚。

为了防止此攻击，随机性 API 调用处理程序应确保调用 **私有入口函数** 的源。
（参考实现点
[实现1](https://github.com/aptos-labs/aptos-core/blob/e5d6d257eefdf9530ce6eb5129e2f6cbbbea8b88/aptos-move/aptos-vm/src/aptos_vm.rs#L806-L811)
[实现2](https://github.com/aptos-labs/aptos-core/blob/e5d6d257eefdf9530ce6eb5129e2f6cbbbea8b88/aptos-move/framework/aptos-framework/sources/randomness.move#L77)）

#### 6.2 低 Gas 攻击
在[低 Gas 攻击](https://github.com/aptos-foundation/AIPs/blob/main/aips/aip-41.md#undergasing-attacks)中，
一个 dApp 为用户定义了一个私有入口函数，用于：

1. 抛硬币（ Gas 费：9），

2. 如果硬币 $=$​ 1，则获得奖励（ Gas 费：10），否则获得多个惩罚（ Gas 费：100）。
    恶意用户可以控制其账户余额，使其最多覆盖 788 个 Gas 单位（或以`max_gas=108`运行交易），然后调用此函数，
    因此它永远不会收到惩罚，因为发布路径会中止。

以下是防止此攻击所做的工作。
- 假设0：一个消耗随机性的事务在任何执行路径中不会使用超过 `X` 个燃气单位。

- 仅当被调用的入口函数具有注释 `#[randomness]` 时，用户交易才被视为 **随机性事务**。

- 在执行*随机性事务*期间，从 Gas 支付方的余额中锁定价值为 `X` 个燃气单位的金额。
  余额不足会导致交易被丢弃。
  
  - 参考实现点
    [实现1](https://github.com/aptos-labs/aptos-core/blob/5a963facfb58ef74611e6fcd6225be4cb2ac7ac2/aptos-move/framework/aptos-framework/sources/transaction_validation.move#L194)
    [实现2](https://github.com/aptos-labs/aptos-core/blob/5a963facfb58ef74611e6fcd6225be4cb2ac7ac2/aptos-move/framework/aptos-framework/sources/transaction_validation.move#L344)
  
- **随机性交易** 需要设置`max_gas=X`。否则，交易将被丢弃。
  
  - 如果允许 `max_gas < X`，那么恶意用户就可以通过控制 `max_gas` 值来强行终止那些需要较多 gas 的执行路径。
  - 如果允许 `max_gas > X`，那么违背了假设 0 的 dApps 将面临风险：恶意用户能够通过操纵账户余额来阻断那些消耗较多 gas 的执行路径。
  - 这一情况破坏了那些在设计上就违背了假设 0 的 dApps：那些消耗超过 `X` 单位 gas 的执行路径会被系统强制中断。
  - [参考实现](https://github.com/aptos-labs/aptos-core/blob/5a963facfb58ef74611e6fcd6225be4cb2ac7ac2/aptos-move/aptos-vm/src/aptos_vm.rs#L2432-L2437)。
  
- 仅允许在 **随机性事务** 中进行随机性 API 调用。
  - [参考实现](https://github.com/aptos-labs/aptos-core/blob/5a963facfb58ef74611e6fcd6225be4cb2ac7ac2/aptos-move/aptos-vm/src/aptos_vm.rs#L806-L810)。
  
- 在参考实现中，`X` 是一个链上配置，设置为10000。
  
  0.01 APTs 在 100 octas / Gas 单位的价格之下。
  
  - 如果`X`太高，使用随机性将会太昂贵。
  - 如果`X`太低，允许单个随机性事务执行的操作太少。
  - [参考实现](https://github.com/aptos-labs/aptos-core/blob/5a963facfb58ef74611e6fcd6225be4cb2ac7ac2/types/src/on_chain_config/randomness_api_v0_config.rs#L14-L18)。



## 七、测试（可选）

单元测试、冒烟测试、锻造测试。

部署一个名为 `randomnet` 的测试网络，该网络持续几个月。

在 `previewnet` 中进行更多的压力 / 性能测试。



## 八、风险和缺陷

### 1. 灾难情景

**DKG 失去活性**。在出现实现错误或糟糕的配置时，DKG 可能永远无法完成，因此纪元更改不会完成，并且缓冲的链上配置更新不会被应用。请注意，在这种情况下链仍然处于活动状态，但验证者无法进入新的纪元。

*缓解措施*：验证者可以强制进行纪元（epoch）变更并禁用随机性功能。在调试之后，需要部署一个热修复程序以实现完全缓解。



**随机性生成失去活性**。如果发生这种情况，区块链将停止，因为每个区块都需要随机性才能继续执行。

*缓解措施*：作为临时的缓解措施，验证者可以通过使用本地配置去覆盖，以禁用随机性功能，并重新启动所有验证者，以暂停随机数生成，可以使链重新活跃。在调试之后，需要部署一个热修复程序以实现完全缓解。



### 2. 其他风险

**MEV 攻击**。当恶意验证者控制超过50%（保密阈值）的总权益时，存在恶意验证者共谋并泄露 VUF 秘钥的风险。如果发生这种情况，共谋的验证者可以预测当前时期的所有随机性。



### 3. 缺陷

**性能影响**。随机性生成阶段会在区块提交延迟中增加小的延迟开销，这是由于加密操作和通信延迟造成的。

**重配置影响**。DKG 阶段将影响重新配置过程。

*链上配置的变更*。对于每两小时就会进行一次的周期性重新配置，所有的验证者必须在下一个纪元周期开始之前完成 DKG。对于那些需要进行重新配置的治理提案，它们引发的变更将被暂存在链上，并在 DKG 完成以及验证者进入新的纪元周期后才被生效。任何以后新增的链上配置都需要遵循同样的规程。 

*验证者集的变更*。根据现行的设计，在 DKG 阶段，所有针对 `ValidatorSet` 的变更都将不被接受。也就是说，根据最新的主网模拟，每两小时的范围中，会有最长 30 秒的时间，试图变更验证者集的交易都会失败。

更多细节可以在规格部分找到。

**关于验证者集大小的约束**。如果验证者集太大，那么在最坏的情况下：当各方的权益分配具有潜在的对立性时，用于舍入计算的算法可能会计算出具有较大总权重的 DKG 参数，这会使得 DKG 的数据量超过了交易被规定的限额。如果出现这种情况，DKG 的正常运作可能会受到影响。 

这里有一些有关交易限额的说明。为了预先防范这种情况的发生，我们将会严密监测 DKG 的交易大小，并设置合适的预警机制。一旦 DKG 交易大小接近上限，我们可能需要考虑放宽以下这些交易的限制。

- `max_bytes_per_write_op`：单个状态项的大小限制，当前为 1MB。
- `ValidatorTxnConfig::V1::per_block_limit_total_bytes`：一个区块中总验证者事务大小的限制，当前为 2MB。

**最低燃气存款**。用户事务如果需要随机性，则需要设置`max_gas=10000`，因为[当前实现了防止低估攻击](#undergasing-attack)。SDK 需要更新以支持此功能。在未来，应支持每个 dApp 的定制存款金额，而不是一个通用常量。

**最低 gas 费**。如果用户的交易需要随机性，那么需要设置 `max_gas=10000`，原因是[我们已经实现了防止低 gas 攻击](#undergasing-attack)。为了支持这个功能，SDK/wallet 也许需要进行相应的更新。未来，我们应该允许每个 dApp 自行设定它的最低费用金额，而不是统一使用一个固定值。



## 九、未来潜力

在本 AIP 中实现的框架为 Aptos 区块链上的阈值密码方案的未来发展提供了机会。例如，一旦相应的高效密码构建模块可用，就可以支持本地阈值加密方案。

## 十、时间表

目标发布 v1.12。

## 十一、安全性考虑

链上随机性的安全假设与区块链安全假设相同，即权益证明。

此实施已经减轻了智能合约层面的几种潜在攻击。详细信息可以在[AIP-41](https://github.com/aptos-foundation/AIPs/blob/main/aips/aip-41.md#security-considerations)中找到。

## 十二、参考资料

[^aip-41]: https://github.com/aptos-foundation/AIPs/blob/main/aips/aip-41.md

[^DPTX24e]: **使用加权 VUF 进行分布式随机性**，作者：Sourav Das、Benny Pinkas、Alin Tomescu 和 Zhuolun Xiang，2024年，[[URL]](https://eprint.iacr.org/2024/198)