---
aip: 24
title: Move 库更新
author: gerben-stavenga
discussions-to (*optional): <a url pointing to the official discussion thread>
Status: 已接受
last-call-end-date (*optional): 
type: 标准
created: 2023年4月16日
updated (*optional): 2023年4月16日
---

[TOC]

# AIP-24 - Move 库更新

## 一、概述

我们增强了库，增加了 FixedPoint64 支持、额外的数学函数、字符串格式化和额外的内联函数。

## 二、动机

为了为Move开发者提供更全面的标准库。

## 三、规范

- 添加了`FixedPoint64`作为`FixedPoint32`的更精确对应物（counterpart），具有18位精度，并覆盖了从0到 $10^{18}$ 的数字。
- 将`sqrt`（平方根）、`exp`（指数）、`log`（对数）、`log2`（以2为底的对数）、`floor_log2`（向下取对数2的值）和`mul_div`（乘除）添加到库中作为标准函数。
- 为向量库添加了额外的内联函数，包括`for_each_reverse`（反向遍历）、`rotate`（旋转）、`partition`（分区）、`foldr`（右折叠）、`stable_partition`（稳定分区）和`trim`（修剪）。
- 在简单表（simple table）中添加了`upsert`功能，允许更新现有键或插入新键。
- 在`string_utils`模块中添加了格式化例程。我们有`to_string(value)`将值转换为人类可读的字符串，以及 `formatX(fmt, val1, val2, ..)`用于类似 Rust 的字符串格式化。

## 四、参考实现

- FixedPoint64 https://github.com/aptos-labs/aptos-core/pull/7074
- 数学库 https://github.com/aptos-labs/aptos-core/pull/6714/
- 内联函数 https://github.com/aptos-labs/aptos-core/pull/6882
- Upsert https://github.com/aptos-labs/aptos-core/pull/6860

## 五、时间线

Devnet：已提供了测试使用的版本

Testnet：约 4 月 24 日

Mainnet：5 月 1 日

