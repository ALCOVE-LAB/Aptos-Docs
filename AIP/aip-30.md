---
aip: 30
title: 实施降低质押奖励
author: michelle-aptos, xindingw, junkil-park
discussions-to (*optional): https://github.com/aptos-foundation/AIPs/issues/119
Status: 已接受
last-call-end-date (*optional): <mm/dd/yyyy 最后一天留下反馈和评论>
type: 框架
created: 5/3/2023
updated (*optional): 7/28/2023
---

[TOC]

# AIP-30 - 实施降低质押奖励

## 一、概述

在[Aptos token 经济概览](https://aptosfoundation.org/currents/aptos-tokenomics-overview)中，Aptos Foundation 介绍了随时间变化的预期 token 供应变化。目前，最高的质押奖励率是一个恒定的年化率 $7\%$；该AIP提议每年将质押奖励降低1.5%，以符合Aptos的 token 经济：

- 最大奖励率每年下降1.5%，直到年化率降至3.25%的下限（预计需要超过50年）。

例如：
- 第1年的最大奖励率（年从起源时间戳2023/10/12开始）：$7\%$
- 第2年的最大奖励率：$7\% * (100\%-1.5\%) = 6.895\%$
- 第3年的最大奖励率：$7\% * (100\%-1.5\%)^2 = 6.791575\%$
- ...
- 第51年的最大奖励率：$7\% * (100\%-1.5\%)^50 \approx 3.28783\%$
- 第52年的最大奖励率：$max(3.25\%, 7\% * (100\%-1.5\%)^{51}) = 3.25\%$​



## 二、动机

完全符合[Aptos token 经济概览](https://aptosfoundation.org/currents/aptos-tokenomics-overview)。



## 三、理由

**考虑因素：**

1. 年份从创世日期开始：基于时间戳（10/12）
2. 每年末尾发生 1.5% 的减少。这意味着在当前奖励率（7%）下，年底的有效奖励率将为 $6.895\%$（7%-1.5%*7%）

**备选方案：**

1. 我们可以在一年中逐渐降低奖励（例如，每30天），但这将使奖励计算更加复杂。



## 四、参考实现

[https://github.com/aptos-labs/aptos-core/pull/7867](https://github.com/aptos-labs/aptos-core/pull/7867)



## 五、未来潜力

所有奖励和奖励机制也可以通过链上治理进行修改



## 六、建议的实施时间表

目标为2023年第三季度末
