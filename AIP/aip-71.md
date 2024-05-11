---
aip: 71
title: 使用模块事件重构 Aptos 框架事件
author: lightmark
discussions-to (*optional): https://github.com/aptos-foundation/AIPs/issues/367
Status: In Review
type: Framework
created: 02/22/2024
---

[TOC]

# AIP-71 - 使用模块事件重构 Aptos 框架事件

## 一、摘要

这个AIP提出了一个迁移计划，将 Aptos 框架中的所有句柄（handle）事件（event v1）迁移到模块（module）事件（event v2），并添加新的事件。

### 1. 目标

- 尽快将所有现有事件迁移到模块事件，以最小程度影响下游自定义索引器。
- 在迁移更改中附加新的有用的模块事件。

## 二、动机

[AIP-44](https://github.com/ALCOVE-LAB/Aptos-Docs/blob/main/AIP/aip-44.md) 引入了模块事件，并解释了与旧事件 `EventHandle` 相比的动机和优势。鉴于其优越性，将 Aptos 框架中的所有事件迁移到模块事件是可取的。其主要优点包括：

- 模块事件不需要事件句柄。
- 模块事件支持并行执行。
- 模块事件更易于索引。

此外，在迁移过程中，也是在框架升级时添加新的、实用的事件的绝佳时机。



## 三、影响

迁移成功后，将使 Aptos Move 开发人员和索引构建者更容易使用事件。 此外，对于普通用户来说，模块事件将更容易理解和使用，因为当前的形式更类似于 Solidity 的语法。



## 四、替代方案

另一种解决方案是直接将所有事件从现有的形式更改为模块事件。这样做的结果是，所有这些更改在主网可用之前未更新的分布式应用（dApps）都会立即出现问题，因为它们依赖于当前的事件形式。



## 五、规范

默认的迁移策略：
- 为事件类型创建新的 `T2` 结构体，并通过每个 `event::emit_event<T1>()` 调用添加一个 `#[event]`，并添加一个 `event::emit<T2>()` 调用。

已迁移的事件

| 模块 | 事件 v1 名称 | 事件 v2 名称 | ABI 更新 |
|:---:|:---:|:---:|:---:|
|account.move|KeyRotationEvent|KeyRotation|+account:address |
|aptos_account.move|DirectCoinTransferConfigUpdatedEvent|DirectCoinTransferConfigUpdated|  + account:address|
|coin.move|DepositEvent|Deposit|+ account:address|
|coin.move|WithdrawEvent|Withdraw|+ account: address|
|object.move|TransferEvent|Transfer| |
|aptos_governance.move|CreateProposalEvent|CreateProposal| |
|aptos_governance.move|VoteEvent|Vote| |
|aptos_governance.move|UpdateConfigEvent|UpdateConfig| |
|block.move|NewBlockEvent|NewBlock| |
|block.move|UpdateEpochIntervalEvent|UpdateEpochInterval| |
|aptos-token-objects/token.move|MutationEvent|Mutation|+ token:address|
|aptos-token-objects/collection.move|MutationEvent|Mutation|+ collection:address|
|aptos-token-objects/collection.move|BurnEvent|Burn| + previous_owner|
|aptos-token-objects/collection.move|MintEvent|Mint|+ collection:address|
|multisig_account.move|AddOwnersEvent|AddOwners|+ account: address|
|multisig_account.move|RemoveOwnersEvent|RemoveOwners|+ account: address|
|multisig_account.move|UpdateSignaturesRequiredEvent|UpdateSignaturesRequired|+ account: address|
|multisig_account.move|CreateTransactionEvent|CreateTransaction|+ account: address|
|multisig_account.move|VoteEvent|Vote|+ account: address|
|multisig_account.move|ExecuteRejectedTransactionEvent|ExecuteRejectedTransaction|+ account: address|
|multisig_account.move|TransactionExecutionSucceededEvent|TransactionExecutionSucceeded|+ account: address|
|multisig_account.move|TransactionExecutionFailedEvent|TransactionExecutionFailed|+ account: address|
|multisig_account.move|MetadataUpdatedEvent|MetadataUpdated|+ account: address|
|reconfiguration.move|NewEpochEvent|NewEpoch| |
|stake.move|RegisterValidatorCandidateEvent|RegisterValidatorCandidate| |
|stake.move|SetOperatorEvent|SetOperator| |
|stake.move|AddStakeEvent|AddStake| |
|stake.move|ReactivateStakeEvent|ReactivateStake| |
|stake.move|RotateConsensusKeyEvent|RotateConsensusKey| |
|stake.move|UpdateNetworkAndFullnodeAddressesEvent|UpdateNetworkAndFullnodeAddresses| |
|stake.move|IncreaseLockupEvent|IncreaseLockup| |
|stake.move|JoinValidatorSetEvent|JoinValidatorSet| |
|stake.move|DistributeRewardsEvent|DistributeRewards| |
|stake.move|UnlockStakeEvent|UnlockStake| |
|stake.move|WithdrawStakeEvent|WithdrawStake| |
|stake.move|LeaveValidatorSetEvent|LeaveValidatorSet| |
|staking_contract.move|UpdateCommissionEvent|UpdateCommission| |
|staking_contract.move|CreateStakingContractEvent|CreateStakingContract| |
|staking_contract.move|UpdateVoterEvent|UpdateVoter| |
|staking_contract.move|ResetLockupEvent|ResetLockup| |
|staking_contract.move|AddStakeEvent|AddStake| |
|staking_contract.move|RequestCommissionEvent|RequestCommission| |
|staking_contract.move|UnlockStakeEvent|UnlockStake| |
|staking_contract.move|SwitchOperatorEvent|SwitchOperator| |
|staking_contract.move|AddDistributionEvent|AddDistribution| |
|staking_contract.move|DistributeEvent|Distribute| |
|staking_contract.move|SwitchOperatorEvent|SwitchOperator| |
|vesting.move|CreateVestingContractEvent|CreateVestingContract| |
|vesting.move|UpdateOperatorEvent|UpdateOperator| |
|vesting.move|UpdateVoterEvent|UpdateVoter| |
|vesting.move|ResetLockupEvent|ResetLockup| |
|vesting.move|SetBeneficiaryEvent|SetBeneficiary| |
|vesting.move|UnlockRewardsEvent|UnlockRewards| |
|vesting.move|VestEvent|Vest| |
|vesting.move|DistributeEvent|Distribute| |
|vesting.move|TerminateEvent|Terminate| |
|vesting.move|AdminWithdrawEvent|AdminWithdraw| |
|voting.move|CreateProposalEvent|CreateProposal| |
|voting.move|RegisterForumEvent|RegisterForum| |
|voting.move|VoteEvent|Vote| |
|voting.move|ResolveProposal| | |
|token_event_store.move|CollectionDescriptionMutateEvent|CollectionDescriptionMutate| |
|token_event_store.move|CollectionUriMutateEvent|CollectionUriMutate| |
|token_event_store.move|CollectionMaxiumMutateEvent|CollectionMaxiumMutate| |
|token_event_store.move|OptInTransferEvent|OptInTransfer| |
|token_event_store.move|UriMutationEvent|UriMutation| |
|token_event_store.move|DefaultPropertyMutateEvent|DefaultPropertyMutate| |
|token_event_store.move|DescriptionMutateEvent|DescriptionMutate| |
|token_event_store.move|RoyaltyMutateEvent|RoyaltyMutate| |
|token_event_store.move|MaxiumMutateEvent|MaximumMutate| |

## 六、参考实现

https://github.com/aptos-labs/aptos-core/pull/10532

https://github.com/aptos-labs/aptos-core/pull/11688



## 七、风险和缺点

在迁移过程中，
- 同时发出 v1 和 v2 事件会导致区块链性能下降 5% - 10%
- 涉及框架模块双重发出事件的相关交易会增加 5% - 15% 的 Gas



## 八、未来潜力

迁移期将需要 3-6 个月，具体取决于过渡到模块事件流的进展情况。之后，可能需要一个新的 AIP 来删除所有旧事件。



## 九、时间表

### 1. 建议的实施时间表

已完成



### 2. 建议的开发者平台支持时间表

在1.11版本发布到主网之前。



### 3. 建议的部署时间表

在 Q1 结束时发布到测试网，然后发布到主网。



## 十、安全注意事项

请参阅[风险和缺点](#七、风险和缺点)。