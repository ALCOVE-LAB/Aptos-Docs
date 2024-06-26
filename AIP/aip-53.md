---
aip: 53
title: 在交易认证器中使费用支付者地址可选
author: davidiw
discussions-to: https://github.com/aptos-foundation/AIPs/issues/257
Status: Draft
last-call-end-date (*optional): <mm/dd/yyyy 最后留下反馈和评论的日期>
type: Framework
created: 10/08/2023
updated (*optional): <mm/dd/yyyy>
requires (*optional): 39
---

[TOC]

# AIP-53 - 在交易认证器中使费用支付者地址可选

## 一、改善

当向区块链提交交易时，客户端用户可能不知道谁是实际的费用支付者。当前生成这些交易的方式要求客户端用户知道谁将为提交到区块链的每个交易支付手续费。这个AIP提议使手续费支付者的知识变成可选，从而提供更好的用户和开发者体验。

当用户向区块链提交接受代付（sponsored）的交易时，他们可能并不清楚谁会支付这笔交易的费用。根据目前的流程，用户在提交交易时必须指定费用支付方。这个改进提议（AIP）旨在让用户无需提前知道费用支付方的身份，此举旨在提供更便捷的用户和开发者体验。

### 1. 目标

这个 AIP 使客户端更容易提交代付交易，并且不知道最终是谁支付了它。因此，这将大大改善 Aptos 上的开发者体验。



## 二、替代方案

尽管目前有些讨论专注于重塑交易的结构设计，以修正 Aptos 代码基础在漫长发展历程中产生的某些技术问题，但相关的改进提案还没有正式发布，并且实现这些提案将需要大量的努力才能使其广泛采用。此外，为了完成这些更新，Aptos 软件系统包括开发工具集 (SDKs)、索引器（ Indexers）和 Move 编程语言中的交易与代码等多个环节都需进行修改，这将需要几个月的从头到尾的工程开发时间，且这还没考虑到推广这样的更新可能带来的各种社会层面的挑战。



## 三、规范

当前，在签署交易时，费用支付者的地址包含在要签署的数据中。因此，所有参与方必须首先知道费用支付者的地址，然后才能提交交易。

本 AIP 规定，对于非费用支付者的签署者，费用支付者的地址可以设置为实际地址，也可以设置为 `0x0`，即 Move 中未分配的地址。费用支付者仍然必须签署其地址以确保其愿意成为费用支付者。因此，在验证过程中，除了费用支付者的签名外，还必须将每个签名与使用实际费用支付者地址和 `0x0` 的原始交易进行比较。



## 四、参考实现

[参考实现](https://github.com/aptos-labs/aptos-core/pull/10443)



## 五、测试

通过端到端测试进行验证。



## 六、风险和缺点

由于现在需要检查许多格式，代付交易可能会略微降低性能。这些成本通常较低，最坏的情况下可能会使验证成本翻倍。由于大多数 SDK可 能会遵循使用 `0x0 `的更简单的方法，因此这最终将成为可以忽略的成本。



## 七、安全考虑

在现有模型里，费用支付者的身份对于所有人来说都是公开的。这种做法似乎提供了一种安全保障，但实际上，由于交易的其他组成要素本身已经保证了其唯一性，因此是否指定一个明确的费用支付者地址对交易的签署来说，并不是必须的。

然而，这确实使用户能够生成一个单独的交易，然后由不同的费用支付者承担。从可见的角度来看，这与今天存在的情况并没有太大差别。在任何情况下，Aptos Mempool 的设计都会阻止具有相同序列号的两个交易被注册或执行，因此从这个变化产生的问题不会发生，既然任何多余的交易都将在收取费用支付者的费用之前被丢弃。



有可能有两个手续费支付者竞争支付交易费用。当前的 mempool 设计一次只支持一个变体。因此，恶意实体可能会拦截交易并用错误的手续费支付者替换它。结果，交易可能在共识中被丢弃，导致一种形式的审查制度。尽管由于固有的信任模型和网络动态性，这种可能性很小，但仍然存在风险。具体来说，通常将交易提交给全节点，这些节点被信任将交易转发给验证器全节点，后者也有相同的期望 -- 设置一个无效的手续费支付者并转发比丢弃交易更多的工作。因此，建议应用程序开发人员在可能的情况下指定手续费支付者，而不是利用可选的手续费支付者模型。

两个费用支付者有可能同时尝试支付同一笔交易费用。当前的内存交易池 (mempool) 设计在任何时候仅支持一个费用支付者。因此，恶意攻击者有可能截获这笔交易，并将其换成一个设定错误的费用支付者。这样一来，该交易可能会因共识验证失败而被网络拒绝，相当于一种交易屏蔽。尽管由于信任模型和网络动态的特性，这种情况发生的概率并不大，但这仍然是一种潜在风险。通常而言，交易是提交给全节点（fullnode）的，这些全节点负责将交易转发给验证节点；验证节点也期望同样的处理 —— 设定一个无效的费用支付者并进行转发的工作量要比简单丢弃交易大得多（将一个无效的手续费支付者设置为交易的发起者，并将其转发到区块链网络中去处理，会比直接丢弃该交易更加耗费资源和工作量）。因此，建议应用程序开发者尽可能指定费用支付者，特别是在可选费用支付者并非必需的场景下。



## 八、时间表

本 AIP 的预期时间表是1.8版本。