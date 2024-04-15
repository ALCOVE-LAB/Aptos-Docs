---
aip: 69
title: 开始在链上复制 Google JWK
author: Zhoujun Ma (zhoujun@aptoslabs.com)
discussions-to (*optional): https://github.com/aptos-foundation/AIPs/issues/349
Status: 待审核
last-call-end-date (*optional): <mm/dd/yyyy 最后反馈和审查日期>
type: <标准（核心，网络，框架）>
created: <02/21/2024>
requires (*optional): 67
---

[TOC]

# AIP-69 - 开始在链上复制 Google JWK

## 一、概述

该 AIP 提议开始使用[原生的 JWK 共识框架](https://github.com/aptos-foundation/AIPs/blob/main/aips/aip-67.md)，在链上复制  [在 Google API 中可用的 ](https://accounts.google.com/.well-known/openid-configuration) Google JWK。



## 二、目标

这将启用基于 Google 的 [无密钥账户](https://github.com/aptos-foundation/AIPs/blob/main/aips/aip-61.md)。



## 三、动机

Google 是最流行的 OIDC 提供商之一。启用基于 Google 的无密钥账户可以极大地扩展 Aptos 的用户基础。

此外，一些最近的观察表明，Google 的 JWK 操作似乎很好地满足了[JWK 共识](https://github.com/aptos-foundation/AIPs/blob/main/aips/aip-67.md)和 [无密钥账户](https://github.com/aptos-foundation/AIPs/blob/main/aips/aip-61.md)的要求。
（注：这是非官方观察，目前没有已知的谷歌文档可以确认！）

- JWK 大约每周轮换一次。
- 通常有 2 个 JWK，`(K[i], K[i+1])`。
  - 很可能，`K[i+1]` 是新的主要 JWK，而 `K[i]` 被保留，以便在最后一次轮换之前签署的签名仍然可以被验证。 
  -  目前尚不清楚谷歌是否在最后一次轮换后立即开始使用`K[i+1]`进行签名。
    - 如果是这样，由于复制延迟（目前约为 10 秒），由`K[i+1]`签署的无密钥交易在轮换后的大约前 10 秒内可能无法验证。  无论如何，复制延迟是不可避免的，可以通过 SDK / 应用程序中的一些重试机制来缓解。
  - 下一次轮换将更新 JWK 集为`(K[i+1], K[i+2])`。



## 四、影响

运营商需要确保其节点可以访问以下 Google API。
- `https://accounts.google.com/.well-known/openid-configuration`
- 上述 API 响应 JSON 的 `jwk_uri`。
  - 当前值为 `https://www.googleapis.com/oauth2/v3/certs`。


## 五、规范

在启用了[本地 JWK 共识框架](https://github.com/aptos-foundation/AIPs/blob/main/aips/aip-67.md)之后，此提案可以通过将 Google 添加到支持的 OIDC 提供程序列表中来实现，该列表是框架的链上配置。



## 六、参考实现

以下是一个治理的示例脚本，将 Google 添加到支持的 OIDC 提供程序列表中。

```
script {
    use aptos_framework::aptos_governance;
    use aptos_framework::jwks;

    fun main(core_resources: &signer) {
        let core_signer = aptos_governance::get_signer_testnet_only(core_resources, @0x1);
        let framework_signer = &core_signer;

        jwks::upsert_oidc_provider(
            framework_signer,
            b"https://accounts.google.com",
            b"https://accounts.google.com/.well-known/openid-configuration"
        );

        aptos_governance::reconfigure(framework_signer);
    }
}
```

## 七、测试（可选）

在本地网络进行了测试，并将在预览网络中进行测试（由 Aptos Labs 提供更真实的环境，请参阅[此处](https://aptoslabs.medium.com/previewnet-ensuring-scalability-and-reliability-of-the-aptos-network-48f0d210e8fe)）。



## 八、时间表

### 1. 建议的实施时间表

> [!TIP]
>
> 原作者注：N/A，因为这是一项链上配置更改。

### 2. 建议的开发者平台支持时间表

> [!TIP]
>
> 原作者注：N/A，因为这是一项链上配置更改。

### 3. 建议的部署时间表

版本 1.10

## 十、安全考虑

一般来说，请参阅[此处](https://github.com/aptos-foundation/AIPs/blob/main/aips/aip-67.md#security-and-liveness-considerations)有关 JWK 共识框架的安全考虑。

一般来说，请参阅[此处](https://github.com/aptos-foundation/AIPs/blob/main/aips/aip-61.md#security-liveness-and-privacy-considerations)有关无密钥账户的安全考虑。







