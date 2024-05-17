---
aip: 41
title: 用于公共随机数生成的Move API
author: Alin Tomescu (alin@aptoslabs.com)
discussions-to (*optional): https://github.com/aptos-foundation/AIPs/issues/185
Status: Accepted
last-call-end-date (*optional): <mm/dd/yyyy the last date to leave feedbacks and reviews>
type: Standard (Framework)
created: 06/27/2023
updated (*optional): <07/28/2023>
requires (*optional): <AIP number(s)>
---

[TOC]



# AIP-41 - 用于公共随机数生成的 Move API

**版本:** 1.3

## 一、概述

> 包含一个简短的描述，总结了拟议的更改。这应该不超过几句话。

本AIP提议引入一个名为`aptos_framework::randomness`的新的Move模块，该模块使智能合约可以**轻松**且**安全**地生成公开可验证的随机数。

已经提出的 `randomness` 模块采用了一个由 Aptos 的验证者所执行的 *链上加密随机性实现*（on-chain cryptographic randomness implementation）。然而，该实现**并不包含在**这份 AIP 的讨论范围之内，而将成为未来[另一份 AIP ](https://github.com/aptos-foundation/AIPs/blob/main/aips/aip-79.md)的重点。

本AIP唯一假设的 *链上加密随机性实现* 是**不可偏倚的**和**不可预测的**，即使被一小部分恶意的验证者（按权益weighed权重stake）控制。

> 该讨论将会影响到业务和商业价值。

我们相信，Move 智能合约内部引入易用的、安全的随机机制，能够为**随机化的dapp（randapps）**开启新的可能性：其核心功能需要一个不可干预和不可预测的熵源（entropy）（例如游戏、随机化的 NFT 空投、抽奖）。



## 二、动机

> 描述这一变化的动机。它实现了什么？

这一变化的动机有两方面：

1. 开启**随机化 dapps**的潜力，如上所定义。
2. 为 Aptos 内部的**更复杂的多方计算（MPC）**协议铺平道路（例如，时间锁加密）。

> 如果我们不接受这个提案会发生什么？

1. **没有（安全的）随机应用程序**：随机应用程序要么（1）无法在 Aptos 上轻松启用，阻碍生态系统的增长，要么（2）无法安全启用，因为在链上导入外部随机性的微妙性（请参阅下文的[“基本原理”](#基本原理)中的讨论）。
2. **创新受到抑制**：不接受这个提案可能是短视的，因为这将关闭其他区块链支持的其他MPC用例的大门，正如上面所暗示的。

## 三、影响

> 哪些受众会受到这一变化的影响？受众需要采取什么样的行动？

只有**Move开发者**会受到“影响”：他们需要学习如何使用这个新的`randomness`模块，这个模块被设计为易于理解和使用。



## 四、理解不同类型的随机性

### 1. 公共随机性与私密随机性

本 AIP 描述了一个用于生成**公共**随机性的API。
这意味着每个人都将了解到生成的随机性。
换句话说，没有办法保持生成的随机性**私密性**。
因此，需要**保密**的应用程序**不应**使用此API。

例如：
  1. 您**不应**使用此 API 生成密钥
  2. 您**不应**使用此 API 生成隐藏承诺的盲因子
  3. 您**不应**使用此 API 生成哈希的秘密原像（例如，[S/KEY](https://en.wikipedia.org/wiki/S/KEY) 类型的方案）

相反，您可以安全地使用此API公开生成随机性：

  1. 您可以使用此API在交互式零知识协议中生成Fiat-Shamir挑战
  2. 您可以使用此API公开挑选抽奖的获胜者
  3. 您可以使用此API向合格接收者列表公开分发空投

### 2. 真随机性与伪随机性

关于是否应该认为**真随机性**优于**伪随机性（密码学 cryptographic）**存在普遍误解。

所谓“真”随机性（"Real" randomness）来自宇宙中的随机事件（例如，[放射性衰变](https://www.fourmilab.ch/hotbits/)）。

但“真”随机性的问题在于，智能合约**无法验证**提供的“真”随机性是否真实。换句话说，恶意的随机性发射源（beacon）可以以任何方式偏向它所希望的"真"随机性，对结果进行操纵，而智能合约根本没办法察觉。

> 原文：
>
> But the problem with "real" randomness is there is **no way** for a smart contract **to verify** that the provided "real" randomness is indeed real. In other words, a malicious randomness beacon could bias the "real" randomness in any way it wants and there would be no way for a smart contract to detect this. 

这就是伪随机性（密码学）发挥作用的地方。

与"真"随机性不同，伪随机性是**可以在密码学上验证的（cryptographically-verifiable）**。这意味着可以在智能合约中编写代码来验证伪随机性（pseudo-randomness）是否真实。此外，在密码学假定了某些难题的前提下，伪随机性几乎无法通过验证的方式与真随机性区分开（**provably-indistinguishable** ）。简单来说，除非有其相关的密码学证明，并且可以验证其为有效的伪随机性，否则没有人能够知道它是否为真正的随机（real randomness ）。

> "provably-indistinguishable"（可证明不可区分）意味着根据已知的密码学理论，我们可以证明无法通过任何测试或方法来区分真正的随机数和伪随机数



## 五、基本原理

> 解释了为什么提出这个提案，而不是其他替代方案。为什么这是最好的可能结果？

与其为**链上随机性**提供一个专门的模块作为 Aptos 框架的一部分，我们可以提供一个**外部随机性**的 Move API，它能被用于验证来自链外信标（如[`drand`](https://drand.love)）或链上预言机（如 Chainlink）。

事实上，我们已经做到了：请参阅这个基于 [drand 的 Move 抽奖](https://github.com/aptos-labs/aptos-core/tree/ad3b32a7686549b567deb339296749b10a9e4e0e/aptos-move/move-examples/drand/sources) 的示例，该示例依赖于 `drand` 随机性信标。

尽管如此，**依赖外部信标存在几个缺点**：

1. 非常**容易误用**外部随机性信标
   1. 例如，合同编写者可能未能承诺应该使用未来某个具体的`drand` 随机数字序列中的数字，而是接受任何序列中的随机数字，这会导致一个致命的偏差攻击。
   2. 例如，在像 `drand` 这样的外部随机数生成系统中，由于系统产生随机数的时间间隔和区块链本身的时间可能存在微小差异（clock drift），开发者就必须为合约指定一个足够未来的数字序列（ a far-enough-in-the-future round #），从而确保随机性的有效应用。这意味着，在去中心化应用（dapp）可以利用这些随机数之前，将不得不等待一个额外的时间延迟。
2. **随机性太昂贵或生成速度太慢**：可以想象一个世界，在这个世界中，许多 dapp 实际上是随机化的应用程序。在这个世界中，随机性需要非常快速且非常便宜地产生，这对于现有的信标来说并不现实。
3. 外部**随机性必须通过一个 交易（TXN）被传送**到合约中，，这使得用户使用起来很别扭（例如，在上述的 [drand 的 Move 抽奖](https://github.com/aptos-labs/aptos-core/tree/ad3b32a7686549b567deb339296749b10a9e4e0e/aptos-move/move-examples/drand/sources) 示例中，某人，可能是中奖用户，必须通过发送一个含有 `drand` 随机性的交易（TXN）来“关闭”抽奖）。



## 六、规范

> 要详尽地阐述这个方案如何实施，我们需要清楚地界定一系列设计原则，这些原则在功能开发过程中必须被严格遵守。提案应该具体到让其他人能够以此为基础进一步开发，乃至创造出具有竞争力的不同版本。

我们提议为 Move 智能合约引入一个新的 `aptos_framework::randomness` 模块，用于生成可公开验证的随机性。

### 1. `randomness` API

提议中的这个模块具有简单而难以误用的接口：

这个功能模块包括了一系列可以用来随机选取不同类型数据的函数，比如整数、字节序列和随机排序等。具体来说：

- `randomness::u64_integer()` 公平地选取一个 64 位的无符号整数，确保每个数被选中的机会都是一样的。

- `randomness::bytes(n)` 公平地选取一个包含 `n` 个字节的向量。
- `randomness::permutation(n)` 返回向量 `[0, 1, 2, ..., n-1]` 的随机排列。

合约可以通过多次调用这些函数安全地采样多个对象。例如，以下代码示例中采样了两个 `u64` 和一个 `u256`：

```rust
let n1 = randomness::u64_integer();
let n2 = randomness::u64_integer();
let n3 = randomness::u256_integer();
```

完整的 `randomness` 模块如下：

```rust
module aptos_framework::randomness {
    use std::vector;
  
    /// 以相同的概率生成 `n` 个字节，字节被生成的机会完全相同。
    public fun bytes(n: u64): vector<u8> { /* ... */ }

    /// 以相同的概率生成一个随机数。
    public fun u64_integer(): u64 { /* ... */ }
    public fun u256_integer(): u256 { /* ... */ }

    /// 以相同的概率生成一个范围在 [min_incl, max_excl) 中的数字。见下文引用的“随机范围”。
    public fun u64_range(min_incl: u64, max_excl: u64): u64 { /* ... */ }
    public fun u256_range(min_incl: u256, max_excl: u256): u256 { /* ... */ }

    /* 对于 u8、u16、u32、u64 和 u128，可以添加类似的方法。 */

    /// 均匀随机生成 `[0, 1, ..., n-1]` 的排列（permutation）。
    public fun permutation(n: u64): vector<u64> { /* ... */ }

    #[test_only]
    /// 用于将 RNG 中的熵设置为特定值的测试专用（Test-only ）函数，对于测试非常有用。
    public fun set_seed(seed: vector<u8>) { /* ... */ }

    //
    // 可以在此添加更多函数以支持其他随机生成操作
    //
}
```

> 随机范围： $n \in [min_incl, max_excl)$ 

#### 1.1 示例：去中心化的抽奖

这个 `raffle` 模块一旦有一定数量的票被购买，就会随机选出一个获胜者。

```rust
module raffle::raffle {
    use aptos_framework::aptos_coin::AptosCoin;
    use aptos_framework::coin;
    use aptos_framework::randomness;
    use aptos_std::smart_vector;
    use aptos_std::smart_vector::SmartVector;
    use aptos_framework::coin::Coin;
    use std::signer;

    // 我们需要这个友元声明，这样我们的测试可以调用 `init_module`。
    friend raffle::raffle_test;

    /// 当用户尝试发起抽奖，但没有用户购买任何票时的错误代码。
    const E_NO_TICKETS: u64 = 2;

    /// 当有人试图抽取已经关闭的抽奖时的错误代码
    const E_RAFFLE_HAS_CLOSED: u64 = 3;

    /// 一张抽奖票的最低价格，以 APT 为单位。
    const TICKET_PRICE: u64 = 10_000;

    /// 一个抽奖：购买了票的用户列表。
    struct Raffle has key {
        // 购买抽奖票的用户列表（允许重复）。
        tickets: SmartVector<address>,
        coins: Coin<AptosCoin>,
        is_closed: bool,
    }

    /// 初始化 `Raffle` 资源，它将维护用户购买的抽奖票列表。
    fun init_module(deployer: &signer) {
        move_to(
            deployer,
            Raffle {
                tickets: smart_vector::empty(),
                coins: coin::zero(),
                is_closed: false,
            }
        );
    }

    #[test_only]
    public(friend) fun init_module_for_testing(deployer: &signer) {
        init_module(deployer)
    }

    /// 购买一张抽奖票的价格。
    public fun get_ticket_price(): u64 { TICKET_PRICE }

    /// 任何用户都可以调用此函数购买抽奖票。
    public entry fun buy_a_ticket(user: &signer) acquires Raffle {
        let raffle = borrow_global_mut<Raffle>(@raffle);

        // 从用户的余额中扣除一张抽奖票的价格，并将其累积到抽奖的奖池中。
        let coins = coin::withdraw<AptosCoin>(user, TICKET_PRICE);
        coin::merge(&mut raffle.coins, coins);

        // 为该用户发放一张票
        smart_vector::push_back(&mut raffle.tickets, signer::address_of(user))
    }

    /// 只能作为 TXN 的顶级调用来调用这个函数，从而防止“测试和中止（test-and-abort）”攻击。
    entry fun randomly_pick_winner() acquires Raffle {
        randomly_pick_winner_internal();
    }

    /// 允许任何人关闭抽奖（如果足够的时间已经过去且有超过 1 个用户购买了票），并抽取一个随机的获胜者。
    public(friend) fun randomly_pick_winner_internal(): address acquires Raffle {
        let raffle = borrow_global_mut<Raffle>(@raffle);
        assert!(!raffle.is_closed, E_RAFFLE_HAS_CLOSED);
        assert!(!smart_vector::is_empty(&raffle.tickets), E_NO_TICKETS);

        // 在 [0, |raffle.tickets|) 中随机选择一个获胜者
        let winner_idx = randomness::u64_range(0, smart_vector::length(&raffle.tickets));
        let winner = *smart_vector::borrow(&raffle.tickets, winner_idx);

        // 支付奖金给获胜者
        let coins = coin::extract_all(&mut raffle.coins);
        coin::deposit<AptosCoin>(winner, coins);
        raffle.is_closed = true;

        winner
    }
}
```

请注意，`raffle::randomly_pick_winner` 函数被标记为 私有 `entry`。这是为了防止测试和中止攻击，我们稍后会讨论。

具体而言，这确保了 `randomly_pick_winner` 这个功能不能从 Move 编程语言所编写的脚本，或 `raffle`（抽奖）模块之外的任何其它函数中被调用。否则，这些调用可能会检测 `randomly_pick_winner` 的运行结果并中断操作，这将导致抽奖结果的不公正（详见 [“测试和中止攻击”](###测试和中止攻击)）。

## 七、未解决的问题

**O1:** `randomness` 模块应该归属于 `aptos_std` 还是 `aptos_framework`？保持它在 `aptos_std` 中的一个理由是，一些加密相关模块可能会用到它——例如，那些可能需要使用到公共随机源（ public coins）的交互式零知识证明 (ZKP) 验证器或许就会应用 `randomness` 模块。

**O2:** 对私有 `entry` 函数的支持可能尚未完全实现：即，私有 `entry` 函数仍然可以从 Move 脚本中调用。或者甚至可能无法从 TXN 中调用。或者当前 SDK 对于创建调用私有 entry 函数的 TXN 可能存在限制。

**O3:** 当调用 `randomness` API 时是否发出事件（例如，解决钱包预览中的[非确定性交易预览问题](###安全考虑-钱包预览中的非确定性交易结果)）？

**O4:**我们是否应该使用一个通用函数 `randomnes::integer<T>(): T` 来替代分别为 `u32_integer`、`u64_integer` 等创建的独立函数？如果给定非整型参数，这个函数将终止执行。



## 八、参考实现

 > 这一部分是可选的，但我们非常推荐你能够提供一个示例，说明你在这个提案中想要实现的目标。示例可以是代码、图示或简单文本的形式。在理想情况下，我们希望有一个连接到活跃的代码仓库的链接，以此来展示这个标准；或者对于更简单的例子，可以直接提供内嵌代码。

- `aptos_framework::randomness` API 的实现在 [这里](https://github.com/aptos-labs/aptos-core/blob/randomnet/aptos-move/framework/aptos-framework/sources/randomness.move)。
- 本 AIP 中抽奖示例的实现在 [这里](https://github.com/aptos-labs/aptos-core/tree/randomnet/aptos-move/move-examples/raffle)。
- 彩票示例的实现在[这里](https://github.com/aptos-labs/aptos-core/tree/randomnet/aptos-move/move-examples/lottery)。

## 九、风险和缺点

 > 在这里说明如果采纳这项提案可能带来的负面影响。该提案存在的潜在风险有哪些？

### 1. 测试和中止攻击

智能合约平台的确定性和随机性天生就是一种对抗性的（inherently-adversarial）环境，，部署随机性面临着固有的挑战。。

**一个问题**是，当 Move 函数的执行受到链上随机性的影响时（例如，通过 `randomness::u64_integer`），它的执行结果可以通过从另一个模块（或从 Move 脚本）调用该函数来进行 **测试**，如果结果不是期望的结果，则 **中止** 操作。

我们在[下面的“安全考虑”部分讨论了](security-consideration-preventing-test-and-abort-attacks)这种攻击的 **缓解措施**。


#### 1.1 一个示例攻击

具体来说，**假设** `raffle::randomly_pick_winner` 是一个 `public entry` 函数而不是 **private** entry 函数:

```rust
public entry fun randomly_pick_winner(): address acquires Raffle, Credentials { /* ... */}
```

那么，一个包含以下 Move 脚本的 TXN 可以通过 **测试和中止（test-and-abort）** 进行攻击：

```rust
script {
    use aptos_framework::aptos_coin;
    use aptos_framework::coin;

    use std::signer;

    fun main(attacker: &signer) {
        let attacker_addr = signer::address_of(attacker);

        let old_balance = coin::balance<aptos_coin::AptosCoin>(attacker_addr);

        // 为了防止此类攻击成功，`randomly_pick_winner` 必须被标记为 *private* entry 函数。
        raffle::raffle::randomly_pick_winner();

        let new_balance = coin::balance<aptos_coin::AptosCoin>(attacker_addr);

        // 攻击者可以查看他的余额是否保持不变。如果是，则攻击者知道他们没有赢得抽奖，并且可以中止一切。
        if (new_balance == old_balance) {
            abort(1)
        };
    }
}
```

### 2. 低 Gas 攻击

**另一个问题** 当 Move 函数的执行结果依赖于随机值来决定不同的执行分支时。
比如说，利用 `if` 判断语句根据一个随机值来决策，就会产生两种可能的执行路径：“then” 分支与 “else” 分支。 问题就在于这些执行路径可能会导致 **不同的 Gas 费用**：某一路径可能手续费可能更低，而另一路径的 Gas 费用则可能更高。

这就使得攻击者可以通过故意对发起函数调用的交易进行 **提供不足的Gas费**（undergasing），来偏置函数的执行路径，确保仅有成本较低的路径能够成功执行（而高成本的路径总是因为Gas不足而失败）。
需要注意的是，攻击者必须不断重试他们的交易，直到执行路径恰好变为成本较低的那一个。
结果是，那些执行了高成本路径的失败交易可能导致攻击者资金的浪费。
然而，如果低成本路径的盈利足够丰厚，这种攻击行为仍是划算的。

我们在下面给出一个易受攻击的应用程序的示例，并在 [“安全注意事项”部分讨论了](security-consideration-preventing-undergasing-attacks)针对此攻击的 **缓解措施**。


#### 2.1 一个在游戏中易受攻击的抛硬币函数示例

一个游戏可能会根据抛硬币的结果采取两条不同的执行路径。
这将容易受到低 Gas 攻击的影响。

*注意:* 并非所有的辅助函数和常量都被定义了，但示例应该还是有意义的。

```rust
entry fun coin_toss(player: signer) {
   let player = get_player(player);
   assert!(!player.has_tossed_coin, E_COIN_ALREADY_TOSSED);

   // 抛一个随机硬币
   let random_coin = randomness::u32_range(0, 2);
   if (random_coin == 0) {
       // 如果是正面，给玩家 100 个硬币（低 Gas 路径；攻击者可以确保此路径总是执行）
       award_hundred_coin(player);
   } else /* random_coin == 1 */ {
       // 如果是反面，惩罚玩家（高 Gas 路径；攻击者可以确保此路径永远不会执行）
       lose_twenty_coins(player);
       lose_ten_health_points(player);
       lose_five_armor_points(player);
   }
   player.has_tossed_coin = true;
}
```

### 3. 表达能力

另一个问题是 **API 的表达能力不够**。

幸运的是，`randomness::bytes` 方法应该允许实现任何其他对象的复杂抽样（例如，512 位数字）。

此外，如果有需要，`randomness` 模块可以升级，使其具备额外功能（functionality）。

## 十、未来潜力

> 对这个提案的未来发展进行深思熟虑。你认为这将会怎样？这个提案一年后会有什么结果？五年后呢？

参见 [“动机”](#Motivation)。

此外，这个提案还可以为外部实体提供一个可信赖的随机性信标（a trustworthy randomness beacon）。

## 十一、时间表

### 1. 建议的实施时间表

> 描述您预计实施工作需要多长时间，可能将其分解为阶段或里程碑。

**不适用**：这个 AIP 严格来说是关于提议的 API，而不是其密码学实现，密码学实现将是另一个未来 AIP 的范围。

### 2. 建议的开发者平台支持时间表

> 如果适用，请描述对于这个功能，有关 SDK、API、CLI、Indexer 支持的计划。

值得研究的是，随机性调用（以及它们的输出）是否需要被索引（例如，似乎有必要在调用 `randomness` API 后发出事件）。

这可能很重要，因为它可以使 randapps 生成的随机性更容易进行 **公开验证**。具体来说，任何人都可以查看合约发出的与 `randomness` 相关的事件，以获取其生成的随机性的可验证历史记录。

### 3. 建议的部署时间表

> 社区何时可以期待在开发网上看到这个功能部署？
>
> 在测试网上？
>
> 在主网上？

参见 [“建议的实施时间表”](#suggested-implementation-timeline)：密码学实现将是另一个未来 AIP 的范围。

## 十二、安全考虑

> 这个更改是否由任何审计公司审计过？

目前还没有，但在部署时将进行审计。更新将在这里发布（**TODO**）。

> 有可以分享的任何安全设计文档或审计资料吗？

没有。目前，这个文档是自包含的（self-contained）。我们让社区审查这个 AIP，以发现 `randomness` API 中的问题或限制，并提出修复建议。

> 有潜在的欺诈行为吗？有什么缓解策略？

任何智能去中心化应用 (dapp) 都可能被故意嵌入恶意漏洞。因此，在开始使用这些应用之前，用户自行审计应用代码，或者确认已有其他人完成审计工作，这一点非常关键。

从这个角度看，randapps 与普通的去中心化应用（dapp）一样，同样易受到恶意注入（maliciously-introduced ）漏洞的影响。

> 有安全影响/考虑吗？

是的。我们在下面讨论它们。

### 1. 安全考虑：意外重新生成相同的随机性

除了自然发生的碰撞之外，误用 API 重新采样先前采样的随机性应该是不可能的。

例如，下面的代码将独立地采样两个 `u64` 整数 `n1` 和 `n2`，这意味着它们很可能是不同的数字，除了一些小的碰撞概率。

```
let n1 = randomness::u64_integer();
let n2 = randomness::u64_integer();

if (n1 == n2) {
   // 这段代码不太可能被执行，除非概率是 2^{-32}，表达式如下引用“概率”。
}
```
>  概率：$2^{-32}$​



### 2. 安全考虑：API 实现

当使用**基于保密性的（ secrecy-based approach）**方法 (例如，阈值可验证随机函数 t-VRF) 而非基于延迟的方法 (例如，可验证延迟函数 VDF) 来实现链上随机性时，大部分持股者（ a majority of the stake ）可能会串通起来，提前预测随机结果或者影响其公正性。

虽然合约可以通过（谨慎地）结合外部随机性或延迟函数来减轻这种情况，但这会消除许多链上（on-chain）随机性的优势。

### 3. 安全考虑：钱包预览中的非确定性交易结果

当发送交易到 randapp 时，钱包将显示该 TXN 的结果的 _预览_（例如，交易用户将从其帐户发送多少硬币）。
当然，这个预览取决于 `randomness` API 调用的随机结果，因此 **绝不能** 信任它作为最终结果。

事实上，用户 **应该** 已经意识到，由于其潜在的依赖关系，由于链上的潜在依赖性，他们的钱包交易预览不是最终的结果。
例如，交易的执行结果可能会与预览中显示的不同，因为执行过程中，它所依赖的区块链状态发生了变化。
同样，可能执行过程依赖于调用 [`aptos_framework::timestamp::now_seconds()`](https://aptos.dev/reference/move/?branch=mainnet&page=aptos-framework/doc/timestamp.md#0x1_timestamp_now_seconds)函数。

为了更明确地表明交易的结果可能与钱包预览中显示的不同，钱包实现可以检查该 TXN 是否发出了 `randomness` 事件，并向用户显示适当的消息。

### 4. 安全考虑：防止测试和中止攻击


在 [“测试与中止攻击”](#test-and-abort-attacks) 中讨论的防御策略假定开发者能够正确地使用 **private** entry 函数作为他们的随机应用程序(randapp)的唯一接入点。
不幸的是，开发者有可能犯错。
因此，要防止 **无意中引入的漏洞** 是极其重要的。

我们在下面讨论两种计划使用的防御措施，以确保 **private** `entry` 函数作为 randapp 的唯一入口。

#### 4.1 基于 linter 的检查

可以实施 linter 检查，以确像 `randomness::u64_integer` 这样的 `randomness` 函数调用生成随机样本，这些函数只能通过`private` entry 函数来调用。

例如，在 `raffle` 示例中就是这种情况：

- 赢家通过 `randomness::u64_range` 抽取，
- ……这是从 private 函数 `randomly_pick_winner_internal` 中调用的，
- ……然后只能从 private `entry` 函数 `randomly_pick_winner` 中调用。

**优势：**

- 这种防御措施可以作为 `aptos move` CLI 编译器默认的 linter 检查的一部分。

**劣势：**

- 一些开发人员可以通过使用自定义编译器或直接编写字节码来跳过这种防御。

### 5. 基于调用堆栈的检查

我们可以检查 Move VM 调用堆栈，以查看是否从 `private entry` 函数之外调用了 `randomness` **native** 函数，从而积极地防止测试和中止攻击。

**优势：**

- 如果 `randomness::u64_integer` 是通过 `SafeNativeContext` -> `NativeContext` -> `Interpreter` -> `CallStack` -> 检查调用堆栈中每个函数的 `def_is_friend_or_private` 来实现的 Move native 函数，那么这可能不是很难实现（需要添加一个 `traverse` 函数）。
- 与基于 linter 的防御措施可能会被一些开发人员跳过相比，这种防御措施将 **始终** 生效。

**劣势：**

-  ~~这种防御方式因为会中断原本正常运行的、存在安全漏洞的随机函数调用而显得有些过于严厉，并且干扰 Move 语言的语义。~~   
   - 随机性函数是 Move 语言的原生函数（ native functions）。为其定义自定义的中止逻辑是完全可以的。这并不会破坏 Move 语言的任何语义。

### 6. 安全考虑：防止低 Gas 攻击

如果开发人员意识到低 Gas 攻击，他们可以小心地编写他们的合约以避免它们（例如，使用提交和执行模式，其中第一个 TXN 提交随机性，而第二个 TXN 执行结果，并根据随机性进行分支）。
不幸的是，攻击和其防御的微妙程度相当高。
因此，我们寻求积极保护开发人员。

在我们提出的防御中，Move VM 将强制随机性 TXN 始终执行以下操作：
 1. 声明其 `max_gas` 量为足够高的金额（例如，TXN 允许的最大 gas 量），
 2. 在执行开始之前，在 TXN prologue 中锁定 gas，
 3. 在 epilogue 中退还剩余的 gas。

如果锁定的金额足够高，使其能够覆盖任何 TXN 的执行，这种防御措施将确保随机性 TXN 永远不会低于 Gas，完全消除了这类攻击。

为了识别 TXN 是否使用随机性，因此是否应进行 gas 锁定，我们建议在任何（私有）入口函数中添加一个 `#[randomness]` 注释来采样随机性：

```rust
#[randomness]
entry fun coin_toss(player: signer)
```

如果开发人员忘记添加此注释，则会中止其随机性 TXN，确保安全抵御低 Gas 攻击。
（为了确保活跃性，linter 可以在他们的代码编译之前帮助他们检测到缺少的注释。）

_注意：_ 如果不进行这样的标注，虚拟机(VM)将不得不为 _所有_ 交易(TXNs)锁定手续费（Gas），这会对 Aptos 生态产生影响（比如，一个进行 APT 转账的分布式应用程序（dapp）可能会一直设定很高的手续费，如 1 APT，它假设手续费不会被锁定，并且交易能够使用大部分；但是在注释之后，这类dapps可能会运行失败，因为它们的 1 APT 将会被预留作为手续费）。

## 十三、测试（可选）

> 测试计划是什么？它是如何进行测试的？

根据[“建议的实施时间表”](#建议的实施时间表)，密码学实现将是另一个未来AIP的范围。

目前的测试计划包括：

- 通过AIP讨论、开发者讨论等方式收集来自Move生态系统对API的反馈。
- 测试在Move虚拟机中对 private `entry`函数的支持是否完全实现。
- 测试基于代码检查（linter-based checks）或调用堆栈检查是否确实捕捉到了对`randomness` API的误用。

## 十四、致谢

感谢[此AIP的讨论](https://github.com/aptos-foundation/AIPs/issues/185)中的个人贡献者。

感谢Move设计审查团队提供的反馈。

感谢Kevin Hoang、Igor Kabiljo和Rati Gelashvili在减轻 *测试和中止* 攻击方面的帮助。

感谢Junkil Park进一步简化API。

感谢Valeria Nikolaenko（a16z Crypto）指出了低 Gas 攻击。

## 十五、更新日志

为了后续查看，该AIP的历史版本为：

- [v1.0](https://github.com/aptos-foundation/AIPs/blob/4577f34c8df6c52a213223cd62472ea59e3861ef/aips/aip-41.md)
  - `randomness` API的初始版本
- [v1.1](https://github.com/aptos-foundation/AIPs/blob/3e40b4e630eb8aa517b617799f8e578f5f937682/aips/aip-41.md)
  - 切换了基于`RandomNumberGenerator`的API设计
  - 查看[v1.0的区别](https://github.com/aptos-foundation/AIPs/compare/4577f34c8df6c52a213223cd62472ea59e3861ef..3e40b4e630eb8aa517b617799f8e578f5f937682?short_path=33dc9b1#diff-33dc9b179818e3c972be69b5f214313b233e2338fef599626ebe4895f4e5dc51)
- v1.2（当前）
  - 增加了对“测试和中止”攻击的防护
  - 进一步简化了API，移除了`RandomNumberGenerator`结构。
  - 讨论了基于代码检查和调用堆栈检查来防御“测试和中止”攻击。
  - 讨论了低 Gas 攻击以及如何避免它们。
  - 查看[v1.1的区别](https://github.com/aptos-foundation/AIPs/compare/3e40b4e630eb8aa517b617799f8e578f5f937682..HEAD)。