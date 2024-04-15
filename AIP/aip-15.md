---
aip: 15
title: token 标准保留属性
author: areshand
discussions-to: https://github.com/aptos-foundation/AIPs/issues/28
Status: Accepted
last-call-end-date (*optional):
type: Standard (framework)
created: 01/06/2023
---

[toc]

# AIP 15 - token 标准保留属性



## 一、概述

这项提案引入了框架保留属性，旨在： 

1. 防止创建者随意修改 token 框架中已使用的属性。 
2. 留出一部分属性，以便后续扩展以支持可编程的 token 行为，例如令牌冻结（token freezing）、灵魂绑定令牌（soulbound token）等。

## 二、动机

在我们的 [Token 标准](https://aptos.dev/concepts/coin-and-token/aptos-token#token-burn)中定义了一些属性，用于控制谁有权限销毁 Token 。不过，如果 Token 的默认属性可以修改，那么创作者在 Token 铸造之后就有可能新增这些控制属性。结果就是，创作者有可能销毁收藏家（collector）手中的 Token 。这在 Token 标准中被指出为一个已知问题，并推荐将令牌的默认属性设置为不可变作为一项最佳实践。为了彻底预防这一问题，本提案旨在确保在创建令牌之后无法更新这些控制属性。 

框架保留的属性可以用于控制 Token 行为，使其具备可编程性。例如，可以设定一个框架保留的属性，在令牌存储中冻结 token（freeze tokens）。

## 三、规范

我们有 3 个现有的控制属性：

- `TOKEN_BURNABLE_BY_CREATOR`
- `TOKEN_BURNABLE_BY_OWNER`
- `TOKEN_PROPERTY_MUTATBLE`

当这 3 个属性存在于 TokenData 的 default_properties 中时，创建者可以使用它们来控制销毁和更改的权限。我们希望以此防止创建者在 token 创建后更改这些框架的保留属性。

为框架使用惯例，标准保留了带有`TOKEN_`前缀的所有键。当创建者更改存储在 token 或 TokenData 中的 token_properties 或 default_properties 时，我们将检查属性名称是否以`TOKEN_`开头，并且，如果任何属性以`TOKEN_`前缀开头，则中止更改。

我们添加了友元函数（friend functions） `add/update_non_framework_reserved_properties` 到 property map 模块。该函数将检查要添加/更新的所有属性，并且只有非框架保留属性可以被添加或更新。

```rust
// 这个特定的函数将由修改令牌属性的功能调用，用于添加或更新令牌的属性。
public(friend) fun add_non_framework_reserved_properties(map: &mut PropertyMap, key: String, value: PropertyValue)
public(friend) fun update_non_framework_reserved_properties(map: &mut PropertyMap, key: String, value: PropertyValue)
```

## 四、风险和缺陷

**令牌默认属性变更的额外成本** 

在进行 `mutate_tokendata/token_property` 方法调用时，对框架预留属性的验证将消耗额外的 Gas 费用。目前，当创作者改动属性时，该函数会检查所有待修改的属性并更新其值。新增的成本在于我们需要验证属性键是否是以 `TOKEN_` 开头。通过采用子字符串匹配和及时终止比较，这部分额外的费用可以保持在最低水平。而且，目前的方法已经在字符串长度、键重复等方面进行了诸多验证，所以增加对属性键前缀的检查，对 Gas 成本和用户体验的影响应该可以忽略不计。

## 五、时间表

这项更改尚未实施。理想情况下，这些更改应在 1 到 2 周内被合并到主分支，然后是 testnet ，最后是mainnet 。

## 六、未来潜力

这将为基于此框架保留属性的关注 AIP（如 soulbound token 和 token freezing）铺平道路。

这为依赖这些框架预留属性的绑定灵魂令牌（soulbound token）和令牌冻结（token freezing）相关后续 AIP (区块链改进提案) 铺平了道路。

## 七、参考资料

- [token 标准](https://aptos.dev/concepts/coin-and-token/aptos-token/)