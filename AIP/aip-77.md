---
aip: 77
title: 多重签名 V2 增强
author: 
  - name: junkil-park
    url: https://github.com/junkil-park
  - name: movekevin
    url: https://github.com/movekevin
discussions-to: https://github.com/aptos-foundation/AIPs/issues/409
status: 草稿
last-call-end-date: <mm/dd/yyyy 最后留下反馈和评论的日期>
type: 标准 (框架)
created: 2024-03-28
updated: <mm/dd/yyyy>
requires: <AIP number(s)>
---

[TOC]

# AIP-77 - 多签 V2 增强

## 一、摘要

本 AIP 提议通过以下方式增强多签 V2：

(1) 限制待处理交易的最大数量，

(2) 引入批量操作功能，

(3) 隐式对交易执行进行投票。



## 二、宏观概述

本 AIP 提议添加了通过以下特性来增强多签 V2：
1. 限制待处理交易的最大数量。通过在多签 V2 账户模块中引入并行新常量 `MAX_PENDING_TRANSACTIONS` 来实现，该常量设置为20。如果待处理交易的数量超过此限制，合约将不会接受任何新的交易提案，直到待处理交易的数量降至限制以下。这是为了防止待处理交易队列被过多的待处理交易过载，从而有效减轻拒绝服务攻击（DoS）的风险（如 https://github.com/aptos-labs/aptos-core/issues/8411 中所述）。

2. 引入批量操作功能。此特性允许多签账户的所有者在单个批量操作中执行多个操作。这是通过在多签 V2 账户模块中引入新的公共入口函数 `vote_transactions` 和 `execute_rejected_transactions` 来完成的。此特性适用于原子性地执行多个交易，从而提高可用性，减少交易数量，节省 Gas 费，并有效对抗潜在的 DoS 攻击（如 https://github.com/aptos-labs/aptos-core/issues/8411 中所述）。

3. 隐式对交易执行进行投票。此特性允许多签账户的所有者在交易执行时隐式投票。多签交易执行首先隐式批准交易，而 `execute_rejected_transaction` 首先隐式拒绝交易。隐式投票并不是一个新概念，因为交易被创建时已经隐式投票以批准交易。这种修改减少了多签名 V2 用户流程中所需的交易数量，从而提高了可用性并节省了 Gas 费。此特性解决了此处描述的用户体验问题：https://github.com/aptos-labs/aptos-core/issues/11011。

## 三、影响

多签名账户不能有超过20个待处理交易，但这个限制对于大多数实际用例来说应该是足够的。如果有些多签名账户已经有超过 20 个待处理交易，它们仍然可以像以前一样继续投票和（拒绝）执行交易。然而，除非待处理交易的数量降至限制以下，否则它们不能有新的交易提案。

批量操作功能和隐式投票特性与现有的多签名 V2 实现向后兼容。现有的多签名 V2 用户可以继续使用现有的工作流程。新特性是可选的，它可以为用户带来好处，简化用户流程并节省 Gas 费。

## 四、规范和实现细节

限制待处理交易最大数量的规范如下：

* 通过在多签名账户模块中引入新常量 `MAX_PENDING_TRANSACTIONS`，将待处理交易的最大数量限制为 20。如果待处理交易的数量超过此限制，合约将不会接受任何新的交易提案，直到待处理交易的数量降至限制以下。
* 如果有些账户已经有超过20个待处理交易，它们仍然可以像以前一样继续投票和（拒绝）执行交易。然而，除非待处理交易的数量降至限制以下，否则它们不能有新的交易提案。

多签名账户模块中新引入的批量操作功能如下：
* `public entry fun vote_transactions(owner: &signer, multisig_account: address, starting_sequence_number: u64, final_sequence_number: u64, approved: bool)`
  * 此入口函数允许 `multisig_account` 的 `owner` 在 `starting_sequence_number` 到 `final_sequence_number` 的范围内批准（`approved = true`）或拒绝（`approved = false`）多个交易。
* `public entry fun execute_rejected_transactions(owner: &signer, multisig_account: address, final_sequence_number: u64)`
  * 此入口函数允许 `multisig_account` 的 `owner` 执行直到 `final_sequence_number` 的拒绝交易。

隐式对交易进行投票的规范如下：
* 当一个所有者执行多签交易时，会自动隐式地表示赞成这笔交易。如果该所有者之前已经投过票反对这笔交易，他/她的投票将会被更新为赞成。若之前尚未对此交易进行过赞成投票，隐式的投票行为将会自动将投票改为赞成，随之触发相应的`VoteEvent`事件。

* 当所有者运行 `execute_rejected_transactions` 时，会隐式地对该交易进行拒绝投票。如果所有者已经投票批准该交易，则投票更新为拒绝该交易。如果尚未对拒绝进行隐式投票，则隐式投票将投票更改为拒绝，从而发出相应的 `VoteEvent`。

* 添加了视图函数 `public fun can_execute(owner: address, multisig_account: address, sequence_number: u64): bool` 以在考虑了隐式投票的情况下, 检查所有者是否可以执行 `multisig_account` 的下一笔交易。

  

* 引入了一个视图函数`public fun can_reject(owner: address, multisig_account: address, sequence_number: u64): bool`，其目的是为了判断所有者在考虑隐式投票情况下，是否有权限拒绝 `multisig_account`  的下一笔交易。



## 五、参考实现

* https://github.com/aptos-labs/aptos-core/pull/12241 
* https://github.com/aptos-labs/aptos-core/pull/11941 



## 六、测试
参考实现包括覆盖端到端场景的单元测试。这些场景将在 devnet 和 testnet 上进行测试。



## 七、安全考虑

本AIP解决了 https://github.com/aptos-labs/aptos-core/issues/8411 中描述的安全问题。



## 八、时间线



## 九、建议的部署时间表

这个特性将是 v1.11 版本的一部分。