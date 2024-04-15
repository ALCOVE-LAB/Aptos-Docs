---
aip: 23
title: 使 Ed25519 公钥验证原生函数在密钥长度错误时返回`false`
author: Michael Straka <michael@aptoslabs.com>
Status: Accepted
type: Standard (Framework)
created: 03/24/2023
---

[TOC]

# AIP-23 - 使 Ed25519 公钥验证原生函数在密钥长度错误时返回`false`

## 一、概述

更改原生Move函数`native fun pubkey_validate_internal`使用的`native_public_key_validate`函数，使其在提供的公钥长度不正确时返回`false`。在此之前，如果提供的密钥长度不正确，该函数会中止。此更改通过功能标志进行控制。



## 二、动机

此功能允许 `native_public_key_validate` 函数的用户更灵活地处理错误。



## 三、理由

通过功能标志的门控确保了与之前调用 `native_public_key_validate` 的行为向后兼容，即在启用该标志之前不会更改之前事务的行为。



## 四、规范

参考实现中的相关代码如下：

```rust
let key_bytes_slice = match <[u8; ED25519_PUBLIC_KEY_LENGTH]>::try_from(key_bytes) {
     Ok(slice) => slice,
     Err(_) => {
         return Err(SafeNativeError::Abort {
             abort_code: abort_codes::E_WRONG_PUBKEY_SIZE,
         });
         if context
             .get_feature_flags()
             .is_enabled(FeatureFlag::ED25519_PUBKEY_VALIDATE_RETURN_FALSE_WRONG_LENGTH)
         {
             return Ok(smallvec);
         } else {
             return Err(SafeNativeError::Abort {
                 abort_code: abort_codes::E_WRONG_PUBKEY_SIZE,
             });
         }
     },
};
```



## 参考实现

https://github.com/aptos-labs/aptos-core/pull/7043



## 风险和缺点

依赖于`native_public_key_validate`之前行为的调用者可能会受到此更改的影响。目前，端到端测试显示没有这样的调用者。此外，如果该函数返回`false`而不是中止，那么过去调用此函数的调用者（如果有的话）不太可能受到影响。这是因为这样的调用者通常需要检查函数的返回值，并且如果返回值为`false`，他们很可能会中止。



## 未来潜力

未来调用 `native_public_key_validate` 的用户将从更灵活的错误处理中受益，如上所述。



## 建议的实现时间线

它已经实现：参见上述参考实现。



## 建议的部署时间线

此 AIP 计划在 4 月部署在开发网和测试网，然后在 5 月初部署在主网。
