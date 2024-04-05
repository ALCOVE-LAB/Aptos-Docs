---
aip: 46
title: 新增 ElGamal、Pedersen 和 Bulletproofs 模块，基于 Ristretto255
author: Michael Straka (michael@aptoslabs.com), Alin Tomescu (alin@aptoslabs.com)
discussions-to (*可选): https://github.com/aptos-foundation/AIPs/issues/222
状态: 草案
最后召集结束日期 (*可选): <mm/dd/yyyy 最后一个留下反馈和审查意见的日期>
类型: 标准 (框架)
创建日期: 07/21/2023
更新 (*可选): <mm/dd/yyyy>
需要 (*可选): <AIP number(s)>
---

[toc]

# AIP-46 - 新增 ElGamal、Pedersen 和 Bulletproofs 模块，基于 Ristretto255

## 一、概述

> 包括了一个简要描述，概述了所计划的更改。这应该不超过几句话。讨论了这个更改对业务的影响和对业务价值的影响。

本 AIP 提议通过三个**新的 Move 模块**来扩展 Move 中的加密操作套件：

- [ElGamal 加密](https://en.wikipedia.org/wiki/ElGamal_encryption)[^elgamal]（基于 Ristretto255）
- [Pedersen 承诺](https://crypto.stackexchange.com/questions/64437/what-is-a-pedersen-commitment)[^pedersen]（基于 Ristretto255）
- 用于 Pedersen 承诺的 Bulletproofs[^bulletproofs] 零知识范围证明验证器（基于 Ristretto255）

此外，本 AIP 还提议**向现有的 [`ristretto255`](https://github.com/aptos-labs/aptos-core/blob/aptos-release-v1.6/aptos-move/framework/aptos-stdlib/sources/cryptography/ristretto255.move) 模块添加几个新函数**[^ristretto255]：

1. 一个原生函数点克隆（point cloning），名为 `point_clone`
2. 一个原生函数用于双倍标量乘法（double scalar multiplications），名为 `double_scalar_mul`
3. 一个*非* 原生函数，用于从 u32 值创建标量（scalars），名为 `new_scalar_from_u32`
4. 一个 *非* 原生函数，用于将 [`CompressedRistretto`](https://aptos.dev/reference/move/?branch=mainnet&page=aptos-stdlib/doc/ristretto255.md#0x1_ristretto255_CompressedRistretto) 转换为字节序列（sequence of bytes），名为 `compressed_point_to_bytes`
5. 一个 *非* 原生函数，通过对 Ristretto255 基点进行哈希来返回一个点（point），名为 `hash_to_point_base`

最后，本 AIP 提议通过重命名两个先前的函数来**弃用**它们，以使名称更清晰：

1. 弃用 `new_point_from_sha512`，改为 `new_point_from_sha2_512`
2. 弃用 `new_scalar_from_sha512`，改为 `new_scalar_from_sha2_512`

## 二、动机

> 描述这一变更的动机。它能达到什么目的？

这一变更的动机是为 Move 开发人员提供一个**更全面的加密工具套件**。具体来说：

- **ElGamal** 是一种特别适用于加密小范围数据字段（如宽度为 40 位的数值 - 使用更大的数据也可，但解密将相对困难）的加法同态和可再随机化的加密方案。
- **Pedersen 承诺** 是一种在理论上完全隐藏信息、在计算上确保承诺的可信度，并且支持同态运算的数据字段承诺方法。
- **Bulletproofs** 是一种**零知识范围证明（ZKRP）**：即，证明一个 Pedersen 承诺 $g^v h^r$ 中的秘密值 $v$ 在特定范围 $v\in [0, 2^n]$ 内。

这些新模块将使得更多种类的加密 DApps 成为可能：

- **Bulletproofs** 对于[保密交易](https://en.bitcoin.it/wiki/Confidential_transactions)、数字身份系统（例如，证明你年龄未满18岁）、偿付能力证明[^provisions]、声誉系统（例如，证明你的声誉足够高）等方面非常有用。
- **ElGamal 加密** 对于保密交易或需要私密、同态可加的值的应用程序非常有用，例如基于卡牌的游戏中的随机洗牌。
- **Pedersen 承诺** 对于保密交易、拍卖协议、类似 RANDAO 的协议生成随机数等非常有用。
- 最后，对 Ristretto255 模块添加的新函数修复了代码中的一些限制。

> 如果我们不接受这个提案，可能会发生什么？

不接受这个提案将阻止依赖于 ZK 范围证明的 gas-efficient DApps 的发展。由于缺少点克隆以及其他一些缺失功能，也将阻止 `ristretto255` 模块的高效使用。

## 三、影响

> 哪些受众群体受到这一变更的影响？受众群体需要采取何种行动？

这个 AIP 影响了 Move 开发人员：

- **编译时间**将增加，因为新模块被添加到了 Move 标准库中。
- 开发人员必须熟悉本 AIP 中引入的新函数以及被弃用的函数。

## 四、基本原理

> 解释为什么你提交了这个提案，而不是其他替代方案。为什么这是最佳的可能结果？

这个提案是针对**新功能**而不是解决问题的方案。尽管如此，我们相信这些新功能将带来良好的结果，因为：

- Bulletproofs[^bulletproofs] 在学术文献中得到了**深入研究**，并且已经在像 [Monero](https://web.getmonero.org/resources/moneropedia/bulletproofs.html) 这样的生产级系统中**部署**。
- Bulletproofs 提供了**小型证明**，且**验证速度快**。
- Bulletproofs 将使一类新的**隐私保护应用程序**成为可能。

- ElGamal 加密和 Pedersen 承诺模块在 `ristretto255` 上的实现将帮助 Move 开发人员减少开发时间，并**防止实现错误**。
- 在 `ristretto255` 中添加点克隆和双倍标量乘法的新函数将使这个模块**更易于使用且更便宜**。

## 五、规范

> 准确描述此提案应该如何实施。包括应该遵循的预期设计原则。

我们的指导原则是：

- 加密 Move 模块的类型安全性
- 实现 Bulletproofs ZK 范围证明验证器的原生函数的 gas 效率
- 难以误用的 Move API 设计

> 使提案具体到足以允许他人构建其上，甚至可能衍生出竞争性实现。

在高层次上，该提案为：

- 在 `ristretto255` 模块之上提供一种类型安全的 ElGamal 加密实现。
- 在 `ristretto255` 模块之上提供一种类型安全的 Pedersen 承诺实现。
- 在上述 Pedersen 承诺模块中提供一种类型安全的 gas 效率高的 ZK 范围证明验证器实现。

## 六、参考实现

> 这是一个可选但非常鼓励的部分，您可以在这里包含您在此提案中寻求的示例。这可以是代码、图表，甚至是纯文本。理想情况下，我们有一个链接到展示标准的代码示例的存储库，或者对于简单情况，是内联代码。

[PR 3444](https://github.com/aptos-labs/aptos-core/pull/3444) 总结了我们的参考实现。为了方便起见，所提议的更改如下所示：

- 可以通过展开 [此处的 PR](https://github.com/aptos-labs/aptos-core/pull/3444/files#diff-24a99571d84934515c6da4d0b6ab75451b1286a3d88126afb92305e2181a5408) 中对 `ristretto255.move` 的更改来查看对 `ristretto255` 模块的更改。
- 最终的 [`aptos_std::ristretto255_bulletproofs`](https://github.com/aptos-labs/aptos-core/blob/main/aptos-move/framework/aptos-stdlib/sources/cryptography/ristretto255_bulletproofs.move) 模块
- 最终的 [`aptos_std::ristretto255_elgamal`](https://github.com/aptos-labs/aptos-core/blob/main/aptos-move/framework/aptos-stdlib/sources/cryptography/ristretto255_elgamal.move) 模块
- 最终的 [`aptos_std::ristretto255_pedersen`](https://github.com/aptos-labs/aptos-core/blob/main/aptos-move/framework/aptos-stdlib/sources/cryptography/ristretto255_pedersen.move) 模块

### 1. 实现细节

- 对 `ristretto255` 模块的更改继续依赖于 [`dalek-cryptography/curve25519`](https://github.com/dalek-cryptography/curve25519-dalek) crate。

- 使用 [`dalek-cryptography/bulletproofs`](https://github.com/dalek-cryptography/bulletproofs) crate 的 [`zkcrypto/bulletproofs`](https://github.com/zkcrypto/bulletproofs) 分支实现了 Bulletproofs 模块。 （不幸的是，`dalek-cryptography/bulletproofs` crate 依赖于一个更旧版本的 `curve25519-dalek` 库，因此无法使用。）
  - Bulletproofs 零知识范围证明验证器实现为一个**Move原生函数**

- ElGamal 加密和 Pedersen 承诺模块直接在 Move 中实现，**没有使用原生函数**，而是在 `ristretto255` Move 模块上进行，展示了其强大性。

## 七、风险和缺点

> 在此处阐明承担此提案可能带来的潜在负面影响。有哪些危险？

### 1. 过时性

在实现加密基础设施时，未来可能淘汰现有技术始终是一种风险。就目前而言，**Ristretto255** 组和 **Bulletproofs** 证明系统已经被使用了数年，这使得它们在可预见的未来继续被采用的可能性较大。同样，ElGamal 加密和 Pedersen 承诺这两种极具灵活性的流行的加密系统，已经广泛应用了多个十年。

## 八、未来潜力

> 深入思考这个提案在未来的演变。你如何看待这个提案的发展？在一年后，五年后，这个提案会带来什么结果？

["动机"](## Motivation) 和 ["基本原理"](## Rationale) 部分已经总结了未来可以在这些模块上开发的许多应用程序。

## 九、时间表

### 1. 建议的实施时间表

> 描述你预计实施工作需要多长时间，可能将其分成阶段或里程碑。

实施工作已经完成；请参阅 [PR 3444]()。

### 2. 建议的开发者平台支持时间表

> 描述有关此功能的 SDK、API、CLI、索引器支持的计划，如果适用。

不适用。

### 3. 建议的部署时间表

> 社区应该何时期望在开发网络上看到此功能部署？

现在应该已经可用：例如，查看 [Bulletproofs 模块](https://explorer.aptoslabs.com/account/0x1/modules/code/ristretto255_bulletproofs?network=devnet) 的开发网络资源视图。

> 在测试网络上？

版本 1.7。
> 在主网上？

版本 1.7。

## 十、安全考虑

> 此变更是否已由任何审计公司审计？

没有，因为所提议的更改要么是：

- 包装现有**经过审计**的加密库的新原生。
- 用于**简单**加密原语的新模块：即 Pedersen 承诺和 ElGamal 加密的实现都很简单。

> 有潜在的诈骗风险吗？有什么缓解策略？

没有潜在的诈骗风险。

> 是否有安全考虑？

值得澄清的是，Bulletproofs ZK 范围证明系统的安全性。

Bulletproofs 的安全性可以在计算某些素数阶群中的**离散对数**和**随机预言模型（ROM）**的困难性下证明，这是由于其使用了 Fiat-Shamir 转换[^fiatshamir]。

这些是用来证明 Aptos 和许多其他区块链中使用的 **Ed25519 签名方案** 的安全性的相同假设。因此，这个 AIP 不引入额外的加密假设。

话虽如此，这并未考虑到我们在 [“实现细节”](#implementation-details) 中使用的底层库中的加密错误。在这里，我们可以更加放心，因为 `dalek-cryptography` 库 [过去已经经过审计](https://blog.quarkslab.com/security-audit-of-dalek-libraries.html)（详细报告请见 [这里](https://blog.quarkslab.com/resources/2019-08-26-audit-dalek-libraries/19-06-594-REP.pdf)）。

> 是否有可以分享的安全设计文档或审计材料？

没有。

## 十一、测试

> 有什么测试计划？这是如何测试的？

- 在上面链接的 `ristretto255` 和 `bulletproofs` 实现中编写了多个单元测试。

- ElGamal 和 Pedersen 模块在 [veiled coin](https://github.com/aptos-labs/aptos-core/blob/main/aptos-move/move-examples/veiled_coin/sources/veiled_coin.move) Move 示例模块中隐式测试。

## 十二、参考文献

[^bulletproofs]: **Bulletproofs: 简洁证明用于机密交易等场景**, B. Bünz 等人，*2018 IEEE Symposium on Security and Privacy (SP)*, 2018, [[URL]](https://ieeexplore.ieee.org/document/8418611)
[^elgamal]: **基于离散对数的公钥密码系统和签名方案**, ElGamal, T., *IEEE信息理论交易*, 1985
[^fiatshamir]: **如何证明你自己：用于身份验证和签名问题的实用解决方案**, Fiat, Amos 和 Shamir, Adi, *加密学进展 --- CRYPTO' 86*, 1987
[^pedersen]: **非交互式和信息论安全的可验证秘密共享**, Pedersen, Torben Pryds, *密码学进展会议论文集*, 1992
[^provisions]: **条款**, Gaby G. Dagher 等人，*第22届ACM SIGSAC计算机与通信安全会议论文集*, 2015, [[URL]](https://doi.org/10.1145%2F2810103.2813674)
[^ristretto255]: [https://ristretto.group](https://ristretto.group)