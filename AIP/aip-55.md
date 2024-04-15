---
aip: 55
title: 泛化交易认证并支持任意 K-of-N 多密钥账户
author: davidiw, hariria
discussions-to: https://github.com/aptos-foundation/AIPs/issues/267
Status: Draft
last-call-end-date (*optional): <mm/dd/yyyy> 留下反馈和审阅的最后日期
type: 平台
created: 10/16/2023
updated (*optional): <mm/dd/yyyy>
requires (*optional): N/A
---

[TOC]

# AIP-55 - 泛化交易认证并支持任意 K-of-N 多密钥账户

## 一、概述

提交给 Aptos 的交易包含一个 `RawTransaction` 和一个 `TransactionAuthenticator`。`TransactionAuthenticator` 授权执行交易的是交易内的账户的发送者（senders）或审批者（approvers）集合（set）。`TransactionAuthenticator` 包含一组通用的认证器，称为 `AccountAuthenticators`，用于费用支付和多代理，以及一些非常特定的类型，用于单个发送者交易，如 Ed25519、MultiEd25519 和 Secp256k1。因此，添加新的用于授权交易的加密证明需要一个新的 `TransactionAuthenticator`、`AccountAuthenticator` 和专门的加密认证器。

除此之外，Aptos 仅支持单一的多密钥方案，即 ed25519。当用户可以针对不同目的利用不同的证明类型时，多密钥方案提供了价值，例如，将 Ed25519 用于其钱包，而从 HSMs 获取 Secp256k1 用于账户恢复。新技术，如 Passkeys 和 OAuth 登录系统，可能需要额外的加密算法。将这些不同的技术结合在一起，可以改善用户管理来自各种设备、平台和环境的单个账户的体验。

本 AIP 引入了一个称为 `SingleSender` 的新的 `TransactionAuthenticator`，支持两种 `AccountAuthenticator`，即 SingleKeyAuthenticator 和 MultiKeyAuthenticator，分别支持单一密钥和 k-of-n 多密钥。这些认证器将证明类型从 `TransactionAuthenticator` 和 `AccountAuthenticator` 中解耦，简化了添加新的用于账户认证的加密证明的过程。

### 1. 目标

* 通过提供一种单一方法来表示其中的身份，消除 `TransactionAuthenticator`、`AccountAuthenticator` 和 `AuthenticationKey` 中的代码重复。
* 在多发送者和单发送者交易中，使添加新的加密协议的过程保持一致。当前的单发送者交易期望 `TransactionAuthenticator` 中具有特定类型的加密算法，而多发送者交易期望它们在 `AccountAuthenticator` 中。
* 提供一个跨单发送者和多发送者交易都适用的统一的多密钥认证框架。



## 二、备选方案

这项工作本可以专注于适用于所有情况的纯多密钥解决方案。这样做将消除针对单发送者应用程序的优化，但会增加添加用于授权的加密算法的负担。

与其在账户级别执行此操作，我们可以使用 Multisigv2。在 Multisigv2 中，多个链上账户共享一个账户。这与此处的模型类似，其中多个密钥共享同一个账户。但要注意的是，如果在 Multisigv2 中错放了密钥，那么这些账户中的资产就会丢失。这些账户最少需要支付 gas 费用，因此不可避免地会有损失。

## 三、规范

### 数据结构

```
TransactionAuthenticator::SingleSender { sender: AccountAuthenticator }
AccountAuthenticator::SingleKey { authenticator: SingleKeyAuthenticator }
AccountAuthenticator::MultiKey { authenticator: MultiKeyAuthenticator }

pub struct SingleKeyAuthenticator {
    public_key: AnyPublicKey,
    signature: AnySignature,
}

pub struct MultiKeyAuthenticator {
    public_keys: MultiKey,
    signatures: Vec<AnySignature>,
    signatures_bitmap: aptos_bitvec::BitVec,
}

pub enum AnySignature {
    Ed25519 {
        signature: Ed25519Signature,
    },
    Secp256k1Ecdsa {
        signature: secp256k1_ecdsa::Signature,
    },
}

pub enum AnyPublicKey {
    Ed25519 {
        public_key: Ed25519PublicKey,
    },
    Secp256k1Ecdsa {
        public_key: secp256k1_ecdsa::PublicKey,
    },
}

pub struct MultiKey {
    public_keys: Vec<AnyPublicKey>,
    signatures_required: u8,
}
```

### 逻辑

`TransactionAuthenticator` 现在包含一个 `SingleSender` 变体。 `SingleSender` 变体支持任何 `AccountAuthenticator`。 `AccountAuthenticator` 现在支持两个新变体 `SingleKey` 和 `MultiKey`，分别支持单个或多个任意密钥类型。支持的密钥类型在 `AnyPublicKey` 和 `AnySignature` 枚举中定义。可以通过更新 `AnyPublicKey` 和 `AnySignature` 来添加新的加密算法。

`AccountAuthenticator::MultiKey` 包含 `MultiKeyAuthenticator`，支持最多 255 个密钥和最多 32 个签名。255 个密钥的数量由对 `MultiKey::public_keys` 和 `signatures_bitmap` 长度的检查所限制。32 个签名的选择符合 Aptos 强制执行的系统标准，即不允许任何交易包含超过 32 个签名。这个 AIP 只是遵循了这个标准。

`MultiKey` 结构实际上是账户的公钥。 `MultiKey::signatures_required` 字段规定了必须包含多少个有效签名才能完全认证签名。

`MultiKeyAuthenticator::signatures` 中包含一组按照 `MultiKeyAuthenticator::signatures_bitmap` 中定义的 `1` 的顺序排列的签名。这样可以方便地通过两者之间进行 zip 操作，生成一个元组 `(index, signature)`，其中 `index` 是存储在 `MultiKeyAuthenticator::public_keys` 中的相应位置的 `AnyPublicKey`。

### 地址和认证密钥推导

在Aptos中，帐户地址和身份验证密钥是由两个部分派生而来：与帐户关联的公钥的哈希值以及与公钥的密钥算法相关联的方案或整数。该方案提供了一个唯一的后缀，以确保两个不同的密钥算法不会导致相同的地址所有权，并减轻了恶意用户可能获取受害者帐户访问权限的情况。

认证密钥是对授权执行账户事务的一组公钥的哈希表示。虽然账户地址是从账户的初始认证密钥派生的，但 Aptos 通过解耦这些值来实现账户密钥轮换。具体来说，认证密钥更新为新公钥的哈希，而地址保持不变。

==本 AIP 引入了 `SingleKey` 和 `MultiKey` 容器来存储公钥，并因此推导认证密钥。 `SingleKey` 和 `MultiKey` 的方案分别为 2 和 3。==

Secp256k1 Ecdsa 密钥的认证密钥可以通过执行以下操作来计算：`sha3_256(1 as u8 | 65 bytes of Secp256k1 Ecdsa Public Key | 2 as u8)`

第一个整数 1 取自 `AnyPublicKey` 结构中的 `Secp256k1Ecdsa` 位置。 第二个整数 2 取自 `Scheme` 枚举中的 `SingleKey` 值。 这直接源于 BCS 实现序列化的方式。

## 四、参考实现

[https://github.com/aptos-labs/aptos-core/pull/10519](https://github.com/aptos-labs/aptos-core/pull/10519)



## 五、测试

通过集成测试和对 Secp256k1 Ecdsa 单密钥变体的端到端测试进行了彻底验证。



## 六、风险和缺点

这增加了密钥可以使用的方式，从而使得同一私钥可以在不同的账户之间使用。然而，这已经存在于 MultiEd25519 中。

目标之一是允许用户在不同的设备上使用同一组密钥来访问他们的账户，这就带来了一个问题，即新设备如何枚举当前包含在 `MultiKey` 中的所有公钥。这个问题在 `MultiEd25519` 密钥中也存在，并且在本 AIP 中没有解决。

在 `TransactionAuthenticator` 中表示每个公钥的现有方式可能导致存储效率低下，因为 255 个 65 字节的 Secp256k1 Ecdsa 密钥的单个交易超过 16KB。通过 Merkle 树或其他形式使之更加高效是可能的，其中只需要数据的子集来进行证明，而其余部分用于验证完整性。目前尚未对此进行探讨，随着这些原始操作的应用逐渐成熟，可以在后续更新中加以解决。



## 七、安全考虑

由于这在很大程度上利用了系统中已经固有的相同功能和属性集，因此没有明显的安全考虑。



## 八、时间表

此 AIP 的预期时间表是发布 1.8 版本。