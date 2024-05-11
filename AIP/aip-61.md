---
aip: 61
title: Keyless accounts
author: Alin Tomescu (alin@aptoslabs.com)
discussions-to (*optional): https://github.com/aptos-foundation/AIPs/issues/297
Status: Approved
last-call-end-date (*optional): 02/15/2024
type: <Standard (Core, Framework)>
created: 01/04/2024
updated (*optional): <mm/dd/yyyy>
requires (*optional): <AIP number(s)>
---

[TOC]

# AIP-61 - Keyless accounts（无秘钥账户）

## 一、摘要

 >  用3~5句话总结我们要解决的问题是什么，我们是如何解决的

目前，唯一确保 Aptos 账户安全的方法是妥善保管与之相连的**秘密密钥（SK）**[^multisig]。然而，这个任务远远不如听起来那么简单。实际情况是，用户很容易遗失秘密密钥（比如，在第一次设置 Aptos 钱包时忘记记录助记词）或者密钥被盗用（例如，被骗子诱导泄露了自己的 SK）。这样一来，让用户开始使用的过程变得异常困难，并且当他们的账户丢失或被盗时，还会导致用户流失。

在本 AIP 中，我们阐述了一种对用户更加友好的账户管理途径，该方法基于未经修改的[^openpubkey] **OpenID Connect （OIDC）** 标准，并结合了关于 **OIDC 签名**的 **零知识证明（ZKPoKs）** 的最新进展[^snark-jwt-verify]$^,$[^nozee]$^,$[^bonsay-pay]$^,$[^zk-blind]$^,$[^zklogin]。

更具体地说，在 Aptos 上，我们提供了不需要传统密钥的**无密钥账户**，这些账户通过用户已有的**OIDC账户**（即他们在 Google、GitHub 或 Apple 等**OIDC提供商**拥有的 Web2 账户）来确保安全，而不是通过难以管理的密钥。总的来说，就是 **”你的区块链账户 = 你的 OIDC 账户**“。



无密钥账户的一个重要特性是，它们不仅与用户的 OIDC 账户**绑定**（例如， `alice@gmail.com` ），同时也与一个在 OIDC 提供商注册的**管理应用程序**绑定（例如， dapp 的`dapp.xyz`网站，或者手机的钱包应用）。换句话说，它们是**特定应用程序**的账户。因此，如果一个账户的管理应用程序消失或者丢失了其 OIDC 提供商的注册凭证，那么与这个应用程序绑定的用户账户将会无法访问，除非提供了可供替代的**恢复路径**（下面将进行讨论）。

### 1. 目标

 > 我们的目标是什么，范围包括什么？有什么评估标准吗？
 > 讨论此更改将影响的业务及业务价值

1. **用户友好性：**
   1. 区块链账户应由用户友好的 OIDC 账户支持，这使得它们易于访问（因此不容易失去其访问权）
   2. 允许用户通过他们的 OIDC 账户与 dapps 交互，而无需安装钱包：即，**无钱包体验**
   3. 允许用户从任何设备轻松访问他们的区块链账户
2. **安全性：**
   1. 无密钥账户的安全性应与其底层的 OIDC 账户一样（参见[此处](#4. OIDC 提供商遭到破坏)的讨论）
   2. 即使是负责管理的应用程序不再可用，无密钥账户也应设置有方法进行账户恢复（详见后续章节中的替代恢复方式讨论）。
3. **隐私：**
   1. 无密钥账户及其相关的交易**不应**泄露用户 OIDC 账户的任何信息（例如， Google 用户的电子邮件地址或他们的 OAuth `sub` 标识符）。
   2. OIDC 提供商（例如，Google）不应能够跟踪用户的交易活动。
   3. 对于同一用户但具有不同管理应用的无密钥账户，不应在链上可链接。
4. **效率：** 无密钥账户的交易应该由钱包或者 DApps 高效地创建（< 1秒）并且能被 Aptos 验证器高效地验证（< 2毫秒）。
5. **抗审查性：**  Aptos 验证节点不应根据交易来自哪个应用程序或哪位用户，对 OpenID 交易进行优先处理。
6. **去中心化：** 无需密钥的账户设计，不该依赖于那些本质上无法实现去中心化的中介方。



### 2. 背景

#### 2.1 OAuth and OpenID Connect (OIDC)

我们假设读者熟悉 OAuth 授权框架[^HPL23]和 OIDC 协议[^oidc]：

- OAuth **隐式授权流**和 OAuth **授权码授权流**的安全性。
- OAuth 客户端注册（例如，管理应用的 `client_id` ）
- **JSON Web Tokens (JWTs)**，它包括：
  - JWT **头部段**
  - JWT **有效载荷**，我们通常简称为“**JWT**”。
  - JWT **签名**，对头部段和有效载荷进行签名，我们常将其称为 **OIDC 签名**
  - 我们通常将头部段、有效载荷及其签名的组合称为**已签名的 JWT**。
  - 参见[这里的示例](#2. JWT头和有效载荷样例)。
- 相关的 JWT 头部字段（例如，`kid`）
- 相关的 JWT 有效载荷字段（例如，`aud`，`sub`，`iss`，`email_verified`，`nonce`，`iat`，`exp`）
- **JSON Web Keys (JWKs)**，由每个 OIDC 提供商发布，发布位置为在其 OpenID 配置的 URL 里指定的 JWK 端点 URL。

#### 2.2 术语

- **OIDC 账户**：由 OIDC 提供商（如 Google ）提供的 Web2 账户（例如，`alice@gmail.com`）
- **无密钥账户**：一个区块链账户，其安全性和有效性由 OIDC 账户（例如， Google 账户）支持而不是由密钥支持。本 AIP 的核心是解释如何安全地实现这样的无密钥账户。
- **特定应用的无密钥账户**：无密钥账户同时**绑定**用户的身份（例如， `alice@gmail.com` ）和负责管理的应用（例如， `dapp.xyz` ）。这意味着，为了访问账户，必须提供一个签名的 JWT 令牌，该令牌覆盖该用户的身份和管理应用的身份。这样的令牌只能通过用户通过其OIDC提供商（例如，谷歌）登录管理应用程序来获得。这有[重要意义](#alternative-recovery-paths-for-when-managing-applications-disappear)。
- **针对特定应用的无密钥账户**：无密钥账户不仅与用户的身份（如`alice@gmail.com`）相关联，也与管理该账户的应用程序身份（如`dapp.xyz`）绑定。这意味着，访问这类账户需要展示一个同时包含用户身份和管理应用程序身份认证的已签名 JWT Token。这种 Token 只能通过在用户的 OIDC 提供商（例如，Google）中对管理应用进行登录认证后获得。这种做法对于应用管理无法访问时的账户恢复路径选择具有[重大影响](#2. 管理应用程序消失的备用恢复机制)。



#### 2.3 关于 OIDC 的概览

在本 AIP 中，理解 OIDC 的最重要的内容是：它能使得**管理应用**（比如 `dapp.xyz`、`some-wallet.org` 或是某个手机应用）通过  OIDC 提供商（例如，Google）让用户登录，而无需要获取用户的 OIDC 登录凭证，也就是说，管理应用并不需要知道用户的谷歌账户密码。需要特别指出的是，当且仅当用户成功登录后，管理应用（除此之外，没有其他人）可以收到来自谷歌的**已签名 JWT**，这是一个公开可查询的用户登录凭证。

本 AIP 旨在演示如何使用这个已签名的 JWT，为那些同时与用户和管理应用关联的无密钥账户进行交易授权。



#### 2.4 零知识证明

本节内容假定你对零知识证明（ZKP）系统有一定了解：简而言之，零知识证明系统针对某个**逻辑关系** $R$，使得**证明者**能够在不透露任何有关其**私人信息** $w$ 的前提下，向一个有**公开信息** $x$ 的**验证者**证明自己确实掌握着 $w$，以满足 $R(x; w) = 1$（即证明关系成立）。

其他ZKP术语：

- **证明密钥** 和 **验证密钥**
- **特定关系的可信设置**

#### 2.5 Aptos 账户和交易验证

回忆一下，在 Aptos 中，一个账户由其 **address** 标识，账户的**认证密钥**存储在该地址下。

认证密钥是一种保密承诺，它与账户的**公钥（PK）**（如，公钥的哈希值）之间有加密的联系。 账户的**持有用户**负责管理相对应的**私钥**。

要**授权**访问 Aptos 账户，交易包括：

1. 对（a）账户的**地址**和（b）**交易有效载荷**（例如，Move 入口函数调用）的**数字签名**
2. 在账户 **地址** 下的 **认证密钥** 中提交的 **公钥**

为了**验证交易**是否取得授权访问账户，验证者会执行以下操作：

1. 从交易中提取**公钥**，并依据公钥的类型，推算出相应的**认证密钥**。
2. 验证这个<u>推导</u>的预期**认证密钥**是否等于存储在地址下的**认证密钥**
3. 检查这个**签名**是否在提取的**公钥**上对交易进行了验证。

### 3. 范围

本 AIP 关注：

1. 阐释无密钥账户是如何运作的详细过程。 
2. 描述我们最早期使用 Rust 语言创建的无密钥交易验证系统的概貌。

### 4. 不在讨论范围内的内容

 > 我们无法承诺什么，为什么它们被排除在范围之外？

1. DNS 和 X.509 证书生态系统的安全性
   - 无密钥账户依赖于 OAuth 和 OIDC ，其安全性反过来依赖于 DNS 和 X.509 证书生态系统的安全性。
   - 鉴于所有互联网软件均未脱离上述体系，故此部分不在本范围内。
2. 冒充真正钱包应用的恶意钱包应用
   - 我们假设攻击者**无法**在（比如说）苹果的 App Store 中发布一个冒充钱包注册的 OAuth `client_id` 的手机应用
   - 我们假设攻击者**无法**欺骗用户安装一个冒充另一个应用的 OAuth `client_id` 的恶意桌面应用。因此，我们**不**推荐对桌面应用的无密钥账户进行管理，因为它们非常容易被冒充。
3. 对无密钥账户所需的辅助后端组件的深入讨论：
   - **Pepper 服务**：将是未来AIP的范围（参见[附录](#3. Pepper 服务)）
   - **ZK 证明服务**：将是未来AIP的范围（参见[附录](#4. (预料之外的) 零知识证明服务)）
   - **对 JSON Web Keys (JWKs)的共识**：将是未来AIP的范围（参见[附录](#5. JWK 共识)）
   - 对我们的 Groth16 ZKP 的**可信设置 MPC 仪式**（参见[Groth16讨论](#1. 选择 ZKP 系统：Groth16)）
4. Pepper 和 ZK 证明服务的去中心化计划

## 二、动机

 > 描述这个改动的动机。它实现了什么？

如摘要中所解释的，这个改动实现了两件事：

1. 通过简单地用（比如说）他们的 Google 账户登录到钱包或 dapp ，使用户的使用过程变得更容易。
2. 由于没有涉及到密钥，所以使用户更不容易丢失他们的账户。

 > 如果我们不接受这个提议，可能会发生什么？

如果拒绝这项提议，我们将维持目前依赖秘密密钥进行的、对用户并不友好的账户管理方式。这可能会妨碍人们对 Aptos 网络的接纳，因为许多用户并不理解公钥加密技术的种种细节，例如如何保护他们的助记词、何为私钥以及公钥与私钥的区别在哪里。

## 三、影响

 > 哪些群体会受到这个变化的影响？需要采取什么类型的行动？

1. Dapp 开发者
   - 熟悉无密钥账户的 SDK
   - 如果需要，dapps 可以启用**无钱包体验**，用户可以直接通过他们的 OpenID 账户（例如，他们的 Google 账户）登录到 dapp ，无需连接钱包。
   - 这将让用户访问他们的**特定于 dapp 的区块链账户**。这样，由于这个账户只限于那个特定的 dapp ， dapp 可以代表用户“盲目地”授权交易，而无需复杂的 TXN 提示。
   
2. 钱包开发者
   - 熟悉无密钥账户的 SDK
   - 考虑将他们的默认用户引导流程切换为使用无密钥账户
   
3. 用户
   - 熟悉无密钥账户的安全模型
   - 理解他们的账户将和他们的 OpenID 账户（例如，他们的 Google 账户）一样安全
   - 了解如果管理应用程序无法使用时，如何采用备选的 **恢复方式** 进行操作。
   
   

## 四、替代方案

 > 解释为什么特别提交了这个提议而不是其他替代方案。为什么这是最好的可能结果？

### 1. Passkeys

一个避免用户需要管理自己密钥的替代方案是 **Passkeys** 和围绕它们构建的 **WebAuthn 标准**。Passkey是与应用程序或网站关联的密钥，它在用户的云中备份（例如， Apple 的 iCloud ，Google 的密码管理器等）。

Passkey 密钥最初是作为替代密码的一种方式引入网站的，方法是网站将每个用户的密码替换为他们的 Passkey 公钥。然后，用户可以使用他们对应的 Passkey 私钥进行安全认证，方法是对一个随机的质疑（a random challenge）进行签名，从而向网站验证他们的身份，同时这个私钥被安全地存储在他们的云储存中。

人们的自然倾向是利用 Passkey 密钥来操作区块链账户，方法是把账户的私钥设置成 Passkey 密钥。不幸的是，Passkey 密钥目前存在两个问题，但这些问题很可能在未来得到解决。具体来说，**在一些平台（例如 Microsoft Windows）上，Passkey 密钥并不总是会自动备份到云储存**。此外，Passkey 密钥还带来了一个**跨设备的问题**：比如，用户在他们的苹果手机上创建了一个区块链账户，他们的 Passkey 密钥会被备份至 iCloud，但是这个备份在那位用户的其他安卓、Linux 或 Windows 设备上却无法获取，因为目前还未实现能跨多个平台备份 Passkey 密钥的功能。



### 2. 多方计算 (MPC)

一个避免用户需要管理自己秘密密钥的替代方案是依赖于**多方计算 (MPC)** 服务来为已经被**某种方式**认证的用户计算签名。

通常情况下，用户必须以用户友好的方式向 MPC 进行认证（即，无需管理私钥 ）。除此之外， MPC 系统不会解决任何用户体验问题，因为那毫无用处。结果是，大多数 MPC 系统在代表用户签名前要么使用 OIDC 要么使用通行密钥来认证用户。

这存在两个问题。首先，MPC 系统将学习谁在什么时候进行交易：即，OIDC 账户及其对应的链上账户地址。

其次，由于用户可以直接通过 OIDC 向验证器进行认证（正如本 AIP 所论述的），或通过通行密钥（如[上文](#1. Passkeys)所述），因此 MPC 在本质上是多余的。

换种方式来说，**无密钥账户免去了对复杂的多方计算 (MPC) 签名服务的需求**（而这种服务可能在安全性和稳健性方面难以实现），因为它通过开放身份认证 (OIDC) 直接进行用户认证。同时，无密钥账户在安全性上与基于 MPC 的账户不相上下，因为它们都采用 OIDC 作为启动安全防护的基础。



尽管如此，无密钥账户仍然依赖于一个分布式的 pepper 服务和一个 ZK 证明服务（见[附录](#十三、附录)）。然而：

- 仅仅在进行零知识证明 (ZKP) 计算时，为了提升浏览器或手机上的性能，才需要使用证明服务。随着 ZKP 系统未来的优化，这种服务预计将不再需要。
- Pepper 服务，与MPC服务不同，对于安全性不敏感：即使攻击者完全占据了 Pepper 服务，也无法偷走用户账户；除非他们也破坏了用户的 OIDC 账户。
- 尽管 Pepper 服务在确保系统活跃的方面非常关键，意味着没有 Pepper 服务，用户无法进行任何操作，我们通过采用一个基于简单的可验证随机函数 (VRF) 的设计来实现去中心化，从而解决了这个问题。

### 3. HSMs 或可信硬件

从根本上讲，这种方法遭受的问题与 MPC 方法相同：它必须使用基于 OIDC 或基于 Passkey 的对用户友好的方法，来认证用户到一个可以代表他们签名的外部（基于硬件的）系统。

如上所述，我们的方法直接将用户认证到区块链验证器，无需额外的基础设施，同时不会损失任何安全性。



## 五、规范

 > 我们将如何解决这个问题？详细描述应如何实施这个提案。包括在实施这个特性时应遵循的设计原则。使提案具体到足以让其他人可以在其基础上进行构建，甚至可能派生出竞争的实现。

### 1. 无密钥账户

以下，我们解释了实现无密钥账户背后的关键概念：

1. 无密钥账户的 **公钥** 是什么？
2. 如何从这个公钥派生出 **认证密钥** ？
3. OIDC 账户交易的 **数字签名** 是什么样的？

#### 1.1 公钥

无密钥账户的 **公钥** 包括：

1. $\mathsf{iss\\_val}$：OIDC 提供者的身份，即在 JWT 的 `iss` 字段中显示的内容（比如，`https://accounts.google.com`），我们将其标记为 $\mathsf{iss\\_val}$
2. $\mathsf{addr\\_idc}$：一个**身份承诺（IDC）**，这是一个<u>隐藏</u>承诺，承诺包括：
   - OIDC 提供商发给拥有用户的标识符（例如， `alice@gmail.com` ），由 $\mathsf{uid\\_val}$ 表示。
   - 存储用户标识符的 JWT 字段的名称，由 $\mathsf{uid\\_key}$ 表示。目前，我们只允许 `sub` 或 `email` [^jwt-email-field]。
   - 在与 OIDC 提供商注册期间发给管理应用程序的标识符（即，存储在 JWT 的 `aud` 字段中的 OAuth  `client_id` ），由 $\mathsf{aud\\_val}$ 表示。

稍微正式一点（但忽略复杂的实现细节），IDC 是通过使用一个对 SNARK 友好的哈希函数 $H'$ 对上述字段进行哈希计算得到的：

$$
\mathsf{addr\_idc} = H'(\mathsf{uid\_key}, \mathsf{uid\_val}, \mathsf{aud\_val}; r),\ \text{where}\ r\stackrel{\$}{\gets} \{0,1\}^{256}
$$

#### 1.2 Peppers

请注意，在上述 IDC 的推导中，我们使用了一个具有高熵性质的盲因子 $r$。这保证了 IDC 真正成为一个对用户及其关联应用程序身份信息的隐藏式承诺（hiding commitment）。在整个 AIP 文档中，这一盲因子 $r$ 被称作保护隐私的 **pepper**。

Pepper 有两个重要的属性：

1. 在签署交易以授权访问账户时，将需要知道 pepper $r$。
2. 如果 pepper 被公开，这**不会**允许攻击者获取与该公钥相关联的账户的访问权限。换言之，不同于密钥，pepper 不需要保密以保护账户的安全性；它只需保护账户的隐私性。

更简单地说：

- 如果 **pepper 丢失**，那么对**账户的访问就丢失了**。
- 如果 **pepper 被公开**（例如：被盗），那么只有**账户的隐私丢失了**（即， $\mathsf{addr\\_idc}$ 中的用户和应用身份可以被暴力破解并最终被公开）。

依赖于用户记住他们的 pepper $r$，这将维持易于丢失的基于密钥的账户的现状，从而 **打破了** 基于OpenID的区块链账户的目标*。

因此，我们引入了一个 **pepper 服务**，可以帮助用户恢复他们的 pepper（我们在[附录](#3. Pepper 服务)中讨论了它的属性）。

#### 1.3 认证密钥

接下来，无密钥账户的**认证密钥**就是其上述公钥的哈希。更正式地说，假设任何加密哈希函数 $H$ ，认证密钥为：

$$
\mathsf{auth\_key} = H(\mathsf{iss\_val}, \mathsf{addr\_idc})
$$

**注意**：在实践中，上述的哈希还包括一个域分隔符，但为了简化阐述，我们忽略了这些细节。

#### 1.4 私钥

在定义了上述的“公钥”之后，一个自然的问题出现了：

> 与这个公钥关联的私钥是什么？

答案是，用户没有额外的密钥需要记忆。相反，这个“私钥”，由用户通过上述的 $\mathsf{auth\\_key}$ 中承诺的管理应用程序登录到OIDC账户的能力构成。

换句话说，这个“私钥”可以被认为是用户对那个账户的密码，用户已经知道了，或者一个预安装的 HTTP cookie ，这可以避免用户需要重新输入密码。尽管如此，这个**密码是不够的**：管理应用程序必须是可用的：它必须允许用户登录到他们的OIDC账户并接收OIDC签名。（我们稍后讨论[如何处理消失的应用程序](#2. 管理应用程序消失的备用恢复机制)。）

更正式地说，如果一个用户可以成功地使用由 $\mathsf{aud\\_val}$ 标识的应用程序登录（通过 OAuth ）到由 $(\mathsf{uid\\_key}, \mathsf{uid\\_val})$ 标识并由 $\mathsf{iss\\_val}$ 标识的 OIDC 提供商发出的 OIDC 账户，那么这个能力就作为那个用户的“私钥”。

#### 1.5 热身：泄露用户和应用身份的签名

在描述我们完全保护隐私的 TXN 签名之前，我们先热身一下，描述一下**泄露签名**，它们会泄露用户和应用的身份：即，它们会泄露 $\mathsf{uid\\_key}, \mathsf{uid\\_val}$ 和 $\mathsf{aud\\_val}$ 。

一个地址的交易 $\mathsf{txn}$ 的**泄露签名** $\sigma_\mathsf{txn}$，该地址的认证密钥为$\mathsf{auth\\_key}$，定义如下：

$$
\sigma_\mathsf{txn} = (\mathsf{uid\_key}, \mathsf{jwt}, \mathsf{header}, \mathsf{epk}, \sigma_\mathsf{eph}, \sigma_\mathsf{oidc}, \mathsf{exp\_date}, \rho, r)
$$

其中:

1. $\mathsf{uid\_key}$ 是JWT字段的名称，用于存储用户的身份，其值在地址IDC中被承诺
2. $\mathsf{jwt}$ 是 JWT 的有效载荷（例如，参见这里的示例）
3. $\mathsf{header}$ 是 JWT 头；表示 OIDC 签名方案和 JWK 的密钥 ID ，这些都是在正确的 PK 下验证 OIDC 签名所必需的
4. $\mathsf{epk}$，是由管理应用程序生成的临时公钥（EPK）（其关联的$\mathsf{esk}$在管理应用程序端保密）
5. $\sigma_\mathsf{eph}$ 是对交易$\mathsf{txn}$的临时签名
6. $\sigma_\mathsf{oidc}$ 是对完整 JWT（即，对$\mathsf{header}$和$\mathsf{jwt}$有效载荷）的 OIDC 签名
7. $\mathsf{exp\_date}$ 是一个时间戳，过了这个时间，$\mathsf{epk}$就被认为过期，不能用来签名TXN。
8. $\rho$是一个高熵的 EPK 盲因子，用于创建对 $\mathsf{epk}$ 和 $\mathsf{exp\_date}$ 的 EPK 承诺，该承诺存储在 $\mathsf{jwt}[\texttt{"nonce"}]$ 字段中
9. $r$ 是地址 IDC 的 pepper，假定在这种"泄露模式"下为零

**简而言之**：为了**验证交易签名 $\sigma_\mathsf{txn}$**，验证者需要检查OIDC提供者是否（1）对用户和应用ID在地址IDC中的承诺进行了签名，以及（2）对EPK进行了签名，此EPK又对交易进行了签名，并且对EPK执行了一些过期策略。

更详细地说，针对公钥 $(\mathsf{iss\\_val}, \mathsf{addr\\_idc})$ 的签名验证包括以下步骤：

1. 如果使用基于 `email` 的ID，确保已验证电子邮件：
   1. 如果 $\mathsf{uid\\_key}\stackrel{?}{=}\texttt{"email"}$，则断言  $\mathsf{jwt}[\texttt{"email\_verified"}] \stackrel{?}{=} \texttt{"true"}$
2. 让 $\mathsf{uid\\_val}\gets\mathsf{jwt}[\mathsf{uid\\_key}]$
3. 让 $\mathsf{aud\\_val}\gets\mathsf{jwt}[\texttt{"aud"}]$
4. 断言 $\mathsf{addr\\_idc} \stackrel{?}{=} H'(\mathsf{uid\\_key}, \mathsf{uid\\_val}, \mathsf{aud\\_val}; r)$，其中 $r$ 来自签名的pepper
5. 验证PK是否与链上的认证密钥匹配：
   1. 断言 $\mathsf{auth\\_key} \stackrel{?}{=} H(\mathsf{iss\\_val}, \mathsf{addr\\_idc})$
6. 检查EPK是否在JWT的`nonce`字段中提交：
   1. 断言 $\mathsf{jwt}[\texttt{"nonce"}] \stackrel{?}{=} H’(\mathsf{epk},\mathsf{exp\\_date};\rho)$
7. 检查EPK的过期日期是否过于遥远（我们在下面详细说明）：
   1. 断言 $\mathsf{exp\\_date} < \mathsf{jwt}[\texttt{"iat"}] + \mathsf{max\\_exp\\_horizon}$，其中 $\mathsf{max\\_exp\\_horizon}$ 是一个链上参数
   2. 我们不断言过期日期不在过去（即，断言 $\mathsf{exp\\_date} > \mathsf{jwt}[\texttt{"iat"}]$）。相反，我们假设JWT的发行时间戳（`iat`）字段是正确的，因此接近当前的区块时间。所以，如果应用程序错误地设置了 $\mathsf{exp\\_date} < \mathsf{jwt}[\texttt{"iat"}]$，那么EPK将过期且无用。
8. 检查EPK是否过期：
   1. 断言 $\texttt{current\_block\_time()} < \mathsf{exp\_date}$
9. 验证交易 $\mathsf{txn}$ 下的 $\mathsf{epk}$ 的临时签名 $\sigma_\mathsf{eph}$
10. 获取 OIDC 提供者的正确 PK ，由 $\mathsf{jwk}$ 表示，通过JWT $\mathsf{header}$ 中的 `kid` 字段识别。
11. 验证 JWT $\mathsf{header}$ 和有效载荷 $\mathsf{jwt}$ 下的  $\mathsf{jwk}$ 的OIDC签名 $\sigma_\mathsf{oidc}$ 。

**JWK 共识**：验证 OIDC 签名的最后一步需要验证者**就 OIDC 提供商的最新 JWKs （即公钥）达成共识**，这些公钥在特定时间周期内会更新。公钥在 **OpenID 配置URL**定期进行更新（参见[附录](#5. JWK 共识)）。

如何在所有支持的 OIDC 提供商的 JWKs 上达成 Aptos 验证者的共识将是另一个 AIP 的讨论主题。目前，本 AIP 假设有一个机制存在，验证者能够通过`aptos_framework::jwks` Move 模块获取到当下供应商的 JWKs。

**设定到期日期的必要性：** 我们认为，对于不了解安全性的 dapps 来说，设定一个过于遥远的 $\mathsf{exp\\_date}$ 将带来风险。这会为攻击者提供更长的时机来攻破已签名的 JWT（及其相关的 ESK）。因此，我们要求到期日期不宜设得过远，在 `iat` JWT 字段的基础上根据一个具体的 "最大到期时限" $\mathsf{max\\_exp\\_horizon}$ 来限制：也就是说，我们确保 $\mathsf{exp\\_date} < \mathsf{jwt}[\texttt{"iat"}] + \mathsf{max\\_exp\\_horizon}$。

另一种方法是确保 $\mathsf{exp\_date} < \texttt{current\_block\_time()} + \mathsf{max\_exp\_horizon}$。然而，这并不理想。攻击者可能会创建一个 $\mathsf{exp\\_date}$，在当前时间 $t_1 = \texttt{current\_block\_time()}$ 时无法通过检查（即，$\mathsf{exp\\_date} \ge t_1 + \mathsf{max\\_exp\\_horizon}$），但在时间变为 $t_2 > t_1$ 时能够通过检查（即，$\mathsf{exp\\_date} < t_2 + \mathsf{max\\_exp\\_horizon}$）。因此，这种设计会允许看似无效（因此无害）的已签名JWT后来变得有效（因此值得攻击）。依赖 `iat` 可以避免这个问题。

接下来将要解决是的**关于模式泄露的警告**：

- Pepper $r$ 会因为交易而泄露，使得地址 IDC 暴露于暴力破解的风险之中。
- 类似地，EPK 的盲化因子 $\rho$ 被交易透露，目前还未能实现其保护隐私的预期功能。
- JWT 有效载荷以明文存在，泄漏了管理应用和用户的身份信息。
- 即便 JWT 有效载荷被隐藏，OIDC 签名同样可能会泄露这些身份信息，因为已签名的 JWT 可能具有低熵。由此，攻击者可以对一组可能被签名的 JWT 进行针对性的暴力破解，以验证签名。

#### 1.6 零知识签名

这引出了本 AIP 的核心内容：现在我们准备好解释隐私保护签名如何在我门的无密钥账户中发挥作用。这些签名不会透露用户 OIDC 账户以及与无密钥账户相关联的管理应用的任何信息。

特定的交易 $\mathsf{txn}$ ，该交易针对拥有认证密钥 $\mathsf{auth\\_key}$ 的地址，其**零知识签名** $\sigma_\mathsf{txn}$  定义如下所示：

$$
\sigma_\mathsf{txn} = (\mathsf{header}, \mathsf{epk}, \sigma_\mathsf{eph}, \mathsf{exp\_date}, \mathsf{exp\_horizon}, \pi)
$$

其中：

1. $(\mathsf{header}, \mathsf{epk}, \sigma_\mathsf{eph}, \mathsf{exp\\_date})$ 如之前的描述。
2. $\mathsf{exp\\_horizon}$ 是一个小于等于 $\mathsf{max\\_exp\\_horizon}$ 的值；$\mathsf{exp\\_date}$ 需要位于 $\mathsf{jwt}[\texttt{"iat"}]$ 和 $\mathsf{jwt}[\texttt{"iat"}] + \mathsf{exp\\_horizon}$ 之间。
3. $\pi$ 是一个对于 ZK 关系 $\mathcal{R}$（将在下面定义）的**零知识证明**（ZKPoK）。

请注意，签名中不再包括除了 $\mathsf{iss\\_val}$ 中的 OIDC 提供商身份外的任何能识别用户的信息。

**总结**：为了**验证 $\sigma_\mathsf{txn}$ 这个签名**，验证者需要核实 OIDC 签名的 ZKPoK，其中包括了：(1) 在认证密钥中记录的用户和应用的 ID；以及 (2) 对交易进行签名的 EPK，其还强制了 EPK 上的一些过期约束。

更为具体地，针对公钥 $\mathsf{iss\\_val}, \mathsf{addr\\_idc}$ 执行签名验证包括以下几个步骤：

1. 断定 $\mathsf{auth\\_key}$ 是否等于 $H(\mathsf{iss\\_val}, \mathsf{addr\\_idc})$，这个过程如之前所述。
2. 核查过期时间阈值是否合规：
   1. 断定 $\mathsf{exp\\_horizon}$ 的取值是否在 $(0, \mathsf{max\\_exp\\_horizon})$ 的范围内，这里 $\mathsf{max\\_exp\\_horizon}$ 是链上预设的参数，如前所述。
3. 校验 EPK 是否已过期：
   1. 断定当前块时间 $\mathsf{exp\\_date}$ 是否小于 $\texttt{current\_block\_time()}$ ，如之前所述。
4. 确认 $\mathsf{epk}$ 对交易 $\mathsf{txn}$ 的短暂签名 $\sigma_\mathsf{eph}$ 的有效性，这依照之前的描述。
5. 确定 OIDC 提供商的确切公钥，这通过 $\mathsf{jwk}$ 表示，按前文所述。
6. 校验零知识证明 $\pi$，它声明了存在一个*私钥输入* $`\textbf{w}=[(\mathsf{aud\\_val}, \mathsf{uid\\_key}, \mathsf{uid\\_val}, r),(\sigma_\mathsf{oidc}, \mathsf{jwt}), \rho]`$，对应于*公共输入* $\textbf{x} = [(\mathsf{iss\\_val}, \mathsf{jwk}, \mathsf{header}), (\mathsf{epk}, \mathsf{exp\\_date}), \mathsf{addr\\_idc}, \mathsf{exp\\_horizon}]$，可满足关系 $\mathcal{R}(\textbf{x}; \textbf{w})=1$
   - 需要重视的是，证明 $\pi$ 不会泄漏任何敏感的私钥输入 $`\textbf{w}`$。

**零知识关系 $\mathcal{R}$** 旨在验证上文提到的模式泄露的隐私敏感部分：

具体而言，当满足以下关系时，即认为验证通过：

$$
\mathcal{R}\begin{pmatrix}
    \textbf{x} = [
        (\mathsf{iss\_val}, \mathsf{jwk}, \mathsf{header}), 
        (\mathsf{epk}, \mathsf{exp\_date}), 
        \mathsf{addr\_idc}, \mathsf{exp\_horizon}
    ],\\ 
    \textbf{w} = [
        (\mathsf{aud\_val}, \mathsf{uid\_key}, \mathsf{uid\_val}, r),
        (\sigma_\mathsf{oidc}, \mathsf{jwt}), 
        \rho]
\end{pmatrix} = 1
$$

当且仅当：

1. 验证 JWT 中 OIDC 提供商的 ID：
   1. 确认 $\mathsf{iss\\_val}$ 是否与 $\mathsf{jwt}[\texttt{"iss"}]$ 相等。
2. 如果采用基于 `email` 的 ID，确保邮箱已经得到验证：
   1. 如果 $\mathsf{uid\\_key}$ 等于 $\texttt{"email"}$，则确认 $\mathsf{jwt}[\texttt{"email\_verified"}]$ 值为 $\texttt{"true"}$。
3. 核对 JWT 中的用户 ID 是否相符：
   1. 确认 $\mathsf{uid\\_val}$ 是否与 $\mathsf{jwt}[\mathsf{uid\\_key}]$ 相等。
4. 核实 JWT 中管理该应用的 ID 是否正确：
   1. 确认 $\mathsf{aud\\_val}$ 是否与 $\mathsf{jwt}[\texttt{"aud"}]$ 相符。
5. 核查地址 IDC 是否使用了 JWT 中的适当值：
   1. 确认 $\mathsf{addr\\_idc}$ 是否等于 $H'(\mathsf{uid\\_key}, \mathsf{uid\\_val}, \mathsf{aud\\_val}; r)$
6. 校验 EPK 是否已在 JWT 的 `nonce` 字段中提交：
   1. 确认 $\mathsf{jwt}[\texttt{"nonce"}]$ 是否等于 $H’(\mathsf{epk},\mathsf{exp\\_date};\rho)$
7. 校验 EPK 的过期日期是否未超出预期：
   1. 确认 $\mathsf{exp\\_date}$ 是否小于 $\mathsf{jwt}[\texttt{"iat"}] + \mathsf{exp\\_horizon}$
8. 验证 OIDC 签名 $\sigma_\mathsf{oidc}$ 是在公钥 $\mathsf{jwk}$ 下对 JWT 的 $\mathsf{header}$ 和载荷 $\mathsf{jwt}$ 进行的。

**注意**：附加的 $\mathsf{exp\\_horizon}$ 变量作为一个中间层存在：它确保即使链上的 $\mathsf{max\\_exp\\_horizon}$ 参数发生了变化，零知识证明（ZKP）依然有效，因为它们把 $\mathsf{exp\\_horizon}$ 作为公共输入，而不是 $\mathsf{max\\_exp\\_horizon}$。

我们稍后将讨论的**零知识模式的考虑因素**包括：

1. 用户/钱包通过 pepper 服务获得 pepper $r$，我们会在[附录](#pepper-service)中对此服务进行简要说明。
2. **计算零知识证明(ZKP)过程缓慢**。这需要专门的证明生成服务，在[附录](#(Oblivious)-ZK-proving-service)中有所简述。
3. **在实施 ZK 关系时出现的错误**可以通过使用[“训练轮”模式](#training-wheels)来纠正。

## 六、参考实现

> 这部分虽然是可选的，但非常建议，您可以在此处加入您想要在此提案中演示的示例。无论是代码、图表还是纯文本都可。理想情况下，应提供一个链接到演示这一标准的活跃代码库；对于更简单的情况，可以是内联代码。

### 1. 选择 ZKP 系统：Groth16

我们的初期部署将使用 Groth16 零知识证明系统[^groth16]，在 BN254[^bn254] 椭圆曲线上进行。选择 Groth16 主要基于以下考虑：

1. 具备**极小化的证明尺寸**（在所有 ZKP 系统中为最小）
2. 验证时间**恒定**
3. 证明生成速度**快**（在 Macbook Pro M2 上运用多线程技术可达到 3.5 秒）
4. 在 `circom`[^circom] 中实现我们所需的关系相对简单
5. `circom` 具备成熟的生态工具支持
6. 可以创建**防伪造**的证明，这对于我们的交易签名的真实性至关重要。

不幸的是，Groth16 属于具有针对性预设的可信设置的前置 SNARK 类型。这意味着，对于我们的 ZK 关系 $\mathcal{R}$，我们需要组织一次**基于多方计算（MPC）的可信设置典礼**，这超出了当前 AIP 文档的范畴。

由此产生的问题是，如果不进行重新设置，我们无法升级或修复我们的零知识证明关系实现。所以，在未来，我们可能会迁移到一个**透明型**SNARK，或者采用与特定关系无关并且一次性的**通用可信设置**。

### 2. 参考实现部分

我们将在下方提供有关我们电路代码和 Rust 交易验证器的更多链接：
 - [Rust 验证器代码，用于处理泄露模式](https://github.com/aptos-labs/aptos-core/pull/11681)
 - [基于 Groth16 的零知识证明模式 Rust 验证器代码](https://github.com/aptos-labs/aptos-core/pull/11772)

## 七、测试（可选项）

> - 计划中包含哪些测试内容？（除了性能测试外，所有测试均应作为实施细节的一部分而进行，并不需要特别强调）
> - 我们可以期待在何时获得测试结果？
> - 测试结果如何，是否达到了我们的预期？如果存在差异，请解释原因。

### 1. 签名验证的正确性与完整性测试

- 受支持的提供商有效的 [身份验证知识证明(ZKPoKs) of] OIDC 签名将通过验证
- 受支持的提供商过期的 [身份验证知识证明(ZKPoKs) of] OIDC 签名将不被验证
- 不受支持的提供商的签名不会通过验证
- 没有临时签名的 [身份验证知识证明(ZKPoKs) of] OIDC 签名将验证失败
- ZKP 是防伪造的

### 2. 其他测试内容

- 能够快速验证包含大量 OIDC 账户的交易

我们将来会对测试计划进行扩展。（**待定**）

## 八、风险与缺点

> - 在此处列举该提案可能带来的潜在负面后果。存在哪些风险？
> - 我们应当关注哪些向后兼容的问题？
> - 如遇到问题，我们将如何减轻或解决？

所有相关风险与缺陷均在[“安全性、活跃性与隐私考量”部分](#Security-Liveness-and-Privacy-Considerations)进行了阐述。

## 九、未来发展潜力

> 对该提案未来的发展进行深思熟虑。您如何预见其未来的成果？一年后该提案将产生怎样的影响？五年后呢？

总体而言，这一方法通过移除与助记词及密钥管理相关的障碍，有助于吸引下一个亿级用户群体进入市场。

在一年之后，这项提案可能促成一个全新的去中心化应用（dapp）生态系统，其中的应用允许用户通过他们的（例如）Google 账号轻松接入，无需手动连接钱包。

钱包生态同样可能因为支持用户无需记下助记词便能接入而实现增长。

同时，这还可能使得 Aptos 网络上的钱包更加安全，因为这样的钱包将不再需要为用户保管长期密钥（或依赖于复杂的多方计算 MPC 或者硬件安全模块 HSM 系统来代替用户执行此项任务）。

## 十、时间线

### 1. 推荐的实施时间线

> 预计需要花费多少时间来完成实施，可以将其分为几个阶段或里程碑进行描述。

鉴于多个组件的相互作用使得逐步开发变得复杂，以下是潜在的里程碑细分：

1. 在 `circom`[^circom] 中实现零知识证明关系。
2. RUST 交易验证器的开发。
3. 中心化的 pepper 服务部署。
4. 中心化的证明服务部署，包括[“辅助轮训”](#training-wheels)。
5. 基于 Aptos 验证节点的 JWK 共识机制。
6. 可信多方计算（MPC）设置仪式的组织。

### 2. 推荐的开发者平台支撑时间线

> 叙述对于此功能的 SDK、API、CLI、索引器等支撑工具的规划安排。

目前，我们正在进行 SDK 的开发和支持。其余工具将在未来进行补充。（**待确定**）

### 3. 推荐的部署时间线

> 提供一个将来的版本发布作为社区预期何时能在我们的三个网络中看到此次部署的*粗略*估计（例如，1.7版发布）。如果 AIP 在设计审查阶段被接受，你需要更新此 AIP 来提供更为精确的时间估计。
>
> - 在 devnet 网络上？
> - 在 testnet 网络上？
> - 在 mainnet 网络上？

计划于二月在 devnet 网络进行部署。

计划于三月在 mainnet 网络进行部署。

计划在 testnet 网络上在接下来的时间内部署。（**具体时间待定**）

## 十一、安全性、活跃性与隐私考量

> - 这是否会改变我们对于安全的基本假设或威胁模型？
> - 是否存在潜在的诈骗行为？相应的缓解措施为何？
> - 是否需考虑其他的安全隐患？
> - 是否有相关的安全设计文档或审计报告可供参考？

### 1. 辅助轮

在我们最初的部署期间，我们打算在证明服务计算证明前，检验其关系准确无误之后为零知识证明（ZKP）添加一个额外的**辅助轮签名**步骤。通过这种方式，我们既可以避免单点的故障风险（即零知识证明关系的实现），同时也不会赋予证明服务额外的权力，以致于干涉账户安全。

具体而言，当启用辅助轮系统时，即便我们在 Circom[^circom] 中实现的零知识证明关系出现漏洞，这也不会导致用户面临灾难性的资产损失。同时，被损害的证明服务本身也无法盗取资产：它必须像其他任何人一样计算（或伪造）一个有效的 ZKP 来攻击受害账户。

其中一个重要的**存活性问题**是，如果证明服务停止运作，用户可能暂时无法访问他们的账户。但是，我们预期此类停机会非常短暂，并且相对于我们初期部署时确保安全性的目标来说，这是一个可以接受的权衡。

### 2. 管理应用程序消失的备用恢复机制

再次说明，无密钥账户不仅与用户绑定，同时还与管理应用（比如dapp或钱包）绑定。

不幸情况下，管理应用有可能会消失，原因多样：

1. 应用的 OAuth `client_id` 可能被 OIDC 提供商封禁（出于各种原因）。
2. 一位疏忽的管理员可能会丢失应用的 OAuth `client_secret`，或是在 OIDC 提供商那里彻底注销了应用。
3. 一位不知情的管理员可能简单地关停了应用（例如，从应用商店中撤下了移动应用，或停止了dapp网站的运作），而没意识到这会导致用户无法再使用应用内的无密钥账户。

无论发生什么，如果**管理应用不再存在，用户将无法访问该应用绑定的无密钥账户**，因为缺少应用，他们无法获得签名的 JWT。

为防止这种情况，我们建议设立一个**备用恢复机制**：

- 对于以**电子邮件**为基础的 OIDC 提供商，可以采用一个 DKIM 签名的电子邮件（内容格式正确的消息）中的 ZKPoK 来取代 OIDC 签名的 ZKPoK。
  - 尽管这可能用户体验不佳，但它提供了一种紧急情况下的恢复方式，还能**保护隐私**。
  - 确保此流程**难以被仿冒**是关键，因为用户可能受到攻击者的诱骗而发送所谓的“账户重置”邮件。
  - 例如，只有在账户长时间未活动时才启用此流程。
  - 或者，此流程可能需要用户与另一个去中心化的验证方沟通，由该验证方适当引导用户完成账户重置流程。
- 对于**非基于电子邮件**的 OIDC 提供商，备用恢复路径可能会因提供商而异：
  - 对于**Twitter**，用户可以通过发布一条特定格式的消息来证明他们拥有自己的 Twitter 账户。
  - 同理，对于**GitHub**，用户可以通过发布一个gist来完成同样的验证。
  - 如果有一个 HTTPS Oracle，验证者可以验证上述 tweet 或 gist，并允许用户进行账户密钥的更换。由于至少验证者可以通过 HTTPS URL 了解用户的身份，这样的操作**不会保护隐私**。
- 另一种方式是，所有无密钥账户都设计为 1 out of 2[^multiauth]模式，并设置一个**恢复用[subaccount](#Passkeys)**。如果 Passkey 进行了自动备份（例如在Apple平台上），那么它可以用来恢复对账户的访问。
  - 也可以选用传统的 SK 子账户，但这需要管理应用提供给用户一个记录SK的选项，这与提升用户体验的目标相悖。
  
- 或者，对于受欢迎的应用，我们可以考虑进行**手动补丁**：例如，为一些特定`client_id`直接在我们的零知识证明关系中添加例外。但这样就引入了集中化的风险。

### 3. OIDC 账户遭到破坏

无密钥账户的核心理念是确保用户的区块链账户**与他们的 OIDC 账户同等安全**。理所当然的是，如果 OIDC 账户（比如 Google）遭到黑客入侵，所有跟用户的 OIDC 账户相关联的无密钥账户同样面临风险。

实际上，即便 OIDC 账户**暂时**被入侵，无密钥账户也可能会**永久**受到损害。因为攻击者可以在短时间内获取到 OIDC 账户的权限，并轻易地更换账户密钥。

尽管如此，这正是我们的设计预期：*“你的区块链账户等同于你的 Google 账户！”*

不喜欢这个设计的用户可以选择（1）不使用此项功能，或者（2）根据个人的选择使用 `t`-out-of-`n` 多因素认证方法[^multiauth]，其中之一可以是无密钥账户。

**注意：**Pepper 服务同样采用 OIDC 认证请求，因此它不能作为二次验证的有效因素；除非破坏了无密钥账户的初衷，即免去了用户需要记忆任何信息的必要。

### 4. OIDC 提供商遭到破坏

再次强调，_“你的区块链账户即是你的 OIDC 账户。”_ 换句话说：

- 如果你的 OIDC 账户受到破坏，你的无密钥账户同样受影响。
- 如果你的 OIDC 提供商（例如 Google）被入侵，所有与该提供商相关联的无密钥账户也处于风险之中。

我们强调这是一项**特性**而非**缺陷**：我们希望Aptos用户借助他们的 OIDC 账户的安全性和便利性来进行无缝交易。然而，对于那些不想完全依赖他们的 OIDC 账户安全性的用户，他们可以升级到$t$ out of $n$ 认证方式，采用不同的 OIDC 账户和/或传统的 SKs[^multiauth]相结合。

**注意**：我们能够通过紧急治理建议来撤销受到破坏的OIDC提供商，屏蔽他们的 JWK。

### 5. OIDC 提供商更改算法为不受支持的签名算法

我们目前实现的零知识证明关系只支持 RSA 签名验证，这是基于其在OIDC提供商中的广泛使用。但如果提供商突然转换到另一种算法（比如，Ed25519），这将导致用户无法生成 ZKPs ，从而无法访问他们的账户。

针对这种不太可能的情况，我们可能需要采取以下措施：

1. 首先，可以启用一个**透露模式**的功能标志。这将在牺牲隐私的情况下恢复对用户账户的访问，虽然不理想，但至少能保证大部分用户资产的安全。
2. 其次，我们会升级零知识证明关系以支持新的签名方案，并部署到生产环境中。

### 6. 基于Web的钱包与无需钱包的Dapps

无密钥账户的出现将促成（1）基于Web的钱包，即用户可通过如Google这样的服务登录，以及（2）允许用户无需传统钱包即可直接通过如Google登录的**无钱包Dapps**。

在Web环境中存在着一些安全挑战：

- 保证 JavaScript 依赖未被篡改，因为这可能导致ESKs、签名的JWTs或两者同时被盗取。
- 在浏览器中管理用户账户的ESKs可能存在风险，这些密钥由于JavaScript的缓存或错误存储而有泄漏的可能性。
  - 值得庆幸的是，使用 Web 加密 API 和/或以 Passkey 代替 ESK 可以有效防止这类问题。

### 7. 针对 SNARK 友好的哈希函数

我们为了确保零知识证明系统（ZKP）的高效运行，使用了一种名为 *Poseidon*[^poseidon]的对SNARK友好的哈希函数[^snark-hash]。然而Poseidon相对较新，并未像SHA2-256等历史较长的哈希函数接受大量密码学分析。

若Poseidon哈希函数存在碰撞漏洞，攻击者可能改动用户或应用ID的已提交值，且在不攻破零知识证明系统或OIDC提供商的前提下挪用资金。

长期看来，我们计划淘汰对SNARK友好的哈希函数，迁移到即便是与对SNARK不友好的哈希函数（如SHA2-256）一同使用，我们的zkSNARK证明系统也能维持高效率的选择。

## 十二、开放性问题（选读）

> 这部分包含了一系列问答，其中部分问题可能已有答案，而另一些问题尚在探讨阶段，我们应当予以明确。

### 1. 安全备用恢复路径的OIDC提供商集合

部署时的一个关键问题是，有哪些OIDC提供商承认安全的备用恢复路径（比如，难以被仿冒的流程）？

### 2. Web开发者应默认在何处存储ESK及签名的JWT？

对于移动应用来说，存储ESK并非问题：所有数据都储存在应用自身的存储中。

但对于Web应用方面，即dapps或Web钱包，存在两个选择：

1. 采用OAuth隐式授权流程，并将所有数据（如ESK和签名的JWT）存于浏览器（比如，本地存储，IndexedDB）。
   - 这种做法的好处是，即便dapp/Web钱包遭受攻击，只要用户不访问被攻击的网站，触发恶意JavaScript，那么资产便安全无虞。
2. 采用OAuth代码授权流程，浏览器存储ESK，而签名的JWT则存储在后端。
   - 不幸的是，一些OIDC提供商允许在后端使用新的`nonce`字段刷新签名的JWT，而无需用户同意。这意味着一旦后端受到攻击，即便是OAuth会话未过期，攻击者也可能在不需要用户访问网站的情况下刷新JWT，并窃取无密钥账户。

### 3. 应设置的最大过期时间阈值  $\mathsf{max\\_exp\\_horizon}$  是多少？

回忆一下， $\mathsf{exp\\_date}$ 的值不应超过 $\mathsf{jwt}[\texttt{"iat"}] + \mathsf{max\\_exp\\_horizon}$ 。

限制因素：

- 时限设定不宜过短：这样会导致用户需要频繁地刷新签名JWTs，进而频繁地重新生成ZKPs，造成不便。

- 时限也不能过长：否则会存在一个较大的风险窗口，签名JWTs和对应的已提交ESK可能在此期间遭到非法盗用。

### 4. 零知识交易仍然会暴露OIDC提供商身份信息

在当前的零知识交易（TXNs）中，通过透露 $\mathsf{iss\_val}$ 字段（例如Google, GitHub）可以知道交易中涉及哪个OIDC提供商。

不过，我们能够修改我们的零知识证明关系，以掩藏OIDC提供商的身份。更具体地，而不是把 $\mathsf{jwk}$ 作为一个*公开的*输入，我们可以将其视为一个*私密的*输入，并确认这个JWK是否在链上的已授权JWKs表中。这是必要的，因为，如果不这样做，用户可以随意输入任何JWK并伪造交易签名。

## 十三、附录

### 1. OpenID连接（OIDC）

[ OIDC规范](https://openid.net/specs/openid-connect-core-1_0.html) 有如下规定：

- 必须用 UNIX 宇宙时间秒来指定`iat`字段（[详情参见](https://openid.net/specs/openid-connect-core-1_0.html#IDToken)）。
- 刷新token的协议**不允许**设定新的nonce值（[详情参见]([url](https://openid.net/specs/openid-connect-core-1_0.html#RefreshTokenResponse))）。然而，基于我们的使用经验，如 Google 这样的提供者实际上确实允许在新的`nonce`上进行OIDC签名的刷新。

### 2. JWT头和有效载荷样例

以下是来自 Google OAuth操场[^oauth-playground]的JWT样例。

JWT 头部段：

```json
{
  "alg": "RS256",
  "kid": "822838c1c8bf9edcf1f5050662e54bcb1adb5b5f",
  "typ": "JWT"
}
```

JWT 有效载荷:

```
{
  "iss": "https://accounts.google.com",
  "azp": "407408718192.apps.googleusercontent.com",
  "aud": "407408718192.apps.googleusercontent.com",
  "sub": "103456789123450987654",
  "email": "alice@gmail.com",
  "email_verified": true,
  "at_hash": "a5z9bu-5jokhN3pmxj2kMg",
  "iat": 1684349149,
  "exp": 1684352749
}
```

### 3. Pepper 服务

Pepper 服务用于在用户丢失他们账户的 pepper 时进行恢复，如若失去 pepper，账户也将随之无法找回。

该服务的具体设计细节不在本 AIP 文稿范围之内，但我们在此强调其几个关键特性：

- 服务需要在透露用户 pepper 前，**验证用户**的身份，这一点与区块链验证者的操作过程极为相似。
  - 验证过程会利用和创建交易签名时相同的OIDC签名来完成。

- 服务采用**可验证随机函数(VRF)**来生成 peppers。
- 它的设计支持轻松**去中心化**，无论是作为一个独立的系统存在，还是作为 Aptos 验证器的上层服务。
- Pepper 服务是**隐私保护型**的：它既不会知悉请求 pepper 的用户身份，也不会知道为该用户生成的具体 pepper 值。
- Pepper 服务主要是“**无状态的**”，意味它仅存储那些用于派生用户 peppers 的 VRF 密钥。

### 4. (预料之外的) 零知识证明服务

当浏览器和移动设备上计算零知识证明（ZKP）的成本仍然较高时，**零知识证明服务**将协助用户快速计算他们的 ZKPs。

该服务的具体设计细节不在本 AIP 文稿范围之内，但我们在此突出其几个关键特性：

- 即使服务停止，用户依然可以通过慢速方式访问他们的账户，因为他们总能自行计算 ZKPs。
  - 除非证明服务正在执行“新手保护模式”（参见[下文](#training-wheels)）。

- 该服务应当是**去中心化的**，无论是在被授权的环境中，还是无需授权的情况下。
- 它应当具备抵御**拒绝服务（DoS）攻击**的弹性。
- 它应当是**无知的**或**保护隐私的**：即不会理解零知识证明中的任何私有输入数据（例如，身份匿名的请求者，或他们的pepper等）。
- 证明服务主要是“**无状态的**”，仅存储那些计算我们的零知识证明关系 $\mathcal{R}$ 的公开证明密钥。

### 5. JWK 共识

不使用密钥的账户在执行交易签名时，涉及到验证 OIDC 签名。这要求验证者**同意 OIDC 提供商的最新 JWKs**（即公钥），这些公钥会在特定提供商的**OpenID 配置 URL** 上定期更新。

虽然 JWK 共识的设计和实践超出了本 AIP 的讨论范围，但我们在这里还是强调一些核心属性：

- 验证者需要定期在每个支持的提供商的**OpenID 配置 URL** 上检查 JWKs 的变动。
- 当一个验证者探测到变动时，他们会通过一种一次性共识机制来提议这些变化。
- 一旦验证者之间达成共识，新的 JWKs 将在 `aptos_framework::jwks` 的公共 Move 模块中反映出来。

## 十四、更新日志

- _2024-02-29_：本 AIP 的[早期版本](https://github.com/aptos-foundation/AIPs/blob/71cc264cc249faf4ac23a3f441fb76a64278b51a/aips/aip-61.md)将无密钥账户称为**基于 OpenID 的区块链 (OIDB)** 账户。

## 十五、引用



[^bn254]: [bn254](https://hackmd.io/@jpw/bn254)
[^bonsay-pay]: [bonsay-pay](https://www.risczero.com/news/bonsai-pay)
[^circom]: [circom](https://docs.circom.io/circom-language/signals/)
[^groth16]: **关于基于配对的非交互式论证的大小**, Jens Groth撰写，收录于《密码学进展 -- EUROCRYPT 2016》，2016年
[^HPL23]: **OAuth 2.1授权框架**, Dick Hardt、Aaron Parecki和Torsten Lodderstedt撰写，2023年，[[URL]](https://datatracker.ietf.org/doc/draft-ietf-oauth-v2-1/08/)
[^jwt-email-field]: 仅允许OIDC提供商使用`email`字段，并且该字段的值永远不会更改（例如，像Google这样的电子邮件服务），因为这可以确保用户不会因为更改Web2账户的电子邮件地址而意外锁定自己的区块链账户。
[^multiauth]: 请参阅[AIP-55：通用交易认证和支持任意K-of-N多密钥账户](https://github.com/aptos-foundation/AIPs/blob/main/aips/aip-55.md)
[^multisig]: 有替代方案可以替代单个SK账户，例如基于多签名的账户（例如通过[MultiEd25519](https://github.com/aptos-labs/aptos-core/blob/main/aptos-move/framework/aptos-stdlib/sources/cryptography/multi_ed25519.move)或通过[AIP-55](https://github.com/aptos-foundation/AIPs/blob/main/aips/aip-55.md)），但它们仍然归结为一个或多个用户保护其秘密密钥免受丢失或盗窃。
[^nozee]: [nozee](https://github.com/sehyunc/nozee)
[^oauth-playground]: [OAuth Playground](https://developers.google.com/oauthplayground/)
[^openpubkey]: **OpenPubkey：利用用户持有的签名密钥增强OpenID Connect**，由Ethan Heilman、Lucie Mugnier、Athanasios Filippidis、Sharon Goldberg、Sebastien Lipman、Yuval Marcus、Mike Milano、Sidhartha Premkumar和Chad Unrein撰写，*收录于密码学预印本，论文2023/296*，2023年，[[URL]](https://eprint.iacr.org/2023/296)
[^poseidon]: **Poseidon：零知识证明系统的新哈希函数**，由Lorenzo Grassi、Dmitry Khovratovich、Christian Rechberger、Arnab Roy和Markus Schofnegger撰写，*收录于USENIX Security’21*，2021年，[[URL]](https://www.usenix.org/conference/usenixsecurity21/presentation/grassi)
[^snark-hash]: [如何选择适合零知识证明的哈希函数](https://www.taceo.io/2023/10/10/how-to-choose-your-zk-friendly-hash-function/)
[^snark-jwt-verify]: [snark-jwt-verify](https://github.com/TheFrozenFire/snark-jwt-verify/tree/master)
[^zk-blind]: [zk-blind](https://github.com/emmaguo13/zk-blind)
[^zklogin]: [zklogin](https://docs.sui.io/concepts/cryptography/zklogin)

```original
[^bn254]: https://hackmd.io/@jpw/bn254
[^bonsay-pay]: https://www.risczero.com/news/bonsai-pay
[^circom]: https://docs.circom.io/circom-language/signals/
[^groth16]: **On the Size of Pairing-Based Non-interactive Arguments**, by Groth, Jens, *in Advances in Cryptology -- EUROCRYPT 2016*, 2016
[^HPL23]: **The OAuth 2.1 Authorization Framework**, by Dick Hardt and Aaron Parecki and Torsten Lodderstedt, 2023, [[URL]](https://datatracker.ietf.org/doc/draft-ietf-oauth-v2-1/08/)
[^jwt-email-field]: Use of the `email` field will be restricted to OIDC providers that never change its value (e.g., email services like Google), since that ensures users cannot accidentally lock themselves out of their blockchain accounts by changing their Web2 account’s email address.
[^multiauth]: See [AIP-55: Generalize Transaction Authentication and Support Arbitrary K-of-N MultiKey Accounts](https://github.com/aptos-foundation/AIPs/blob/main/aips/aip-55.md)
[^multisig]: There are alternatives to single-SK accounts, such as multisignature-based accounts (e.g., via [MultiEd25519](https://github.com/aptos-labs/aptos-core/blob/main/aptos-move/framework/aptos-stdlib/sources/cryptography/multi_ed25519.move) or via [AIP-55](https://github.com/aptos-foundation/AIPs/blob/main/aips/aip-55.md)), but they still bottle down to one or more users protecting their secret keys from loss or theft.
[^nozee]: https://github.com/sehyunc/nozee
[^oauth-playground]: https://developers.google.com/oauthplayground/
[^openpubkey]: **OpenPubkey: Augmenting OpenID Connect with User held Signing Keys**, by Ethan Heilman and Lucie Mugnier and Athanasios Filippidis and Sharon Goldberg and Sebastien Lipman and Yuval Marcus and Mike Milano and Sidhartha Premkumar and Chad Unrein, *in Cryptology ePrint Archive, Paper 2023/296*, 2023, [[URL]](https://eprint.iacr.org/2023/296)
[^poseidon]: **Poseidon: A New Hash Function for Zero-Knowledge Proof Systems**, by Lorenzo Grassi and Dmitry Khovratovich and Christian Rechberger and Arnab Roy and Markus Schofnegger, *in USENIX Security’21*, 2021, [[URL]](https://www.usenix.org/conference/usenixsecurity21/presentation/grassi)
[^snark-hash]: https://www.taceo.io/2023/10/10/how-to-choose-your-zk-friendly-hash-function/
[^snark-jwt-verify]: https://github.com/TheFrozenFire/snark-jwt-verify/tree/master
[^zk-blind]: https://github.com/emmaguo13/zk-blind
[^zklogin]: https://docs.sui.io/concepts/cryptography/zklogin
```
