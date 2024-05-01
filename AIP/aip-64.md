---
aip: 64
title: Validator Transaction Type
author: zhoujun@aptoslabs.com, daniel@aptoslabs.com
discussions-to (*optional): https://github.com/aptos-foundation/AIPs/issues/327
Status: Accepted
last-call-end-date (*optional): <mm/dd/yyyy the last date to leave feedbacks and reviews>
type: Standard (Core/Framework)
created: <10/17/2023>
updated (*optional): <mm/dd/yyyy>
requires (*optional): <AIP number(s)>
---

[TOC]

原文链接：https://github.com/aptos-foundation/AIPs/blob/main/aips/aip-64.md


# AIP-64 - Validator Transaction Type 验证者交易类型

## 一、概述

Aptos 区块链目前主要处理两种类型的交易：`UserTransaction`和特定的 validator 交易（`BlockMetadata`和`StateCheckpoint`）。考虑到未来发展的需要，我们需要整合更多类型的 validator 交易以支持新的应用场景。为此，本 AIP 建议引入一个名为＇ValidatorTransaction＇的新交易类型。这个类型将作为枚举（enum）设计，以便在未来可以轻松地进行扩展，加入更多的 validator 交易类型。

### 1. 目标

本 AIP 有两个主要目标：
- 引入一种新的交易类型 `ValidatorTransaction`。
- 执行所有必需的修改，以确保新引入的 ValidatorTransaction 与现有系统兼容。

### 2. 在范围之外

关于未来可能扩展的 validator 交易类型的具体细节不在本文档讨论范围之内。这些细节由未来的开发者根据需要提出并处理。

## 二、动机

未来的应用需要区块链能够高效地对链上特定提案达成共识并更新。Validator 交易能够使验证器快速提议更改，并在几秒钟内达成共识，这得益于 Aptos 区块链的低延迟特性。如果没有 Validator 交易，要达成这样的共识只能依赖于 [Aptos Governance](https：//aptos.dev/concepts/governance/)，这将是一个更长周期的流程，不仅需要几天甚至几周的时间，而且每次都需手动提交提案。

Validator 交易框架的一个典型应用场景是，验证器可以以最小的延迟更新链上另一个特性的配置。如下 Aptos 特性已经使用了 Validator 交易：
- 在[链上随机性](https://github.com/aptos-foundation/AIPs/pull/321/files)的设计中，用于 epoch `e`的验证器随机生成的密钥份额是链上配置的一部分。这些密钥份额在 epoch `e-1`结束时链下生成，随后通过 ValidatorTransaction 上传到链上。

- 在[无密钥账户](https://github.com/aptos-foundation/AIPs/blob/main/aips/aip-61.md)设计中，每个 OIDC 提供商的最新 JWKs 是特性配置的一部分，需要尽快在链上进行更新。通过子特性[JWK 共识](https：//github.com/aptos-foundation/AIPs/blob/main/aips/aip-67.md)，`ValidatorTransaction` 框架被用来发布已获得必要认证的链上 JWK 更新。

## 三、影响

如果下游消费者（如索引器、浏览器、SDKs 等）无法正确处理未知的交易类型，新的交易类型可能会对它们产生破坏性的影响。

## 四、可替代方案


我们也有可能通过[Aptos Governance](https://aptos.dev/concepts/governance/)进行反复执行来实现类似的功能，但这个过程可能需要几天甚至几周的时间。然而，对于许多需要效率的应用情境而言，这并不现实。


为了发布链上配置更新，还存在一些其他的替代方案。其中一种是在已有的`BlockMetadata transactions`中发布更新。然而，尽管在一开始这省去了添加新交易类型的麻烦，但随着越来越多的功能采用这种方法，会出现一些长期问题：
- `BlockMetadata`交易是关键的交易类型，它需要在执行时永不中断。在`BlockMetadata`交易中添加特性相关的更新需要对这些更新进行强验证，这本质上是无必要的。
- 如果在单个`BlockMetadata`交易中顺序执行所有更新可能会导致处理速度过慢。正如在[安全性考虑](#security_considerations)中所提到的那样，每次更新都需要进行加密验证，这可能进一步消耗系统。将更新分解到独立的交易中可以帮助加快执行速度。

第二种替代方案是将更新作为用户交易提交。但需注意一点：
- 这些更新是针对系统配置的并通常需要比用户交易更高的优先级。
- 与用户交易不同，这些更新的处理过程中并不涉及 gas 费。
- 这些更新的执行结果需要采取与用户交易不一样的处理方法(详情请查看[安全性考虑](#security_considerations))。

因此，虽然第二种替代方案也能避免添加新的交易类型，但它无疑会增加现有用户交易流程的复杂性。为更新创建一个独立的处理流程应能确保代码更好的可维护性。

## 五、规范

### 1. 节点交易 API 的变动

节点交易 API 应支持`ValidatorTransaction`作为新的交易变体。

```rust
/// 在aptos_api_types::transaction中
pub enum Transaction {
    PendingTransaction(PendingTransaction),
    UserTransaction(Box<UserTransaction>),
    GenesisTransaction(GenesisTransaction),
    BlockMetadataTransaction(BlockMetadataTransaction),
    StateCheckpointTransaction(StateCheckpointTransaction),
    ValidatorTransaction(ValidatorTransaction), // 新增！
}
```

应该公开`ValidatorTransaction`的以下属性。
```rust
/// 在aptos_api_types::transaction中
pub struct ValidatorTransaction {
    pub info: TransactionInfo,
    pub events: Vec<Event>,
    pub timestamp: U64,
}
```

节点交易 API `/v1/transactions/by_version/<version>` 返回的 JSON 示例：
```json
{
  "version": "19403939",
  "hash": "0x8853505309d4f5b97a6ac3004f2410df055cdd507c99fce3e0318f7deeef43ff",
  "state_change_hash": "0x4a70ca32dfb51227fca6c7d430f49cc9e822e96b1ffcff8b4d8fd2ae3ae53ecb",
  "event_root_hash": "0xc49747cf5fa2dcb6351a7039a6604b63d73d6f4de3e2e14bee88f8ff5f346229",
  "state_checkpoint_hash": "0x9e67939845db211fb5596733391bbdeda0dcbf4d2bf7fe72496cab6eef23daff",
  "gas_used": "0",
  "success": true,
  "vm_status": "Executed successfully",
  "accumulator_root_hash": "0x50d548f5f7a73f3e2d1bc62a91abca9fead3bcec3dc590793521f21f2092d143",
  "changes": [
    {
      "address": "0x1",
      "state_key_hash": "0x2287692c179002981a8fe19f6def197ed5ff452848659e2e091ead318d719050",
      "data": {
        "type": "0x1::reconfiguration::Configuration",
        "data": {
          "epoch": "5098",
          "events": {
            "counter": "5098",
            "guid": {
              "id": {
                "addr": "0x1",
                "creation_num": "2"
              }
            }
          },
          "last_reconfiguration_time": "1709688193025121"
        }
      },
      "type": "write_resource"
    },
  ],
  "events": [
    {
      "guid": {
        "creation_number": "2",
        "account_address": "0x1"
      },
      "sequence_number": "5097",
      "type": "0x1::reconfiguration::NewEpochEvent",
      "data": {
        "epoch": "5098"
      }
    }
  ],
  "timestamp": "1709688193025121",
  "type": "validator_transaction"
}
```

请注意，当前 API 规范对于执行`ValidatorTransaction`的具体上层逻辑是不透明的。选择这样做的原因如下：
- 这允许我们在添加新特性（这些特性需要使用`ValidatorTransaction`）的时候，避免每次都更改交易 API 规范。
- 目前，并不确定是否有需求要求透明化那些触发`ValidatorTransaction`的上层特性。
  （鉴于`ValidatorTransaction`不是向公众开放的，保持它们不被公开某种程度上是合理的，但目前仍未将其设为标准。）
- 实际上，`changes`和`events`字段已经足够用来识别发起交易的上层特性：
  通常每个特性都有一些特定的事件产生或特定的资源变动。

### 2. 验证器网络/存储的内部数据格式变更

用于验证器网络/存储的内部`Transaction`枚举同样需要更新，以便支持新的交易类型。
```rust
/// 在 aptos_types::transaction 中
pub enum Transaction {
    UserTransaction(SignedTransaction),
    GenesisTransaction(WriteSetPayload),
    BlockMetadata(BlockMetadata),
    StateCheckpoint(HashValue),
    ValidatorTransaction(ValidatorTransaction), // New!
}
```

更新内部数据格式比更新 API 规范要简单得多，因此在这里，`ValidatorTransaction`明确指出了其上层特性。
```rust
/// 在 aptos_types::validator_txn 中
pub enum ValidatorTransaction {
    DKGResult(DKGTranscript),
    ObservedJWKUpdate(jwks::QuorumCertifiedUpdate),
    // 未来的特性...
}
```

为了让 Validator 在 Joteon 共识中提出 validator transactions，同样需要一个适当的`Proposal`扩展（即`ProposalExt`）。
```rust
/// In aptos_consensus_types::proposal_ext
pub enum ProposalExt {
    V0 {
        validator_txns: Vec<ValidatorTransaction>, // the old `Proposal` doesn't have this
        payload: Payload,
        author: Author,
        failed_authors: Vec<(Round, Author)>,
    },
}
```

### 3. Validator 中的高级数据流

在系统的高层次中，参与过程如下：
- 上层特性作为 validator 交易的生产者。
- 共识机制作为 validator 交易的消费者。
- Validator 交易池为生产者和消费者之间提供了解耦。
  - 为确保处理的公平性，交易会在池中排队。

以下是关键的互动步骤：
- 生产者将 validator 交易发送到池中。
- 当共识机制准备好建议时，它会从交易池中拉取 validator 交易。
  - 拉取交易的数量和大小需有共识机制上的限制。

### 4. 执行细节

每一个特性都需要定义 validator 交易的职责及其在提议阶段和执行阶段的验证方法。

安全实践的详细定义应该在[安全考量](#security-considerations)部分进行讨论。

## 六、参考实现


- [https://github.com/aptos-labs/aptos-core/pull/10963](https://github.com/aptos-labs/aptos-core/pull/10963)
- [https://github.com/aptos-labs/aptos-core/pull/10971](https://github.com/aptos-labs/aptos-core/pull/10971)

## 七、测试（可选）

鉴于此 AIP 主要关注格式变化，而不是新增 validator 交易类型，现有测试应注重保持兼容性。但是，对于未来可能新增的 validator 交易类型，应彻底进行测试以维护系统的安全和功能完整性。

## 八、风险和缺点

引入 Validator 交易类型的可能性提高了恶意的区块链核心开发者通过提出有害的 Validator 交易扩展来破坏区块链系统的可能性。因此，任何未来添加新的 Validator 交易类型都必须伴随着单独的 AIP，并经过彻底和仔细的审查，以减轻此类风险。

## 九、未来潜力

这个 AIP 有可能使需要区块链有效且定期地就提案达成共识的各种用例成为可能。

## 十、时间线

### 1. 建议的部署时间线

这个 AIP 的建议时间线是 1.10 版本。

## 十一、安全性考虑

没有什么能阻止恶意的 Validator 提出无效的 Validator 交易。因此，需要安全地实施 Validator 交易框架以减轻负面影响。
- 一个区块中的 Validator 交易的数量和总大小应该受到限制。

  （[参考实现](https://github.com/aptos-labs/aptos-core/blob/d4fdb8f08929903044673d03e79c9f118a6c714a/consensus/src/payload_client/mixed.rs#L82-L96)）
- 如果特性 X 被禁用，那么在区块提案中应该不期望有特性 X 的 Validator 交易。
  （[示例实现](https://github.com/aptos-labs/aptos-core/blob/7b2b2332f1f865b1ec367601045b2e0cd836a15d/consensus/src/round_manager.rs#L665-L673)）
- 任何 Validator 交易中嵌入的更新应该能够获得群体认证。
    - 执行任何 Validator 交易应该涉及到群体认证的验证。
      （[示例实现](https://github.com/aptos-labs/aptos-core/blob/d4fdb8f08929903044673d03e79c9f118a6c714a/aptos-move/aptos-vm/src/validator_txns/jwk.rs#L119-L127)）
    - 验证失败应该导致 Validator 交易被丢弃。
      （[示例实现](https://github.com/aptos-labs/aptos-core/blob/d4fdb8f08929903044673d03e79c9f118a6c714a/aptos-move/aptos-vm/src/validator_txns/jwk.rs#L68-L75)）

