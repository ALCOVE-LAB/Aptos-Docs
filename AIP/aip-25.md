---
aip: 25
title: 结构体的交易参数支持
author: gerben-stavenga
discussions-to (*optional): <a url pointing to the official discussion thread>
Status: 已接受
last-call-end-date (*optional): 
type: 标准
created: 2023年4月16日
updated (*optional): 2023年4月16日
---

[TOC]

# AIP-25 - 结构体的交易参数支持

## 一、概述

我们使得某些选定的结构体可以作为入口函数和视图函数（ entry and view functions）的有效参数。



## 二、动机

到目前为止，只有原始值可以传递到入口函数。主要是因为允许通用结构体作为输入将使得重要的不变量失效，比如无中生有地创建Coins。

到目前为止，只有原始值可以传递给入口函数。主要是因为允许将通用结构体作为输入会破坏重要的不变量，比如无中生有地创建 Coin 。



## 三、规范

- 入口函数可以接受 `String` 作为输入参数。这已经是一个特殊情况，并且将继续得到支持。该字符串以一个必须是有效UTF-8的vector<u8>形式给出，否则交易验证将失败。
- 入口函数可以接受 `FixedPoint32`、`FixedPoint64` 作为输入参数。传递的值分别为 `u64` 或 `u128` ，分别代表固定点数值的原始值。
- 入口函数可以接受`Option<T>`作为输入参数，前提是`T`是有效的输入参数类型。格式是：一个空的向量表示`None`，以及一个单元素向量表示`Some(x)`，其中`x`的类型为`T`。超过一个元素的向量将在输入验证时会被拒绝。
- 入口函数可以接受`Object<T>`作为输入参数。格式将仅是对象的地址。资源`T`必须在指定地址存在，否则输入验证将失败。



## 四、参考实现

- 启用通用结构体的代码已合并至 https://github.com/aptos-labs/aptos-core/pull/7090



# 五、时间线

Devnet：已经可用于测试

Testnet：约 4 月 24 日

Mainnet：5 月 1 日
