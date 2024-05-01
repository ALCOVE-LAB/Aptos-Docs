---
aip: 62
title: Wallet Standard
author: 0xmaayan, hardsetting, NorbertBodziony
discussions-to (*optional): https://github.com/aptos-foundation/AIPs/issues/306
Status: Adoption
last-call-end-date (*optional): 19/02/2024
type: Ecosystem Standard
created: 29/01/2024
---

> [!TIP]
> 译者注：
> 原文链接：https://github.com/aptos-foundation/AIPs/blob/main/aips/aip-62.md

[TOC]

# AIP-62 - Wallet Standard (钱包标准)

## 一、概述

钱包标准定义了钱包与 dapp 之间的通用 API 交互模式。此 AIP 引入了一个为 Aptos 生态系统量身定制的新钱包互操作标准。



## 二、动机

当前大部分网络钱包采用浏览器扩展形式，通过注入脚本到全局窗口对象中，并期望 dapp 通过这一对象来感知它们的存在，以实现与 dapp 的交互。然而，此做法存在几个问题：

1. 这要求每个 dapp 必须学会如何定位这些对象，并有选择性地支持一系列可能与用户不相关的钱包。
2. 为了侦测钱包，dapp 必须运行一个持续的监控进程，不停地扫描窗口对象，以识别出是在 dapp 加载前还是加载后被注入的钱包。
3. 如果 dapp 依靠自己的检测逻辑而钱包在 dapp 之后加载或dapp 不知情新钱包存在，将存在创建竞态条件的风险。

在 Aptos 生态中，现行的标准实施方法也面临一些挑战。

1. 标准深度整合于 Aptos 钱包适配器，任何变动都可能会导致破坏性改变，要求 dapp 或钱包应对这些变更，从而产生持续的维护压力。
2. 因为每个 dapp 都需要安装和维护钱包插件的依赖，这种情况增加了遭遇供应链攻击的潜在风险。
3. 现有标准仅支持旧版 TS SDK 的输入、类型和逻辑，这意味着它无法享受新版 TS SDK 提供的特性与增强功能，而且旧版 TS SDK 已不再支持更新。

本提案推荐在 Aptos 钱包与 dapp 之间引入基于事件的通讯模式，以消除上述的各类问题。



## 三、影响

1. Dapp 开发者需：

- 了解并熟悉[新钱包标准](https://github.com/aptos-labs/wallet-standard/tree/main)
- 将自身代码迁移到[新的 TypeScript SDK](https://github.com/aptos-labs/aptos-ts-sdk)
- 实装钱包可发现功能，以屏蔽非 Aptos 钱包

2. 钱包开发者需：

- 了解并熟悉[新钱包标准](https://github.com/aptos-labs/wallet-standard/tree/main)
- 根据需要迁移到[新的 TypeScript SDK](https://github.com/aptos-labs/aptos-ts-sdk) 或维护从新 SDK 到旧 SDK 的类型转换层
- 注册自己的钱包，确保其能够被 dapp 正确发现
- 按照新标准实施符合 Aptos 钱包协议的 AptosWallet 类



## 四、核心概念

此[链无关的解决方案](https://github.com/wallet-standard/wallet-standard)已作为钱包与 dapps 通信的通用标准而推出，并已经在 [Solana](https://github.com/wallet-standard/wallet-standard) 及 [Sui](https://docs.sui.io/standards/wallet-standard) 获得实施，而 [Ethereum](https://eips.ethereum.org/EIPS/eip-6963) 社群最近也提出了相似的解决思路。



## 五、规范详情

[钱包互操作标准](https://github.com/aptos-labs/wallet-standard)提出了一系列区块链网络无关的接口与协议，目的是为了优化应用程序与注入式钱包之间的互动体验。

**标准化的功能特性**

以下列出了钱包需遵循的标准化功能，包括必须与建议实现的方法。
查阅建议的 [Aptos 功能清单](https://github.com/aptos-labs/wallet-standard/tree/main/src/features)

> 标有 `*` 的功能为可选实现

`aptos:connect` 方法旨在建立 dapp 与钱包的连接关系。

```ts
// `silent?: boolean` - 允许在不引起用户注意的情况下触发连接（如用于自动连接功能）
// `networkInfo?: NetworkInfo` - 指定 dapp 所选的网络信息（方便连接和切换网络操作）

connect(silent?: boolean, networkInfo?: NetworkInfo): Promise<UserResponse<AccountInfo>>;
```

`aptos:disconnect` 方法用于断开 dapp 和钱包之间建立的连接
```ts
disconnect(): Promise<void>;
```

`aptos:getAccount` 用于获取钱包中当前连接的账户

```ts
getAccount():Promise<UserResponse<AccountInfo>>
```

`aptos:getNetwork` 用于获取钱包中的当前网络

```ts
getNetwork(): Promise<UserResponse<NetworkInfo>>;
```

`aptos:signTransaction` 用于当前连接的账户在钱包中签署消息。

```ts
// `transaction: AnyRawTransaction` - a generated raw transaction created with Aptos’ TS SDK

signTransaction(transaction: AnyRawTransaction):AccountAuthenticator
```

`aptos:signMessage` 用于当前连接的账户在钱包中签署消息。
```ts
// `message: AptosSignMessageInput` - a message to sign

signMessage(message: AptosSignMessageInput):Promise<UserResponse<AptosSignMessageOutput>>;
```

`aptos:onAccountChange` 事件，当钱包中的账户发生变化时，钱包触发。

```ts
// `newAccount: AccountInfo` - The new connected account

onAccountChange(newAccount: AccountInfo): Promise<void>
```

`aptos:onNetworkChange` 事件，当钱包中的网络发生变化时，钱包触发。
```ts
// `newNetwork: NetworkInfo` - The new wallet current network

onNetworkChange(newNetwork: NetworkInfo):Promise<void>
```

`aptos:signAndSubmitTransaction*` 方法，使用钱包中当前连接的账户签署并提交交易。
```ts
// `transaction: AnyRawTransaction` - a generated raw transaction created with Aptos’ TS SDK

signAndSubmitTransaction(transaction: AnyRawTransaction): Promise<UserResponse<PendingTransactionResponse>>;
```

`aptos:changeNetwork*` 事件，dapp 发送给钱包，以更改钱包的当前网络

```ts
// `network:NetworkInfo` - The network for the wallet to change to

changeNetwork(network:NetworkInfo):Promise<UserResponse<{success: boolean,reason?: string}>>
```

`aptos:openInMobileApp*` 函数，支持将用户从移动设备的网络浏览器重定向到原生移动应用。钱包插件应添加钱包应在应用内浏览器打开的位置 URL。

```ts
openInMobileApp(): void
```

**类型**

> 注意：`UserResponse` 类型用于用户拒绝可拒绝的请求。例如，当用户想要连接但却关闭了窗口弹出时。

```ts
export enum UserResponseStatus {
  APPROVED = 'Approved',
  REJECTED = 'Rejected'
}

export interface UserApproval<TResponseArgs> {
 status: UserResponseStatus.APPROVED
 args: TResponseArgs
}

export interface UserRejection {
 status: UserResponseStatus.REJECTED
}

export type UserResponse<TResponseArgs> = UserApproval<TResponseArgs> | UserRejection;

export interface AccountInfo = { account: Account, ansName?: string }

export interface NetworkInfo {
  name: Network
  chainId: number
  url?: string
}

export type AptosSignMessageInput = {
  address?: boolean
  application?: boolean
  chainId?: boolean
  message: string
  nonce: string
}

export type AptosSignMessageOutput = {
  address?: string
  application?: string
  chainId?: number
  fullMessage: string
  message: string
  nonce: string
  prefix: 'APTOS'
  signature: Signature
}
```

**错误**

钱包必须抛出一个 [AptosWalletError](https://github.com/aptos-labs/wallet-standard/blob/main/src/errors.ts)。标准要求支持 `Unauthorized` 和 `InternalError`，但钱包可以抛出一个自定义的 `AptosWalletError` 错误

使用默认消息

```ts
if (error) {
  throw new AptosWalletError(AptosWalletErrorCode.Unauthorized);
}
```

使用自定义消息

```ts
if (error) {
  throw new AptosWalletError(
    AptosWalletErrorCode.Unauthorized,
    "My custom unauthorized message"
  );
}
```

使用自定义错误

```ts
if (error) {
  throw new AptosWalletError(-32000, "Invalid Input");
}
```



## 六、参考实现

该标准公开了一个[detect](https://github.com/aptos-labs/wallet-standard/blob/main/src/detect.ts#L17)功能，通过验证钱包中是否存在必需的函数来检测现有的钱包是否符合Aptos标准。这些函数被称为[features](https://github.com/aptos-labs/wallet-standard/tree/main/src/features)。每个特性应以 `aptos` 命名空间、`colon` 和 `{method}` 名称定义，即 `aptos:connect`。

**钱包提供者**

<ins>AptosWallet接口实现</ins>

钱包必须实现一个[AptosWallet接口](https://github.com/aptos-labs/wallet-standard/blob/main/src/wallet.ts)，并提供钱包提供者的信息和特性：

```ts
class MyWallet implements AptosWallet {
  url: string;
  version: "1.0.0";
  name: string;
  icon:
    | `data:image/svg+xml;base64,${string}`
    | `data:image/webp;base64,${string}`
    | `data:image/png;base64,${string}`
    | `data:image/gif;base64,${string}`;
  chains: AptosChain;
  features: AptosFeatures;
  accounts: readonly AptosWalletAccount[];
}
```

<ins>AptosWalletAccount接口实现</ins>

钱包必须实现一个 [AptosWalletAccount接口](https://github.com/aptos-labs/wallet-standard/blob/main/src/account.ts) ，该接口代表了已被dapp授权的账户。

```ts
enum AptosAccountVariant {
  Ed25519,
  MultiEd25519,
  SingleKey,
  MultiKey,
}

class AptosWalletAccount implements WalletAccount {
  address: string;

  publicKey: Uint8Array;

  chains: AptosChain;

  features: AptosFeatures;

  variant: AptosAccountVariant;

  label?: string;

  icon?:
    | `data:image/svg+xml;base64,${string}`
    | `data:image/webp;base64,${string}`
    | `data:image/png;base64,${string}`
    | `data:image/gif;base64,${string}`
    | undefined;
}
```

<ins>注册钱包</ins>

为了通知 dapp 其已准备就绪，钱包通过使用 [registerWallet](https://github.com/wallet-standard/wallet-standard/blob/master/packages/core/wallet/src/register.ts#L25) 方法来注册自身。

```ts
const myWallet = new MyWallet();

registerWallet(myWallet);
```

**Dapp**

<ins>获取钱包</ins>

Dapp使用[getAptosWallets()](https://github.com/aptos-labs/wallet-standard/blob/main/src/detect.ts#L30) 函数，该函数获取所有符合Aptos标准的钱包。

```ts
import { getAptosWallets } from "@aptos-labs/wallet-standard";

let { aptosWallets, on } = getAptosWallets();
```

<ins>注册事件</ins>

在初次加载与 dapp 启动之前，系统将获取所有已注册的钱包信息。为了在此时间点之后继续监测新注册的钱包，dapp 需使用事件监听器来订阅新钱包的注册，该监听器提供了一个取消订阅的函数，可用于之后撤销该监听器。

```ts
const removeRegisterListener = on("register", function () {
  // Dapp可以将新注册的aptos钱包添加到其自身的状态上下文中。
  let { aptosWallets } = getAptosWallets();
});

const removeUnregisterListener = on("unregister", function () {
  let { aptosWallets } = getAptosWallets();
});
```

Dapp现在有了一个事件监听器，所以它可以立即看到新的钱包，不需要再次轮询或列出它们。 如果dapp在任何钱包加载之前加载，这也是有效的（它将初始化，但看不到任何钱包，然后在它们加载时看到钱包）。

<ins>钱包请求</ins>

Dapp通过调用与所需操作相对应的特性名称来发出钱包请求。

```ts
const onConnect = () => {
  this.wallet.features["aptos:connect"].connect();
};
```



## 七、风险和缺点

新的标准使用了[新的TypeScript SDK](https://github.com/aptos-labs/aptos-ts-sdk)类型，因此要求dapps和钱包使用/迁移到新的TypeScript SDK，或者持有从新的TypeScript SDK类型到旧的TypeScript SDK类型的转换层。



## 八、未来潜力

该解决方案作为一个通用的实现，已被多种链和钱包采用，并有潜力被更广泛的项目所接受。因此，钱包从一个区块链迁移到另一个的工作量极小。同时，支持多链的 dApps 能够轻松识别遵循此标准的全部钱包。

对于 dApps 和钱包的集成与实施过程来说，都极为简便而无需付出巨大努力。在大部分情况下，仅需通过使用已提供的方法完成注册和检测流程。

任何未来的新功能或改进都应避免导入破坏性更改，这是因为每个钱包都运行其私有的插件代码，并在独立的上下文中实现各项功能。



## 九、时间线

### 1. 建议的实施时间线

一旦 AIP（Aptos Improvement Proposal）获得批准，dapps 和钱包就应开始实施必要的更改（如“参考实现”部分所述），以遵循新标准。



## 十、安全考量

新的发现机制旨在减轻 dApps 在安装及维护不同钱包包时面临的责任，进一步降低了供应链攻击的风险。

新方法已经在 [Solana](https://github.com/wallet-standard/wallet-standard) 和 [Sui](https://docs.sui.io/standards/wallet-standard) 上得到采纳并实施，而 [Ethereum](https://eips.ethereum.org/EIPS/eip-6963) 社区最近也提出了相似的方案。更进一步，多个不同钱包如 Phantom、Nightly、Brave Wallet 和 Martian 等也已经集成并采用了这一新标准。