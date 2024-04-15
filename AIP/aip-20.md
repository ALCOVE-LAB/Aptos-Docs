---
AIP: 20
标题: 通用密码学代数和BLS12-381实现
作者: zhoujun-ma，alin
讨论 (*可选): https://github.com/aptos-foundation/AIPs/issues/94
状态: 已接受
类型: 标准 (框架)
创建日期: 2023/03/15
---

[TOC]

# AIP-19 - 通用密码学代数和 BLS12-381 实现

## 一、概述

本 AIP 提议在 Aptos 标准库中支持通用密码学代数操作。

支持的通用操作初始列表，包括群 / 域元素的序列化 / 反序列化、基本算术、配对、哈希到结构的转换。

支持的代数结构初始列表包括 BLS12-381 中使用的群 / 域，这是一种流行的配对友好曲线，如[此处](https://datatracker.ietf.org/doc/html/draft-irtf-cfrg-pairing-friendly-curves-11#name-bls-curves-for-the-128-bit-)所述。

未来的 AIP 可以扩展到操作列表或结构列表中。



## 二、动机

代数结构是许多密码学方案的基本构建模块，但在纯 Move 中实现效率较低。

此更改应该使 Move 开发人员能够实现这些方案的通用构造，然后通过仅切换类型参数来获得不同的实例化。

例如，如果支持了 BLS12-381 群和BN254群，可以实现通用的 Groth16 证明验证器构造，然后能够同时使用基于 BLS12-381 和基于BN254的Groth16证明验证器。

基于 BLS12-381 的 Groth16 证明验证器已作为参考实现的一部分以此方式实现。



## 三、原因

### 1. 通用 API

另一种非通用方法是直接在 aptos_stdlib 中公开已实例化的方案。
例如，我们可以为每个配对友好椭圆曲线 `<curve>` 定义 Groth16 证明验证函数 `0x1::groth16_<curve>::verify_proof(vk, proof, public_inputs): bool`。

对于需要哈希函数和群的 ECDSA 签名，我们可以为每对适当的哈希函数 `<hash>` 和群 `<group>` 定义 `0x1::ecdsa_<hash>_<group>::verify_signature(pk, msg, sig):bool`。

与所提出的方法相比，另一种方法节省了为 Move 开发人员构建方案的工作。然而，aptos_stdlib 的大小在未来可能会呈指数级增长。
此外，非通用方法从开发的角度来看不可扩展：每种加密系统及其底层代数结构的组合都需要一个新的本地函数（例如，椭圆曲线）。

为了使 Aptos stdlib 简洁，同时尽可能涵盖更多的用例，应选择所提出的通用方法，而不是另一种方法。

### 2. 后端调度框架

在后端，通用 API 的本地函数实现了动态调度：它们根据给定的标记类型进行操作。
例如，在 BLS12-381 的上下文中，对于 `add<S>(x: &Element<S>, y:&Element<S>): Element<S>` 的本地函数将计算 `x*y`（如果 `S` 是代表 BLS12-381 中的乘法子群 `Gt`），或者计算 `x+y`（如果 `S` 是标量域 `Fr`）。这是通用 API 设计所必需的。



## 四、规范

### 1. 通用操作

#### 1.1 结构体和函数

模块 `aptos_std::crypto_algebra` 设计了以下定义。

- 一个通用结构体 `Element<S>`，表示代数结构 `S` 的一个元素。
- 表示群 / 域操作的通用函数。

以下是伪 Move 代码中的完整规范。

```rust
module aptos_std::crypto_algebra {
    /// 群（group） `G` 的一个元素。
    struct Element<S> has copy, drop;

    /// 检查代数结构 `S` 的元素 `x` 和 `y` 是否相等（x == y）。
    public fun eq<S>(x: &Element<S>, y: &Element<S>): bool;

    /// 将 u64 转换为代数结构 `S` 的元素。
    public fun from_u64<S>(value: u64): Element<S>;

    /// 返回域 `S` 的加法单位元，或群 `S` 的单位元。（ Return the additive identity of field `S`, or the identity of group `S`）
    public fun zero<S>(): Element<S>;

    /// 返回域 `S` 的乘法单位元，或群 `S` 的固定生成元。
    public fun one<S>(): Element<S>;

    /// 计算代数结构 `S` 的元素 `x` 的负。
    public fun neg<S>(x: &Element<S>): Element<S>;

    /// 计算代数结构 `S` 的元素 `x` 和 `y` 的和。
    public fun add<S>(x: &Element<S>, y: &Element<S>): Element<S>;

    /// 计算代数结构 `S` 的元素 `x` 和 `y` 的差。
    public fun sub<S>(x: &Element<S>, y: &Element<S>): Element<S>;

    /// 尝试计算代数结构 `S` 的元素 `x` 的倒数。
    /// 如果 `x` 在结构 `S` 中没有乘法逆元（例如，当 `S` 是域时，且 `x` 为零时），则返回 none。
    public fun inv<S>(x: &Element<S>): Option<Element<S>>;

    /// 计算代数结构 `S` 的元素 `x` 和 `y` 的乘积。
    public fun mul<S>(x: &Element<S>, y: &Element<S>): Element<S>;

    /// 尝试计算代数结构 `S` 的元素 `x` 和 `y` 的商。
    /// 如果 `y` 在结构 `S` 中没有乘法逆元（例如，当 `S` 是域时，且 `y` 为零时），则返回 none。
    public fun div<S>(x: &Element<S>, y: &Element<S>): Option<Element<S>>;

    /// 计算代数结构 `S` 的元素 `x` 的平方。比 `mul(x, x)` 更快更便宜。
    public fun sqr<S>(x: &Element<S>): Element<S>;

    /// 计算代数结构 `S` 的元素 `P` 的倍增。比 `add(P, P)` 更快更便宜。
    public fun double<G>(element_p: &Element<G>): Element<G>;

    /// 计算 `k*P`，其中 `P` 是群 `G` 的元素，而 `k` 是群 `G` 的标量域 `S` 的元素。
    public fun scalar_mul<G, S>(element_p: &Element<G>, scalar_k: &Element<S>): Element<G>;

    /// 计算 `k[0]*P[0]+...+k[n-1]*P[n-1]`，其中
    /// `P[]` 是由参数 `elements` 表示的群 `G` 的 `n` 个元素，而
    /// `k[]` 是由参数 `scalars` 表示的群 `G` 的标量域 `S` 的 `n` 个元素。
    ///
    /// 如果 `elements` 和 `scalars` 的大小不匹配，则以代码 `std::error::invalid_argument(E_NON_EQUAL_LENGTHS)` 中止。
    public fun multi_scalar_mul<G, S>(elements: &vector<Element<G>>, scalars: &vector<Element<S>>): Element<G>;

    /// 在群 `G1` 和 `G2` 到群 `Gt` 的预编译配对函数（即，双线性映射）上高效计算 `e(P[0],Q[0])+...+e(P[n-1],Q[n-1])`，
    /// `P[]` 是由参数 `g1_elements` 表示的群 `G1` 的 `n` 个元素，而
    /// `Q[]` 是由参数 `g2_elements` 表示的群 `G2` 的 `n` 个元素。
    ///
    /// 如果 `g1_elements` 和 `g2_elements` 的大小不匹配，则以代码 `std::error::invalid_argument(E_NON_EQUAL_LENGTHS)` 中止。
    public fun multi_pairing<G1,G2,Gt>(g1_elements: &vector<Element<G1>>, g2_elements: &vector<Element<G2>>): Element<Gt>;

    /// 在元素 `element_1` 和 `element_2` 上计算预编译配对函数（即，双线性映射）。
    /// 返回目标群 `Gt` 中的一个元素。
    public fun pairing<G1,G2,Gt>(element_1: &Element<G1>, element_2: &Element<G2>): Element<Gt>;

    /// 尝试将字节数组序列化为代数结构 `S` 的元素，使用给定的序列化格式 `F`。
    /// 如果反序列化失败，则返回 none。
    public fun deserialize<S, F>(bytes: &vector<u8>): Option<Element<S>>;

    /// 使用给定的序列化格式 `F` 将代数结构 `S` 的元素序列化为字节数组。
    public fun serialize<S, F>(element: &Element<S>): vector<u8>;

    /// 获取结构 `S` 的阶，以字节序编码的大整数形式。
    public fun order<G>(): vector<u8>;

    /// 将元素 `element` 的结构 `S` 强制转换为超结构 `L`。
    public fun upcast<S,L>(element: &Element<S>): Element<L>;

    /// 尝试将结构 `L` 的元素 `x` 转换为子结构 `S`。
    /// 如果 `x` 不是 `S` 的成员，则返回 none。
    ///
    /// 注意：内部执行成员检查，具体取决于结构 `L` 和 `S`。
    public fun downcast<L,S>(element_x: &Element<L>): Option<Element<S>>;

    /// 使用给定的域分隔标记 `dst` 和给定的哈希到结构套件 `H`，将任意长度的字节数组 `msg` 哈希为结构 `S`。
    public fun hash_to<St, Su>(dst: &vector<u8>, msg: &vector<u8>): Element<St>;

    #[test_only]
    /// 生成结构 `S` 的随机元素。
    public fun rand_insecure<S>(): Element<S>;

```

一般来说，每个结构都实现了基本操作，如（反）序列化、相等性检查、随机抽样。



例如，一个群体也可以实现以下操作。（使用加法概念。）
- `order()` 用于获取群体的阶数。
- `zero()` 用于获取群体的单位元。
- `one()` 用于获取群体的生成元（如果存在）。
- `neg()` 用于群元素求逆。
- `add()` 用于基本群操作。
- `sub()` 用于群元素减法。
- `double()` 用于高效群元素加倍。
- `scalar_mul()` 用于群标量乘法。
- `multi_scalar_mul()` 用于高效群多标量乘法。
- `hash_to()` 用于哈希到群。

另一个例子，一个字段也可以实现以下操作。
- `order()` 用于获取字段的阶数。
- `zero()` 用于字段加法单位元。
- `one()` 用于字段乘法单位元。
- `add()` 用于字段加法。
- `sub()` 用于字段减法。
- `mul()` 用于字段乘法。
- `div()` 用于字段除法。
- `neg()` 用于字段取负。
- `inv()` 用于字段求逆。
- `sqr()` 用于高效字段元素平方。
- `from_u64()` 用于从 u64 快速转换为字段元素。

类似地，对于3个允许双线性映射的群体 `G1`、`G2`、`Gt`，也可以实现 `pairing<G1, G2, Gt>()` 和 `multi_pairing<G1, G2, Gt>()`。

对于两个结构之间的子集 / 超集关系，也可以实现 `upcast()` 和 `downcast()`。
例如，在 BLS12-381 中，`Gt` 是 `Fq12` 的一个乘法子群，因此可以支持从 `Gt` 到 `Fq12` 的向上转换和从 `Fq12` 到 `Gt` 的向下转换。

#### 1.2 共享标量场

一些群体共享相同的群体阶数，为此，设计一个符合人体工程学的方法是允许多个群体共享相同的标量场（主要用于标量乘法），如果它们具有相同的阶数。

换句话说，应该支持以下内容。

```rust
// algebra.move
pub fun scalar_mul<G,S>(element: &Element<G>, scalar: &Scalar<S>)...

// user_contract.move
**let k: Scalar<ScalarForBx> = somehow_get_k();**
let p1 = one<GroupB1>();
let p2 = one<GroupB2>();
let q1 = scalar_mul<GroupB1, ScalarForBx>(&p1, &k);
let q2 = scalar_mul<GroupB2, ScalarForBx>(&p2, &k);
```

#### 1.3 处理不正确的类型参数

目前没有简单的方法来确保对于泛型操作的类型安全性。
例如，即使群体 `A`、`B` 和 `C` 不管理一个双线性映射，`pairing<A,B,C>(a,b,c)` 也可以编译。

因此，后端应在运行时处理类型检查。
例如，如果调用需要两个或更多类型参数的群体操作时提供了不兼容的类型参数，它必须中止。
例如，当 `GroupA` 和 `ScalarB` 具有不同阶数时，`scalar_mul<GroupA, ScalarB>()` 将中止并显示“未实现（not implemented）”的错误。

使用用户定义类型调用操作函数也应中止并显示“未实现（not implemented）”的错误。
例如，`zero<std::option::Option<u64>>()` 将中止。

### 2. BLS12-381 结构的实现

为了能够通过 `aptos_std::crypto_algebra` API 支持广泛而多样的 BLS12-381 加密操作，我们实现了多种标记类型，分别用于表示阶为 `r` 的群（如 `G1`、`G2` 及 `Gt`）以及不同的域（例如 `Fr`、`Fq12`）。

我们还为常见的序列化格式和哈希到群组套件实现了标记类型。

以下，我们描述了我们 *可能* 实现的所有标记类型，以及我们实际上 *已经* 实现的标记类型。

我们相信，这些标记类型应足以支持大多数 BLS12-381 应用程序。

#### `Fq`
在 BLS12-381 曲线中使用的有限域 $F_q$，其中素数阶 $q$ 等于
0x1a0111ea397fe69a4b1ba7b6434bacd764774b84f38512bf6730d2a0f6b0f6241eabfffeb153ffffb9feffffffffaaab。

#### `FormatFqLsb`
`Fq` 元素的一种序列化格式，其中一个元素由大小为 48 的字节数组 `b[]` 表示，最低有效字节（LSB）排在前面。

#### `FormatFqMsb`
`Fq` 元素的一种序列化格式，其中一个元素由大小为 48 的字节数组 `b[]` 表示，最高有效字节（MSB）排在前面。

#### `Fq2`
在 BLS12-381 曲线中使用的有限域 $F_{q^2}$，是 `Fq` 的扩展域，构造方式为 $F_{q^2}=F_q[u]/(u^2+1)$。

#### `FormatFq2LscLsb`
`Fq2` 元素的一种序列化格式，其中形式为 $(c_0+c_1\cdot u)$ 的元素由大小为 96 的字节数组 `b[]` 表示，
它是其系数序列化的串联，最低有效系数（LSC）排在前面：

- `b[0..48]` 是使用 `FormatFqLsb` 序列化的 $c_0$。
- `b[48..96]` 是使用 `FormatFqLsb` 序列化的 $c_1$。

#### `FormatFq2MscMsb`
`Fq2` 元素的一种序列化格式，其中形式为 $(c_0+c_1\cdot u)$ 的元素由大小为 96 的字节数组 `b[]` 表示，
它是其系数序列化的串联，最高有效系数（MSC）排在前面：
- `b[0..48]` 是使用 `FormatFqLsb` 序列化的 $c_1$。
- `b[48..96]` 是使用 `FormatFqLsb` 序列化的 $c_0$。

#### `Fq6`
在 BLS12-381 曲线中使用的有限域 $F_{q^6}$，是 `Fq2` 的扩展域，构造方式为 $F_{q^6}=F_{q^2}[v]/(v^3-u-1)$。

#### `FormatFq6LscLsb`
`Fq6` 元素的一种序列化格式，其中形式为 $(c_0+c_1\cdot v+c_2\cdot v^2)$ 的元素由大小为 288 的字节数组 `b[]` 表示，
它是其系数序列化的串联，最低有效系数（LSC）排在前面：
- `b[0..96]` 是使用 `FormatFq2LscLsb` 序列化的 $c_0$。
- `b[96..192]` 是使用 `FormatFq2LscLsb` 序列化的 $c_1$。
- `b[192..288]` 是使用 `FormatFq2LscLsb` 序列化的 $c_2$。

#### `Fq12` (已实现)
在 BLS12-381 曲线中使用的有限域 $F_{q^12}$，是 `Fq6` 的扩展域，构造方式为 $F_{q^12}=F_{q^6}[w]/(w^2-v)$。

#### `FormatFq12LscLsb` (已实现)
`Fq12` 元素的一种序列化格式，其中一个元素 $(c_0+c_1\cdot w)$ 由大小为 576 的字节数组 `b[]` 表示，
它是其系数序列化的串联，最低有效系数（LSC）排在前面。
- `b[0..288]` 是使用 `FormatFq6LscLsb` 序列化的 $c_0$。
- `b[288..576]` 是使用 `FormatFq6LscLsb` 序列化的 $c_1$。

#### `G1Full`
在 BLS12-381 曲线上的点和无穷远点构成的群，采用椭圆曲线点加法。
它包含用于配对的素数阶子群 $G_1$。

#### `G1` (已实现)
在 BLS12-381 基于配对的群 $G_1$。
它是 `G1Full` 的子群，其素数阶 $r$ 等于
0x73eda753299d7d483339d80809a1d80553bda402fffe5bfeffffffff00000001。
（所以 `Fr` 是相关的标量域）。

#### `FormatG1Uncompr` (已实现)
从 https://www.ietf.org/archive/id/draft-irtf-cfrg-pairing-friendly-curves-11.html#name-zcash-serialization-format- 派生的用于 `G1` 元素的序列化方案。

以下是将 `G1` 元素 `p` 序列化为大小为 96 的字节数组的序列化过程。
1. 如果 `p` 在曲线上，则将 `(x,y)` 作为其坐标，否则为 `(0,0)`。
2. 分别使用 `FormatFqMsb` 将 `x` 和 `y` 序列化为 `b_x[]` 和 `b_y[]`。
3. 将 `b_x[]` 和 `b_y[]` 连接为 `b[]`。
4. 如果 `p` 是无穷远点，则设置无穷远标志位：`b[0]: = b[0] | 0x40`。
5. 返回 `b[]`。

以下是从字节数组 `b[]` 反序列化出 `G1` 元素或无的反序列化过程。
1. 如果 `b[]` 的大小不是 96，则返回无。
2. 计算压缩标志位：`b[0] & 0x80 != 0`。
3. 如果压缩标志位为真，则返回无。
4. 计算无穷远标志位：`b[0] & 0x40 != 0`。
5. 如果无穷远标志位被设置，则返回无穷远点。
6. 使用 `FormatFqMsb` 将 `[b[0] & 0x1f, b[1], ..., b[47]]` 反序列化为 `x`。如果 `x` 是无，则返回无。
7. 使用 `FormatFqMsb` 将 `[b[48], ..., b[95]]` 反序列化为 `y`。如果 `y` 是无，则返回无。
8. 检查 `(x,y)` 是否在曲线 `E` 上。如果不是，则返回无。
9. 检查 `(x,y)` 是否在阶为 `r` 的子群中。如果不是，则返回无。
10. 返回 `(x,y)`。

注意：其他使用此格式的实现：ark-bls12-381-0.4.0。

#### `FormatG1Compr` (已实现)
从 https://www.ietf.org/archive/id/draft-irtf-cfrg-pairing-friendly-curves-11.html#name-zcash-serialization-format- 派生的用于 `G1` 元素的序列化方案。

以下是将 `G1` 元素 `p` 序列化为大小为 48 的字节数组的序列化过程。
1. 如果 `p` 在曲线上，则将 `(x,y)` 作为其坐标，否则为 `(0,0)`。
2. 使用 `FormatFqMsb` 将 `x` 序列化为 `b[]`。
3. 设置压缩位：`b[0] := b[0] | 0x80`。
4. 如果 `p` 是无穷远点，则设置无穷远位：`b[0]: = b[0] | 0x40`。
5. 如果 `y > -y`，则设置词典排序位：`b[0] := b[0] | 0x20`。
6. 返回 `b[]`。

以下是从字节数组 `b[]` 反序列化出 `G1` 元素或无的反序列化过程。
1. 如果 `b[]` 的大小不是 48，则返回无。
2. 计算压缩位：`b[0] & 0x80 != 0`。
3. 如果压缩位为假，则返回无。
4. 计算无穷远位：`b[0] & 0x40 != 0`。
5. 如果无穷远位被设置，则返回无穷远点。
6. 计算词典排序位：`b[0] & 0x20 != 0`。
7. 使用 `FormatFqMsb` 将 `[b[0] & 0x1f, b[1], ..., b[47]]` 反序列化为 `x`。如果 `x` 是无，则返回无。
8. 用 `x` 解曲线方程得到 `y`。如果不存在这样的 `y`，则返回无。
9. 如果词典排序位被设置，则 `y'` 为 `max(y,-y)`，否则为 `min(y,-y)`。
10. 检查 `(x,y')` 是否在阶为 `r` 的子群中。如果不是，则返回无。
11. 返回 `(x,y')`。

注意：其他使用此格式的实现的实现：ark-bls12-381-0.4.0。

#### `G2Full`
在曲线 $E'(F_{q^2}): y^2=x^3+4(u+1)$ 上构建的群，以及无穷远点，采用椭圆曲线点加法。
它包含用于配对的素数阶子群 $G_2$。

#### `G2` (已实现)
在基于 BLS12-381 的配对 $G_1 \times G_2 \rightarrow G_t$ 中的群 $G_2$。
它是 `G2Full` 的子群，素数阶为 $r$
等于 0x73eda753299d7d483339d80809a1d80553bda402fffe5bfeffffffff00000001。
(因此 `Fr` 是关联的标量域)。

#### `FormatG2Uncompr` (已实现)
从  https://www.ietf.org/archive/id/draft-irtf-cfrg-pairing-friendly-curves-11.html#name-zcash-serialization-format-  派生的用于 `G2` 元素的序列化方案。

以下是将 `G2` 元素 `p` 序列化为大小为 192 的字节数组的序列化过程。
1. 如果 `p` 在曲线上，则将 `(x,y)` 作为其坐标，否则为 `(0,0)`。
2. 使用 `FormatFq2MscMsb` 将 `x` 和 `y` 序列化为 `b_x[]` 和 `b_y[]`。
3. 将 `b_x[]` 和 `b_y[]` 连接为 `b[]`。
4. 如果 `p` 是无穷远点，则在 `b[]` 中设置无穷远位：`b[0]: = b[0] | 0x40`。
5. 返回 `b[]`。

以下是从字节数组 `b[]` 反序列化出 `G2` 元素或无的反序列化过程。
1. 如果 `b[]` 的大小不是 192，则返回无。
2. 计算压缩标志位：`b[0] & 0x80 != 0`。
3. 如果压缩标志位为真，则返回无。
4. 计算无穷远标志位：`b[0] & 0x40 != 0`。
5. 如果无穷远标志位被设置，则返回无穷远点。
6. 使用 `FormatFq2MscMsb` 将 `[b[0] & 0x1f, ..., b[95]]` 反序列化为 `x`。如果 `x` 是无，则返回无。
7. 使用 `FormatFq2MscMsb` 将 `[b[96], ..., b[191]]` 反序列化为 `y`。如果 `y` 是无，则返回无。
8. 检查 `(x,y)` 是否在曲线 `E'` 上。如果不是，则返回无。
9. 检查 `(x,y)` 是否在阶为 `r` 的子群中。如果不是，则返回无。
10. 返回 `(x,y)`。

注意：其他使用此格式的实现：ark-bls12-381-0.4.0。

#### `FormatG2Compr` (已实现)
从 https://www.ietf.org/archive/id/draft-irtf-cfrg-pairing-friendly-curves-11.html#name-zcash-serialization-format- 派生的用于 `G2` 元素的序列化方案。

以下是将 `G2` 元素 `p` 序列化为大小为 96 的字节数组的序列化过程。
1. 如果 `p` 在曲线上，则将 `(x,y)` 作为其坐标，否则为 `(0,0)`。
2. 使用 `FormatFq2MscMsb` 将 `x` 序列化为 `b[]`。
3. 设置压缩位：`b[0] := b[0] | 0x80`。
4. 如果 `p` 是无穷远点，则设置无穷远位：`b[0]: = b[0] | 0x40`。
5. 如果 `y > -y`，则设置词典排序位：`b[0] := b[0] | 0x20`。
6. 返回 `b[]`。

以下是从字节数组 `b[]` 反序列化出 `G2` 元素或无的反序列化过程。
1. 如果 `b[]` 的大小不是 96，则返回无。
2. 计算压缩位：`b[0] & 0x80 != 0`。
3. 如果压缩位为假，则返回无。
4. 计算无穷远位：`b[0] & 0x40 != 0`。
5. 如果无穷远位被设置，则返回无穷远点。
6. 计算词典排序位：`b[0] & 0x20 != 0`。
7. 使用 `FormatFq2MscMsb` 将 `[b[0] & 0x1f, b[1], ..., b[95]]` 反序列化为 `x`。如果 `x` 是无，则返回无。
8. 用 `x` 解曲线方程得到 `y`。如果不存在这样的 `y`，则返回无。
9. 如果词典排序位被设置，则 `y'` 为 `max(y,-y)`，否则为 `min(y,-y)`。
10. 检查 `(x,y')` 是否在阶为 `r` 的子群中。如果不是，则返回无。
11. 返回 `(x,y')`。

注意：其他使用此格式的实现：ark-bls12-381-0.4.0。

### 3. 用法示例

#### 3.1 序列化和反序列化

```rust
// G1 元素的序列化和反序列化
let p1: Element<G1> = ...;
let bytes_uncompressed_g1 = serialize::<G1, FormatG1Uncompr>(&p1);
let bytes_compressed_g1 = serialize::<G1, FormatG1Compr>(&p1);

let p1_uncompressed = deserialize::<G1, FormatG1Uncompr>(&bytes_uncompressed_g1);
let p1_compressed = deserialize::<G1, FormatG1Compr>(&bytes_compressed_g1);

// G2 元素的序列化和反序列化
let p2: Element<G2> = ...;
let bytes_uncompressed_g2 = serialize::<G2, FormatG2Uncompr>(&p2);
let bytes_compressed_g2 = serialize::<G2, FormatG2Compr>(&p2);

let p2_uncompressed = deserialize::<G2, FormatG2Uncompr>(&bytes_uncompressed_g2);
let p2_compressed = deserialize::<G2, FormatG2Compr>(&bytes_compressed_g2);
```

#### 3.2 基本群操作

```rust
// G1 元素相加
let p1: Element<G1> = ...;
let p2: Element<G1> = ...;
let sum = p1 + p2;

// G2 元素相乘
let q1: Element<G2> = ...;
let q2: Element<G2> = ...;
let product = q1 * q2;

// 计算配对
let pairing_result = pair(&p1, &q1);
```

这些示例演示了在 Rust 中使用 BLS12-381 群进行序列化、反序列化和基本群操作的方法。

#### 3.3 使用示例

##### 3.3.1 序列化和反序列化

```rust
// Gt 元素的序列化和反序列化
let p_gt: Element<Gt> = ...;
let bytes_gt = serialize::<Gt, FormatGt>(&p_gt);
let p_gt_deserialized = deserialize::<Gt, FormatGt>(&bytes_gt);

// Fr 元素的序列化和反序列化
let scalar_fr: Element<Fr> = ...;
let bytes_fr = serialize::<Fr, FormatFrLsb>(&scalar_fr);
let scalar_fr_deserialized = deserialize::<Fr, FormatFrLsb>(&bytes_fr);
```

##### 3.3.2 哈希到群

```rust
// 哈希到 G1 元素
let msg: Vec<u8> = ...;
let hash_to_g1 = hash_to::<G1, HashG1XmdSha256SswuRo>(&msg);

// 哈希到 G2 元素
let hash_to_g2 = hash_to::<G2, HashG2XmdSha256SswuRo>(&msg);
```

以上示例展示了在 Rust 中使用 BLS12-381 群进行序列化、反序列化以及哈希到群操作的方法。

## 五、参考实现

[https://github.com/aptos-labs/aptos-core/pull/6550/files](https://github.com/aptos-labs/aptos-core/pull/6550/files)

## 六、风险和缺陷

无论是在 Move 中还是在任何其他语言中开发加密方案都非常困难，因为这些方案固有的数学复杂性，以及安全地使用加密库的难度。

因此，我们告诫 Move 应用程序开发人员，使用 `crypto_algebra.move` 和/或 `bls12381_algebra.move` 模块实现加密方案将容易出错，并可能导致存在漏洞的应用程序。

正因如此，`crypto_algebra.move` 和 `bls12381_algebra.move` 这两个 Move 模块在设计时就特别注重安全性。

首先，我们提供了一个最小化的、难以被错误使用的代数结构抽象，比如群（group）和域（field）。

其次，我们的 Move 模块是类型安全的（例如，群 G 中的倒数返回一个 Option<G>）。

第三，我们在 BLS12-381 的实现中，在将群元素从数据格式转换回可计算状态的过程中（反序列化），始终会进行质数阶子群的检查，这样做能够有效避免实现上的重大缺陷。

## 七、未来潜力

`crypto_algebra.move` Move 模块可以扩展以支持更多的结构（例如，新的椭圆曲线）和操作（例如，域元素的批量倒数）。这将:
1. 允许将现有的构建在此模块之上的通用密码系统移植到新的、潜在更快或更小的曲线上。
2. 允许构建更复杂的密码应用程序，利用新的操作或新的代数结构（例如，环、隐藏阶群等）。

一旦 Move 语言升级支持某种类型的接口，就可以使用它来重写 Move 侧的规范，以确保在编译时实现类型安全性。

## 八、建议的实现时间表

该更改应于 2023 年 4 月在 devnet 上可用。