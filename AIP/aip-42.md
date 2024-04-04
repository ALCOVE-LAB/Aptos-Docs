---
aip: 42
title: Aptos TypeScript SDK V2
author: 0xmaayan
discussions-to: https://github.com/aptos-foundation/AIPs/issues/193
Status: Draft
last-call-end-date (*optional):
type: Standard
created: 07/07/2023
updated (*optional):
requires (*optional):
---

[TOC]

# AIP-42 - Aptos TypeScript SDK V2

## 一、概述

该 AIP 介绍了 Aptos TypeScript SDK 的新版本开发。

## 二、动机

鉴于当前版本中存在的几项重大问题，我们已经着手开发 TypeScript SDK 新版本。现行版本的整体结构和设计问题严重影响了其使用的效率和效果。而且，SDK 内部采用的命名规则和类型的不一致使得开发者感到困惑，降低了他们的工作效率。此外，现有的客户端支持的功能缺乏可定制性，并且在提供高级客户端特性支持方面也有所不足，这些问题增加了调试和问题解决的难度。

总的来说，这些问题共同导致用户的开发体验不佳。因此，新版本的 SDK 旨在解决这些问题，并为开发人员提供改进、简化和更用户友好的开发体验。

## 三、影响

新版本将在发布标签 `^2.0.0` 下提供。将升级其包到此版本的 TS SDK 用户需要对其代码进行更改以与 TS SDK V2 兼容。当前和现有的 `^1.x.x` 版本将可用，但将不会增加新功能。

## 四、规范

变更如下：

1. **用法：**引入全局的 `AptosConfig` 配置文件，作为一种便捷灵活的方式，让最终用户能够在不直接初始化 SDK 中的不同类的情况下配置和自定义项目。这降低了引入错误的可能性，简化了维护工作。同时，它还使得在不同环境中共享和分发项目配置变得更加容易。

```ts
class AptosConfig = {
  readonly network: string;

  constructor(config: AptosConfig){
    this.network = config.network ?? DEFAULT_NETWORK
  }
}

class Aptos {
  readonly transaction: Transaction;
  readonly account: AptosAccount;

  constructor(settings?: AptosConfig){
    this.config = new AptosConfig(settings);
    this.transaction = new Transaction(this.config)
    this.account = new AptosAccount(this.config);
  }
}

// 用法示例
const settings: AptosConfig = {
  network: Network.DEVNET,
};

const aptos = new Aptos(settings);
const transactions = await aptos.transaction.getTransactions()
```

2. **客户端：**在目前的架构中，所有的客户端请求操作都封装在 `AptosClient` 或 `IndexerClient` 类中，以及扩展这两个类的 `Provider` 类。但随着越来越多的请求不断增加到这两个类别中，它们变得异常庞杂并且令人迷惑。此外，用户实际上无需也不应该去考虑执行请求所依赖的服务类型，也不必劳心于选择哪一种类别。因此，我们决定废弃这两个类别，并将各个请求分散到各自相关的文件或上下文中。例如，所有和 `transaction` 相关的请求将被归纳至 `Transaction` 接口之下，该接口整合了所有与交易（读取/写入）相关的操作。

```ts
const transactions = await aptos.transaction.getTransactions();
```

3. **类型：**SDK 集成了一系列类型，涵盖了对象、请求及响应的表达方式。但是，目前SDK中定义这些类型的方式却常常令人迷惑。这部分混乱源于同时存在由服务器架构自动生成的类型和本地定义的类型，如 Types 和 TxnBuilderTypes。此外，这些类型的命名习惯也存在提升空间，例如采用比 MaybeHexString 更加描述性和直观的命名方式。
4. **生成交易载荷：**SDK 提供了一种功能性插件，通过该插件，用户只需传入必要的参数即可轻松调用功能并提交交易。然而，对于钱包开发者来说，他们首先需要生成交易，再进行模拟，之后才能签名并提交。针对这类功能插件，可以定义一个类，其中包含一组静态函数。这些静态函数接收特定的函数参数，并生成交易载荷，此载荷可用于生成待提交的原始交易，并对其进行模拟或直接提交至区块链。

```ts
const createCollectionPayload: TransactionPayload = AptosToken.generateCreateCollectionPayload(...);
// 模拟
const rawTransaction = aptos.transaction.generateRawTransaction(createCollectionPayload)
const txn = await aptos.transaction.simulate(sender, rawTransaction)
// 提交
const rawTransaction = aptos.transaction.generateRawTransaction(createCollectionPayload)
const signedTransaction = aptos.transaction.sign(sender,generateCreateCollectionPayload)
const hash = await aptos.transaction.submit(signedTransaction)
```

5. **日志服务：**通过在 SDK 中增加日志服务的支持，我们为开发人员提供了一个强大的工具，可以增强其应用程序的性能，排除问题，确保合规性，并为最终用户提供更强大和可靠的软件体验。

- 打印完整的堆栈跟踪
- 支持不同类型的日志（信息，警告，错误，调试）（info, warning, error, debug）
- 支持使用开关来禁用 / 启用日志记录

## 五、风险和缺点

- TS SDK 版本 `^1.x.x` 将不会获得新功能。
- 升级到新版本的现有用户需要进行代码更改以与新版本兼容。

## 六、测试

TS SDK 测试套件
