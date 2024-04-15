---
aip: 49
title: 用于交易认证的 secp256k1 ECDSA
author: davidiw
discussions-to (*optional): https://github.com/aptos-foundation/AIPs/issues/247
Status: 草案
last-call-end-date (*optional): <mm/dd/yyyy 最后留下反馈和审查的日期>
type: 接口
created: 09/28/2023
updated (*optional): <mm/dd/yyyy>
---

[TOC]

# AIP-49 - 用于交易认证的 Secp256k1 ECDSA

尽管我们期望硬件加密设备能支持更多的密钥算法，然而，Aptos 的主要密钥算法 Ed25519 还没有在整个生态系统中广泛被接受。`secp256k1` ECDSA 作为现行算法，已经得到了广泛支持。本 AIP 提议将 `secp256k1` ECDSA 引入作为 Aptos 交易的认证机制。

## 一、概述

在 Aptos 中，每笔交易都含有一个交易认证器，它包含了数字签名和公钥信息，而交易本身则携带着发起者的信息。为了核实交易是否经过恰当的授权签名，验证者会确认公钥能否有效验证交易上的签名，以及该公钥的哈希是否已作为哈希值保存在链上的账户之下。完成这一验证之后，验证者便能够断定该账户的主人实际上已授权此项交易。本 AIP 新增了对 `secp256k1` ECDSA 在交易认证中的支持。

## 二、动机

- 许多组织已经支持 `secp256k1` Ecdsa，但尚未支持 Ed25519
- 硬件加密尚未广泛采用 Ed25519，但兼容 `secp256k1` ECDSA

## 三、规范

虽然大部分内容都是关于 `secp256k1` ECDSA 的应用，但以下是与 Aptos 相关的不同方面：

- 所有签名都是标准化的，即 s 被设置为低阶。
- 非标准化的签名将被拒绝。
- 由于 `secp256k1` ECDSA 对 32 字节消息进行签名和验证，我们的框架通过将 Sha3-256 应用于消息来生成 32 字节的消息摘要。

## 四、参考实现

https://github.com/aptos-labs/aptos-core/pull/10207

## 五、测试

已在端到端测试中完全实现和验证。

## 六、时间表

### 1. 建议的实施时间表

已完全实施，待外部反馈。



### 2. 建议的开发者平台支持时间表

- API 支持已经验证
- 索引器代码已经更新，正在等待在开发网络上的验证
- SDK 即将更新



### 3. 建议的部署时间表

预计于十月初可用于开发网络，并打算在 1.8 版中发布。