---
aip: 66
title: 通行证账户
author: hariria
discussions-to (*optional): https://github.com/aptos-foundation/AIPs/issues/322
Status: In Review
last-call-end-date (*optional): <mm/dd/yyyy the last date to leave feedback and reviews>
type: Standard (Core/Framework)
created: 12/14/2023
updated (*optional): <mm/dd/yyyy>
requires (*optional): <AIP number(s)>
---

[TOC]

# AIP-66 - 通行证账户

## 一、概述

本AIP提议了 Aptos 的首个 [WebAuthn](https://www.w3.org/TR/webauthn-3/) 验证器，使用户能够使用通行证（passkeys）和其他 WebAuthn 凭据进行交易认证。

[通行证（Passkeys）](https://fidoalliance.org/passkeys/)旨在取代密码，作为一种防钓鱼、更快速、更安全的用户认证形式。当用户注册通行证时，在其设备的 [验证器（authenticator）](https://www.w3.org/TR/webauthn-3/#authenticator) 上创建一个新的网站特定的 [公钥凭证（public key credential）](https://www.w3.org/TR/webauthn-3/#public-key-credential) 。WebAuthn 验证器安全地存储通行证，并使用户可以通过 [授权手势（authorization gestures）](https://www.w3.org/TR/webauthn-3/#authorization-gesture) （如 Face ID 或 Touch ID）访问它们。在未来的会话中，可以使用该网站的通行密钥代替密码来生成一个数字签名，以验证用户的身份。

在 Aptos 上，通过 [WebAuthn](https://www.w3.org/TR/webauthn-3/) 特定的 [`AccountAuthenticator`](#交易提交) 对通行证交易进行认证。由于其在大多数现代操作系统中的广泛支持，Aptos 目前支持 NIST P256 (`secp256r1`) 作为唯一有效的 WebAuthn 签名方案。WebAuthn 的 [`AccountAuthenticator`](#4. 交易提交) 使 Aptos 用户可以使用任何兼容的 WebAuthn 凭据进行交易签名和提交，包括在 iOS、MacOS 和 Android 设备上注册的多设备凭据，以及像 Yubikey 等设备上的单设备、硬件绑定的凭据。

## 二、动机

区块链通过提供用户完全控制其账户而无需集中托管人的能力，彻底改变了数字资产所有权。然而，这种去中心化模型也有缺点：如果用户自行托管他们的密钥，他们将完全负责账户的管理，如果用户丢失了他们的私钥，就没有恢复路径。 

通行密钥通过提供一种机制来创建和恢复账户，而无需依赖明文助记词或私钥，使用户能够无缝地进入Web3。具体来说，在苹果或安卓生态系统中的通行密钥可以轻松地恢复或同步到新设备，只要该设备存在于同一生态系统中（例如，iOS 和 MacOS 设备可以通过 iCloud 同步通行密钥）。在 Aptos 中，通过其原生密钥轮换和对 K-of-N 多密钥账户的支持，存在进一步的恢复可能性，如[AIP-55](https://github.com/aptos-foundation/AIPs/blob/main/aips/aip-55.md )中所讨论的。



## 三、目标

本AIP的目标有两个：

1. 使 Aptos 上的用户能够创建与其 WebAuthn 凭据关联的链上账户
2. 使 Aptos 上的用户能够使用 WebAuthn 凭据对交易进行签名
## 四、影响

本AIP将通过为开发者和用户提供一种简单的方法来创建不可钓鱼、可恢复的私钥，从而惠及他们。 

1. **用户友好性：**
   1. WebAuthn 凭证注册（私钥创建）和交易签名可以通过用户认证（如设备生物识别）简单完成。
   2. 使用户能够通过他们的通行密钥账户与 DApp 交互，而无需安装移动或扩展钱包：即，**无头钱包体验**。    
   3. 通过将私钥安全地存储在 WebAuthn 认证器中，而不是浏览器存储（例如，本地存储），通行密钥消除了传统上为加密浏览器中的私钥而设置钱包密码的需要。    
   4. 在某些操作系统上，通行密钥会被备份并与多个设备无缝同步（参见[备份资格](#备份状态和资格 )）。 
2. **安全性：**
   1. 在通行密钥中，私钥不会被保存在浏览器的存储空间（例如：localStorage）中，恶意方可能通过物理访问设备或通过恶意依赖的供应链攻击来访问它。
   2. 默认情况下，通行密钥提供一种基于同意的签名体验，并在每次签名时提示用户进行身份验证，类似于 Apple Pay 或 Google Pay。
   3. 通行密钥绑定到一个依赖方（例如，一个像基于 Web 钱包这样的网站），并且不易受到钓鱼攻击。



## 五、可替代方案

请参阅 [AIP-61](https://github.com/aptos-foundation/AIPs/blob/main/aips/aip-61.md#alternative-solutions) 中关于多方计算（MPC）、硬件安全模块（HSM）和 OpenID 区块链（Keyless）账户各自权衡的更深入探讨。

### 1. 无密钥账户

无密钥账户提供了一种独特且新颖的方式来生成账户：通过OpenID账户在链上生成账户。这是非常可取的，因为无密钥账户可以在所有设备上通过浏览器访问。此外，大多数人都有一个或多个与 OIDC 提供者（例如Google）关联的账户，并且熟悉 OAuth 登录流程。最后，可恢复性不像通行密钥那样仅限于某些操作系统。 

另一方面，也需要考虑权衡：

1. 无论是 OIDC 提供者还是无密钥账户在隐私保护方式下运行所需的服务（例如，pepper服务、验证服务），都存在中心化和活跃性方面的问题。
2. 除非用户的无密钥账户关联的临时密钥（$\mathsf{esk}$）被加密（即，使用密码）或本身就是一个通行证，否则浏览器上仍然可以获取明文的临时密钥，这使其在短暂的有效期内可能受到恶意行为者的威胁。另一方面，通行证私钥通常安全存储在设备上的验证器中，并在备份过程中进行端到端加密。

## 六、用户流程概述

本 AIP 的目标不是规定将通行证集成到您的 Aptos 应用程序中的方法，而是为了演示目的。这里是一个示例，说明了如何使用通行证在网站上进行交易签名。

假设一个新用户 Alice 想要在 Aptos 上创建一个账户，并使用该账户向她的朋友 Bob 发送资金。

1. Alice访 问一个钱包网站并创建一个账户。
2. 在创建账户时，钱包提示 Alice 为该网站注册一个通行证。
3. Alice 通过授权手势（生物特征识别，如 Face ID 或 Touch ID ）同意，并创建一个绑定到该钱包网站的通行证。
4. 钱包从 Alice 的通行证公钥生成一个 Aptos 地址。
5. Alice 向她的地址转账一些 APT 以在链上创建账户。
6. 在 Web 钱包上，Alice 签署一个交易，将 1 个 APT 发送到 Bob 的地址。
7. dApp 请求 Alice 通过她的通行证签署交易。
8. Alice 通过授权手势（生物特征识别，如 Face ID 或 Touch ID ）同意。
9. 交易（带有通行证签名和公钥）成功提交到 Aptos 区块链，并 APT 转入 Bob 的账户。

## 七、规范

通行证使用了由 [FIDO联盟](https://fidoalliance.org/) 和 [万维网联盟（W3C）](https://www.w3.org/) 共同创建的 [WebAuthn规范](https://www.w3.org/TR/webauthn-3/) 进行通行证注册和认证。在下文中，本 AIP 讨论了WebAuthn 规范在 Aptos 上用于交易认证的方式。

### 1. 术语

- **[无密钥账户](https://github.com/aptos-foundation/AIPs/blob/main/aips/aip-61.md): **无密钥账户的安全性和活跃性由 OIDC 账户支持（例如，Google 账户），而不是由密钥支持。
- **[依赖方](https://www.w3.org/TR/webauthn-3/#relying-party): ** 用户试图访问的网站、应用程序或服务，负责验证用户通行密钥的签名，以确保用户是他们声称的那个人。为了简化，我们可以说“依赖方”在这个上下文中与**钱包**同义，因为两者都负责管理用户对私钥的访问。
- **[WebAuthn验证器](https://www.w3.org/TR/webauthn-3/#authenticator):** 存在于硬件或软件中的加密实体，可以为给定的依赖方注册用户，并在稍后断言拥有注册的公钥凭证，并可选地向依赖方验证用户。
- **[账户验证器](https://github.com/aptos-foundation/AIPs/blob/main/aips/aip-55.md#summary):** 通过交易中账户的发送者或批准者的集合授权在Aptos上执行交易。



### 2. 使用通行证创建Aptos账户

注册新的 WebAuthn 凭证期间，通过该设备上的 WebAuthn 认证器安全地生成了一对非对称密钥（即**私钥**及其关联的**公钥**）。与 WebAuthn 凭证关联的公钥可以用来推导链上关联的账户。控制账户的私钥将由设备上的 WebAuthn 认证器进行保护。 

注册过程特别重要，原因有几个： 

1. 与凭证关联的公钥仅在注册期间被公开。断言响应（使用通行密钥签名）**不会**返回公钥。因此，必须解析注册响应，并且与凭证相关的相关信息应该被适当地存储并由钱包在后端持久化。
2. 注册响应包括描述凭证备份特性的`flags`。如果通行密钥备份是一个重要的考虑因素，那么在创建链上账户之前，应该评估这些标志。



#### 2.1 [`PublicKeyCredentialCreationOptions`](https://www.w3.org/TR/webauthn/#dictdef-publickeycredentialcreationoptions)

每个WebAuthn凭证都由钱包使用一组选项创建，称为`PublicKeyCredentialCreationOptions`。这些选项有助于配置WebAuthn凭证的许多方面，包括但不限于：

- 凭证的非对称密钥类型（`Ed25519`、`ECDSA`、`RSA`等）
- 在断言期间的用户验证要求
- WebAuthn凭证是否应绑定到一个设备或可在多个平台上使用。

仔细考虑选择哪些选项非常重要，因为它们对通行证用户体验有重大影响，并且在凭证创建后**无法**重新配置。

下面突出显示了`PublicKeyCredentialCreationOptions`中的一些字段。

每个 WebAuthn 凭证都是由钱包使用一组选项创建的，这些选项被称为 `PublicKeyCredentialCreationOptions`。这些选项有助于配置 WebAuthn 凭证的许多方面，包括但不限于： 

- 凭证的非对称密钥类型（`Ed25519`、`ECDSA`、`RSA`等） 
- 断言期间的用户验证要求
- WebAuthn 凭证是否应该绑定到一个设备或跨平台可用。 

仔细考虑选择哪些选项非常重要，因为它们对通行密钥用户体验有重大影响，并且**不能**在凭证创建后重新配置。 

在`PublicKeyCredentialCreationOptions`中，以下是一些被突出显示的字段。

**[`PublicKeyCredentialParameters`](https://www.w3.org/TR/webauthn-3/#dictdef-publickeycredentialparameters)**

为使 WebAuthn 凭证与 Aptos 兼容，`pubKeyCredParams` 数组应仅包含受支持的签名方案。目前只支持 `secp256r1`，因此数组必须有一个元素，其 `alg: -7`，以表示 `secp256r1` 是凭证类型的选择。

**[`PublicKeyCredentialUserEntity`](https://www.w3.org/TR/webauthn-3/#dictdef-publickeycredentialuserentity)**

每个认证器存储一个[凭证映射](https://www.w3.org/TR/webauthn-3/#authenticator-credentials-map)，从（[`rpId`](https://www.w3.org/TR/webauthn-3/#public-key-credential-source-rpid)、[`userHandle`](https://www.w3.org/TR/webauthn-3/#public-key-credential-source-userhandle)）到公钥凭证源的映射。为了提供上下文，[用户句柄](https://www.w3.org/TR/webauthn-3/#user-handle)是凭证的唯一标识，类似于`credentialId`，但由钱包而不是 WebAuthn 验证器选择。如果用户在同一验证器上（在同一用户的设备/平台上）使用与现有凭证相同的`userHandle`创建另一个凭证，它将**覆盖**现有凭证，从而覆盖与通行证账户关联的私钥。为了避免这种情况发生，钱包必须维护一个已注册的凭证及其对应的用户句柄列表。



#### 2.2 注册响应

成功注册凭证后，验证器将返回一个 [`PublicKeyCredential`](https://www.w3.org/TR/webauthn-3/#publickeycredential)。在 `PublicKeyCredential` 中将包含一个 [`AuthenticatorAttestationResponse`](https://www.w3.org/TR/webauthn-3/#authenticatorattestationresponse) 类型的 `response` 字段，其中将包含大部分用于验证注册响应的信息。

对 `PublicKeyCredential` 中的 `response` 字段进行完全解码将得到一个 `AuthenticatorAttestationResponse`。下面是其 Rust 表示形式的示例：

[code reference](https://github.com/1Password/passkey-rs/blob/c23ec42da5d3a9399b358c5d38f4f5b02f48b792/passkey-types/src/webauthn/attestation.rs#L497)

```rust 
pub struct AuthenticatorAttestationResponse {
    /// 这个属性包含了客户端传递给认证器的`CollectedClientData`的JSON序列化，用于生成这个凭证。确切的 JSON 序列化必须被保留，因为已经对序列化的客户端数据进行了哈希计算。
    #[serde(rename = "clientDataJSON")]
    pub client_data_json: Bytes,

    /// 这是包含在认证对象中的验证器数据。
    pub authenticator_data: Bytes,

    /// 这是新凭证的 DER [SubjectPublicKeyInfo]。如果不可用，则为 None。
    ///
    /// [SubjectPublicKeyInfo]: https://datatracker.ietf.org/doc/html/rfc5280#section-4.1.2.7
    #[serde(skip_serializing_if = "Option::is_none")]
    pub public_key: Option<Bytes>,

    /// 这是新凭证的 [CoseAlgorithmIdentifier]
    ///
    /// [CoseAlgorithmIdentifier]: https://w3c.github.io/webauthn/#typedefdef-cosealgorithmidentifier
    #[typeshare(serialized_as = "I54")] // because i64 fails for js
    pub public_key_algorithm: i64,

    /// 这个属性包含一个认证对象，对于客户端来说是不透明的，并且被加密保护，以防止客户端篡改。认证对象包含了  [`AuthenticatorData`] 和一个认证声明。前者包含了 [`Aaguid`]、唯一凭证 ID 和凭证公钥的 [`AttestedCredentialData`]。认证声明的内容由验证器使用的认证声明格式确定。它还包含了依赖方服务器验证认证声明所需的任何附加信息，以及解码和验证 [`AuthenticatorData`] 以及客户端数据的 JSON 兼容序列化。
    pub attestation_object: Bytes,

	/// 此字段包含零个或多个唯一的 [`AuthenticatorTransport`] 值的序列，按字典顺序排列。这些值是认证器（被认为）支持的传输方式，或者如果信息不可用，则为空序列。这些值应该是 [`AuthenticatorTransport`] 的成员，但依赖方应该接受并存储未知值。
    #[serde(default, skip_serializing_if = "Option::is_none")]
    pub transports: Option<Vec<AuthenticatorTransport>>,
}
```

#### 2.3 [`Authenticator Data`](https://www.w3.org/TR/webauthn-3/#table-authData)


| 名称                   | 长度（字节）     | 描述                                                         |
| ---------------------- | ---------------- | ------------------------------------------------------------ |
| rpIdHash               | 32               | RP ID 的 SHA-256 哈希，表示凭证所在范围的 RP ID。            |
| flags                  | 1                | 标志位（位 0 是最不重要的位）：<br>• 位 0: 用户存在（User Present / UP）结果。<br>  ◦ 1 表示用户存在。<br>  ◦ 0 表示用户不存在。<br>• 位 1: 保留供将来使用（RFU1）。<br>• 位 2: 用户已验证（User Verified / UV）结果。<br>  ◦ 1 表示用户已验证。<br>  ◦ 0 表示用户未验证。<br>• 位 3: 备份资格（ Backup Eligibility / BE）。<br>  ◦ 1 表示公钥凭证源具有备份资格。<br>  ◦ 0 表示公钥凭证源没有备份资格。<br>• 位 4: 备份状态（Backup State / BS）。<br>  ◦ 1 表示公钥凭证源当前已备份。<br>  ◦ 0 表示公钥凭证源当前未备份。<br>• 位 5: 保留供将来使用（Reserved for future use / RFU2）。<br>• 位 6: 包含已验证凭证数据（Attested credential data / AT）。<br>  ◦ 表示验证器是否添加了已验证凭证数据。<br>• 位 7: 包含扩展数据（Extension data  / ED）。<br>  ◦ 表示验证器数据是否具有扩展。 |
| signCount              | 4                | 签名计数器，32 位无符号大端整数。                            |
| attestedCredentialData | 可变（如果存在） | 已验证的凭证数据（如果存在）。详见 § 6.5.2 已验证凭证数据。其长度取决于被验证的凭证 ID 和凭证公钥的长度。 |
| extensions             | 可变（如果存在） | 扩展定义的验证器数据。这是一个 CBOR [RFC8949] 映射，扩展标识符为键，验证器扩展输出为值。详见 § 9 WebAuthn 扩展。 |

在认证响应（`AuthenticatorAttestationResponse`）中的`authenticatorData`包含`flags`，提供了 WebAuthn 凭证的**备份资格**和**备份状态**信息。如果钱包认为 WebAuthn 凭证应该需要备份，钱包应该检查认证响应中的 Backup Eligibility（`BE`）和 Backup State（`BS`）标志，以确保凭证已备份。如果`BE`和`BS`都设置为 true，意味着 [凭证是多设备凭证，并且当前已备份](https://www.w3.org/TR/webauthn-3/#sctn-credential-backup)。 

对于那些希望使用 passkeys 作为 Aptos 账户的可恢复私钥替代品的用户，强烈建议在创建链上账户**之前**，将`BE`和`BS`标志都设置为`true`，以确保在设备丢失的情况下账户可恢复。





[`AttestedCredentialData`](https://w3c.github.io/webauthn/#attested-credential-data) 包含了 `key`，这是与 WebAuthn 凭证关联的公钥。

[code reference](https://github.com/1Password/passkey-rs/blob/c23ec42da5d3a9399b358c5d38f4f5b02f48b792/passkey-types/src/ctap2/attestation_fmt.rs#L206C1-L225C2)

```rust 
pub struct AttestedCredentialData {
    /// 验证器的 AAGUID。
    pub aaguid: Aaguid,

    /// 凭证 ID，其长度被前置到字节数组中。这不是公开的，因为它不应该被修改为超过 u16 的长度。
    credential_id: Vec<u8>,

    /// 使用 CTAP2 规范的 CBOR 编码格式编码的 COSE_Key 格式的凭证公钥，如 [RFC9052] 的第 7 节所定义。
    /// COSE_Key 编码的凭证公钥必须包含“alg”参数，且不得包含任何其他可选参数。
    /// “alg”参数必须包含一个 [coset::iana::Algorithm] 值。
    /// 编码的凭证公钥还必须包含相关密钥类型规范中规定的任何其他必需参数，即“kty”和算法“alg”（见 [RFC9053] 的第 2 节）。
    ///
    /// [RFC9052]: https://www.rfc-editor.org/rfc/rfc9052
    /// [RFC9053]: https://www.rfc-editor.org/rfc/rfc9053
    pub key: CoseKey,
}
```

凭证公钥以 [CBOR Object Signing and Encryption format (COSE)](https://datatracker.ietf.org/doc/html/rfc8152) 格式存储。`CoseKey` 包括公钥的 `x` 和 `y` 坐标，组合起来形成公钥。

有了原始公钥，就可以推导出 Aptos 账户地址。请注意，根据创建的账户类型（`SingleKey` 或 `MultiKey`），默认的身份验证密钥派生方法会有所不同。

### 3. 签署交易

在传统的客户端-服务器WebAuthn规范实现中，注册新凭证和身份验证断言都使用[挑战-响应认证（challenge-response authentication）](https://en.wikipedia.org/wiki/Challenge%E2%80%93response_authentication)来避免多种类型的攻击。随机化的挑战对于保护免受[重放攻击](https://en.wikipedia.org/wiki/Replay_attack)（[§13.4.3](https://www.w3.org/TR/webauthn-3/#sctn-cryptographic-challenges)）特别重要。

Aptos 正在将 WebAuthn 规范调整为用于链上账户和交易。正如许多人已经知道的那样，链上交易包括一个序列号，以避免在 Aptos 上发生。因此，如果交易被用作 WebAuthn 断言的挑战，即使挑战不是由钱包随机生成，重放攻击的风险也会得到缓解。



在传统的客户端-服务器WebAuthn规范实现中，注册新凭证和认证断言都使用[挑战-响应认证]()来避免多种类型的攻击。随机化的挑战（challenges）对于防止[重放攻击]()特别重要[(§13.4.3)]()。 



Aptos 正在将 WebAuthn 规范整合到链上账户和交易验证中。通常，链上交易会包含一个序列号，用以防止对 Aptos 网络进行重放攻击。这样一来，即便挑战并非由钱包随机生成，使用交易作为 WebAuthn 断言的验证挑战也能有效增强安全性，降低重放攻击的可能性。



#### 3.1 [`challenge`](https://www.w3.org/TR/webauthn-3/#authenticatorassertionresponse)

[code reference](https://www.w3.org/TR/webauthn-3/#dictdef-publickeycredentialrequestoptions)

```js 
dictionary PublicKeyCredentialRequestOptions {
    required BufferSource                   challenge;
    unsigned long                           timeout;
    USVString                               rpId;
    sequence<PublicKeyCredentialDescriptor> allowCredentials = [];
    DOMString                               userVerification = "preferred";
    sequence<DOMString>                     hints = [];
    DOMString                               attestation = "none";
    sequence<DOMString>                     attestationFormats = [];
    AuthenticationExtensionsClientInputs    extensions;
};
```

在断言过程中，`signing_message(raw_transaction)` 的 `SHA3-256` 摘要被提供为 [`PublicKeyCredentialRequestOptions`](https://www.w3.org/TR/webauthn-3/#dictdef-publickeycredentialrequestoptions) 中的 `challenge`。在计算 `raw_transaction` 的 `SHA3-256` 之前，[`signing_message`](https://github.com/aptos-labs/aptos-core/blob/main/crates/aptos-crypto/src/traits.rs#L151) 函数被应用，以实现域分离，而不会进一步增加挑战的大小。

换句话说，`challenge` 是：

$$
challenge = H({signing\\_message(raw\\_transaction)})
$$

其中 $H$ 是 `SHA3-256` 哈希。

#### 3.2 [`AuthenticatorAssertionResponse`](https://www.w3.org/TR/webauthn-3/#authenticatorgetassertion)

在通过 `navigator.credentials.get()` 成功提交 `PublicKeyCredentialRequestOptions` 后，将向客户端返回 `AuthenticatorAssertionResponse`，如下所示：

[code block](https://www.w3.org/TR/webauthn-3/#authenticatorassertionresponse)

```js
interface AuthenticatorAssertionResponse : AuthenticatorResponse {
    [SameObject] readonly attribute ArrayBuffer      authenticatorData;
    [SameObject] readonly attribute ArrayBuffer      signature;
    [SameObject] readonly attribute ArrayBuffer?     userHandle;
    [SameObject] readonly attribute ArrayBuffer?     attestationObject;
};
```



#### 3.3 [`签名`](https://www.w3.org/TR/webauthn-3/#dom-authenticatorassertionresponse-signature)

`AuthenticatorAssertionResponse` 中的 `signature` 是在消息 `m` 上计算的，其中 `m` 是 `authenticatorData` 和 `clientDataJSON` .[^verifyingAssertion]的 SHA-256 哈希值的二进制连接。`clientDataJSON` 是 [`CollectedClientData`](https://www.w3.org/TR/webauthn-3/#dictdef-collectedclientdata) 的 JSON 序列化，包括 `challenge` 和关于请求的其他数据，例如请求的 `origin` 和 [`type`](https://www.w3.org/TR/webauthn-3/#dom-collectedclientdata-type)。

换句话说，对于签名 $\sigma$：

$$
\begin{aligned}
m &= \text{authenticatorData} \ || \ H(\text{clientDataJSON})\\
\sigma &= \text{sign}(\text{sk}, m)
\end{aligned}
$$

其中 $H$ 是 `SHA-256` 哈希（注意：不是 `SHA3-256`），$sk$ 是 passkey 私钥，$||$​ 是二进制链接。



#### 3.4 [`AuthenticatorAssertionResponse`](https://www.w3.org/TR/webauthn-3/#authenticatorassertionresponse)

```js
interface AuthenticatorAssertionResponse : AuthenticatorResponse {
    [SameObject] readonly attribute ArrayBuffer authenticatorData;
    [SameObject] readonly attribute ArrayBuffer signature;
    [SameObject] readonly attribute ArrayBuffer? userHandle;
};
```



### 4. 交易提交

这个 AIP 基于 [AIP-55](https://github.com/aptos-foundation/AIPs/blob/main/aips/aip-55.md) 中提出的新的 `TransactionAuthenticator` 和 `AccountAuthenticator` 构建，使用了 `SingleKeyAuthenticator` 和 `MultiKeyAuthenticator`，分别支持单一密钥和 `k-of-n` 多密钥。在 `AnySignature` 下新增了一个 `WebAuthn` 变体，在 `AnyPublicKey` 下新增了一个 `Secp256r1Ecdsa` 变体，使用户可以使用 `Secp256r1` WebAuthn 凭证从 `SingleKey` 或 `MultiKey` 账户提交交易。

```rust
AccountAuthenticator::SingleKey { authenticator: SingleKeyAuthenticator }
AccountAuthenticator::MultiKey { authenticator: MultiKeyAuthenticator }

pub struct SingleKeyAuthenticator {
    public_key: AnyPublicKey,
    signature: AnySignature,
}

pub struct MultiKeyAuthenticator {
    public_keys: MultiKey,
    signatures: Vec<AnySignature>,
    signatures_bitmap: aptos_bitvec::BitVec,
}

pub struct MultiKey {
    public_keys: Vec<AnyPublicKey>,
    signatures_required: u8,
}

pub enum AnySignature {
    ...,
    WebAuthn {
        signature: PartialAuthenticatorAssertionResponse,
    },
}

pub enum AnyPublicKey {
    ...,
    Secp256r1Ecdsa {
        public_key: secp256r1_ecdsa::PublicKey,
    },
}
```

`AnySignature` 中的每个 `signature` 变体都结构化为 `PartialAuthenticatorAssertionResponse`。`PartialAuthenticatorAssertionResponse` 包含验证 WebAuthn 签名所需的所有信息。

```rust
pub enum AssertionSignature {
    Secp256r1Ecdsa {
        signature: secp256r1_ecdsa::Signature,
    },
}

pub struct PartialAuthenticatorAssertionResponse {
    /// 此属性包含从认证器返回的原始签名。
    /// 注意：从 WebAuthn 断言返回的许多签名不是原始签名。
    /// 例如，secp256r1 签名被编码为 [ASN.1 DER Ecdsa-Sig_value](https://www.w3.org/TR/webauthn-3/#sctn-signature-attestation-types)
    /// 如果签名已编码，则预期客户端在将其包含在交易中之前将编码签名转换为原始签名。
    signature: AssertionSignature,
    /// 此属性包含通过客户端传递给认证器的 [`CollectedClientData`](CollectedClientData) 的认证器数据的字节序列化。
    /// 必须保留确切的 JSON 序列化，因为已对序列化的客户端数据计算了哈希。
    authenticator_data: Vec<u8>,
    /// 此属性包含通过客户端传递给认证器的 [`CollectedClientData`](CollectedClientData) 的 JSON 字节序列化。
    /// 必须保留确切的 JSON 序列化，因为已对序列化的客户端数据计算了哈希。
    client_data_json: Vec<u8>,
}
```

### 5. 签名验证

一旦交易达到签名验证步骤，WebAuthn 认证器将执行以下步骤来验证签名 $\sigma$：

1. 验证实际的 `RawTransaction` 是否与 `clientDataJSON` 中期望的 `challenge` 匹配，方法是计算 
   $$
   H(signing_{message}(raw_{transaction}))
   $$
   其中 $H$​​ 是 `SHA3-256` 哈希，并验证它是否等于 `clientDataJSON` 中的 `challenge`。
   
   > [!tip]
   >
   > 译者注：原公式为`H(signing\\_message(raw\\_transaction))`
   
2. 使用 `clientDataJSON` 和 `authenticatorData` 重构消息 `m`，

3. 针对消息 `m`、公钥 `pk` 和原始签名 $\sigma$，应用验证函数以确定 $V(σ, m, pk)$ 是否评估为 `true`

假设其他所有内容都是正确的，验证通过，并且其他验证器同意结果，那么这笔交易将会成功，并且会被添加到区块链状态中。

### 6. 交易提交摘要

总结一下，在 Aptos 区块链对 WebAuthn 规范的实现中，身份验证断言的高级协议如下：

1. 客户端以 `signing_message` 的 `RawTransaction` 的 `SHA3-256` 形式提供一个 `challenge`。我们使用 `RawTransaction` 的 `SHA3-256` 而不是 `RawTransaction` 本身来限制 `challenge` 的大小，这是 `clientDataJSON` 的一部分。
2. 如果用户同意，用户代理（浏览器）将请求 WebAuthn 认证器使用 Passkey 凭证在包含挑战的消息 `m` 上生成断言签名，并将响应返回给客户端。
3. 然后客户端将 `SignedTransaction` 发送到区块链，其中 `SignedTransaction` 的签名字段包含 WebAuthn 认证器响应中的相关字段
4. 区块链将验证 `SignedTransaction` 的签名是否与相同的消息 `m`（其中包含 `RawTransaction`）并验证断言签名是否与相应的公钥有效

## 八、参考实现

本 AIP 中的工作实现可在 [Passkeys Authenticator PR](https://github.com/aptos-labs/aptos-core/pull/10755) 中找到。

## 九、测试（可选）

测试可以在上述 PR 链接中的参考实现中找到。

## 十、风险和缺陷

> [!WARNING]  
>
> 这份AIP并不旨在规定或强制执行特定的解决方案来应对下面提到的风险和缺点。然而，最终，解决这些风险的责任在于钱包提供者。
>
> 一种可能的方式是，钱包可以通过允许用户设置一个与他们的密码关联的k-of-n `MultiKey`账户（如[AIP-55](https://github.com/aptos-foundation/AIPs/blob/main/aips/aip-55.md )所述）来解决下面提到的一些可恢复性风险。一个额外的非密码签名者使用户即使在钱包不可用的情况下也能访问他们的账户。例如，配置一个 1-of-2 `MultiKey`账户，使用密码和 `ed25519` 私钥，即使钱包不可用也能进行交易签名。但是，钱包需要提供一种方法，让用户在钱包不可用的情况下也能访问密码公钥。这是因为`MultiKey` Authenticator需要在交易中使用所有关联的公钥。有关更多信息，请参阅[密码公钥丢失](#Loss-of-Passkey-Public-Key)部分。

### 1. 备份状态和资格

正如 AIP 61 中所述：

> **Passkey 在某些平台上（例如 Microsoft Windows）不一定会备份到云端**。

因此，重要的是要检查注册响应中 `authenticatorData` 中的 `backupEligible` 和 `backupState` 标志，以确保 passkey 已经正确备份。

#### 1.1 操作系统

请参阅 [操作系统参考](https://passkeys.dev/docs/reference/) 以更好地了解哪些操作系统支持 passkey 以及它们的限制。值得注意的是，根据 2023 年 2 月的情况，注册在 Windows 设备上的 passkey 将无法备份。如果用户希望备份 passkey，则建议他们使用允许多设备凭证并支持 passkey 备份的设备。

#### 1.2 浏览器

有关哪些浏览器和操作系统支持可备份的 passkey 的更多信息，请参阅此 [设备支持矩阵](https://passkeys.dev/device-support/)。

#### 1.3 不称职/恶意认证器

如果用户的认证器符合备份条件，在许多情况下，passkey 将备份到其 iCloud 或 Google 密码管理器帐户中。但在某些情况下，用户可能有一个自定义认证器，如密码管理器（例如 1Password），该认证器存储 passkey。

在向密码管理器注册 passkey 时，passkey 将备份到用户的密码管理器帐户，而不是他们的 iCloud 或 Google 密码管理器帐户中。这对于备份资格和备份状态是一个重要考虑因素，因为用户信任密码管理器及其对客户端到认证器协议（[CTAP](https://fidoalliance.org/specs/fido-v2.0-ps-20190130/fido-client-to-authenticator-protocol-v2.0-ps-20190130)）的实现以进行 passkey 备份。如下文 [云提供商被入侵](#Compromised-Cloud-Provider) 部分所述，如果密码管理器被入侵，您的 passkey 帐户也会被入侵。

为了减轻潜在恶意认证器的风险，钱包应该解析 Authenticator Data 中的 Attested Credential Data，以获取与凭证相关联的 [`AAGUID`](https://www.w3.org/TR/webauthn-2/#sctn-authenticator-model)。

[`AAGUID`](https://www.w3.org/TR/webauthn-2/#sctn-authenticator-model) 是一个128位的标识符，指示认证器的类型（例如制造和型号）。与 Google 密码管理器、iCloud 和其他几个密码管理器相关联的 [`AAGUID`](https://www.w3.org/TR/webauthn-2/#sctn-authenticator-model) 可以在 [passkey-authenticator-aaguids](https://github.com/passkeydeveloper/passkey-authenticator-aaguids/blob/main/aaguid.json) 仓库中找到。

### 2. 钱包活跃性

一个重要的 **活跃性考虑因素** 是与您的 passkey 关联的钱包必须可用，用户才能访问他们的帐户。

许多支持的浏览器，如 Google Chrome，实现了客户端到认证器协议 ([CTAP](https://fidoalliance.org/specs/fido-v2.0-ps-20190130/fido-client-to-authenticator-protocol-v2.0-ps-20190130.html))，该协议规定了漫游认证器和浏览器之间的通信协议，在允许用户注册或使用 passkey 进行断言之前检查钱包是否可用。

换句话说，如果钱包不可用或返回类似于 404 的错误，则用户将无法使用与该钱包关联的 passkey，直到钱包重新上线。这包括使用您的 passkey 进行交易签名。

要了解更多关于 Chromium 如何处理断言和注册请求的信息，请参阅 [`webauthn_handler.h`](https://github.com/chromium/chromium/blob/95bb60bf7fd3d18f469f050b60663b3dbdfa0402/content/browser/devtools/protocol/webauthn_handler.h#L19)。

### 3. 不称职或恶意钱包

#### 3.1 Passkey 公钥丢失

- 如果钱包丢失了与 passkey 关联的公钥，用户将无法向区块链提交交易。这是因为交易需要用于签名验证的公钥。有两种主要策略可以减轻这种风险：
  1. **（推荐）** 确保 passkey 关联了一个 k-of-n `MultiKey` 帐户。在链上存储 passkey `credentialId`、`publicKey` 和 `address` 之间的映射关系。如果钱包不可用，用户可以从区块链检索与 passkey 凭证关联的公钥，并提交由非 passkey 签名者签名的 `MultiKey` 交易。
  2. 与 `secp256r1` passkey 凭证关联的 passkey 公钥可以从 ECDSA 签名中恢复（[ECDSA 公钥恢复](https://wiki.hyperledger.org/display/BESU/SECP256R1+Support#SECP256R1Support-PublicKeyRecoveryfromSignature)）。然而，这假设 passkey 提供者可用。
     - 注意：公钥恢复是未来支持的 WebAuthn 签名方案的重要考虑因素，因为其他签名方案如 `ed25519` 并不支持与 ECDSA 相同的公钥恢复方式。

#### 3.2 覆盖 Passkey

如果用户使用与现有凭证相同的 `userHandle` 创建另一个凭证，则它将 **覆盖** 现有凭证。钱包必须跟踪现有的 `userHandle`，以确保不会发生这种情况。

### 4. 云提供商被入侵

如果您的 passkey 是多设备凭证，并备份到 iCloud 或 Google 密码管理器，并且相关的云账户被入侵，那么您的 passkey 帐户也将被入侵。

请注意，这仅适用于多设备凭证，而不适用于硬件绑定凭证，如 Yubikeys，因为它们没有备份。

### 5. 授权提示

passkey 的一个缺点是，通过 WebAuthn API 无法自定义 WebAuthn Authenticator  的授权提示。尽管这可能对许多支付应用来说更安全，但对于某些应用（例如游戏）来说，可能并不理想，因为这些应用可能不希望每次用户签署交易时都要求用户进行授权操作。此外，提示中使用的授权消息会根据 WebAuthn Authenticator  的不同而有所变化。理想情况下，授权提示可能会包含有关交易的信息（例如，交易模拟结果），并提示用户“使用您的 passkey 进行交易签名”，而不是提示用户“使用你的 `<PASSKEY_NAME>` passkey 为`<WALLET_NAME>`”或“使用 Touch ID 登录”。 

钱包可以通过预先向用户介绍 passkey 交易签名流程来解决这个问题。例如，钱包可以将“签署交易”的按钮更改为“认证并签署”或“认证并支付”，以向用户表明使用密码进行认证将签署交易。此外，在用户入门过程中，钱包可以提供教育资源，以向用户解释 passkey 交易签名流程。

## 十一、未来潜力

总的来说，passkey 提供了一个无缝且直观的方式，让用户生成区块链上的账户和交易。随着 passkey 的不断普及和流行，我们很高兴看到这种方法如何改变人们对数字所有权的看法。


## 十二、时间表

### 1. 建议的部署时间表

截至 2023 年 1 月，Passkeys 已经在 Devnet 上可用。

此 AIP 的建议时间表为 1.10 版本发布。



## 十三、参考资料

[^verifyingAssertion]: "WebAuthn Level 3 规范，§ 7.2  验证断言，第23条要点。" [W3C](https://www.w3.org/TR/webauthn-3/#sctn-verifying-assertion)。