---
aip: 82
title: 交易上下文扩展
author: 
  - name: junkil-park
    url: https://github.com/junkil-park
  - name: lightmark
    url: https://github.com/lightmark
  - name: movekevin
    url: https://github.com/movekevin
discussions-to: <指向官方讨论帖的URL>
status: 草稿
last-call-end-date: <mm/dd/yyyy 最后留下反馈和评论的日期>
type: 标准 (框架)
created: 2024-04-02
updated: <mm/dd/yyyy>
requires: <AIP number(s)>
---

[TOC]

# AIP-82 - 交易上下文扩展

## 一、摘要

本AIP提议在Aptos框架内扩展交易上下文模块。增强功能将使用户能够在智能合约中直接检索用户交易信息。这一特性对于支持自定义用户序言和尾声的机制至关重要，这些将在以后的AIP中单独处理。

### 1. 不讨论的部分

本AIP不包括检索用户交易签名的API，尽管它们是已签名交易的一部分。这是因为目前还没有识别到对这种功能的需求。然而，如果将来需要，可以引入此功能。

此扩展仅适用于用户交易，不适用于其他类型的系统交易。



## 二、宏观概述

此特性通过在 Aptos 框架内扩展交易上下文模块，为智能合约提供对当前用户交易信息的访问权限。信息包括发送者、次要签名者、Gas 支付者、最大 Gas 量、Gas 价格、链 ID、入口函数有效载荷和多签有效载荷。

本 AIP 在交易上下文模块内引入了一个API，它由一组公共函数组成，用于检索交易上下文信息。例如，在智能合约执行期间，可以通过调用相应的函数来检索交易发送者的地址或 Gas 支付者的地址。有关提议 API 的更全面解释，请参见[规范和实现细节](#四、规范和实现细节)部分。

这一增强对于促进自定义用户开始和结束（ prologues and epilogues）的机制至关重要，目前这些正在开发中。自定义开始和结束将能够使用提案的 API 检索交易信息并根据用户交易上下文执行特定逻辑。有关自定义用户开始和结束的设计和实现的详细信息将在未来的 AIP 中讨论。



## 三、影响

智能合约开发者可以利用这一特性，使他们的智能合约能够直接从当前用户交易中访问上下文信息。



## 四、规范和实现细节

本 AIP 通过引入交易上下文 API 扩展了 Aptos 框架的交易上下文模块（即`aptos_framework::transaction_context`）。这个API包括一系列公共函数，允许智能合约访问当前用户交易的信息。API规范如下：

* `public fun sender(): address`
  * 返回当前交易的发送者地址。
* `public fun secondary_signers(): vector<address>`
  * 返回当前交易的次要签名者列表。
* `public fun gas_payer(): address`
  * 返回当前交易的 Gas 支付者地址。
* `public fun max_gas_amount(): u64`
  * 返回当前交易所指定的最大 gas 单位数。
* `public fun gas_unit_price(): u64`
  * 返回当前交易中指定的以 Octas 计价的 gas 单位价格。
* `public fun chain_id(): u8`
  * 返回为当前交易指定的链 ID。
* `public fun entry_function_payload(): Option<EntryFunctionPayload>`
  * 如果当前交易存在有效载荷，则返回这个交易中入口函数的有效载荷。否则，返回`None`。
* `public fun multisig_payload(): Option<MultisigPayload>`
  * 如果当前交易存在有效载荷，则返回这个交易中多签的有效载荷。否则，返回`None`。

入口函数有效载荷定义如下：

```
struct EntryFunctionPayload has copy, drop {
    account_address: address,
    module_name: String,
    function_name: String,
    ty_args_names: vector<String>,
    args: vector<vector<u8>>,
}
```

以下是访问入口函数有效载荷字段的函数：

* `public fun account_address(payload: &EntryFunctionPayload): address`
* `public fun module_name(payload: &EntryFunctionPayload): String`
* `public fun function_name(payload: &EntryFunctionPayload): String`
* `public fun type_arg_names(payload: &EntryFunctionPayload): vector<String>`
* `public fun args(payload: &EntryFunctionPayload): vector<vector<u8>>`

多签有效载荷定义如下：

```
struct MultisigPayload has copy, drop {
    multisig_address: address,
    entry_function_payload: Option<EntryFunctionPayload>,
}
```

以下是访问多签有效载荷字段的函数：

* `public fun multisig_address(payload: &MultisigPayload): address`
* `public fun inner_entry_function_payload(payload: &MultisigPayload): Option<EntryFunctionPayload>`

## 五、参考实现

https://github.com/aptos-labs/aptos-core/pull/11843



## 六、测试

参考实现包括多个单元测试和端到端测试，涵盖各种正面和负面场景。这些场景将在 devnet 和 testnet 上进行测试。



## 七、安全考虑

本 AIP 允许智能合约访问当前执行的交易信息。由于用户交易数据已经存储在链上并且是公开可用的，本 AIP 不引入与秘密信息披露相关的任何额外安全风险。



## 八、未来潜力

这一特性对于启用将在单独的 AIP 中详细说明的自定义用户序言和尾声至关重要。



## 九、时间线

### 1. 建议的实现时间表

实现已在v1.12分支截止前登陆到`main`分支。

### 2. 建议的开发者平台支持时间表

N.A.

### 3. 建议的部署时间表

* 在devnet上：随版本v1.12发布
* 在 testnet 和 mainnet 上：取决于 AIP 批准进程

