---
aip: 39
标题: 分离 Gas 付款人
作者: gerben-stavenga, movekevin, davidiw, kslee8224, neoul, styner32
讨论区: https://github.com/aptos-foundation/AIPs/issues/173
状态: 草案
最后调用结束日期: TBD
类型: 标准 (核心)
创建日期: 06/14/2023
---

[TOC]

# AIP 39 - 分离Gas付款人
## 一、概述

这个 AIP 提出了一种机制，用于指定一个与交易发送者不同的 Gas 付款人账户。交易的 Gas 从 Gas 付款人账户中扣除，而不给其签名者任何其他用途的访问权限 - 无论谁支付 Gas，交易都会以完全相同的方式执行。

## 二、动机

应用程序可以代表其用户支付 Gas，这是一个常见的用例。目前， Aptos 上的交易总是从正常的单一发送者交易或多代理交易的主要发送者那里扣除 Gas 。没有直接支持指定应从其中扣除 Gas 的不同账户。用户目前可以构造多代理交易，并调用自定义的 Move 代理函数，以便：

1. 主要发送者是 Gas 付款人， Gas 从这个账户中扣除。
2. 第二位发送者是将在交易执行中使用的实际签名者。代理调用将丢弃第一个签名者(来自 Gas 付款人)，并将第二个签名者传递给目标函数调用。

尽管这可以达到分离 Gas 付款人的期望效果，但这种方法很麻烦，需要自定义的链上和链下代码。此外，这将使用 Gas 付款人的随机数，这为扩展 Gas 支付操作创建了瓶颈，其中 Gas 付款人账户可能为许多不同用户的许多交易支付 Gas。

## 三、提案

我们可以正确地支持将 Gas 付款人账户与发送者分离，同时保留与发送者自己发送交易相同的效果。这意味着：

- 应使用发送者账户的随机数，而不是 Gas 付款人的。这允许更容易地扩展 Gas 支付操作。
- 带有单独 Gas 付款人的交易应使用与普通交易相同的有效载荷(入口函数和脚本调用)，并且不应需要中间代理代码来处理多个签名者。

### 1. 通用化多代理交易

多代理交易过去一直作为扩展标准交易(带有一系列次要签名者)的基本构造很有用。具体来说，多代理交易引入了 RawTransactionWithData - 一个围绕标准 SignedTransaction 的包装数据构造，向其添加更多数据。这允许签名验证然后验证用户的已签名交易也包含额外的数据(例如，次要签名者)。我们可以利用这个相同的数据结构来扩展一个与支付 Gas 相关的数据的交易：

```
pub enum RawTransactionWithData {
    MultiAgent {
        raw_txn: RawTransaction,
        secondary_signer_addresses: Vec<AccountAddress>,
    },
    MultiAgentWithFeePayer {
        raw_txn: RawTransaction,
        secondary_signer_addresses: Vec<AccountAddress>,
        fee_payer_address: AccountAddress,
    },
}
```

这可以被认为是 MultiAgent 交易数据的一种概括，允许一个包含新的交易数据类型，可以向所有种类的交易添加一个单独的费用付款人地址和签名，包括 MultiAgent。流程如下：

1. 要发送从 0xsender 到 0xreceiver 的 USDC 转账，其中一个单独的账户 0xpayer 支付 Gas，应用程序可以首先构造一个带有标准有效载荷的多代理交易(入口函数 0x1::aptos_account::transfer_coins， 其中发送者是 0xsender)。 0xpayer 在 RawTransactionWithData::MultiAgentWithFeePayer 中被指定为 fee_payer_address。所有这些都可以通过 SDK 支持轻松完成。
2. 应用程序可以提示用户使用他们的账户 0xsender 签署交易有效载荷。用户可以清楚地看到他们正在签署一个带有单独 Gas 费用付款人地址的交易。
3. 有效载荷和签名然后可以传递到服务器端，那里 0xpayer 将审查并签署交易。
4. 交易现在已经完成。 0xpayer 可以自己发送交易，或者将其传回到客户端，让用户 (0xsender) 自己提交。无论哪种方式， Gas 都将从 0xpayer 中扣除，交易将在 0xsender 的上下文中执行，使用他们账户的随机数和签名者。

这种实施相对直接：

1. 创建新的多代理序言和尾言函数，其中应在单独的 Gas 付款人账户上应用随机数/ Gas 验证和最终 Gas 收费，而不是发送者。执行流程还应验证 Gas 付款人签名是否有效，以及发送者和 Gas 付款人是否都在新引入的 RawTransactionWithData::MultiAgentWithFeePayer 交易数据上签名。
2. 更新 aptos_vm，当存在单独的 Gas 付款人地址时，正确地调用新的序言/尾言流程，而不是原来的流程。
3. 更新 SDK 以支持在发送者地址列表的末尾添加单独的 Gas 付款人。
4. 更改在 Indexer 中的默认硬币处理器中使用的 coin_model，以正确地将 Gas 燃烧活动归因于 Gas 付款人，而不是发送者。

## 四. 对索引的影响

在任何一种方法中，硬币处理器/模型都需要更新，以正确地归因 Gas 燃烧事件(当前硬编码为发送者)。运行自己的索引的索引服务提供商和生态系统项目需要升级他们的代码以获取这个更改。

## 五、参考实现

https://github.com/aptos-labs/aptos-core/pull/8904

## 六、风险和缺点

主要的风险是智能合约风险，其中可能存在虚拟机(Gas 补充)和 Move (交易序言/尾言)更改中的漏洞。

## 七、未来潜力

Gas 付款人为未来的许多令人兴奋的扩展铺平了道路，例如:

1. 使用对象而不是账户支付 Gas
2. 更通用的账户认证，可以完全自定义控制哪个账户/对象可以用于 Gas，或者账户的签名者是否可以用于特定目的。这与这里的 Gas 更改有关，因为这是对自 Diem 时代以来存在的严格单一发送者机制的交易序言/尾言流程的第一个非琐碎修改。

## 八、建议的实施时间表

将包含在v1.6版本中，该版本将在2023年6月底进入测试网，2023年7月进入主网。