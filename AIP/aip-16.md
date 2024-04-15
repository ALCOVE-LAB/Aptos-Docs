---
aip: 16
title: 新密码学原生功能用于哈希和 MultiEd25519 PK 验证
author: Alin Tomescu <alin@aptoslabs.com>
discussions-to (*optional): https://github.com/aptos-foundation/AIPs/issues/57
Status: Accepted
last-call-end-date (*optional):
type: Standard Framework # <Standard (Core, Networking, Interface, Application, Framework) | Informational | Process>
created: 02/02/2022
updated (*optional):
requires (*optional):
---

[TOC]

# AIP - 16 - 新密码学原生功能用于哈希和 MultiEd25519 PK 验证

## 一、概述

这是一系列包含三个变更的计划：

 - 在 Move 智能合约中添加支持计算 [Blake2b-256 哈希函数](https://github.com/aptos-labs/aptos-core/pull/5436)
 - 在 Move 智能合约中添加支持计算 [SHA2-512、SHA3-512 和 RIPEMD-160 哈希函数](https://github.com/aptos-labs/aptos-core/pull/4181)
 - 升级了 [MultiEd25519 PK (Public Key) 验证](https://github.com/aptos-labs/aptos-core/pull/5822) 功能，以改正一个错误，此错误导致即使是一个没有任何子公钥 (sub-PKs) 的公钥，也会错误地被判定为有效。具体来说：
   - 弃用API： `0x1::multi_ed25519::public_key_validate` 
   - 弃用API： `0x1::multi_ed25519::new_validated_public_key_from_bytes` 
   - 引入API： `0x1::multi_ed25519::public_key_validate_v2` 
   - 引入API： `0x1::multi_ed25519::new_validated_public_key_from_bytes_v2`



## 二、动机

### 1. 哈希

当构建跨链桥时，支持新的哈希函数将非常有用。 例如，生成比特币地址时需要进行 RIPEMD-160 哈希计算（参见 [这里](https://en.bitcoin.it/wiki/Protocol_documentation#Addresses)）。 如果我们不采纳这项提案，就会像 Composable.Finance 这样的公司所说的一样，搭建到其他链的跨链桥时（以及其他密码学应用），将在执行成本（即 Gas 费用）上变得更加高昂。

### 2. MultiEd25519 PK 验证 V2

PK 验证的错误本会危及我们的 `0x1::multi_ed25519::ValidatedPublicKey` 结构类型的类型安全，该数据结构旨在保证一个公钥是格式正确的。正确格式的要求之一是，子公钥的数量必须超过 0。 

虽然格式错误的 `0x1::multi_ed25519::ValidatedPublicKey` 结构在进行多重签名认证时**不会**引起安全问题（因为在用于验证签名时，如通过 `0x1::multi_ed25519::signature_verify_strict` 时，它们很容易就被识别并排除），但它们会强迫开发者必须思考在他们的 Move 模块全程使用这些不规范的公钥可能带来的后果，这无疑增加了他们的心智负担。

 如果我们不通过这项提案，就可能允许用户滥用这类 `0x1::multi_ed25519::ValidatedPublicKey` 结构，进而引发其他安全隐患。 比如说，签名的不可否认性特性可能会如下被破坏： 设想有一个智能合约，它（1）手动反序列化这样一个结构体，（2）解析所有子公钥，并（重新）实施自己的签名验证逻辑来进行检查：



PK 验证的 bug 将威胁我们的 `0x1::multi_ed25519::ValidatedPublicKey` 结构类型（struct type）的类型安全性，该类型应保证公钥是格式良好的，其中包括但不限于要求子公钥的数量大于 0。

虽然格式错误的 `0x1::multi_ed25519::ValidatedPublicKey` 结构在进行多重签名认证时**不会**引起安全问题（因为在用于验证签名时，如通过 `0x1::multi_ed25519::signature_verify_strict` 时，它们很容易就被识别并排除），但它们会强迫开发者必须思考在他们的 Move 模块全程使用这些不规范的公钥可能带来的后果，这无疑增加了他们的心智负担。

如果我们不接受这个提案，用户可能会错误地使用 `0x1::multi_ed25519::ValidatedPublicKey` 结构进而引发其他安全问题。
比如说，签名的不可否认（non-repudiation）性特性可能会如下被破坏： 设想有一个智能合约，它（1）手动反序列化这样一个结构体，（2）解析所有子公钥，并（重新）实施自己的签名验证逻辑来进行检查：

```
// 从恶意字节构造一个格式不正确的 n-out-of-n 公钥
let ill_formed_validated_pk = multi_ed25519::new_validated_public_key_from_bytes(evil_bytes);

let sub_sigs = vector<vector<u8>> = /* 一些签名 */;

// 反序列化这样一个公钥，隐含地假设子公钥的数量大于 0。
let sub_pks : vec<vector<u8>> = deserialize_multi_ed25519_pk(ill_formed_validated_pk);

for i in 0..sub_pks.len() {
	if !verify_sub_sig(sub_sigs[i], sub_pks[i]) {
		return false;
	}
}

return true;
```

要指出的是，任何一个包含 0 个子公钥的不规范公钥，使用任何多重签名进行验证都会通过，这意味着签名的不可否认性会丧失。

尽管这个例子有些牵强（因为开发人员很可能依赖于 `0x1::multi_ed25519::signature_verify_strict`函数，它会直接拒绝这样的公钥和任何基于它的签名），但这个例子强调了保持结构类型安全性的重要性，确保反序列化后的数据格式对用户具有实际意义。



## 三、基本原理

**Q：** 解释为什么您提交了这个提案，而不是其他解决方案。为什么这是最好的可能结果？

**A：** 关于新的哈希函数，我们只是添加了新功能。不需要考虑其他替代方案。

MultiEd25519 的变更是一个 bug 修复。另一种选择是不修复它，正如上文所述，这不是理想的选择。



## 四、规范

哈希函数的 API 和 MultiEd25519 的 API 在下面的参考实现中已经得到了充分的文档支持。



## 五、参考实现

- [Blake2b-256 哈希函数](https://github.com/aptos-labs/aptos-core/pull/5436)
- [SHA2-512、SHA3-512 和 RIPEMD-160 哈希函数](https://github.com/aptos-labs/aptos-core/pull/4181)
- [MultiEd25519 PK 验证 V2](https://github.com/aptos-labs/aptos-core/pull/5822)



## 六、风险和缺陷

- 在 Move 中缺少 `#[deprecated]` 或 `#[deprecated_by=new_func_name]` 注解，使得很难轻松地警告用户不要使用已过时的 MultiEd25519 PK 验证 API。例如：
  - 弃用的API： `0x1::multi_ed25519::public_key_validate` 
  - 弃用的API： `0x1::multi_ed25519::new_validated_public_key_from_bytes` 

- 这就意味着用户可能仍会调用这些 API，并且无意中发现了滥用结构缺陷公钥的方法。



## 七、未来潜力

这些哈希函数可以用于链上的许多密码应用程序（例如，验证 zk-STARK 证明）。



## 八、建议的实施时间表

一切都已经实现了。



## 九、建议的部署时间表

这可能会在三月初进入测试网。