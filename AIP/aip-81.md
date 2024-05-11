---
aip: 81
title: 为无密钥账户提供 Pepper 服务
author: 
  - name: Zhoujun Ma
    email: zhoujun@aptoslabs.com
discussions-to: "<指向官方讨论主题的 URL>"
status: 起草中
last-call-end-date: "<最后留下反馈和评论的日期 (mm/dd/yyyy)>"
type: 标准 (核心, 网络, 接口, 应用, 框架) | 信息性 | 流程
created: 2024-03-17
updated: "<更新日期 (mm/dd/yyyy)>"
requires: "<AIP 号码>"
---

[TOC]

# AIP-81 - 无私钥账户的 Pepper 服务

## 一、摘要

在 [无私钥账户](https://github.com/aptos-foundation/AIPs/blob/main/aips/aip-61.md) 中，最终用户需要一个私有盲因子（pepper）作为输入，以隐私保护账户地址的派生：
只要 pepper 没有泄露，账户与其背后的提供者或 dApp 所有者之间的链接就会保持隐藏。

这个 AIP 提出了一种解决方案，即通过部署一个公共服务（由 Aptos Labs 运营），以某些会话数据（即来自最终用户的临时公钥和 OIDC 提供者的授权令牌（JWT））的可验证不可预测函数（VUF）来计算 pepper，从而为最终用户管理 pepper，而无需实际存储它们。

### 1. 目标

这是 [无私钥账户](https://github.com/aptos-foundation/AIPs/blob/main/aips/aip-61.md) 基础架构的基本构建块。



## 二、动机

这是 [无私钥账户](https://github.com/aptos-foundation/AIPs/blob/main/aips/aip-61.md) 基础架构的基本构建块。



## 三、影响

DApp/SDK 开发人员需要实现与 pepper 服务的交互，以支持无私钥流程。



## 四、替代方案

### 1. 让 dApp 最终用户管理自己的 pepper？

使用无私钥账户的整个目的是为了让用户免于记忆额外的密钥（secrets），这打破坏了这个目的。

### 2. 让 dApp 服务所有者管理 pepper？

这将使无私钥账户依赖于 dApp：
如果依赖的 dApp 停止运行，所有相关账户将无法访问。



## 五、规范

### 1. 准备工作

以下简要解释了 pepper 服务交互中涉及的一些数据项。
有关完整术语表，请参见[无私钥账户规范](https://github.com/aptos-foundation/AIPs/blob/main/aips/aip-61.md#specification)。

每次用户登录 OIDC 提供者以使用支持无私钥流程的应用程序时，都会生成一对*临时私钥（ESK）和临时公钥（EPK）*。
无私钥交易需要使用 ESK 生成的临时签名。

*EPK 过期日期*是为每个 EPK 指定的时间戳。如果 EPK 的过期日期已过，则将视为过期 EPK，无法用于签署交易。

*EPK blinder* 是在将 EPK 和其过期日期提交到发送到 OIDC 提供者的签名请求的 `nonce` 字段时用于盲化 EPK 和其过期日期的一些随机字节。
（发出的 `JWT` 将包含相同的 `nonce`。）

### 2. 公共参数

#### 2.1 Nonce derivation

需要一个函数 `derive_nonce()` 来将用户的 EPK、EPK blinder 和过期时间戳提交到一个 nonce 中。
然后将该 nonce 发送到 OIDC 提供者，并作为 JWT 载荷的一部分进行签名。

同样的函数需要作为[ZK 关系的一部分](https://github.com/aptos-foundation/AIPs/blob/main/aips/aip-61.md#snark-friendly-hash-functions)进行实现，
因此应优先选择 SNARK 友好的哈希函数。
参考实现使用了 Poseidon 哈希函数覆盖了 BN254。

#### 2.2 可验证不可预测函数（VUF）

粗略地说，VUF 是一个映射 `f(x)`，其隐式确定了一对密钥 `sk, pk`，使得：

- 对于随机输入 `x`，如果没有私钥 `sk`，则很难预测 `f(x)`；
- 对于声称是 `f(x)` 的输出 `y`，任何人都可以使用公钥 `pk` 验证该声明是否为真。

这些属性使得 VUF 输出成为无私钥账户流程中使用的 pepper 的理想候选。

适用于此处的一些具体 VUF 方案：

- BLS VUF（在 BLS12-381 G1 群上，以下简称为 `Bls12381G1Bls`）。
- [Pinkas VUF](https://eprint.iacr.org/2024/198.pdf)。

下面我们将用 `vuf` 来指代用于 pepper 计算的 VUF 方案，用 `vuf.eval` 来指代评估函数。

#### 2.3 非对称加密

此外，为了确保只有最终用户能够查看到（pepper），我们需要使用一种可以处理可变长度输入数据的非对称加密技术，通过终端用户的 ESK 来对密钥剂进行加密。如果没有这样的措施，任何一位接触到泄漏出的 JSON 网络令牌（JWT）的人都可能通过与 pepper 服务进行交互，而得到该账户的地址。

以下是一个结合使用了 ElGamal 非对称加密和 AES-256-GCM `(Aes256Gcm.enc, Aes256Gcm.dec)` 对称加密的加密方案示例，该方案支持不同长度的输入数据（假设其中的一次性密钥对采用 Ed25519 算法生成，下文将此方案简称为 `ElGamalCurve25519Aes256Gcm`）。

- 让 `x, Y` 为 Ed25519 密钥对。
- 对于要加密的可变长度（variable-lenth）输入 `ptxt`：
  - 从 Ed25519 的基础素数阶群中随机选择一个点 `M`。
  - 通过公钥 `Y` 对 `M` 使用 El-Gamal 加密。用 `C0, C1` 表示密文，其中 `C0, C1` 是 Curve25519 上的 2 个点。
  - 将 `M` 哈希转换为 AES-256-GCM 密钥 `aes_key`。
  - 使用加密密钥 `aes_key` 对 `ptxt` 进行 AES-256-GCM 加密。用 `ctxt` 表示可变长度的 AES 密文。
  - 完整的密文为 `C0, C1, ctxt`。
- 解密 `C0, C1, ctxt`：
  - 使用私钥 `x` 对 `C0, C1` 进行 El-Gamal 解密以恢复 `M`。
  - 将 `M` 哈希为AES-256-GCM密钥 `aes_key`。
  - 使用 `aes_key` 对 `ctxt` 进行 AES-256-GCM 解密以从中解密 `ptxt`。
  - 最终输出为 `ptxt`。

下文中我们将使用`asymmetric`来指代所使用的非对称加密方案，将其加密算法称为`asymmetric.enc`。


### Pepper 服务设置

在部署 pepper 之前，应选择临时密钥对的方案以及参数 `derive_nonce`、`vuf`、`asymmetric`。

此外，根据选择的 `vuf`，还应生成并安全地存储一个 VUF 密钥对，以供 pepper 服务读取。

### 发布 VUF 公钥

Pepper 服务应该通过开放一个端点让任何人获取其公钥来发布其 VUF 公钥。

以下是一个示例响应，其中 VUF 公钥被序列化、转换为十六进制，并包装在 JSON 中，假设 `vuf=Bls12381G1Bls`。

```JSON
{
  "public_key": "b601ec185c62da8f5c0402d4d4f987b63b06972c11f6f6f9d68464bda32fa502a5eac0adeda29917b6f8fa9bbe0f498209dcdb48d6a066c1f599c0502c5b4c24d4b057c758549e3e8a89ad861a82a789886d69876e6c6341f115c9ecc381eefd"
}
```

### Pepper 请求（输入）

用户到 pepper 服务的 pepper 请求应该如下所示。
有关详细说明，请参见示例中的注释。

```JSON
{
  /// 用户的临时公钥（EPK），序列化并转换为十六进制。
  "epk": "002020fdbac9b10b7587bba7b5bc163bce69e796d71e4ed44c10fcb4488689f7a144",

  /// EPK 的过期日期，以自 Unix 纪元以来的秒数表示。
  "exp_date_secs": 1710800689,

  /// EPK blinding 因子，十六进制。
  "epk_blinder": "00000000000000000000000000000000000000000000000000000000000000",

  /// 由 OIDC 提供者为最终用户登录到具有指定 EPK 的 dApp 时发出的 JWT。
  "jwt_b64": "xxxx.yyyy.zzzz",

  /// 可选项。确定从哪个 JWT claim 中读取用户 ID。默认为 `"sub"`。
  "uid_key": "email",
  
  /// 可选项。如果 JWT 中的 `aud` claim 代表一个特殊的账户恢复应用程序，并且请求中指定了 `aud_override`，
  /// 则 pepper 服务应该计算 `aud_override` 的 pepper，而不是 JWT 中原始的 `aud`。
  ///
  /// 这有助于最终用户在 OIDC 提供者阻止该应用并停止为其生成 JWT 后，
  /// 仍然可以与该账户进行交易。
  "aud_override": "some-client-id-of-some-banned-google-app",
}
```

### Pepper 请求验证

如果以下任何检查失败，应拒绝 pepper 请求。

- `epk` 应该是有效的。
- 将 `jwt_b64` 解码为 JSONs `jwt.header`、`jwt.payload`、`jwt.sig` 应该成功，
  并且 `jwt.payload` 应该包含以下声明。
  - `jwt.payload.nonce`：keyless 账户方案用来放置用户的临时公钥承诺的可定制字段。
  - `jwt.payload.sub`：由提供者签发的用户 ID（或请求中指定的任何其他字段）
  - `jwt.payload.iss`：提供者
  - `jwt.payload.aud`：由提供者签发的应用程序的客户端 ID
  - `jwt.payload.iat`：JWT 生成时间
- 确保 `epk` 已经承诺在 `jwt.nonce` 中，即 `jwt.nonce == derive_nonce(epk_blinder, exp_date_secs, epk)`。
- `jwt.payload.iss` 和 `jwt.header` 应该标识当前活动的 JWT 验证密钥（也称为 JWK）。
- 使用 `jwt.payload` 和引用的 JWK 验证 `jwt.sig` 应该通过。
- `exp_date_secs` 既不应该是过去的，也不应该远在 `jwt.iat` 加上配置的持续时间之后（默认为 10,000,000 秒或约 115.74 天）。

### Pepper 处理

- 让 `uid_val` 是 `jwt.payload` 中指示的用户 ID，由 `uid_key` 指定。
- 让 `aud` 是由 `jwt.payload.aud` 和 `aud_override` 决定的有效客户端 ID。
- 让 `input` 是数据项 `jwt.payload.iss, uid_key, uid_val, aud` 的规范化序列化。
- 将 `pepper` 计算为 `vuf.eval(vuf_sk, input)`。
- 让 `pepper_bytes` 是 VUF 输出 `pepper` 的序列化。
- 将 `pepper_encryped` 计算为 `asymmetric.enc(epk, pepper_bytes)` 。

### Pepper 响应（输出）

来自 pepper 服务到用户的 pepper 响应应该简单地是由 EPK 加密的 pepper，转换为十六进制，并包装在 JSON 中。

以下是一个示例 pepper 响应，假设 `vuf=Bls12381G1Bls` 和 `asymmetric=ElGamalCurve25519Aes256Gcm`。

```JSON
{
  "signature_encrypted": "6e0cf0dfdffbd22d0108195b54949b7840c3b7e4c9168f0899b60950f16e52cd54f0306cb8e76eda9fb3d5be6890cd3fc2c0cb259e3578dab6c7496b6d64553d4463a18aecf5bc3fc629cd88f9a78221b05b4d04d1a0a20292ed11f4197c5169227c86a6775b1c2990709cadf010cbf624763d68783eb466892d69a70c3c95a9fdffe5917e4554871db915b0"
}
```

## 参考实现

https://github.com/aptos-labs/aptos-core/tree/main/keyless/pepper

## 测试（可选）

测试应作为 [无私钥账户](https://github.com/aptos-foundation/AIPs/blob/main/aips/aip-61.md) 的端对端测试的一部分进行。

## 风险和缺陷

所提议的 pepper 服务是集中化的，在无私钥交易流程中是一个单点故障。
目前，这可以通过多个部署来缓解，由于其无状态特性，易于维护。
将来，作为 Aptos 验证器的一部分运行的去中心化 pepper 服务可能会完全解决这个问题。

如果 VUF 私钥丢失，则所有无私钥账户都无法访问（除非 pepper 在用户端被本地缓存）。

如果 VUF 私钥泄露，则无私钥账户与 OIDC 提供者/dApp 所有者之间的链接不再是私密的。 例如，现在 Google 可能会知道哪个无私钥账户是从它签署的 JWT 派生的。

## 未来潜力

如果 pepper 服务实例化为加权 VUF（例如 Pinkas VUF），
它有可能是去中心化的：

- VUF 私钥需要使用阈值加权密钥共享方案在 Aptos 验证器之间共享。
- pepper 请求将被广播到所有验证器，并且 pepper 是通过验证和聚合来自验证器的足够多的 pepper 响应来获取的。

## 时间表

### 建议的实施时间表

2024.01-03

### 建议的开发平台支持时间表

SDK 支持将与 pepper 服务的开发同步进行。

### 建议的部署时间表

版本 1.10 发布

## 安全性考虑

请参见[风险和缺陷](#risks-and-drawbacks)。





