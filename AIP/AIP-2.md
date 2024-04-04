|aip|标题|作者|讨论|状态|最后讨论时间（*可选）|类型|创建时间|
| -----| ----------------------------------------| ----------| ------| --------| -----------------------| --------------| ------------|
|1|Multiple Token Changes（多个代币更改）|areshand|[#2](https://github.com/aptos-foundation/AIPs/issues/2)|已接受||标准（框架）|2022/07/12|

# AIP-2 - Multiple Token Changes（多个代币更改）

## 摘要

本提案包含以下更改：

1. 代币集合数据（CollectionData）突变函数：根据代币突变性设置，提供集合函数来突变代币数据字段
2. 代币 TokenData 元数据突变函数：根据集合突变性设置，提供集合函数来突变 CollectionData 字段
3. 修复集合供应下溢的错误：修复在燃烧无限制集合的 TokenData 时导致下溢错误的错误
4. 修复事件的顺序：在铸造代币时，存入（deposit）事件先于铸造（mint）事件进入队列。此更改可纠正顺序，使代币存入事件排在铸造事件之后
5. 为 opt-in 转账提供公共入口：提供一个入口函数，允许用户在 opt-in 直接转账时直接转移代币
6. 确保版权费（royalty）分子小于分母：我们观察到约 0.004% 的代币版税大于 100%。这一改动引入了一个断言，以确保版税总是小于或等于 100%。

### 动机

更改 1、2：动机是支持基于可变性配置的 CollectionData 和 TokenData 字段变更，以便创建者可以根据自己的应用逻辑更新字段，以支持新的产品功能

更改 3、4：动机是修复现有问题，使代币合约正常工作

更改 5：目的是允许 dapp 直接调用函数，而无需部署自己的合约或脚本

更改 6：这是为了防止潜在的恶意代币收取高于代币价格的费用。

## 理由

更改 1、2 是对现有代币标准规范的补充，没有引入新的功能

更改 3、4、5 和 6 是小修小补和硬性更改

## 参考实施

上述变更的 PR 如下：

变更 1, 2：[aptos-labs/aptos-core#5382](https://github.com/aptos-labs/aptos-core/pull/5382) [aptos-labs/aptos-core#5265](https://github.com/aptos-labs/aptos-core/pull/5265) [aptos-labs/aptos-core#5017](https://github.com/aptos-labs/aptos-core/pull/5017)

更改 3：[aptos-labs/aptos-core#5096](https://github.com/aptos-labs/aptos-core/pull/5096)

更改 4：[aptos-labs/aptos-core#5499](https://github.com/aptos-labs/aptos-core/pull/5499)

更改 5：[aptos-labs/aptos-core#4930](https://github.com/aptos-labs/aptos-core/pull/4930)

更改 6：[aptos-labs/aptos-core#5444](https://github.com/aptos-labs/aptos-core/pull/5444)

## 风险和缺点

变更 1、2 经过内部审核和审计，以全面审查风险

变更 3、4 和 6 是为了降低已确定的风险和缺点

变更 5 是为了提高可用性，此变更不引入新功能

### 时间表

参考实施变更将于 11/14（太平洋标准时间）部署到 devnet，以便于测试和提供反馈/讨论。本 AIP 将在 11/17 前公开征求公众意见。讨论结束后，参考实施更改将于 11/17 部署到 testnet 中进行测试。



原文链接：[https://github.com/aptos-foundation/AIPs/blob/main/aips/aip-2.md](https://github.com/aptos-foundation/AIPs/blob/main/aips/aip-2.md)
