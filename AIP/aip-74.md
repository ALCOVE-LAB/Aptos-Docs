---
aip: 74
title: 增加区块燃气限制以适应并发增加
author: 
  - name: Sital Kedia
    email: skedia@aptoslabs.com
  - name: Igor Kabiljo
    email: ikabiljo@aptoslabs.com
discussions-to: https://github.com/aptos-foundation/AIPs/issues/375
status: 已接受
type: 标准
created: 2024-03-12
---

[TOC]

# AIP-74 - 为了应对并发性的增加，需要增加区块的 Gas 限制

## 一、摘要

我们升级了主网验证节点的硬件规格，以实现主网的更高吞吐量，并且因此还将BlockSTM中使用的默认并发性增加到32，在https://github.com/aptos-labs/aptos-core/pull/12200中。为了充分利用更高的并发性，我们还需要增加区块大小和区块气体限制。本AIP建议将区块气体限制和相关配置增加50％，以应对并发性增加。

为了提升主网的处理能力，我们提升了主网验证节点的硬件配置。相应地，在 https://github.com/aptos-labs/aptos-core/pull/12200 中，我们将 BlockSTM 默认的并发数提升至 32。要想充分发挥提高后的并发能力，我们还需要增加区块的大小和块 Gas 限制。因此，本次AIP（Aptos Improvement Proposal）建议将块 Gas 限制及相关配置提升 50%，以匹配并发水平的提升。

### 1. 目标

在硬件升级后提高主网吞吐量。



## 二、动机

为实现主网更高的数据处理能力，我们对主网验证节点的硬件配置进行了升级。相应地，我们也提高了 BlockSTM 的默认并行数至 32，具体内容见https://github.com/aptos-labs/aptos-core/pull/12200。为了完整地利用提高后的并行性，我们还需要扩大区块的大小以及提高区块的 Gas 限制。因此，本次 AIP（Aptos Improvement Proposal）提议将区块 Gas 限制及相关配置提升 50%，以匹配并行度的提升。通过这一变更，将使主网的数据处理能力提升 50%，使网络能够在每秒钟处理更多的交易量。



## 三、影响

这与增加区块大小一起将使得主网吞吐量增加50％。



## 四、测试

增加区块 Gas 限制已在 Forge 以及 devnet 和 testnet 中进行了测试，并产生了预期的更高吞吐量。



## 五、时间表

### 1. 建议的部署时间表

计划是在面向 1.10 的治理提案中启用它。
