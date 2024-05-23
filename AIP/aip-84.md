---
aip: 84
title: 提高 Gas 覆盖率
author: vgao1996
discussions-to:: https://github.com/aptos-foundation/AIPs/issues/427
Status: 已接受
last-call-end-date: 待定
type: 标准 (Gas)
created: 2024/03/24
updated (*optional): 2024/03/24
requires (*optional): N/A
---

[toc]

# AIP-84 提高燃 Gas 覆盖率

>[!NOTE]
>
>该提案提出了在交易执行过程中对类型创建和模块加载施加 Gas 费用的措施。它具有追溯效力，因为其解决的主要问题是一个已知的安全漏洞。



## 一、摘要

通过引入以下措施，我们提高了gas费用的应用范围：

- **类型创建所需的Gas**    
    - 字节码指令和其他执行类型实例化的操作，例如 `vec_push<T>`，会导致需要支付gas费用。 
- **依赖项的Gas与限制**    
    - 对于一项交易中可能包含的依赖项的总数量及其总大小（以字节计）会有所限制。    
    - 如果一个依赖项被交易直接或间接引用，那么这个依赖项也需要支付gas费用。   
    - 但是，框架模块的相关计算则被排除在外。

通过这些新的费用措施，我们可以实现 100% 的 Gas 覆盖率，以此确保网络计算资源的公平分配，保障了网络的安全性。



## 二、规范

以下是成本的公式：

- 每种类型创建的成本：
    - `per_type_node_cost * num_nodes(type)`
- 每个依赖项的成本：
    - `per_module_cost + per_module_byte_cost * module_size_in_bytes`

其中

- `per_type_node_cost = 0.0004  Gas 单位`
- `per_module_cost = 0.07446  Gas 单位`
- `per_module_byte_cost = 0.000042  Gas 单位`

这些参数的值通过基准测试进行校准。还应用了折扣以摊销冷热加载成本。

硬性限制：

- `max_num_dependencies = 512`
- `max_total_dependency_size = 1.2 MB`

这些限制基于主网上最大的传递闭包进行校准，并为升级留出了一些额外的空间。



## 三、不在讨论范围内的内容

- 解决模块冷热加载之间的差异
- 完美的 Gas 校准



## 四、影响

- **为类型创建所需的Gas**
  
    - 这将导致执行 Gas 成本增加，但通常可以忽略不计。
- **依赖项的 Gas **
    - **依赖项 Gas 将导致某些类型的交易成本增加，尤其是那些拥有大量依赖项但计算量相对较少的交易**。
        - 例如，套利机器人（arbitrage bots）或涉及多个协议的交换
        - 我们建议受影响的用户考虑优化各自的依赖结构。  
        - 从长远来看，我们打算引入模块的延迟加载功能，通过这种方式，只有真正被使用的部分才会计入费用，从而有望大幅降低依赖成本
- **依赖项的限制**
  - 包含过多依赖项的交易将不予执行。
    - 不过，目前主网络上的所有模块均符合提出的限制条件，因此现有模块将不受此影响。



## 五、替代解决方案

没有明显的替代解决方案 —— 虚拟机需要为其所做的所有工作收费，否则网络可能会变慢。



## 六、参考实现

- [[https://github.com/aptos-labs/aptos-core-private/pull/79](https://github.com/aptos-labs/aptos-core-private/pull/79)](类型创建的 Gas )
- [[https://github.com/aptos-labs/aptos-core/pull/12166](https://github.com/aptos-labs/aptos-core/pull/12166)](依赖项的 Gas 和限制)



## 七、测试

- 各种级别的测试（单元，集成和属性），确保实现的正确性。
- 在测试网上进行测试



## 八、风险和缺点

- 当前模型可能过于粗糙，无法解释冷热加载之间的差异。
- 如前所述，某些类型的交易可能会看到他们的 Gas 费略有增加。



## 九、未来潜力

- 更精细的 Gas 模型
- 更精确的校准



## 十、时间表

作为v1.10版本的一部分，已于4月15日通过以下治理提案启用

- [https://governance.aptosfoundation.org/proposal/69](https://governance.aptosfoundation.org/proposal/69)
