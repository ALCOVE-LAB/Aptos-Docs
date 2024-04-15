---
aip: 14
title: 更新锁仓合约
author: movekevin
Status: Accepted
last-call-end-date:
type: Standard (Framework)
created: 2023/02/10
updated: 2023/02/10
---

[TOC]

# AIP - 14 - 更新锁仓合约

## 一、动机

本提案旨在更新当前的锁定合约中的奖励分配逻辑：https://github.com/aptos-labs/aptos-core/blob/496a2ce5481360e555b670842ef63e6fcfbc7926/aptos-move/framework/aptos-framework/sources/vesting.move#L420

当前的奖励计算使用的是基础的  `staking_contract` 的记录本金，每次调用 `staking_contract::request_commission` 时都会更新。

目前的奖励计算方法是基于底层质押合约`staking_contract` 记录的本金，这个记录会在每次执行质押合约的请求佣金函数`staking_contract::request_commission` 时进行更新。



## 二、提议

通过质押合约（staking_contract）的本金来计算奖励的方法比较间接。理想的方式是，锁仓合约（vesting_contract）使用剩余的授权资金额作为基础，以此来计算迄今为止累积的实际奖励金额。下面通过一个例子来进一步说明这个概念：

1. 为了简单起见，锁定合约有100 APT和 1 个抵押参与者。这里的 100 APT 同时也表示尚未分配的金额。而底层质押合约的初始质押金额也是 100 APT。
2. 抵押池赚取了50 APT。
3. 在向锁定池计算尚未支付的奖励前，`vesting::unlock_rewards` 首先要求支付给运营商佣金，即 5 APT（级赚取的 50 APT 的10%）。剩余的总活跃质押金额为145 APT，全部属于契约锁定池。
4. 鉴于目前还没有 token 被锁定，当前的 `vesting::unlock_rewards`  将计算尚未结算的奖励，以分配给权益池的参与者。计算公式为：`145(总活跃份额 total active in stake pool) - 100（未分配的份额 remaining_grant）= 45`。



## 三、参考实现

https://github.com/aptos-labs/aptos-core/pull/6106



## 四、时间安排

目标测试网发布日期：2023年2月