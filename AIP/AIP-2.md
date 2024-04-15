---
aip: 2
title: 多个 Token 更改
author: areshand
discussions-to: https://github.com/aptos-foundation/AIPs/issues/2
Status: 已接受
last-call-end-date (*optional):
type: 标准（框架）
created: 12/7/2022
---

[TOC]

# AIP-2 - 多个 Token 更改

## 一、概述

本提案包含以下更改：


1. token CollectionData 修改函数（mutation functions）：提供设定函数（set functions），根据 token 的可变配置（mutability setting），来更新 TokenData 的各个字段。
2. token TokenData 元数据修改函数：提供设定函数（set functions），根据 collection 的可变配置（mutability setting），来更新 CollectionData 的各个字段。。
3. 修复 collection 供应下溢（supply underflow）的错误：修复了一个导致无限 collection 的 TokenData 销毁时出现下溢（小于可表示的最小正值）错误的 bug。
4. 修复事件（event）的顺序：在铸造（mint） token 时，存入（deposit）事件先于铸造（mint）事件进入队列。此更改可纠正顺序，使 token 存入事件排在铸造事件之后
5. 创建一个公开的转移（transfer）入口，支持用户选择（opt-in）直接转移 token ：提供一个入口函数，允许用户在选择直接转移 token 时直接进行 token转移。
6. 确保版税（royalty）分子小于分母：我们观察到约 0.004% 的 token 版税大于 100%，这一改动引入了一个断言，以确保版税总是小于或等于 100%。

## 二、动机

更改 1、2：动机是支持基于可变性配置的 CollectionData 和 TokenData 字段变更，以便创建者可以根据自己的程序逻辑更新字段，以支持新的产品功能

更改 3、4：动机是修复现有问题，使 token 合约正常工作

更改 5：目的是允许 dapp 直接调用函数，而无需部署自己的合约或脚本

更改 6：这是为了防止潜在的恶意 token 收取高于 token 价格的费用。

## 三、理由

更改 1、2 是对现有 token 标准规范的补充，没有引入新的功能

更改 3、4、5 和 6 是小修复和简单的更改。

## 四、参考实施

上述变更的 PR 如下：

变更 1, 2：[aptos-labs/aptos-core#5382](https://github.com/aptos-labs/aptos-core/pull/5382) [aptos-labs/aptos-core#5265](https://github.com/aptos-labs/aptos-core/pull/5265) [aptos-labs/aptos-core#5017](https://github.com/aptos-labs/aptos-core/pull/5017)

更改 3：[aptos-labs/aptos-core#5096](https://github.com/aptos-labs/aptos-core/pull/5096)

更改 4：[aptos-labs/aptos-core#5499](https://github.com/aptos-labs/aptos-core/pull/5499)

更改 5：[aptos-labs/aptos-core#4930](https://github.com/aptos-labs/aptos-core/pull/4930)

更改 6：[aptos-labs/aptos-core#5444](https://github.com/aptos-labs/aptos-core/pull/5444)

## 五、风险和缺点

变更 1、2 经过内部审核和审计，以全面审查风险

变更 3、4 和 6 是为了降低已确定的风险和缺点

变更 5 是为了提高可用性，此变更不引入新功能

## 六、时间表

参考实施变更将于 11/14（太平洋标准时间）部署到 devnet，以便于测试和提供反馈/讨论。
本 AIP 将在 11/17 前公开征求公众意见。
讨论结束后，参考实施更改将于 11/17 部署到 testnet 中进行测试。

原文链接：[https://github.com/aptos-foundation/AIPs/blob/main/aips/aip-2.md](https://github.com/aptos-foundation/AIPs/blob/main/aips/aip-2.md)
