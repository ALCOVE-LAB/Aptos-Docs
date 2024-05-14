---
aip: 73
title: 可分派代币标准
author: 
  - name: Runtian Zhou
discussions-to: "https://github.com/aptos-foundation/AIPs/issues/374"
status: 审核中
last-call-end-date: 04/08/2024
type: 框架
created: 2024-03-08
---

[TOC]

# AIP-73 - 可调度 token 标准

## 一、摘要

目前，Aptos 框架定义了一个单一的 `fungible_asset.move` 作为我们的可替代资产标准，这使得其他开发者难以自定义他们所需的逻辑。通过这项 AIP，我们希望开发者能够自定义他们自己的取出和存入方式 —— 允许以一种可扩展性更好的方式使用我们的 Aptos 框架。



### 1. 目标

该 AIP 的目标是允许第三方开发者在可替代资产的存取过程中加入自他们的自定义逻辑。这将允许以下情况：

1. 通缩 Token ：在转移时销毁固定比例的 Token 。
2. 转账白名单： token 只能转账到白名单上的地址。
3. 预测转移：只有在满足某些特定条件时才能进行转移。
4. 忠诚度 Token ：在可替代资产发生转移时，将向指定地址支付固定的忠诚度奖励。

请注意，上述所有逻辑都可以由 Aptos 上的任何开发者开发和扩展！这将极大地增加我们框架的可扩展性。



### 2. 不在讨论范围内的内容

我们不会修改任何核心 Move VM / 文件格式逻辑。我们将使用此 AIP 作为我们计划在未来的 Move 版本中支持的动态 / 静态分派的前期工作。

此处的 AIP 可能也适用于我们的 NFT 标准。然而，在此 AIP 的范围内，我们不会考虑这样的情况。



## 二、动机

目前，Aptos Framework 管理着可替代资产的全部逻辑，每个 DeFi 模块都需要静态链接到这样的模块上。我们可能无法满足来自所有开发人员的各种功能需求，因此，在我们的网络上，一个可扩展的可替代资产标准是必不可少的。



## 三、影响

我们希望允许代币开发者在代币提取和存入过程中注入自定义逻辑，以提供更灵活的操作方式。这将对我们的 DeFi 开发人员产生一些相关影响。



## 四、替代解决方案

我们正在使用这个 AIP 作为未来在 Aptos 上 Move 语言中支持调度的前期工作。因此，我们将通过一个原生函数而不是在 Move 编译器和 VM 中进行全面更改来实现一个有限作用域的调度函数（function），这样我们就有更多时间评估 Move 中调度逻辑的安全影响。

作为 `overloaded_fungible_asset.move` 的一个备选方案，我们可以考虑将调度功能直接集成到现有的 `fungible_asset.move` 中。不过，按照当前提出的运行时规则，这样做初始使用起来会非常不便。要使调度功能正常工作，我们需要对运行时的安全规则作出一定的调整，以允许重入 `fungible_asset.move`。这意味着框架的开发者需要格外警惕重入可能带来的问题。




## 五、规范

我们将在 Aptos Framework 中添加两个模块。

- `function_info.move`。该模块将模拟一个运行时函数指针，可用作分派。该模块将如下所示：
```
struct FunctionInfo has copy, drop, store {
    module_address: address,
    module_name: string,
    function_name: string,
}

public fun new(
    module_address: address,
    module_name: string,
    function_name: string,
): FunctionInfo

// 检查 lhs 的函数签名是否等于 rhs。
// 这可以作为类型检查器，以确保分派函数的类型与分派函数相同
public(friend) native fun check_dispatch_function_info_compatible(
    lhs: &FunctionInfo,
    rhs: &FunctionInfo,
): bool;
```

- `overloaded_fungible_asset.move`。该模块将作为可替代资产的新入口点。该模块将作为我们现有的 `fungible_asset.move` 的包装模块，并具有类似的 API 。之所以需要一个额外的模块而不是在 `fungible_asset.move` 中添加分派逻辑，是因为下面提到的运行时规则。该模块将具有以下 API：
```
#[resource_group_member(group = aptos_framework::object::ObjectGroup)]
struct WithdrawalFunctionStore has key {
    // 每个不同的 Metadata 可以有一个断言（predicate）
    function: FunctionInfo
}

// 基于第一个参数的可分派调用。
//
// MoveVM 将使用 FunctionInfo 来确定分派目标。
native fun dispatchable_withdraw(
    function: &FunctionInfo,
	// 安全讨论：我们是否需要在这里添加 owner 字段？
    owner: [address,&signer],
    store: Object<T>,
	amount: u64,
): FungibleAsset;

// withdraw 的调度版本。此提款将调用断言函数，而不是 fungible_asset.move 中的默认提款函数
public fun withdraw<T: key>(
    owner: &signer,
    store: Object<T>,
    amount: u64,
): FungibleAsset acquires FungibleStore

// 存储函数信息，以便 withdraw 可以调用自定义的 dispatchable_withdraw
public fun register_withdraw_epilogue(
    owner: &ConstrutorRef,
    withdraw_function: FunctionInfo,
)
```

在 Move 虚拟机中也将引入新的运行时检查：

- 对于调用堆栈上的每一个新层次，MoveVM 需要保证这个函数不会在调用关系图中创建循环依赖。具体来说，假设模块 A 的一个函数调用了模块 B，相当于跳出了原本的作用域，在模块 B 的这个函数返回前，即我们没有回到模块 A 的作用域之前，是不允许再有任何对模块 A 里函数的调用的。

由于此 AIP 可能会启用潜在的重新进入问题，因此需要此运行时检查。此检查不会导致链上任何现有 Move 程序失败。有关为什么需要此类运行时检查的安全讨论，请参见安全讨论。



## 六、参考实现

尚未实现。



## 七、风险和缺点

这里最大的风险是调度逻辑可能引入的潜在重新进入问题。有关详细信息，请参阅安全考虑部分。



## 八、安全考虑

### 1. Move 当前的重新进入和引用安全状态
最大的安全关注点是此举可能如何改变 Move 的重新进入和引用安全性。在我们深入探讨问题之前，让我们先来看看几个 Move 的设计目标：

1. 引用安全性：在任何给定时间，对任何值只能有一个可变引用，或多个不可变引用。
2. 重新进入安全性：重新进入攻击发生在合同在单个交易中被多次调用时，可能导致合同在完成先前操作之前重新进入自身。这可能会导致意外行为，并可能利用合同逻辑中的漏洞，使恶意行为者能够操纵资金或干扰合同的预期操作。

请注意，这两个属性是由 Move 字节码验证器强制执行的，它是在模块发布时进行的静态分析，因此任何违反这些属性的模块都将在模块发布时立即被拒绝。问题是我们如何实际推断 Move 字节码验证器中的这两个安全性属性？

Move 字节码验证器所做的第一个重要假设是：任何 Move 程序的依赖图必须是无环的，这意味着两个模块不能相互直接或间接地依赖。在模块发布时会进行特定的检查来验证这个属性。这导致了一个重要的观察结果：如果模块 A 调用另一个模块 B 中定义的函数，这样的函数将无法调用定义在模块A 中的任何函数，因为有这个无环属性。所以考虑以下程序：

```move
public fun supply(): u64 acquires Lending {
  borrow_global<Lending>(@address).supply
}

public fun borrow(amount: u64) {
  let supply = supply();
  
  // 调用其他模块的函数
  another_module::bar();
  
  supply += amount;
  set_supply(supply)
}
```

由于非循环性，Move 字节码验证器知道 `another_module::bar()`无 法以任何方式调用可能改变 `Lending` 资源中的`supply`字段的其他函数。因此，改变`Lending`资源的唯一方式是调用在你自己的模块中定义的函数，而移动字节码验证器将执行静态分析以确保不会有两个可变引用。具体来说，我们可以查看以下示例：

```
module 0x1.A {
    import 0x1.signer;
    struct T1 has key {v: u64}
    struct T2 has key {v: u64}

    // 所有有效的获取

    public test1(account: &signer) acquires T1, T2 {
        let x: &mut Self.T1;
        let y: &mut u64;
    label b0:
        x = borrow_global_mut<T2>(signer.address_of(copy(account)));
        _ = move(x);
		// 获取 T2 是安全的，因为 x 已被丢弃。
        Self.acquires_t2(copy(account));
        return;
    }

    public test2(account: &signer) acquires T1, T2 {
        let x: &mut Self.T1;
        let y: &mut u64;
    label b0:
        x = borrow_global_mut<T1>(signer.address_of(copy(account)));
		// 获取 T2 是不安全的，将被字节码验证器拒绝
        Self.acquires_t2(copy(account));
        return;
    }

    public test3(account: &signer) acquires T1, T2 {
        let x: &mut Self.T1;
        let y: &mut u64;
    label b0:
        x = borrow_global_mut<T1>(signer.address_of(copy(account)));
		// 调用外部函数是安全的，因为有无循环属性。
        aptos_framework::....
        return;
    }

    acquires_t2(account: &signer) acquires T2 {
        let v: u64;
    label b0:
        T2 { v } = move_from<T2>(signer.address_of(move(account)));
        return;
    }
}
```

在上述所有测试函数中，一旦借用了可变引用，字节码验证器将确保只有在第一个可变引用被丢弃后，才能借出后续的引用。在`test1`中，调用`acquires_t2`是允许的，因为可变引用已经被丢弃。然而，在`test2`中，调用`acquires_t2`将被严格禁止，包含此类代码的模块将无法发布，因为当`acquires_t2`试图获取另一个引用时，可变引用仍然被持有。然而，在`test3`中，由于 Move 依赖关系的非循环性，Move 字节码验证器可以静态地假设这个函数调用将无法调用能够生成你当前持有状态的引用的函数。因此，字节码验证器在静态分析期间将简单地将此调用视为无操作。

### 2. 调度逻辑的任何更改会如何改变这里的情况？

调度过程中最大的假设是，Move 字节码验证器不能再假设一个函数只能调用那些已经被发布的其他函数。因此，对 Move 至关重要的引用安全属性和重入属性所依赖的重要非循环性将被破坏。考虑以下示例：

```
public fun borrow(amount: u64) {
  let supply = borrow_global_mut<Lending>(@address).supply
  
  // 调用分派版本的可互换资产。
  //
  // MoveVM 将控制流导向上面提到的 `dispatchable_withdraw` 函数。
  aptos_framework::overloadable_fungible_asset::withdraw()
  
  supply += amount;
  set_supply(supply)
}

public fun dispatchable_withdraw(...) {
    // 创建了两个可变引用
	let supply_2 = borrow_global_mut<Lending>(@address).supply
    
}
```
在此示例中，字节码验证器不知道调用 `aptos_framework::overloadable_fungible_asset::withdraw()` 将返回到同一模块中定义的 `dispatchable_withdraw` 函数。因此，它不知道当 `supply_2` 被借用时，`supply_1` 中已经存在一个可变引用，这实际上破坏了 Move 的引用安全假设。

以下是另一个关于重新进入的问题的示例：

```
#[view]
public fun supply(): u64 acquires Lending {
  borrow_global<Lending>(@address).supply
}

public fun borrow(amount: u64) {
  let supply = supply();
  
  // 调用其他模块中的函数
     ()
  
  supply += amount;
  set_supply(supply)
}

public fun dispatchable_withdraw(...) {
    // 变更 supply 字段
    ...
}
```

在这一场景下，Move 保持了引用安全性，因为在调用 `supply()` 之后，`Lending` 的引用就已经被释放了。尽管如此，代码还是有潜在的问题。根据当前 Move 的设计，调用其他模块中的函数是无法修改你所关注的状态的。因此，你只需考虑原生函数可能对这些状态造成的变化。但是，一旦引入了调度机制，这样的假设将不再成立。这将显著增加智能合约开发者在理解代码重入特性时的难度。

### 3. 本提议的解决方案：对循环依赖进行新的运行时检查

上述分析中，我们阐明了非循环结构假设对 Move 静态引用安全性分析及其重入特性的关键影响。考虑最糟糕的情况，开发者可能会创建针对同一全局变量的多个可变引用，而 Move 的字节码验证程序却未能发出警告。为了降低风险，我们建议需要在运行时加强这一原则的执行。也就是说，函数在调用依赖关系图中不能创建[[后向边（Back Edge）]]的依赖。就重入问题而言，具体的调用栈结构将会如下所示：


```
[
    some_module::borrow,
    aptos_framework::overloadable_fungible_asset::withdraw,
    some_module::dispatchable_withdraw,          <-- 形成了一个循环。因为 `some_module` 已经在调用堆栈的顶部
]
```

当将 `dispatchable_withdraw` 推送到调用堆栈时，运行时规则将导致程序中止。其他区块链系统也对模块级重新进入问题进行类似的运行时检查。需要注意的一点是，此检查在任何现有的 Move 代码上都不会失败，因为我们在模块发布时已经检查了此属性。这样的检查只有在引入调度机制后才会失败。

这种运行时检查的一个缺点是，它使得将调度功能直接集成到 `fungible_asset.move` 中变得非常困难。原因是，可以想象调度后的 `withdraw` 函数可能需要调用在 `fungible_asset.move` 中定义的函数。在通缩 token （deflation token）示例中，开发人员很可能需要调用 `fungible_asset.move` 中的分割和销毁（split and burn）函数。如果将 `withdraw` API 添加到 `fungible_asset.move` 中，则调用栈将如下所示：

```
[
    A::some_function
    aptos_framework::fungible_asset::withdraw,
    third_party_token::dispatchable_withdraw,
    aptos_framework::fungible_asset::split,     <-- 形成了一个循环。因为 `fungible_asset` 已经在调用堆栈的顶部 
]
```

这将会直接违背我们之前提出的运行时检查规则。为了解决这个问题，我的建议是将调用入口迁移到一个名为 `overloadable_fungible_asset.move` 的新模块中，取代 `fungible_asset.move` 中原有的 `withdraw` API。另一个解决方案是让 `fungible_asset.move` 模块成为一个不受此检查影响的特例。这要求框架开发人员必须考虑 `fungible_asset.move` 是否存在重入性问题，这在先前并不是什么难题。



## 九、未来潜力

我们将利用从此 AIP 中学到的经验，帮助实现 Move 中提议的高阶函数系统，如 [Aptos 上的 Move 的未来](https://medium.com/aptoslabs/the-future-of-move-at-aptos-17d0656dcc31#e1b1)



## 十、时间表

### 1. 建议的实施时间表

我们计划在即将发布的版本中实现此功能。

### 2. 建议的开发人员平台支持时间表

> [!TIP]
>
> 原作者注：无

### 3. 建议的部署时间表

我们希望在 1.11 版本中实现它。

 > 在开发网络上？
 > 在测试网络上？
 > 在主网上？


> [!TIP]
>
> 原作者注：...


## 十一、开放问题（可选）

我们需要一些关于 `overloaded_fungible_asset.move` 中模块的公共接口的反馈。看看这如何与 `fungible_asset.move` 正确工作。
