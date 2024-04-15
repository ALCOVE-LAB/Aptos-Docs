---
aip: 0
title: Aptos 优化提案
authors: aptos 基金会
status: 已接受
type: 信息
created: 12/05/2022
updated: 12/07/2022
---

[TOC]

# AIP-0 - Aptos Improvement Proposals（Aptos 优化提案）

Aptos Improvement Proposals（AIP）描述了 Aptos 网络的标准，包括核心区块链协议和开发平台（Move），智能合约及用于智能合约验证的系统，Aptos 网络的部署和运行标准，用于访问 Aptos 网络并处理来自 Aptos 网络的信息的 API。

## 一、AIP 流程

正式的 AIP 流程通常（建议）在提案的发起人已经与 Aptos 社区和 Aptos 基金会讨论并推广了提案之后开始。它包括以下步骤：

  * **想法（Idea）** – 作者将通过撰写 GitHub Issue 并获得反馈，与开发者社区和 Aptos 维护者们交流自己的想法。如果可能（且相关），作者应在讨论中提供支持其提议的实现。

    一旦讨论进入成熟阶段，可向 aptos-foundation/AIPs 文件夹提交拉取请求，正式的 AIP 流程就会开始，。此时文件的状态栏应为 "草案（Draft）"。AIP 编号与上述分配的初始提案中的问题编号相同。AIP 管理员将审核/同意/批准/拒绝拉取请求，将 AIP 草案提交到 AIP repo 需要两个 maintainer 的批准。

    * ✅ 草案（Draft） – 如果同意，AIP 管理人员将批准拉取请求。
    * 🛑 草稿（Draft） – 拒绝草稿状态的原因包括：与 Aptos 使命或 Aptos 基金会政策不符、重点不突出、过于宽泛、重复劳动、技术上不合理、未提供适当的动机或未解决向后兼容性问题。作者可以尝试完善并重新提交其 AIP 想法，以供再次审核。

  * **草案（Draft）** – 草案合并后，可通过拉取请求提交更多修改。当 AIP 已经完成并稳定时，作者可以要求将状态更改为审核（Review），以便与社区更广泛地共享。

    * ✅ 审核（Review） – 如果同意，AIP 管理人员将批准审核状态，并设置一个合理的时间（通常为 1-2 周）进行社区审核以收集 Aptos 社区的反馈。 如果有需要，AIP 管理人员可以授予额外时间。
    * 🛑 审核（Review） – 如果草案仍需要重大更改，则将拒绝审核请求。

  * **审核（Review）** – 合并草案后，应该广泛与 Aptos 社区分享以收集反馈。 可以通过拉取请求提交额外的更改。 作者和维护者们应监控社区的反馈，解决可能出现的有关提案的任何疑虑和问题。

    * ✅ 已接受（Accepted） – 成功的审核没有任何重大更改或未解决的技术问题将变为已接受状态。 此状态表示不太可能发生重大更改， Aptos Maintainers 应支持推动此 AIP 以纳入生态。
    * 🛑 已拒绝（Rejected） – 如果草案仍需要重大更改，或者社区有强烈反馈要拒绝它，或者在 AIP 中发现了一个重大，且无法纠正的缺陷。

  * **已接受（Accepted）** –  处于已接受状态的 AIP **意味着 Aptos 维护者们已确定该 AIP 可供积极实施**

    * ✅ 最终（Final） – AIP 部署到主网。 当实施完成并部署 / 投票到主网时，状态将更改为 “最终”。
    * 🛑 草案（Draft） –  如果有了新发现（new information），可将已接受的 AIP 移回草案状态，进行必要的修改。
    * 🛑 已弃用 – 如果 AIP 被后续提案取代或变得不相关，则 AIP 维护者们可能会标记为已弃用。

  * **最终（Final）** – AIP 的最终状态意味着 AIP 的必要实现已经完成，并已部署到 Aptos 主网。该 AIP 代表了当前最先进的技术。最终 AIP 只应更新以纠正错误。

一个 AIP 可以关联相关 / 依赖的 AIP。每个 AIP 在发展过程中将被分配一个状态标签。在下一阶段之前，每个阶段都可能有多次修订/审核。

原文链接：[https://github.com/aptos-foundation/AIPs/blob/main/aips/aip-0.md](https://github.com/aptos-foundation/AIPs/blob/main/aips/aip-0.md)