---
aip: 67
title: 为JSON Web Key（JWK）提供原生一致性
author: Zhoujun Ma (zhoujun@aptoslabs.com)
discussions-to (*optional): https://github.com/aptos-foundation/AIPs/issues/331
Status: 已接受
type: <标准（核心、网络、框架）>
created: <2024年2月10日>
---

[TOC]

# AIP-67 - 为JSON Web Key（JWK）提供原生一致性

## 一、概述

OpenID Connect （OIDC）协调认证，使用户能够通过信任的身份提供者的中介，利用 OAuth 2.0 框架进行安全交互，向客户端应用程序证明其身份。

通常，此过程涉及使用身份提供者的加密公钥对其进行签名验证，这些公钥以**JSON Web Key (JWK)**格式发布。出于安全目的，JWK定期轮换，但每个提供者可能有自己的轮换时间表，并且提供者通常不会提供官方文档或通知：客户端应用程序预期以临时方式获取JWK。

[AIP-61: 无密钥账户](https://github.com/aptos-foundation/AIPs/blob/main/aips/aip-61.md)引入了一种新的 Aptos 账户类型，这种账户通过所有者的现有 OIDC 账户（即与 OIDC 提供商如 Google、GitHub 或 Apple 的 Web2 账户）来保障安全。从这样的 OIDC 账户验证交易涉及验证提供商与其 JWK 的签名。这要求验证者**同意每个需要支持的提供者的最新JWK**。

本AIP提出的解决方案是，验证者：
- 通过直接获取它们来监视 OIDC 提供者的 JWK；
- 一旦检测到 JWK 更改，与同行协作形成一个经过法定人数认证（quorum-certified）的 JWK 更新。
- 通过[验证者交易](https://github.com/aptos-foundation/AIPs/blob/main/aips/aip-64.md)将更新发布到链上。



### 1. 目标

建立一个在链上复制外部 JWK 的框架。

- 功能性：对于给定提供者集合（可能存储在一个链上资源中）中的每个提供者，确保验证者对其最新 JWK 达成一致（并可能将其作为另一个链上资源发布）。
- 非功能性：
  - 安全与去中心化：JWK 应以安全且去中心化的方式得到验证者网络的达成一致，理想情况下引入零额外安全假设。
  - 低复制延迟：一旦提供者轮换其 JWK，区块链应尽快捕获它们。
    - 若缺少来自提供商 X 的最新 JWK，基于 X 的无密钥账户发起的交易可能会被拒绝。
  - 提供者独立性：对OIDC提供商端所需进行的额外操作应尽可能减少（理想情况下为零），以便用极低成本支持新的 OIDC 提供商。
  - DevOps 复杂性：开发与运维复杂度较低的解决方案被视为成本更低、风险更小。



### 2. 不在范围内

以下细节不在本 AIP 的讨论范围内，可能需要单独的 AIP 来处理。
- 支持的 OIDC 提供者列表。
- 每个 OIDC 提供者的实际 Open ID 配置。
- 每个 OIDC 提供者的实际 JWK 操作。
- 运营商的实际影响和推荐操作。



## 二、动机

如摘要所述，[AIP-61: 无密钥账户](https://github.com/aptos-foundation/AIPs/blob/main/aips/aip-61.md)要求验证者就 OIDC 提供商的最新 JWK 达成一致。

本 AIP 提出了一种平衡所有非功能性需求的功能性目标解决方案：

- 假定 OIDC 提供商集在链上资源中进行维护；
- 验证者将 OIDC 提供商集（provider set）作为输入，对每个提供商监控并发布经法定人数认证的（quorum-certified）JWK 更新至链上；
- 在节点环境中引入相对较小且实用的安全假设；
- 可实现几秒钟的复制延迟；
- 对 OIDC 提供商没有要求（除了要求其符合 OIDC 规范）；
- DevOps 复杂度相对较低。



## 三、影响

验证者运营商需要确保其验证节点处于一个可以安全访问 OIDC 提供者 API 的环境中。

每次更新支持的 OIDC 提供者列表或任何其 OpenID 配置 URL 时，都需要进行此配置。
但是希望每个操作都应该是容易解决的。



## 四、替代方案

### 1. 替代方案1：使用单独的JWK预言机
一个替代方案是运行一个独立的受信任的预言机来监视 JWK
（或者运行一个独立的受信任的预言机网络来监视 JWK 并以法定人数认证更新）
并通过常规的 Aptos 交易在链上发布更新。

**不考虑此方案的原因：**安全性和去中心化 + DevOps复杂性。 此解决方案引入了一种需要被验证者网络信任的新节点/网络类型，这意味着额外的安全假设和运维复杂度。



### 2. 替代方案2：在链下监视 JWK 并使用治理提案发布
另一种替代方案是：一旦以任何方式在链下检测到JWK更新，通过治理提案发布，并等待大多数质押者投票通过。

**不考虑此方案的原因：**延迟。投票可能持续长达7天（根据当前的设置），这意味着新 JWK 可能直到 7 天后才在链上可用，进一步意味着使用新密钥签署的无密钥交易在7天内将不被接受。



### 3. 替代方案3：让 OIDC 提供者在链上更新 JWK
一种可行的方案是，各个 OIDC 提供商分别建立自己的 Aptos 账户，在自身的 JWK URL 更新新密钥的同时，也将这些新密钥发布到区块链上。

安全性+提供者独立性。此替代方案要求 OIDC 提供商有足够的动机参与 Aptos 协议，但这在今天并非如此，也不清楚如何确保这一点。即使提供商被激励，它们的任何 Aptos 账户密钥 / JWK 实际上都会成为整个系统的一个新的“单一妥协点”：如果遭到妥协，所有相关的无密钥账户都将遭到妥协。这是一个新的（并且可能是不受欢迎的）安全假设。此外，现有的 OIDC 提供商将被要求更新它们的密钥轮换逻辑。



### 4. 替代方案4：让验证者在执行过程中获取JWK，并在结果出现分歧时丢弃交易
一种替代方案是在交易验证过程中获取JWK，并在验证结果出现分歧时丢弃交易。

不考虑此方案的原因：DevOps复杂性。这需要重构 Aptos 的一些基本构建块（例如，解耦执行），这可能带来过多的实现/测试复杂性。



## 五、规范

### 1. 假设

OIDC 提供商可以根据需要以任意频率更换其 JWK 。然而，只要某个特定版本被维持了足够长的时间，并且至少有超过三分之二总投票权的诚实验证者（honest validators）已经确认了这个版本，那么这个版本的密钥信息就可以被可靠地记录在区块链上。

这一假设对当今大多数 OIDC 提供商都有效。（已知最快的轮换者是Google，其几乎每周都会轮换一次JWK）

以下是JWK共识在提供商处可能陷入停滞的一些情况：

- 当验证者对该提供者的JWKs持有均匀的观点时。
- 当提供者轮换太快时。
- 当总投票权的1/3被恶意验证者控制时，
  并且至少有一个诚实的验证者无法访问提供者的JWK API。



一个 OIDC 提供商可能以任意频率轮换其 JWK。但只要它最终在一个版本上停留足够长的时间，并且这个版本可以被持有超过2/3总投票权的诚实验证者子集观察到，那么这个版本就可以成功地在链上复制。

这一假设对当今大多数OIDC提供商都有效。（已知最快的轮换者是Google，其几乎每周都会轮换一次JWK。）

以下是JWK共识在提供商处可能陷入停滞的一些情况。

- 当验证者之间对于提供者的 JWK 没有达成一致的意见，而且这种分歧是在各个验证者之间相对平衡的。
- 当提供商轮换速度过快时。
- 当总投票权的1/3被恶意验证者控制，并且至少有一个诚实验证者无法访问提供商的 JWK API 时。



### 2. 链上状态

- `SupportedOIDCProviders`: 一个从 OIDC 提供者（例如 `https://accounts.google.com`）到其 OpenID 配置 URL 的映射。
- `ObservedJWKs`: 一个从提供者到 `(jwk_set, version)` 对的映射，验证者观察（observed）并达成一致意见。
- 一个功能标志，用于开启/关闭该功能。

### 3. 链上事件

- `ObservedJWKsUpdated`
    - 可以包含 `ObservedJWKs` 的副本。
    - 当 `ObservedJWKs` 通过交易更新时，应该发出此事件。
    - 验证者应该订阅此事件以重置其内存中的状态。

### 4. 治理操作

1. 开启/关闭该功能。
1. 开始/停止监视一个 OIDC 提供者（即更新 `SupportedOIDCProviders`）。
1. 删除一个 OIDC 提供者的所有观察到的 JWK。
    - 通常与“停止监视一个 OIDC 提供者”一起使用，以完全禁用一个提供者（从无密钥账户的角度看）。 这可以使验证者端的逻辑更简单。
1. 对链上的 JWK 集合打补丁。
    - 示例用例：发现单个密钥被泄露，但提供者尚未进行轮换。

### 5. 验证者内存状态

如果一个验证者是当前时期的验证者集合的成员，则应该对链上 `SupportedOIDCProviders` 中的每个 `provider` 有以下状态。
- `latest_on_chain: LastestOnChainValue { jwks, version }`：应该维护为链上 `ObservedJWKs` 中 `provider` 的最新值，并用作检测 JWK 更新的基准。
- `consensus_state: PerProviderConsensusState`：应该捕获每个提供者的 JWK 共识状态。以下是变体。
    - `NotStarted`：等待被观察到的新版本 JWK。
    - `InProgress{ob_jwks, version, task_handle}`：表示观察到一个新的JWKs版本（值为`ob`），正在与对等节点并行运行一个任务`task_handle`来执行认证过程。
    - `Finished{ob_jwks, version}`：更新 `(ob_jwks, version)` 已经被认证，并作为交易提议更新链上 `ObservedJWKs`。

### 6. 验证者操作

如果一个验证者是当前时期的验证者集合的成员，则对链上 `SupportedOIDCProviders` 中的每个 `provider`，验证者应执行以下操作：

对于链上`SupportedOIDCProviders`中的每个`provider`来说，如果一个验证者是当前时期的验证者集的成员，验证者应该执行以下操作：

- 在新的时期：
    - 定期从`provider`的 OpenID 配置中获取最新的 JWK；
    - 加载`ObservedJWKs`并用键`provider`对应的值初始化`latest_on_chain`；
    - 将 `consensus_state` 初始化为 `NotStarted`。
- 在获取结果 `ob_jwks` 上，如果 `ob_jwks != latest_on_chain.jwks`：
    - 开始一个异步任务`task_handle`来运行共识认证过程：广播请求每个对等节点签署其观察到的`provider`的`(ob_jwks, version)`，并收集足够多的响应`(ob_jwks, version, sig)`，这些响应与本地`(ob_jwks, latest_on_chain.version + 1)`匹配，以形成一个共识认证（超过时期总投票权的2/3的多重签名）；
    - 将 `consensus_state` 切换为 `InProgress(ob, latest_on_chain.version + 1, task_handle)`。
- 一旦为 `(ob_jwks, version)` 形成了共识认证，如果状态为 `InProgress{ob_jwks, version, _}`：
    - 提出一个交易以更新链上的 `ObservedJWKs`，使用观察到并共识认证的 `(ob_jwks, version)` 的 `provider`。
        - 可以利用[验证者交易](https://github.com/aptos-foundation/AIPs/blob/main/aips/aip-64.md)。
    - 将 `consensus_state` 切换为 `Finished{ob_jwks, version}`。
- 在 `ObservedJWKsUpdated` 事件中，如果 `provider` 的值与 `latest_on_chain` 不同：
    - 使用新的链上值重新初始化 `latest_on_chain`；
    - 将 `consensus_state` 切换为 `NotStarted`。
- 每当从 `InProgress(ob, version, task_handle)` 切换 `consensus_state` 时，中止认证进程 `task_handle`。
- 如果从对等节点收到认证请求，且`consensus_state`是`InProgress{ob_jwks, version, task_handle}`或`Finished{ob_jwks, version}`，签署`(ob_jwks, version)`并以`(ob_jwks, version, sig)`的形式响应，其中`sig`是签名。

### 7. JWK更新事务执行

一个 JWK 更新的交易应包含以下参数：
- `epoch`：生成和认证此更新的时期。
- `provider`：正在更新 JWK 的提供者。
- `ob_jwks`：JWK 的新值。
- `version`：新版本号，应该是旧版本号加1。
- `multi_sig`：签署更新 `(ob_jwks, version)` 的签名者集合及其签名的聚合。

在更新映射 `ObservedJWKs` 之前，执行应进行以下检查：
- `epoch` 是当前时期。
- `version == on_chain_version + 1`，其中 `on_chain_version` 是当前与 `provider` 在 `ObservedJWKs` 中关联的版本号，或者在没有关联的情况下为 0 。
- `multi_sig` 中的签名者集合是当前验证者集合的有效子集，并且该子集的投票权超过总投票权的2/3。
- `multi_sig`  本身是签名者集合对更新 `(ob_jwks, version)` 的有效签名。

### 8. 备注

验证者不需要持久化任何离线状态。

## 六、参考实现

 https://github.com/aptos-labs/aptos-core/pull/11528

## 七、风险和缺点

引入了节点环境的一些新假设。 有关详细信息，请参阅 [安全性和存活性考虑](#十、安全性和存活性考虑)。 

操作员需要采取行动，但希望它们是容易解决的。

## 八、未来潜力

这个 AIP 的目的是支持[AIP-61: 无密钥账户](https://github.com/aptos-foundation/AIPs/blob/main/aips/aip-61.md)的实现。

然而，本 AIP 为 Move 中对 HTTP 预言机（Oracle ）的**原生**支持铺平了道路。具体而言，JWK 共识基础设施可用于赋予 Move 合约**异步**地获取 HTTP 页面内容的能力。这将开启许多应用场景，如价格预言机、原生桥接、预测市场等。

## 九、时间表

### 1. 建议的实施时间表

实施和测试应在2024年2月底之前完成。

### 2. 建议的开发者平台支持时间表

这主要是一个验证者的变更 + 新的链上状态，不需要 SDK / API / CLI / Indexer 支持。

### 3. 建议的部署时间表

1.10版本

## 十、安全性和存活性考虑

在此，我们假设以下两个基本假设是有效的，并且仅讨论 JWK 共识所需的假设。
- 区块链主共识的假设。
- OIDC 提供者本身（主要是其 JWK 和 HTTPS 私钥）是安全的。
    - 受损的 OIDC 提供者在[AIP-61: 无密钥账户](https://github.com/aptos-foundation/AIPs/blob/main/aips/aip-61.md)中进行了讨论。

### 1. 支持单个OIDC提供者

当超过三分之二的验证者投票权集中在那些运行在安全环境中的节点上时（这些节点通常通过 HTTPS 访问提供商的 API），提出的方案应该是可行的。在这里，对于提供商来说，**安全环境**指的是：

- 用于解析提供者 API 域名的 DNS 服务器（以及与它们的连接）是安全的；
- 用于验证 HTTPS 连接的 CA 证书是安全的。

更多含义：
- 如果安全环境对于提供者没有超过总验证者投票权的2/3，提供者的新密钥无法发布到链上。（从无密钥账户的角度来看，这是一个存活性问题。）
- 如果恶意环境对于提供者拥有超过总验证者投票权的2/3（其中 DNS 服务器和 CA 证书都由攻击者控制），恶意的 JWK 可以发布到链上。（从无密钥账户的角度来看，这意味着与此提供者相关的任何用户账户都可能被破坏。）

一些假设可能无效的现实场景。
- 非常不平衡的质押分布。如果大量质押被分配给一小部分验证器节点，一般来说更容易受到攻击。
- 地理限制。如果太多质押被分配给在阻止与提供者 API 通信的地区（甚至操纵通信）运行的验证器节点，那么 JWK 共识可能会停滞（甚至让恶意 JWK 通过）。
- DNS 服务器 / 包管理器作为潜在的单点故障。即使没有不平衡的质押分布或地理限制，验证器节点仍然很可能共享相同的 DNS 服务器 / 包管理器配置，将 DNS 服务器 / 包管理器作为潜在的单点攻击。 

### 2. 支持多个OIDC提供者

因为解决方案可以以每个提供者的方式实现，所以支持多个提供者的假设可以简单地是每个个体提供者的假设的组合。唯一的注意事项是，两个提供者可能以不同的方式进行地理限制（例如，提供者 1 在区域 2 被封锁，提供者 2 在区域 1 被封锁），因此不能同时支持。在这种情况下，我们可能希望切换到每个提供者的预言机（Oracle）网络。

