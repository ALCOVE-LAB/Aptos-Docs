---
aip: 61
title: 无秘钥账户
author:
  name: Alin Tomescu
  email: alin@aptoslabs.com
discussions-to: https://github.com/aptos-foundation/AIPs/issues/297
status: 已接受
last-call-end-date: '2024-02-15'
type: "Standard (Core, Framework)"
created: '2024-01-04'
requires: "<AIP number(s)>"
original:
  en: https://github.com/aptos-foundation/AIPs/blob/main/aips/aip-61.md
  zh: https://github.com/ALCOVE-LAB/Aptos-Docs/blob/main/AIP/aip-61.md
---

[TOC]

# AIP-61 - 无秘钥账户

## 一、摘要

 >  用3~5句话总结我们要解决的问题是什么，我们是如何解决的

目前，唯一确保 Aptos 账户安全的方法是妥善保管与之相连的**私钥（secret key，SK）**[^multisig]。然而，这个任务远远不如听起来那么简单。实际情况是，用户很容易遗失私钥（比如，在第一次设置 Aptos 钱包时忘记记录助记词）或者私钥被盗用（例如，被骗子诱导泄漏了自己的私钥）。这样一来，让用户开始使用的过程变得异常困难，并且当他们的账户丢失或被盗时，还会导致用户流失。

在这份 AIP 中，我们描述了一种更用户友好的账户管理方法，该方法依赖于：

1. **未修改的**[^openpubkey] **OpenID Connect （OIDC）**标准，
2. 其区块链应用[^openzeppelin],[^eip-7522],[^zkaa]，
3. **零知识证明知识（ZKPoKs）** 在 **OIDC 签名**[^snark-jwt-verify]，[^nozee]，[^bonsay-pay],[^zk-blind]，[^zklogin]的最新进展。

更具体地说，在 Aptos 上，我们提供了不需要传统私钥的**无密钥账户**，这些账户通过用户已有的 **OIDC 账户**（即他们在 Google、GitHub 或 Apple 等 **OIDC 提供商**拥有的 Web2 账户）来确保安全，而不是通过难以管理的私钥去确保你的账户安全。总的来说，就是 **”你的区块链账户 = 你的 OIDC 账户**“。

> [!WARNING]
> 无密钥账户的一个重要特性是，它们不仅与用户的 OIDC 账户**绑定**（例如， `alice@gmail.com` ），同时也与一个在 OIDC 提供商注册的**管理应用程序**绑定（例如， dapp 的`dapp.xyz`网站，或者手机的钱包应用）。换句话说，它们是**特定应用程序**的账户。因此，如果一个账户的管理应用程序消失或者丢失了其 OIDC 提供商的注册凭证，那么与这个应用程序绑定的用户账户将会无法访问，除非提供了可供替代的**恢复路径**（下面将进行讨论）。



### 1. 目标

1. **用户友好性：**
   1. 区块链账户应由用户友好的 OIDC 账户支持，这使得它们易于访问（因此不容易失去其访问权）
   2. 允许用户通过他们的 OIDC 账户与 dapps 交互，而无需安装钱包：即，**无钱包体验**
   3. 允许用户从任何设备轻松访问他们的区块链账户
2. **安全性：**
   1. 无密钥账户的安全性应与其底层的 OIDC 账户一样（参见[此处](##4-oidc-账户遭到破坏)的讨论）
   2. 即使是负责管理的应用程序不再可用，无密钥账户也应设置有方法进行账户恢复（详见后续章节中的替代恢复方式讨论）。
3. **隐私：**
   1. 无密钥账户及其相关的交易**不应** 泄漏用户 OIDC 账户的任何信息（例如， Google 用户的电子邮件地址或他们的 OAuth `sub` 标识符）。
   2. OIDC 提供商（例如，Google）不应能够跟踪用户的交易活动。
   3. 对于同一用户但具有不同管理应用的无密钥账户，在区块链上是不可链接的。
4. **效率：** 无密钥账户的交易应该由钱包或者 DApps 高效地创建（$<1$秒）并且能被 Aptos 验证器高效地验证（$< 2$毫秒）。
5. **抗审查性：**  Aptos 验证节点不应根据交易来自哪个应用程序或哪位用户，对 OpenID 交易进行优先处理。
6. **去中心化：** 无密钥账户设计，不该依赖于那些本质上无法实现去中心化的中介方。



### 2. 背景

#### 2.1 OAuth and OpenID Connect (OIDC)

我们假设读者熟悉 OAuth 授权框架[^HPL23]和 OIDC 协议[^oidc]：

- OAuth **隐式授权流**和 OAuth **授权码授权流**的安全性。
- OAuth 客户端注册（例如，管理应用的 `client_id` ）
- **JSON Web Tokens (JWTs)**，它包括：
  - JWT **头部段**（header）
  - JWT **有效载荷**（payload），我们通常简称为“**JWT**”。
  - JWT **签名**（signature），对头部段和有效载荷进行签名，我们常将其称为 **OIDC 签名**
  - 我们通常将头部段、有效载荷及其签名的组合称为**已签名的 JWT**。
  - 参见[JWT 头和有效载荷样例](#2-jwt头和有效载荷样例)。
- 相关的 JWT header 字段（例如，`kid`）
- 相关的 JWT payload 字段（例如，`aud`，`sub`，`iss`，`email_verified`，`nonce`，`iat`，`exp`）
- **JSON Web Keys (JWKs)**，由每个 OIDC 提供商发布，发布位置为在其 OpenID 配置的 URL 里指定的 JWK 端点 URL。



#### 2.2 术语

- **OIDC 账户**：由 OIDC 提供商（如 Google ）提供的 Web2 账户（例如，`alice@gmail.com`）
- **无密钥账户**：一个区块链账户，其安全性和有效性由 OIDC 账户（例如， Google 账户）支持而不是由密钥支持。本 AIP 的核心是解释如何安全地实现这样的无密钥账户。
- **特定应用的无密钥账户**：无密钥账户同时**绑定**用户的身份（例如， `alice@gmail.com` ）和负责管理的应用身份（例如， `dapp.xyz` ）。这意味着，为了访问账户，必须提供一个已签名的 JWT Token，该 token 包含该用户的身份和管理应用的身份。这种 Token 只能通过在用户的 OIDC 提供商（例如，Google）中对管理应用进行登录认证后获得。这种做法对于应用管理无法访问时的账户恢复路径选择具有[重要意义](#2-恢复服务)。



#### 2.3 关于 OIDC 的概览

在本 AIP 中，理解 OIDC 的最重要的内容是：它能使得**管理应用**（比如 `dapp.xyz`、`some-wallet.org` 或是某个手机应用）通过  OIDC 提供商（例如，Google）让用户登录，而无需要获取用户的 OIDC 登录凭证，也就是说，管理应用并不需要知道用户的谷歌账户密码。需要特别指出的是，当且仅当用户成功登录后，只有管理应用程序可以收到来自谷歌的 **已签名 JWT**，这是一个公开可查询的用户登录凭证。

这个 AIP 旨在演示如何利用这个签名过的 JWT 来为该用户，以及它所关联的无密钥账户授权执行交易，同时也服务于管理应用程序。



#### 2.4 零知识证明

本节内容假定你对零知识证明（ZKP）系统有一定了解：简而言之，零知识证明系统针对某个**逻辑关系** $R$，使得**证明者**能够在不透露任何有关其**私人信息** $w$ 的前提下，向一个有**公开信息** $x$ 的**验证者**证明自己确实掌握着 $w$，以满足 $R(x; w) = 1$（即证明关系成立）。

其他ZKP术语：

- **证明密钥（Proving key）** 和 **验证密钥（verification key）**
- **特定关系的可信设置（Relation-specific trusted setup）**

我们假设一个对 SNARK 友好的哈希函数 $H_\mathsf{zk}$​ ，它在我们的零知识证明 (ZKP) 内部容易被证明。



#### 2.5 Aptos 账户和交易验证

回忆一下，在 Aptos 中，一个账户由其 **address** 标识，账户的**认证密钥**存储在该地址下。

认证密钥是一种保密承诺，它与账户的**公钥（PK）**（如，公钥的哈希值）之间有加密的联系。 账户的**持有者**负责管理相对应的**私钥**。

要**授权**访问 Aptos 账户，其包括的交易如下：

1. 对（a）账户的**地址**和（b）**交易有效载荷**（例如，Move 入口函数调用）的**数字签名**
2. 在账户 **地址** 下的 **认证密钥** 中提交的 **公钥**

为了**验证交易**是否取得授权访问账户，验证者会执行以下操作：

1. 从交易中提取**公钥**，并依据公钥的类型，推算出相应的**认证密钥**。
2. 验证这个<u>派生</u>的**认证密钥**是否与地址下已存储的**认证密钥**匹配
3. 验证**签名**是否能够被交易中取出的**公钥**成功校验。



### 3. 范围

本 AIP 关注：

1. 阐释无密钥账户运作的详细过程。 
2. 描述我们最早期使用 Rust 语言创建的无密钥交易验证系统的概貌。



### 4. 不在讨论范围内的内容

1. DNS 和 X.509 证书生态系统的安全性
   - 无密钥账户依赖于 OAuth 和 OIDC ，其安全性反过来依赖于 DNS 和 X.509 证书生态系统的安全性。
   - 鉴于所有互联网软件均未脱离上述体系，故此部分不在本范围内。

2. 冒充真正钱包应用的恶意钱包应用
   - 我们假设攻击者**无法**在（比如说）苹果的 App Store 中发布一个冒充钱包注册的 OAuth `client_id` 的手机应用
   - 我们假设攻击者**无法**欺骗用户安装一个冒充另一个应用的 OAuth `client_id` 的恶意桌面应用。因此，我们**不**推荐对桌面应用的无密钥账户进行管理，因为它们非常容易被冒充。

3. 对无密钥账户所需的辅助后端组件的深入讨论：
   - **Pepper 服务**：这是 AIP-81[^aip-81] 讨论的主题，相关内容的概述可以在 [Pepper 服务](#3-pepper-服务)中找到。
   - **ZK 证明服务**：这是 AIP-75[^aip-75] 讨论的主题，相关内容的概述可以在[附录](#（预料之外的）零知识证明服务）找到。
   - **JSON Web 密钥 (JWK) 的共识协议**：这是 AIP-67[^aip-67] 讨论的主题，相关内容的概述可以在 [JWK 共识](#5-jwk-共识)中找到。
   - 对我们的 Groth16 ZKP 的**可信设置 MPC 仪式**（参见 [Groth16 讨论](#1-选择-zkp-系统groth16)）
4. Pepper 和 ZK 证明服务的去中心化计划



## 二、动机

如摘要中所解释的，这个改动实现了两件事：

1. 通过简单地用（比如说）他们的 Google 账户登录到钱包或 dapp ，使用户的使用过程变得更容易。
2. 由于没有涉及到密钥，所以用户更不容易丢失他们的账户。

如果拒绝这项提议，我们将维持目前依赖于密钥的、对用户并不友好的账户管理方式。这可能会妨碍人们对 Aptos 网络的接纳，因为许多用户并不理解公钥加密技术的种种细节，例如：用户如何保护他们的助记词、何为私钥以及公钥与私钥的区别在哪里。



## 三、影响

1. Dapp 开发者
   - 熟悉无密钥账户的 SDK
   - 如果需要，dapps 可以启用**无钱包体验**，用户可以直接通过他们的 OpenID 账户（例如，他们的 Google 账户）登录到 dapp ，无需连接钱包。
   - 这将让用户访问他们的**特定于 dapp 的区块链账户**。这样，由于这个账户只限于那个特定的 dapp ， dapp 可以代表用户“无脑地”授权交易，而无需复杂的 TXN 提示。
   
2. 钱包开发者
   - 熟悉无密钥账户的 SDK
   - 考虑将他们的默认用户引导流程切换为使用无密钥账户引导流程
   
3. 用户
   - 熟悉无密钥账户的安全模型
   - 认识到他们的账户的很安全，就像他们的 OpenID 账户（比如 Google 账户）一样。
   - 如果管理应用无法使用了，知道如何[恢复账户](#2-恢复服务)。
   
   

## 四、替代方案

### 1. Passkey

一个避免用户需要管理自己密钥的替代方案是 **Passkeys** 和围绕它们构建的 **WebAuthn 标准**。Passkey 是与应用程序或网站关联的密钥，它在用户的云中备份（例如， Apple 的 iCloud ，Google 的 Password Manager 等）。

Passkeys 最初被设计为取代传统的网站密码，它的工作原理是：网站不再保存用户的密码，而是保存与用户唯一对应的 Passkey 公钥。当用户需要登录网站时，只需用自己的 Passkey 私钥对网站提供的一个随机验证信息进行签名，就能安全地证明其身份，而用户的 Passkey 私钥则安全地存储在云端备份中。

人们的自然倾向是利用 Passkey 私钥来操作区块链账户，方法是把账户的私钥设置成 Passkey 私钥。不幸的是，Passkey 私钥目前存在两个问题，但这些问题很可能在未来得到解决。具体来说，**在一些平台（例如 Microsoft Windows）上，Passkey 私钥并不总是会自动备份到云储存**。此外，Passkey 私钥还带来了一个**跨设备的问题**：比如，用户在他们的苹果手机上创建了一个区块链账户，他们的 Passkey 私钥会被备份至 iCloud，但是这个备份在该用户的其他安卓、Linux 或 Windows 设备上却无法获取，因为目前还未实现能跨多个平台备份 Passkey 私钥的功能。

将来，一种增强 passkeys 应用的可考虑的方法是通过 passkey 对用户的助记词进行加密，这样就能提供一个额外的备份选项。然而遗憾的是，目前还不明确 WebAuthn PRF 扩展 [^webauthn-prf] 的支持范围有多广泛。



### 2. 多方计算 (MPC)

一个避免用户需要管理自己私钥的替代方案是依赖于**多方计算 (MPC)** 服务来为已经被**某种方式**认证的用户计算签名。

通常情况下，用户需要以方便的方式（也就是说，无需管理私钥）向多方计算（MPC）系统进行身份验证。如果不是这样，多方计算系统就无法解决用户体验上的问题，因此会变得毫无价值。正因此，绝大多数的多方计算系统在代替用户签名前，都会采用 OIDC 或 Passkey 密钥来验证用户的身份。

这存在两个问题。首先，MPC 系统将学习谁在什么时候进行交易：即，OIDC 账户及其对应的链上账户地址。

其次，由于用户可以直接通过 OIDC 向验证器进行认证（正如本 AIP 所论述的），或通过 Passkey（如[上文](#1-passkey)所述）进行认证，因此 MPC 在本质上是多余的。

换句话说，**无密钥账户（keyless accounts）避免了对复杂的多方计算（MPC）签名服务的需求**（安全且稳定地搭建此类服务可能颇具挑战），它们通过 OIDC 直接完成对用户的认证。

事实上，无密钥账户的安全性**更高**，因为一旦 MPC 系统出现安全漏洞，账户就有被盗的风险。而无密钥账户是**不会**被盗的，除非出现这样的情况：

（1）用户的 OIDC 账户出现安全问题，

（2）OIDC 服务提供商本身遭受攻击。

尽管如此，无密钥账户仍然依赖于一个分布式的 Pepper 服务和一个 ZK 证明服务（见[附录](#十三附录)）。然而：

- 为了提升浏览器或手机上的性能，仅仅在进行零知识证明 (ZKP) 计算时，才需要使用证明服务。随着 ZKP 系统未来的优化，这种服务预计将不再需要。
- Pepper 服务与 MPC 服务不同，Pepper 服务对于安全性不敏感：即使攻击者完全占据了 Pepper 服务，也无法偷走用户的账户；除非他们也破坏了用户的 OIDC 账户。
- 虽然 Pepper 服务对于系统的即时响应性有较高要求，也就是说，用户在没法使用自己的 Pepper 的情况下无法完成交易，但我们可以通过采用基于 VRF 的简易设计方案将其分布化，从而解决这个问题。
- MPC 系统的一个**优点**在于，MPC 节点可以在执行交易（TXN）签名前进行**诈骗检测**（fraud detection）。至于进行诈骗检测是在 MPC 系统中好些，还是在钱包中或通过账户抽象好些，这一点需要更深入的研究。



### 3. HSM 或可信硬件

从根本上来说，这个方法存在的问题与多方计算（MPC）的问题一样：为了验证用户身份，其必须采用便捷的基于 OIDC 或 Passkey 的认证方式，以便用户可以通过一个外部（以硬件为基础的）系统代替用户进行签名。

因此，该外部系统（即硬件安全模块，HSM）可能成为一个不必要的风险点 —— 一个可能被入侵的风险点。

正如前文所述，我们的策略可以直接让用户向区块链验证器进行认证，避免了引入额外的基础架构而带来的风险。

硬件安全模块（HSM）或者其它可信硬件的一个优势在于它们相对较简单的实现过程。更重要的是，不同于多方计算（MPC）的方法，使用可信硬件更有助于确保隐私安全。



## 五、规范

以下，我们解释了实现无密钥账户背后的关键概念：

1. 无密钥账户的 **公钥**（*public key*）是什么？
2. 如何从这个公钥派生出 **认证密钥**（*authentication key* ） ？
3. OIDC 账户交易的 **数字签名**（*digital signature*）是什么样的？



### 1. 公钥

无密钥账户的 **公钥** 包括：

1. $\mathsf{iss_{val}}$：OIDC 提供者的身份，即在 JWT 的 `iss` 字段中显示的内容（比如，`https://accounts.google.com`），我们将其标记为 $\mathsf{iss_{val}}$
2. $\mathsf{addr_{idc}}$：一个**身份承诺（identity commitment，IDC）**，这是一个<u>隐藏</u>的承诺，承诺包括：
   - OIDC 提供商发给拥有用户的标识符（例如， `alice@gmail.com` ），由 $\mathsf{uid_{val}}$ 表示。
   - JWT 中用于存储用户唯一标识的字段名被指定为 $\mathsf{uid_{key}}$；Typescript SDK 为了避免初学者选用错误的 JWT 字段，所以当前只支持 `sub` 或 `email` 字段[^jwt-email-field]。尽管如此，验证器能够接受任意字段，这样做是为了不限制那些希望在现有功能基础上创新的经验丰富的用户。
   - 在管理应用注册到 OIDC (OpenID Connect) 提供商时，会被授予一个标识，这个标识即 OAuth 中的 `client_id`，它被存储在 JWT 的 `aud` 字段中，并用 $\mathsf{aud_{val}}$ 表示。

更严格地说（此处省略那些复杂的实施细节），IDC 是通过采用一种适合 SNARK 技术的哈希函数 $H'$ 对前述各字段进行哈希处理得出的：
$$
\mathsf{addr_{idc}} = H'(\mathsf{uid_{key}}, \mathsf{uid_{val}}, \mathsf{aud_{val}}; r),\ \text{where}\ r\stackrel{\$}{\gets} \{0,1\}^{256}
$$

### 2. Pepper

在上述内容中，我们采用了一个具有高熵（high-entropy）的盲化因子（blinding factor） $r$ 来生成 IDC。这确保 IDC 真实隐藏了用户和管理应用程序的身份信息。在这份 AIP 报告中，这种盲化因子被称作保护隐私的 **Pepper**。

Pepper 有两个重要的属性：

1. 在签署交易以授权访问账户时，需要知道 pepper $r$
2. 即便 Pepper 公开了，攻击者也 **不能** 获取账户的访问权限。换言之，不同于密钥，Pepper 无需通过加密来确保账户的安全性；它只关注账户隐私的保护。

更简单地说：

- 如果 **Pepper 丢失**，那么对**账户的访问权就丢失了**。
- 如果**pepper 被 leaky **（例如，被盗），那么只会**丢失账户的隐私**（即，$\mathsf{addr\_idc}$ 中的用户和应用身份可能会被暴力破解，并最终暴露）。

依赖于用户记住他们的 pepper $r$，这将维持易于丢失的基于密钥的账户的现状，从而 **违背了基于OpenID 的区块链账户的目标**。

因此，我们引入了一种 **Pepper Service**，旨在帮助用户生成并记忆他们的 Pepper（有关这项服务的特点，我们在[附录](#3-pepper-服务)中有简要介绍，并在 AIP-81[^aip-81] 文档中进行了深入探讨）。

### 3. 认证密钥

接下来，无密钥账户的**认证密钥（authentication key）**就是其上述公钥的哈希。更正式地说，假设加密哈希函数为 $H$ ，则认证密钥为：

$$
\mathsf{auth_{key}} = H(\mathsf{iss_{val}}, \mathsf{addr_{idc}})
$$

**注意**：在实践中，上述的哈希还包括一个域分隔符，但为了简化阐述，我们忽略了这些细节。

### 4. 私钥

在定义了上述的“公钥”之后，一个自然的问题出现了：

> 与这个公钥关联的私钥是什么？

答案是，用户没有额外的私钥需要记忆。相反，他们的“私钥”实际上就是他们登录到 OIDC 账户的能力，这是由用户通过上述的 $\mathsf{auth_{key}}$ 中提交的管理应用程序来实现的。

换句话说，"私钥" 其实就是用户用来登录账户的密码，用户本人已经知道了这个密码，或者它可以是一个预先安装的 HTTP cookie，从而省去了用户重复输入密码的步骤。然而，单单有这个**密码是不够的**：管理应用程序必须处于可用状态，它得允许用户登录他们的 OIDC 账户并获取 OIDC 签名。（关于如何应对应用程序不可用的问题，我们将在后文详细介绍[当管理应用不可用时的恢复服务](#2-恢复服务)）

更正式地说，如果一个用户可以成功地使用由 $\mathsf{aud_{val}}$ 标识的应用程序（通过 OAuth ）登录到由 $(\mathsf{uid_{key}}, \mathsf{uid_{val}})$ 标识并由 $\mathsf{iss_{val}}$​ 标识的 OIDC 提供商给的 OIDC 账户，那么这个成功使用的能力就作为那个用户的“私钥（secret key）”。



### 5. 签名

我们先从“热身”开始，讲述我们所开发的“ leaky 模式”（leaky mode）无密钥签名方案，该方案并不保障用户隐私。紧接着，我们将介绍“零知识”（zero-knowledge）签名方案，该方案能够有效保护用户隐私。



### 6. 热身： leaky 用户身份和应用身份的签名

在描述我们完全保护隐私的 TXN 签名之前，我们先热身一下，描述一下**leaky 签名（leaky signatures）**，它们会暴露用户的身份和应用的身份：即，它们会暴露 $\mathsf{uid_{key}}, \mathsf{uid_{val}}$ 和 $\mathsf{aud_{val}}$ 。

一个地址交易 $\mathsf{txn}$ 的 **leaky 签名**为 $\sigma_\mathsf{txn}$，该地址的认证密钥（authentication key）为$\mathsf{auth\\_key}$，则定义如下：

$$
\sigma_\mathsf{txn} = (\mathsf{uid_{key}}, \mathsf{jwt}, \mathsf{header}, \mathsf{epk}, \sigma_\mathsf{eph}, \sigma_\mathsf{oidc}, \mathsf{exp_{date}}, \rho, r, \mathsf{idc_{aud\_val}})
$$

其中:

1. $\mathsf{uid_{key}}$ 是JWT字段的名称，用于存储用户的身份，其值在地址 IDC 中被提交
2. $\mathsf{jwt}$ 是 JWT 的有效载荷（例如，参见[示例](#2-jwt头和有效载荷示例)）
3. $\mathsf{header}$ 是 JWT 头；表示 OIDC 签名方案和 JWK 的密钥 ID ，这些都是在正确的 PK 下验证 OIDC 签名所必需的
4. $\mathsf{epk}$，是由管理应用程序生成的临时公钥（EPK），其关联的$\mathsf{esk}$在管理应用程序端保密
5. $\sigma_\mathsf{eph}$ 是对交易$\mathsf{txn}$的临时签名
6. $\sigma_\mathsf{oidc}$ 是对完整 JWT（即，对$\mathsf{header}$和$\mathsf{jwt}$有效载荷）的 OIDC 签名
7. $\mathsf{exp_{date}}$ 是一个时间戳，过了这个时间，$\mathsf{epk}$就被认为过期，不能用来签名TXN。
8. $\rho$ 是一个关键的高熵度变量，用于生成与 $\mathsf{epk}$ 和 $\mathsf{exp\\_date}$ 相关的 **EPK 承诺**。为了保证数据的安全，这个承诺被存储在 $\mathsf{jwt}[\texttt{"nonce"}]$ 字段中。

 9. $r$ 是一个 Pepper ，用于地址的 IDC，但在所谓的 “leaky 模式” 中，这个值通常被设置为零。

10. $\mathsf{idc_{aud\_val}}$ 是一个选填字段，主要用于账户的[恢复操作](#2-恢复服务)。如果被设置，它将包含一个与 IDC 中相同的 `aud` 值。这是在恢复时需要包含在 TXN 签名中的重要信息，因为在那种情况下，$\mathsf{jwt}$ 的负载将会包含恢复服务的 `aud` 值，而非原本在 IDC 中承诺的 `aud` 值。

> [!TIP]
> **概括来说**：为了**核实 $\sigma_\mathsf{txn}$ 签名**，验证者需要确认：首先，OIDC 提供者是否（1）签署了地址 IDC（或恢复服务的 ID）中记录的用户与应用ID；其次，是否（2）对 EPK 进行了签名，而 EPK 又签署了这笔交易，同时还要确保 EPK 的有效期得到了严格限制。

更详细地说，针对公钥 $(\mathsf{iss_{val}}, \mathsf{addr_{idc}})$ 的签名验证包括以下步骤：

1. 如果使用基于 `email` 的 ID，则需要确保是已验证电子邮件：
   1. 也就是说，如果$\mathsf{uid_{key}}\stackrel{?}{=}\texttt{"email"}$，我们就验证（断言 assert） $\mathsf{jwt}[\texttt{"email\_verified"}] \stackrel{?}{=} \texttt{"true"}$。
2. 让 $\mathsf{uid\\_val}\gets\mathsf{jwt}[\mathsf{uid\\_key}]$
3. 当设置了 $\mathsf{idc_{aud\_val}}$ 时，我们需要确认 $\mathsf{jwt}[\texttt{"aud"}]$ 是一个被批准的恢复服务 ID，这个ID应在链上的 [`aud` 重写列表](#3-aud-重写列表)里面。
4. 当设置了 $\mathsf{idc_{aud\_val}}$，我们就让 $\mathsf{aud_{val}}\gets \mathsf{idc_{aud\_val}}$；如果没设置，那么就让 $\mathsf{aud_{val}}\gets\mathsf{jwt}[\texttt{"aud"}]$。
5. 验证（断言 assert） $\mathsf{addr_{idc}} \stackrel{?}{=} H'(\mathsf{uid_{key}}, \mathsf{uid_{val}}, \mathsf{aud_{val}}; r)$，其中 $r$ 来自签名的 Pepper
6. 验证 PK 是否与链上的认证密钥匹配：
   - 验证 $\mathsf{auth_{key}} \stackrel{?}{=} H(\mathsf{iss_{val}}, \mathsf{addr_{idc}})$
7. 检查 EPK 是否在 JWT 的 `nonce` 字段中提交：
   - 验证  $\mathsf{jwt}[\texttt{"nonce"}] \stackrel{?}{=} H’(\mathsf{epk},\mathsf{exp_{date}};\rho)$
8. 检查 EPK 的过期日期是否过于遥远（我们在下面会详细说明）：
   - 确保 $\mathsf{exp_{date}} < \mathsf{jwt}[\texttt{"iat"}] + \mathsf{max_{exp_horizon}}$ ，其中 $\mathsf{max_{exp_horizon}}$  是在链上的参数（详情参见[此处](#2-keyless_accountmove-move-模块)）。  
   - 我们不要求过期时间必须设置在未来的某个时间，也就是说，不检查 $\mathsf{exp_{date}} > \mathsf{jwt}[\texttt{"iat"}]$ 。相反，我们假设 JWT 的签发时间戳（`iat`）字段是正确的，并且与当前区块时间相近。因此，如果应用程序错误地设置为 $\mathsf{exp_{date}} < \mathsf{jwt}[\texttt{"iat"}]$ ，那么从区块链的视角看，EPK 会被认为已过期。
9. 检查 EPK 是否过期：
   - 断言 $\texttt{current\_block\_time()} < \mathsf{exp_{date}}$
10. 验证交易 $\mathsf{txn}$ 下的 $\mathsf{epk}$ 的临时签名 $\sigma_\mathsf{eph}$
11. 获取 OIDC 提供者的正确 PK ，由 $\mathsf{jwk}$ 表示，通过 JWT $\mathsf{header}$ 中的 `kid` 字段标识。
12. 验证 JWT $\mathsf{header}$ 和有效载荷 $\mathsf{jwt}$ 下 $\mathsf{jwk}$ 的 OIDC 签名 $\sigma_\mathsf{oidc}$​ 

> [!NOTE]
> **JWK共识**: 在验证 OIDC 数字签名的最后一步，要求所有验证节点**对 OIDC 提供商的最新 JSON Web Keys（公钥）达成共识**，这些公钥由提供商在特定的**OpenID 配置 URL**上发布。
>
> 为此，验证节点需要就提供商的公钥达成共识，并在 `aptos_framework::jwks`  Move  模块中展示这些信息（具体细节见[附录](#5-jwk-共识)以及 AIP-67[^aip-67]文档）。

>[!NOTE]
>**设定到期日期的必要性：** 我们认为，对于不了解安全性的 dapps 开发者来说，设定一个过于遥远的 $\mathsf{exp_{date}}$ 会带来风险。这让攻击者有更长的时间来攻破已签名的 JWT（及其相关的 ESK）。因此，我们要求到期日期不宜设得过远 —— 在 JWT 的 `iat` 字段的基础上定义一个的 "最大到期时限" $\mathsf{max_{exp\_horizon}}$ 来限制，也就是说，确保 $\mathsf{exp_{date}} < \mathsf{jwt}[\texttt{"iat"}] + \mathsf{max_{exp\_horizon}}$。
>
>另一种方法是确保 $\mathsf{exp\_date} < \texttt{current\_block\_time()} + \mathsf{max\_exp\_horizon}$；然而，这并不理想。攻击者可能会创建一个 $\mathsf{exp_{date}}$，在当前时间 $t_1 = \texttt{current\_block\_time()}$ 时无法通过检查（即，$\mathsf{exp_{date}} \ge t_1 + \mathsf{max\_exp\_horizon}$），但在时间变为 $t_2 > t_1$ 时能够通过检查（即，$\mathsf{exp_{date}} < t_2 + \mathsf{max\_exp\_horizon}$）。因此，一些起初被认为是无效（似乎也无害）的签名 JWT 后来有可能变得有效（变成了值得攻击的目标）。依赖于 JWT 的 'iat' 字段，就可以避免这种情况的发生。



接下来将要解决是的**关于模式 leaky 的警告**：

- Pepper $r$​ 会因为交易而 leaky ，使得地址 IDC 暴露于暴力破解的风险之中。
- 类似地，EPK 的盲化因子 $\rho$ 被 TXN 透露了，因此，目前还未能实现其保护隐私的目的。
- JWT 有效载荷以明文存在， leaky 了管理应用和用户的身份信息。
- 即便 JWT 有效载荷被隐藏，OIDC 签名同样可能会 leaky 这些身份信息，因为已签名的 JWT 可能具有低熵。由此，攻击者可以对一组可能被签名的 JWT 进行针对性的暴力破解，以验证签名。



### 7. 零知识签名

这引出了本 AIP 的核心内容：现在我们准备好解释隐私保护签名如何在我门的无密钥账户中发挥作用。这些签名不会透露用户 OIDC 账户以及与无密钥账户相关联的管理应用的任何信息。

特定的交易 $\mathsf{txn}$ ，该交易针对拥有认证密钥 $\mathsf{auth\\_key}$ 的地址，其**零知识签名** $\sigma_\mathsf{txn}$  定义如下所示：

$$
\sigma_\mathsf{txn} = (\mathsf{header}, \mathsf{epk}, \sigma_\mathsf{eph}, \mathsf{exp\_date}, \mathsf{exp\_horizon}, \mathsf{extra\_field}, \mathsf{override\_aud\_val}, \sigma_\mathsf{tw}, \pi)
$$

其中：

1. $(\mathsf{header}, \mathsf{epk}, \sigma_\mathsf{eph}, \mathsf{exp\\_date})$ 如[之前](#6-热身-leaky-用户和应用身份的签名)所述，此外还有以下变化： 
   - 临时签名 $\sigma_\mathsf{eph}$ 现在还额外包含了零知识证明 (Zero-Knowledge Proof) $\pi$​，以此来抵御更改攻击（malleability attacks）。

2. $\mathsf{extra\_field}$ 是一个**可选**字段 (optional field)，它需要在 JSON Web Token（JWT）中匹配，并且会被**公开地** (publicly) 展示（例如，用户如果想公开他们的电子邮箱地址，就将 `extra_field` 设置为 `"email": "alice@gmail.com"`）。
3. $\mathsf{override\_aud\_val}$ 是[账户恢复](#2-恢复服务)服务中使用的一个**可选**字段 (optional field)。当该字段包含了一个替代 `aud` 值（这是恢复服务的 `client_id`），这通常会被 JWT 载荷的 ZKP 遮蔽掉。在设置此字段的情况下，ZKP 关系（接下来将描述）会核对这个重写的 `aud` 值是否与 JWT 中的 `aud` 相符，而不是与身份验证服务（IDC）的 `aud` 相比较，这与之前提到的 leaky 模式（leaky mode）类似。 
4. $\sigma_\mathsf{tw}$ 是针对[training wheels 模式](#1-training-wheels)下零知识证明（ZKP）的一个**可选的** **training wheels 签名**。
5. $\pi$ 是针对零知识（ZK）关系 $\mathcal{R}$​ 的一个**零知识证明 (ZKPoK)**，用于执行针对隐私保护的验证（这部分内容将在随后详细讨论）。

>[!NOTE]
> 请注意，$\sigma_\mathsf{txn}$ 交易的签名现在不再含有用户的任何可识别个人信息（除了 $\mathsf{iss_{val}}$ 中 OIDC 提供方身份的信息）。

>[!TIP]
>**简而言之**：为了**验证 $\sigma_\mathsf{txn}$ 签名**，验证者需要核实一个对 OIDC 签名的零知识证明 (ZKPoK)。此 OIDC 签名包括：（1）用户和应用程序 ID —— 在地址 IDC 中或者在恢复服务的 ID 中进行了承诺；（2）EPK 进而对交易进行签名，并为 EPK 设定了过期时间。

更进一步，针对公钥 $(\mathsf{iss_{val}}, \mathsf{addr_{idc}})$ 的签名验证包含了以下几个步骤:

1. 验证 $\mathsf{auth_{key}} \stackrel{?}{=} H(\mathsf{iss_{val}}, \mathsf{addr_{idc}})$ ，这个过程如之前所述。
2. 核查过期时间阈值是否合规：
   - 验证 $\mathsf{exp_{horizon}}$ 的取值是否在 $(0, \mathsf{max\_exp\_horizon})$ 的范围内，即 $\mathsf{exp\_horizon} \in (0, \mathsf{max\_exp\_horizon})$，这里的 $\mathsf{max\_exp\_horizon}$ 是链上预设的参数，如前所述。
3. 校验 EPK 是否已过期：
   - 检查当前块时间 $\mathsf{exp_{date}}$ 是否小于 $\texttt{current\_block\_time()}$ ，即 $\texttt{current\_block\_time()} < \mathsf{exp_date}$，如之前所述。
4. 确认 $\mathsf{epk}$ 对交易 $\mathsf{txn}$ 的短暂签名 $\sigma_\mathsf{eph}$​ 的有效性，这依照之前的描述。
5. 获取 OIDC 提供商的正确公钥 (Public Key)，其表示为 $\mathsf{jwk}$​ ，如前所述。
6. 计算**公共输入哈希 (public inputs hash)**：    $$\mathsf{pih} = H_\mathsf{zk}(\mathsf{epk}, \mathsf{addr_{idc}}, \mathsf{exp_{date}}, \mathsf{exp_{horizon}}, \mathsf{iss_{val}}, \mathsf{extra_{field}}, \mathsf{header}, \mathsf{jwk}, \mathsf{override_{aud\_val}})$$    
7. 倘若**training wheels 公钥 ( public key)** 通过治理（governance）的方式被设置在区块链上（具体参见[这里](#1-training-wheels)），则：
   - 需要对零知识证明 (ZKP) $\pi$ 及公共输入哈希 $\mathsf{pih}$ 的 **training wheels 签名** $\sigma_\mathsf{tw}$ 进行验证
8. 验证（ZKPoK）$\pi$，它验证了一个**密钥输入 (secret input)** $\textbf{w}$ 的存在，这个输入满足下文定义的**零知识相关性 (ZK relation)** $\mathcal{R}$：

$$
\mathcal{R}\begin{pmatrix}
    \mathsf{pih};\\
    \textbf{w} = [
      \textbf{w}_\mathsf{pub} = (
        \mathsf{epk},
        \mathsf{addr\_idc}, 
        \mathsf{exp\_date}, 
        \mathsf{exp\_horizon}, 
        \mathsf{iss\_val}, 
        \mathsf{extra\_field}, 
        \mathsf{header}, 
        \mathsf{jwk}, 
        \mathsf{override\_aud\_val}
      ),\\
      \textbf{w}_\mathsf{priv} = (
        \mathsf{aud\_val}, 
        \mathsf{uid\_key}, 
        \mathsf{uid\_val}, 
        r,
        \sigma_\mathsf{oidc},
        \mathsf{jwt},
        \rho
      )
    ]
\end{pmatrix}
$$

**零知识相关性 (ZK relation) $\mathcal{R}$** 主要负责从之前提到的** leaky 模式 (leaky mode)** 中执行涉及隐私的核心验证过程：

1. 验证 —— 通过用 $H\_\mathsf{zk}$ （如前文所述）对 $\textbf{w}\_\mathsf{pub}$ 中的输入进行哈希处理，得到的公开输入哈希 $\mathsf{pih}$​ —— 是否正确。
2. 检查JWT中的OIDC提供商ID：
   - 验证 $\mathsf{iss_{val}}$ 是否与 $\mathsf{jwt}[\texttt{"iss"}]$ 相等，即：$\mathsf{iss_{val}}\stackrel{?}{=}\mathsf{jwt}[\texttt{"iss"}]$
3. 如果采用基于 `email` 的 ID，则要确保邮箱已经得到验证：
   1. 如果 $\mathsf{uid_{key}}$ 等于 $\texttt{"email"}$，则确认 $\mathsf{jwt}[\texttt{"email\_verified"}]$ 值为 $\texttt{"true"}$ ，即：$\mathsf{jwt}[\texttt{"email\_verified"}] \stackrel{?}{=} \texttt{"true"}$
4. 核对 JWT 中的用户 ID ：
   1. 确认 $\mathsf{uid_{val}}$ 是否与 $\mathsf{jwt}[\mathsf{uid_{key}}]$ 相等，即：$\mathsf{uid_{val}}\stackrel{?}{=}\mathsf{jwt}[\mathsf{uid_{key}}]$
5. 检查地址 IDC 使用的值是否正确：
   1. 确认 $\mathsf{aud_{val}}$ 是否与 $\mathsf{jwt}[\texttt{"aud"}]$ 相符，即：$\mathsf{addr_{idc}} \stackrel{?}{=} H'(\mathsf{uid_{key}}, \mathsf{uid_{val}}, \mathsf{aud_{val}}; r)$。
6. 我们是否处于恢复模式？（即，$\mathsf{override_{aud_{val}}} \stackrel{?}{=} \bot$）
   - 如果是，则检查 JWT 中的管理应用程序 ID：验证 $\mathsf{aud_{val}}\stackrel{?}{=}\mathsf{jwt}[\texttt{"aud"}]$
   - 否则，不进行操作，允许JWT中的 `aud` 与IDC中的 `aud` 不同。
7. 校验 EPK 是否已在 JWT 的 `nonce` 字段中提交：
   1. 确认 $\mathsf{jwt}[\texttt{"nonce"}]$ 是否等于 $H’(\mathsf{epk},\mathsf{exp_{date}};\rho)$，即：$\mathsf{jwt}[\texttt{"nonce"}] \stackrel{?}{=} H’(\mathsf{epk},\mathsf{exp_{date}};\rho)$
8. 校验 EPK 的过期日期是否过于遥远：
   1. 确认 $\mathsf{exp_{date}}$ 是否小于 $\mathsf{jwt}[\texttt{"iat"}] + \mathsf{exp_{horizon}}$，即： $\mathsf{exp_{date}} < \mathsf{jwt}[\texttt{"iat"}] + \mathsf{exp_{horizon}}$
9. 解析  $\mathsf{extra_{field}}$ 为 $\mathsf{extra_{field\_key}}$ 和 $\mathsf{extra_{field\_val}}$ 并验证 $\mathsf{extra_{field\_val}}\stackrel{?}{=}\mathsf{jwt}[\mathsf{extra_{field\_key}}]$​
10. 验证 OIDC 签名 $\sigma_\mathsf{oidc}$ 基于 $\mathsf{jwk}$ 对JWT的 $\mathsf{header}$ 和负载 $\mathsf{jwt}$.

>[!TIP]
> 零知识证明 $\pi$ 不会 leaky  $\textbf{w}$​ 中的任何隐私敏感输入。

> [!NOTE]
>
> 附加的 $\mathsf{exp\\_horizon}$ 变量作为一个中间层存在：它确保即使链上的 $\mathsf{max_{exp\_horizon}}$ 参数发生了变化，零知识证明（ZKP）依然有效，因为它们把 $\mathsf{exp_{horizon}}$ 作为公共输入，而不是 $\mathsf{max_{exp\_horizon}}$​。

我们稍后将讨论的**零知识模式的考虑因素**包括：

1. 用户/钱包通过 pepper 服务获得 pepper $r$，我们会在[附录](#3-pepper-服务)中对此服务进行简要说明。
2. **计算零知识证明(ZKP)过程缓慢**。这需要专门的证明生成服务，在[附录](#4-预料之外的-零知识证明服务)中有所简述。
3. 对于**零知识关系实现的 Bugs**，可以通过采用 [“training-wheels” 模式](#1-training-wheels) 来有效规避。



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



### 2. `keyless_account.move` Move 模块

无密钥账户需要在链上进行以下配置：

1. Groth16验证密钥，零知识证明（ZKP）需在该密钥下完成验证； 

2. 特定的 Circuit 的常量值；

3. training wheels 的公钥（如果有提供）；

4. `aud` 的重写清单列表； 

5. 单个交易（TXN）中允许的无密钥签名的最大数量，以此减少拒绝服务（DoS）攻击的风险。 

相关配置被存放在 `aptos_framework::keyless_account` 内的两个资源中：


```rust
/// 用于实现无密钥账户的零知识关系的 288 字节 Groth16 验证密钥（VK）
struct Groth16VerificationKey has key, store {
    /// `alpha * G`的32字节序列化，其中`G`是`G1`的生成器。
    alpha_g1: vector<u8>,
    /// `alpha * H`的64字节序列化，其中`H`是`G2`的生成器。
    beta_g2: vector<u8>,
    /// `gamma * H`的64字节序列化，其中`H`是`G2`的生成器。
    gamma_g2: vector<u8>,
    /// `delta * H`的64字节序列化，其中`H`是`G2`的生成器。
    delta_g2: vector<u8>,
    /// `\forall i \in {0, ..., \ell}, 64-byte serialization of gamma^{-1} * (beta * a_i + alpha * b_i + c_i) * H`, 其中`H`是`G1`的生成器，`\ell`对于零知识关系是 1 。
    gamma_abc_g1: vector<vector<u8>>,
}

struct Configuration has key, store {
    /// 重写`aud`用于身份恢复服务的身份，这将帮助用户恢复与已经消失的dapps或钱包关联的无密钥账户
    /// 重要提示：此恢复服务**不能**单独接管用户账户；用户必须首先通过 OAuth 在恢复服务中登录
    /// 方可允许它轮换该用户的任何无密钥账户。
    override_aud_vals: vector<String>,
    /// 任何交易都不能拥有超过这个数量的无密钥签名。
    max_signatures_per_txn: u16,
    /// 从 JWT 签发时间起，EPK 过期时间可以被设置的最大时长（秒）。
    max_exp_horizon_secs: u64,
    /// 如果启用了 training wheels ，则是 training wheels 公钥
    training_wheels_pubkey: Option<vector<u8>>,
    /// 我们的 Circuit 支持的临时公钥的最大长度（93字节）
    max_commited_epk_bytes: u16,
    /// 我们的 Circuit 支持的JWT `iss` 字段的最大长度（例如 `"https://accounts.google.com"`）
    max_iss_val_bytes: u16,
    /// 我们的 Circuit 支持的JWT字段名称和值的最大长度（例如 `"max_age":"18"`）
    max_extra_field_bytes: u16,
    /// 我们的 Circuit 支持的base64url编码JWT头部的最大字节长度
    max_jwt_header_b64_bytes: u32,
}
```

此 Move 模块同样提供治理功能（functions ），用于更新链上的`Groth16VerificationKey`和`Configuration`配置项：

```rust
/// 将对Groth16验证密钥的更改排队。更改仅在重新配置后生效。
/// 只能通过治理提案调用。
///
/// 警告：为了减轻 DoS 攻击的风险，VK更改应与训练轮公钥更改一起进行，
/// 这样旧的VK的旧ZKP就不能作为潜在有效的ZKP重放。
///
/// 警告：如果设置了恶意密钥，这将导致资金被盗。
public fun set_groth16_verification_key_for_next_epoch(fx: &signer, vk: Groth16VerificationKey) { /* ... */ }

/// 将对无密钥配置的更改排队。更改仅在重新配置后生效。只能
/// 通过治理提案调用。
///
/// 警告：恶意的`Configuration`可能导致 DoS 攻击，造成活性问题，或使恶意
/// 恢复服务提供商能够网络钓鱼用户的账户。
public fun set_configuration_for_next_epoch(fx: &signer, config: Configuration) { /* ... */ }

/// 便捷方法，用于排队更改训练轮公钥。更改仅在重新配置后生效。
/// 只能通过治理提案调用。
///
/// 警告：如果设置了恶意密钥，这*可能*导致资金被盗。
public fun update_training_wheels_for_next_epoch(fx: &signer, pk: Option<vector<u8>>) acquires Configuration { /* ... */ }

/// 便捷方法，用于排队更改最大过期时间范围。更改仅在重新配置后生效。
/// 只能通过治理提案调用。
public fun update_max_exp_horizon_for_next_epoch(fx: &signer, max_exp_horizon_secs: u64) acquires Configuration { /* ... */ }

/// 便捷方法，用于排队清除覆盖`aud`的集合。更改仅在重新配置后生效。
/// 只能通过治理提案调用。
///
/// 警告：当没有设置覆盖`aud`时，与已消失的应用关联的无密钥账户的恢复
/// 将不再可能。
public fun remove_all_override_auds_for_next_epoch(fx: &signer) acquires Configuration { /* ... */ }

/// 便捷方法，用于排队追加到覆盖`aud`的集合。更改仅在重新配置后生效。
/// 只能通过治理提案调用。
///
/// 警告：如果设置了恶意覆盖`aud`，这*可能*导致资金被盗。
public fun add_override_aud_for_next_epoch(fx: &signer, aud: String) acquires Configuration { /* ... */ }
```



### 3. `aud` 重写列表

请注意，为了配合后文将会讨论的[恢复服务（见“恢复服务”一节）](#2-恢复服务)，Move 模块在`Configuration`中保留了一个`aud`的替代列表。正如在 [“签名”](#5-签名) 以及 [“恢复服务”](#2-恢复服务) 两节中所解释的那样，当应用程序（dapp）消失时，这些`aud`对应的 OIDC 签名可允许用户恢复与任意`client_id` 相关联的无密钥账户。



### 4. Rust 结构体

无密钥公钥定义为：

```rust
pub struct KeylessPublicKey {
    /// JWT中`iss`字段的值，表示OIDC提供商。
    /// 例如：https://accounts.google.com
    pub iss_val: String,

    /// 对以下内容的 SNARK 友好型承诺：
    /// 1. 应用程序的ID；即签名OIDC JWT中的`aud`字段，代表OAuth客户端ID。
    /// 2. OIDC提供商的用户内部标识符；例如JWT中签名的`sub`字段，
    ///   它是Google对bob@gmail.com的用户内部标识符，或者是`email`字段。
    ///
    /// 例如：H(aud || uid_key || uid_val || pepper)，其中`pepper`是用于隐藏
    ///  `aud`和`sub`的承诺随机性。
    pub idc: IdCommitment,
}

pub struct IdCommitment(#[serde(with = "serde_bytes")] pub(crate) Vec<u8>);
```

无密钥签名定义为：

```rust
pub struct KeylessSignature {
    pub cert: EphemeralCertificate,

    // 解码后的明文JWT头部（即，不是base64url编码的），包含两个相关字段：
    //  1. `kid`，指明应该使用OIDC提供商的哪个JWK来验证
    //     [零知识证明的] OpenID签名。
    //  2. `alg`，指明签署JWT时使用了哪种类型的签名方案
    pub jwt_header_json: String,

    // `ephemeral_pubkey`的过期时间，以UNIX纪元时间戳（秒）表示。
    pub exp_date_secs: u64,

    // 用于验证`ephemeral_signature`的短期公钥。
    pub ephemeral_pubkey: EphemeralPublicKey,

    // 在`ephemeral_pubkey`下对交易以及ZKP（如果存在）的签名。
    // 将ZKP包含在此签名中，以防止可更改攻击。
    pub ephemeral_signature: EphemeralSignature,
}

pub enum EphemeralCertificate {
    ZeroKnowledgeSig(ZeroKnowledgeSig),
    OpenIdSig(OpenIdSig),
}

pub struct ZeroKnowledgeSig {
    pub proof: ZKP,
    //  Circuit 应在nonce中承诺的过期日期上执行的过期范围。
    // 这必须 <= `Configuration::max_expiration_horizon_secs`。
    pub exp_horizon_secs: u64,
    // 一个可选的额外字段（例如，`"<name>":"<val>"`），将在JWT中公开匹配
    pub extra_field: Option<String>,
    // 将设置为覆盖`aud`值， Circuit 应该匹配此值，而不是IDC中的`aud`。
    // 这将允许用户恢复与不再在线的应用程序绑定的无密钥账户。
    pub override_aud_val: Option<String>,
    // 通过训练轮SK对证明和声明进行签名，以减轻我们 Circuit 中的缺点。
    pub training_wheels_signature: Option<EphemeralSignature>,
}

pub struct OpenIdSig {
    // JWT中JWS签名的解码字节(https://datatracker.ietf.org/doc/html/rfc7515#section-3)
    #[serde(with = "serde_bytes")]
    pub jwt_sig: Vec<u8>,
    // JWT的解码/明文JSON有效负载(https://datatracker.ietf.org/doc/html/rfc7519#section-3)
    pub jwt_payload_json: String,
    // 映射到用户标识的声明中的键的名称；例如，"sub"或"email"
    pub uid_key: String,
    // 用于在nonce字段中混淆来自OIDC提供商的EPK的随机值
    #[serde(with = "serde_bytes")]
    pub epk_blinder: Vec<u8>,
    // 用于计算身份承诺的隐私保护值。它通常从`(iss, client_id, uid_key, uid_val)`唯一派生。
    pub pepper: Pepper,
    // 当使用覆盖aud_val时，签名需要包含IDC中承诺的aud_val，
    // 因为JWT将包含覆盖。
    pub idc_aud_val: Option<String>,
}

pub enum EphemeralPublicKey {
    Ed25519 {
        public_key: Ed25519PublicKey,
    },
    Secp256r1Ecdsa {
        public_key: secp256r1_ecdsa::PublicKey,
    },
}

pub enum EphemeralSignature {
    Ed25519 {
        signature: Ed25519Signature,
    },
    WebAuthn {
        signature: PartialAuthenticatorAssertionResponse,
    },
}
```



### 4. PR

我们将在下方添加更多链接到我们的 Circuit 代码和Rust交易认证器：

- 为 leaky 模式（leaky mode）编写的 Rust 认证器代码 [#11681](https://github.com/aptos-labs/aptos-core/pull/11681)

- 基于 Groth16 的 ZKP 模式的 Rust 认证器代码 [#11772](https://github.com/aptos-labs/aptos-core/pull/11772)

- 从链上获取 Groth16 VK [#11895](https://github.com/aptos-labs/aptos-core/pull/11895)

- 在 Rust 中正确处理 exp_horizon、VK 和配置初始化 [#11966](https://github.com/aptos-labs/aptos-core/pull/11966)

- 可选的 training wheels 签名 [#11986](https://github.com/aptos-labs/aptos-core/pull/11986)

- 修复烟雾测试（smoke test） [#11994](https://github.com/aptos-labs/aptos-core/pull/11994)

- 使用基数为 10 的字符串作为 nonce [#12001](https://github.com/aptos-labs/aptos-core/pull/12001)

- 移除 uid_key 限制 [#12007](https://github.com/aptos-labs/aptos-core/pull/12007)

- 优化`iss`存储并重构 [#12017](https://github.com/aptos-labs/aptos-core/pull/12017)

- 重命名为 AIP-61 术语 [#12123](https://github.com/aptos-labs/aptos-core/pull/12123)

- 重命名为无密钥（keyless） [#12285](https://github.com/aptos-labs/aptos-core/pull/12285)

- 修复 base64 ，修复 training wheels 签名 [#12287](https://github.com/aptos-labs/aptos-core/pull/12287)

- 为某些无密钥结构体自定义十六进制序列化 [#12295](https://github.com/aptos-labs/aptos-core/pull/12295)

- 端到端测试特性门控 [#12296](https://github.com/aptos-labs/aptos-core/pull/12296)

- 支持基于密码的 EPK 和修复不可更改（non-malleability）签名 [#12333](https://github.com/aptos-labs/aptos-core/pull/12333)

- 更新验证密钥和测试证明 [#12413](https://github.com/aptos-labs/aptos-core/pull/12413)

- 修复公共输入哈希生成，修复 serde 反序列化，将 Google 添加为 devnet 中的默认 OIDC 提供商 [#12476](https://github.com/aptos-labs/aptos-core/pull/12476/files)

- 修复特性门控（gating ）并匹配 alg 字段 [#12521](https://github.com/aptos-labs/aptos-core/pull/12521)

  

## 七、测试（可选项）

### 1. 签名验证的正确性与完整性测试

- <u>未过期的</u> 来自支持的提供商的 [ ZKPoKs 的 ] OIDC 签名
- <u>已过期的</u> 来自支持的提供商的 [ ZKPoKs 的 ] OIDC 签名不通过验证。
- 来自**不支持的**提供商的签名不通过验证。
- 没有临时签名的 [ ZKPoKs 的 ] OIDC 签名验证失败。
- 零知识证明（ZKPs）是不可更改的。

### 2. 其他测试内容

- 能够快速验证包含大量 OIDC 账户的交易

我们将来会对测试计划进行扩展。（**待定**）



## 八、风险与缺点

> - 在此处列举该提案可能带来的潜在负面后果。存在哪些风险？
> - 我们应当关注哪些向后兼容的问题？
> - 如遇到问题，我们将如何减轻或解决？

所有相关风险与缺陷均在[“安全性、活跃性与隐私考量”部分](#十一安全性活跃性与隐私考量)进行了阐述。



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
4. 中心化的证明服务部署，包括[“training-wheels”](#1-training-wheels)。
5. 基于 Aptos 验证节点的 JWK 共识机制。
6. 可信多方计算（MPC）设置仪式的组织。

### 2. 推荐的开发者平台支撑时间线

目前，我们正在进行 SDK 的开发和支持。其余工具将在未来进行补充。（**待确定**）

### 3. 推荐的部署时间线

计划于二月在 devnet 网络进行部署。

计划于三月在 mainnet 网络进行部署。

计划在 testnet 网络上在接下来的时间内部署。（**具体时间待定**）



## 十一、安全性、活跃性与隐私考量

> - 这是否会改变我们对于安全的基本假设或威胁模型？
> - 是否存在潜在的诈骗行为？相应的缓解措施为何？
> - 是否需考虑其他的安全隐患？
> - 是否有相关的安全设计文档或审计报告可供参考？

### 1. Training wheels

在我们最初的部署期间，我们打算在证明服务计算证明前，检验其关系准确无误之后为零知识证明（ZKP）添加一个额外的**辅助轮签名**步骤。通过这种方式，我们既可以避免单点的故障风险（即零知识证明关系的实现），同时也不会赋予证明服务额外的权力，以致于干涉账户安全。

具体来说，即使在初学者阶段（training wheels），我们的零知识证明（ZK）关系实现出错也不会导致用户资金遭受重大损失。同时，一个受到攻击的证明服务也无法独立盗取资金：它还必须要获取受害者的 OIDC 账户以及一份有效的 OIDC 签名。

其中一个重要的**存活性问题**是，如果证明服务停止运作，用户可能暂时无法访问他们的账户。但是，我们预期此类停机会非常短暂，并且相对于我们初期部署时确保安全性的目标来说，这是一个可以接受的权衡。

Training wheels 模式的公钥将被保存为链上的一个参数，在`aptos_framework::keyless_account::Configuration`资源中可以查到（详情见[此处](#2-keyless_accountmove-move-模块)），并且可以通过治理来修改。 

>[!NOTE] 
> Training wheels 模式还起到了**防止DoS攻击的作用**，因为无效的零知识证明（ZKP）不会被证明服务签名。因此，由于我们的 Rust 验证代码首先会检查 Training wheels 的签名，它可以轻松地识别并排除无效的 ZKP，无需耗费资源进行验证。 
> 一个关键的实现细节是，如果 Groth16 验证密钥（VK）发生变更，Training wheels 模式公钥（twPK）也必须同时更新，因为 Training wheels 的签名并不涵盖 VK。如果忘记更新 twPK，可能会让攻击者利用旧的 VK 重放（replaying）旧的 ZK 证明，发起 DoS 攻击。

### 2. 恢复服务

再次说明，无密钥账户不仅与用户绑定，同时还与管理应用（比如dapp或钱包）绑定。

不幸情况下，管理应用有可能会消失，例如：

1. 应用的 OAuth `client_id` 可能被 OIDC 提供商封禁（出于各种原因）。
2. 一位疏忽的管理员可能会丢失应用的 OAuth `client_secret`，或是在 OIDC 提供商那里彻底注销了应用。
3. 一位不知情的管理员可能简单地关停了应用（例如，从应用商店中撤下了移动应用，或停止了dapp网站的运作），而没意识到这会导致用户无法再使用应用内的无密钥账户。

换言之，**如果管理应用不存在，用户就无法再访问他们的无密钥账户**，因为他们无法在应用缺失的情况下获取到所需的已签名 JWT。

不过，有了**恢复服务**，通过辅助用户（helping users）重新获得账户访问权限，就可以避免这种账户丢失的情况。具体而言，

1. 用户将使用他们的 OIDC 账号登录到恢复服务。
2. 随后，恢复服务会获取基于它的 `client_id` 和用户的个人识别号 `sub` 的 OIDC 签名。
3. 恢复服务的 `client_id` 将被列入 [`aud` 替换列表](#3-aud-重写列表)，这是由之前成功的治理提案所确定的。
4. 区块链验证者会承认这份 [零知识证明的] OIDC 签名，将其视为对 **任何** 管理应用的有效的无密钥签名。
5. 因此，用户就可以利用这份 [零知识证明的] OIDC 签名，为他们那些无法进入的账户执行密钥更新操作。

> [!WARNING]
>
> 恢复路径只有在通过治理手段将恢复服务的`client_id`添加到链上[`aud`替换列表](#3-aud-重写列表)之后才能使用。

这种做法的**优点**包括：

1. 它减少了中心化的风险，因为恢复服务的`client_id`是可以依据治理提案来设置的。
2. 用户可以选择多个提供商的恢复服务，并使用他们最信赖的那一个。
3. 恢复服务本身不保留状态，也不储存用户以往登录时的已签名 JWT。
4. 不守信的恢复服务不能单独更改用户的账户，用户必须先登入恢复服务。

但这种方法也有一个**弊端**：如果用户登录了一个**恶意**的恢复服务，那么该服务可能会窃取用户所有的无密钥账户。因此，用户必须谨慎选择他们要使用的恢复服务。

为了减轻这种风险，我们可能会通过多方计算（MPC）技术来分布式部署恢复服务。这将确保已签名的 JWT 不会被服务器内部少数串通的人员盗取。这方面的工作还有待未来进一步探索。



### 3. 其他恢复方式

尽管还有**其他的恢复方式**可行，但是它们可能会存在一些问题：用户体验不佳，易受钓鱼攻击影响，不支持所有平台，或者需要中心化管理：

- 对于**基于邮箱的 **OIDC 服务提供商，我们可以将 OIDC 签名的 ZKPoK 替换为 DKIM 签名的电子邮件的 ZKPoK[^zkemail]（且该信息必须是格式正确的）。

  - 尽管这可能用户体验不佳，但它提供了一种紧急情况下的恢复方式，还能**保护隐私**。

  - 确保此流程**难以被仿冒**是关键，因为用户可能受到攻击者的诱骗而发送所谓的“账户重置”邮件。

  - 例如，只有在账户长时间未活动时才启用此流程。

  - 或者，此流程可能需要用户与另一个去中心化的验证方沟通，由该验证方适当引导用户完成账户重置流程。

- 对于**非基于电子邮件**的 OIDC 提供商，备用恢复路径可能会因提供商而异：
  - 对于**Twitter**，用户可以通过发布一条特定格式的消息来证明他们拥有自己的 Twitter 账户。
  - 同理，对于**GitHub**，用户可以通过发布一个gist来完成同样的验证。
  - 如果有一个 HTTPS Oracle，验证者可以验证上述 tweet 或 gist，并允许用户进行账户密钥的更换。由于至少验证者可以通过 HTTPS URL 了解用户的身份，这样的操作**不会保护隐私**。
  - 这个流程也有被钓鱼攻击利用的可能性。 
  
- 另一种方案是，将所有无密钥账户配置成1-out-of-2模式[^multiauth]，并设有一个**恢复用的[Passkey](#1-passkey)**子账户。这样，在自动备份 Passkey 的情形下（比如在苹果平台上），就能利用它来恢复账户访问权限。  
    - 同样可以考虑使用传统的密钥（SK）子账户，但这会要求管理应用提示用户记录密钥，这样做违背了[原定目标](#-目标)中提到的用户友好原则。


- 或者，对于受欢迎的应用，我们可以考虑进行**手动补丁**：例如，为一些特定`client_id`直接在我们的零知识证明关系中添加例外。但这样就引入了集中化的风险。

对于不习惯这种方式的用户，他们可以选择（1）完全不使用这项功能，或（2）根据自己的喜好选择使用` t`-out-of-`n` 方法[^multiauth]，其中一个`n`因素是无密钥账户。

> [!NOTE] 
>
> 该服务同样使用OIDC账户进行请求验证，所以它不能有效地作为第二验证因素；至少不能在不违背无需记忆任何信息的无密钥账户的初衷下使用。



### 4. OIDC 账户遭到破坏

无密钥账户的核心理念是确保用户的区块链账户**与他们的 OIDC 账户同等安全**。理所当然的是，如果 OIDC 账户（比如 Google）遭到黑客入侵，所有跟用户的 OIDC 账户相关联的无密钥账户同样面临风险。

实际上，即便 OIDC 账户**暂时**被入侵，无密钥账户也可能会**永久**受到损害。因为攻击者可以在短时间内获取到 OIDC 账户的权限，并轻易地更换账户密钥。

尽管如此，这正是我们的设计预期：*“你的区块链账户等同于你的 Google 账户！”*

不喜欢这个设计的用户可以选择（1）不使用此项功能，或者（2）根据个人的选择使用 `t`-out-of-`n` 多因素认证方法[^multiauth]，其中之一可以是无密钥账户。

**注意：**Pepper 服务同样采用 OIDC 认证请求，因此它不能作为二次验证的有效因素；除非破坏了无密钥账户的初衷，即免去了用户需要记忆任何信息的必要。



### 5. OIDC 提供商遭到破坏

再次强调，_“你的区块链账户即是你的 OIDC 账户。”_ 换句话说：

- 如果你的 OIDC 账户受到破坏，你的无密钥账户同样受影响。
- 如果你的 OIDC 提供商（例如 Google）被入侵，所有与该提供商相关联的无密钥账户也处于风险之中。

我们强调这是一项**特性**而非**缺陷**：我们希望Aptos用户借助他们的 OIDC 账户的安全性和便利性来进行无缝交易。然而，对于那些不想完全依赖他们的 OIDC 账户安全性的用户，他们可以升级到$t$ out of $n$ 认证方式，采用不同的 OIDC 账户和/或传统的 SKs[^multiauth]相结合。

> [!NOTE]
>
> 我们可以通过一个紧急治理提案来撤销被破坏的 OIDC 提供商的 JWK，以此减少相关的风险。



### 6. OIDC 提供商更改算法为不受支持的签名算法

我们目前实现的零知识证明关系只支持 RSA 签名验证，这是基于其在OIDC提供商中的广泛使用。但如果提供商突然转换到另一种算法（比如，Ed25519），这将导致用户无法生成 ZKPs ，从而无法访问他们的账户。

针对这种不太可能的情况，我们可能需要采取以下措施：

1. 首先，可以启用一个**透露模式**的功能标志。这将在牺牲隐私的情况下恢复对用户账户的访问，虽然不理想，但至少能保证大部分用户资产的安全。
2. 其次，我们对零知识证明（ZK）协议进行升级，使之适配新的方案，并进行一次短期的信任设定，最后将其部署到实际的产品环境中去。



### 7. 基于Web的钱包与无需钱包的Dapps

无密钥账户的出现将促成（1）基于Web的钱包，即用户可通过如Google这样的服务登录，以及（2）允许用户无需传统钱包即可直接通过如Google登录的**无钱包Dapps**。

在Web环境中存在着一些安全挑战：

- 保证 JavaScript 依赖未被篡改，因为这可能导致ESKs、签名的JWTs或两者同时被盗取。
- 在浏览器中管理用户账户的ESKs可能存在风险，这些密钥由于JavaScript的缓存或错误存储而有 leaky 的可能性。
  - 值得庆幸的是，使用 Web 加密 API 和（或）以 Passkey 代替 ESK 可以有效防止这类问题。
- 防止跨站脚本攻击（XSS）和跨站请求伪造攻击（CSRF） 
  - 在网页浏览器中管理 JWT 极具风险，尤其在采用 OAuth 授权的隐式流程时更为明显，因为 JWT 可能会出现在 URL 和 HTTP `Referer` 字段中，容易发生信息 leaky 。因此，应用程序的开发者们必须特别谨慎。



### 8. 对 SNARK 友好的哈希函数

为了保证生成证明的时间在合理范围内，我们采用了专为 SNARK 设计的友好型哈希函数 *Poseidon*[^poseidon]。遗憾的是，与诸如 SHA2-256 这样历史较长的哈希函数相比，Poseidon 尚未经历太多密码学上的详细分析。

若Poseidon哈希函数存在碰撞漏洞，攻击者可能改动用户或应用ID的已提交值，且在不攻破零知识证明系统或OIDC提供商的前提下挪用资金。

长期看来，我们计划淘汰对SNARK友好的哈希函数，迁移到即便是与对SNARK不友好的哈希函数（如SHA2-256）一同使用，我们的zkSNARK证明系统也能维持高效率的选择。



## 十二、开放性问题（选读）

> 这部分包含了一系列问答，其中部分问题可能已有答案，而另一些问题尚在探讨阶段，我们应当予以明确。



### 1. 安全备用恢复路径的 OIDC 提供商集合

>  部署时的关键问题是？有哪些 OIDC 提供商承认安全的备用恢复路径（比如，难以被仿冒的流程）？



### 2. Web 开发者应默认在何处存储ESK及签名的JWT？

对于移动应用来说，存储ESK并非问题：所有数据都储存在应用自身的存储中。

但对于Web应用方面，即dapps或Web钱包，存在两个选择：

1. 使用 **OAuth 隐式授权** 流程，并将加密签名密钥（Encryption Signing Key, ESK）和签名过的 JWT 存放在浏览器内（如本地存储，`IndexedDB`）。   

    - 这种方法的优势在于，即便用户使用的去中心化应用程序（dapp）或网络钱包的后端服务被入侵，用户的资产也不会被窃取，除非用户打开并操作了受入侵的网站，触发恶意的JavaScript代码，。 

2. 使用 **OAuth 授权码** 流程，把加密签名密钥（ESK）存储在浏览器中，而将签名过的 JWT 保留在服务器后台。

   - 不幸的是，一些OIDC提供商允许在后端使用新的`nonce`字段刷新签名的 JWT，而无需用户同意。这意味着一旦后端受到攻击，即便是 OAuth 会话未过期，攻击者也可能在不需要用户访问网站的情况下刷新 JWT，并窃取无密钥账户。



### 3. 应设置的最大过期时间阈值 $\mathsf{max\\_exp\\_horizon}$ 是多少？

回忆一下， $\mathsf{exp\\_date}$ 的值不应超过 $\mathsf{jwt}[\texttt{"iat"}] + \mathsf{max_{exp\_horizon}}$ 。

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

### 2. JWT头和有效载荷示例

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

Pepper 服务用于在用户丢失他们账户的 pepper 时进行恢复，如若失去 pepper，账户也将无法找回。

Pepper 服务旨在帮助用户找回他们账户的 Pepper，如果 Pepper 遗失，那么账户也无法被找回。 

该服务的具体设计细节不在本 AIP 文稿范围之内，但我们在此强调其几个关键特性：

- 服务需要在透露用户 pepper 前，**验证用户**的身份，这一点与区块链验证者的操作过程极为相似。
  - 验证过程会利用和创建交易签名时相同的OIDC签名来完成。

- 服务采用**可验证随机函数(VRF)**来生成 peppers。
- 它的设计支持轻松**去中心化**，无论是作为一个独立的系统存在，还是作为 Aptos 验证器的上层服务。
- Pepper 服务是**隐私保护型**的：它既不会知悉请求 pepper 的用户身份，也不会知道为该用户生成的具体 pepper 值。
- Pepper 服务主要是“**无状态的**”，意味它仅存储那些用于派生用户 peppers 的 VRF 密钥。

### 4. (预料之外的) 零知识证明服务

当浏览器和移动设备上计算零知识证明（ZKP）的成本仍然较高时，**零知识证明服务**将协助用户快速计算他们的 ZKPs。

该服务的设计细节详见AIP-81[^aip-81]文档。下面，我们将重点介绍其几个关键特性：

- 它无法盗取用户的无密钥账户；除非先入侵与之关联的 OIDC（OpenID Connect）账户。

- 即使服务停止，用户依然可以通过慢速方式访问他们的账户，因为他们总能自行计算 ZKPs。
  - 除非证明服务正在执行“新手保护模式”（参见[下文](#1-training-wheels)）。

- 该服务应当是**去中心化的**，无论是在被授权的环境中，还是无需授权的情况下。
- 它应当具备抵御**拒绝服务（DoS）攻击**的弹性。
- 它应当是**无知的**或**保护隐私的**：即不会理解零知识证明中的任何私有输入数据（例如，身份匿名的请求者，或他们的pepper等）。
- 证明服务主要是“**无状态的**”，仅存储那些计算我们的零知识证明关系 $\mathcal{R}$ 的公开证明密钥。

### 5. JWK 共识

不使用密钥的账户在执行交易签名时，涉及到验证 OIDC 签名。这要求验证者**同意 OIDC 提供商的最新 JWKs**（即公钥），这些公钥会在特定提供商的**OpenID 配置 URL** 上定期更新。

关于JWK（JSON Web Key）共识机制的设计与实施细节，请参见AIP-67[^aip-67]文档。现在，让我们着重介绍其中的几个核心特性：

- 验证者需要定期在每个支持的提供商的**OpenID 配置 URL** 上检查 JWKs 的变动。
- 当一个验证者探测到变动时，他们会通过一种一次性共识机制来提议这些变化。
- 一旦验证者之间达成共识，新的 JWKs 将在 `aptos_framework::jwks` 的公共 Move 模块中反映出来。



## 十四、更新日志

- _2024-02-29_：本 AIP 的[早期版本](https://github.com/aptos-foundation/AIPs/blob/71cc264cc249faf4ac23a3f441fb76a64278b51a/aips/aip-61.md)将无密钥账户称为**基于 OpenID 的区块链 (OIDB)** 账户。
- _2024-03-14_：为无需密钥的签名和给公钥添加了 Rust 的数据结构支持。
- _2024-03-14_：增强了无需密钥的交易签名功能，加入了账户恢复服务和新增的公共字段功能。
- _2024-05-08:_ 进行了多项更新
  - 作为解决 dapps / 钱包消失问题的主要方法，引入了[恢复服务](#2-恢复服务)。
  - 对 ZK 关系 $\mathcal{R}$ 进行了更新，补充了最新的细节（`aud`字段的重写，新增的`extra_field`字段，公共输入的哈希值）。
  - 更新了passkey、MPC（多方计算）和 HSM（硬件安全模块）的对比分析。
  - 引入了“初学者（training wheels）签名”机制。
  - 提供了 Rust 和 Move 语言的代码示例。
  - 添加了指向相关 AIP 文档的链接（如，账户恢复服务、证明服务、JWK 共识机制等）。 
  
  

## 十五、引用


[^aip-67]: https://github.com/aptos-foundation/AIPs/blob/main/aips/aip-67.md
[^aip-75]: https://github.com/aptos-foundation/AIPs/blob/main/aips/aip-75.md
[^aip-81]: https://github.com/aptos-foundation/AIPs/blob/main/aips/aip-81.md
[^bn254]: [bn254](https://hackmd.io/@jpw/bn254)
[^bonsay-pay]: [bonsay-pay](https://www.risczero.com/news/bonsai-pay)
[^circom]: [circom](https://docs.circom.io/circom-language/signals/)
[^eip-7522]: **EIP-7522: 针对AA账户的OIDC ZK验证器**, 作者: dongshu2013, [链接](https://eips.ethereum.org/EIPS/eip-7522)

[^groth16]: **关于基于配对的非交互式论证的大小**, Jens Groth撰写，收录于《密码学进展 -- EUROCRYPT 2016》，2016年
[^HPL23]: **OAuth 2.1授权框架**, Dick Hardt、Aaron Parecki和Torsten Lodderstedt撰写，2023年，[[URL]](https://datatracker.ietf.org/doc/draft-ietf-oauth-v2-1/08/)
[^jwt-email-field]: 仅允许OIDC提供商使用`email`字段，并且该字段的值永远不会更改（例如，像Google这样的电子邮件服务），因为这可以确保用户不会因为更改Web2账户的电子邮件地址而意外锁定自己的区块链账户。
[^multiauth]: 请参阅[AIP-55：通用交易认证和支持任意K-of-N多密钥账户](https://github.com/aptos-foundation/AIPs/blob/main/aips/aip-55.md)
[^multisig]: 有替代方案可以替代单个SK账户，例如基于多签名的账户（例如通过[MultiEd25519](https://github.com/aptos-labs/aptos-core/blob/main/aptos-move/framework/aptos-stdlib/sources/cryptography/multi_ed25519.move)或通过[AIP-55](https://github.com/aptos-foundation/AIPs/blob/main/aips/aip-55.md)），但它们仍然归结为一个或多个用户保护其秘密密钥免受丢失或盗窃。
[^nozee]: [nozee](https://github.com/sehyunc/nozee)
[^oauth-playground]: [OAuth Playground](https://developers.google.com/oauthplayground/)
[^openpubkey]: **OpenPubkey：利用用户持有的签名密钥增强OpenID Connect**，由Ethan Heilman、Lucie Mugnier、Athanasios Filippidis、Sharon Goldberg、Sebastien Lipman、Yuval Marcus、Mike Milano、Sidhartha Premkumar和Chad Unrein撰写，*收录于密码学预印本，论文2023/296*，2023年，[[URL]](https://eprint.iacr.org/2023/296)
[^openzeppelin]: **使用Google登录到你的身份合约（为娱乐和盈利）**, 作者: Santiago Palladino, [链接](https://forum.openzeppelin.com/t/sign-in-with-google-to-your-identity-contract-for-fun-and-profit/1631)
[^poseidon]: **Poseidon：零知识证明系统的新哈希函数**，由Lorenzo Grassi、Dmitry Khovratovich、Christian Rechberger、Arnab Roy和Markus Schofnegger撰写，*收录于USENIX Security’21*，2021年，[[URL]](https://www.usenix.org/conference/usenixsecurity21/presentation/grassi)
[^snark-hash]: [如何选择适合零知识证明的哈希函数](https://www.taceo.io/2023/10/10/how-to-choose-your-zk-friendly-hash-function/)
[^snark-jwt-verify]: [SNARK-JWT验证](https://github.com/TheFrozenFire/snark-jwt-verify/tree/master)
[^webauthn-prf]: [PRF 拓展详解](https://github.com/w3c/webauthn/wiki/Explainer:-PRF-extension)
[^zk-blind]: [zk-blind](https://github.com/emmaguo13/zk-blind)
[^zkaa]: **区块链地址的扩展：零知识地址抽象化**; 作者: Sanghyeon Park, Jeong Hyuk Lee, Seunghwa Lee, Jung Hyun Chun, Hyeonmyeong Cho, MinGi Kim, Hyun Ki Cho, Soo-Mook Moon; 发表在Cryptology ePrint Archive, 2023/191论文, 2023年; [链接](https://eprint.iacr.org/2023/191)
[^zkemail]: https://github.com/zkemail
[^zklogin]: [zklogin](https://docs.sui.io/concepts/cryptography/zklogin)

> 原作者未定义的脚注

[^oidc]: 未定义（译者注）
