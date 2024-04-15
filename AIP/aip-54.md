---
aip: 54
title: 对象代码部署
author: movekevin, xbtmatt, johnchanguk
discussions-to (*可选): https://github.com/aptos-foundation/AIPs/issues/259
状态: 已接受
类型: 标准（框架）
创建时间: 10/09/2023
---

[TOC]

# AIP-54 - 对象代码部署

## 一、概述

本 AIP 提出了一个简化的代码部署流程，旨在改善在 Aptos 上部署和升级 Move 模块的当前开发者体验。具体而言：
1. 开发者可以将一个 Move 模块包部署到新创建的对象（object）而不是他们的账户。
2. 如果包是可升级的，开发者将收到一个 PublisherRef 对象。
3. PublisherRef 可用于将来更新该包。
4. PublisherRef 可以转移到另一个账户，包括多签账户，或由模块管理。

这套新的简化且改良的代码部署流程将消除当前代码部署中的绝大多数问题和困惑。

另外，它还将减少代码部署所需的存储空间，同时继续保持发布或升级时方便地添加程序化控制功能，因为它无需新增资源账户。



## 二、目标

1. 简化 Aptos 上的代码部署流程，提高整体开发者体验。
2. 提供一个强大的基于对象的框架来管理代码部署和升级。
3. 减少代码部署的存储占用



## 三、现有解决方案
当前的解决方案有：
1. 将代码直接部署到开发者拥有的账户。这有两个主要缺点：
  - 在开发过程中，如果开发者需要进行不兼容的更改，他们需要放弃该账户，并部署到另一个账户以避免命名冲突。或者，他们需要更改他们的包和模块名称。
  - 如果代码直接部署在某个账户上，这个账户就拥有了完全的代码升级控制权，并且不能把这个权限委托出去，比如给一个治理机构。这种做法限制了通过 Aptos 部署的协议在程序控制、灵活性方面的表现，也削弱了去中心化的特性。
2. 将代码部署到一个新创建的资源账户。这与对象部署类似，但存在以下问题：
   - 在开发时处理不兼容的升级也存在类似问题。开发人员需要尝试使用不同的初始参数（种子）来创建一个新的资源账户，这个过程可能会导致混乱。
   - 存储效率低 - 创建一个资源账户来托管代码意味着至少创建了2个资源（特别是当不需要 Account 资源时）。
   - 升级流程使用起来相当棘手。因为升级权限是由资源账户持有的签名能力（SignerCapability）来控制的，理解并运用这个能力来升级代码过程中可能会出现混淆。



## 四、规范
新的对象部署流程将包含3个具体组件：
1. 一个新的 `object_code_deployment` 模块，提供不同的 API 来使用对象进行部署、升级等操作。
2. 一个新的资源`PublisherRef`，包含用于升级代码的扩展引用。
3. 对 Aptos CLI 的改进，以简化代码部署和升级流程。

此外，可以考虑以下改进：
1. 在 `PackageRegistry` 中添加额外的元数据，支持添加自定义元数据，例如最新代码版本的 Git 提交信息。
2. 允许在 `Move.toml` 中指定当前代码对象地址，以便更轻松地进行管理升级。
3. 允许将 `PackageMetadata` 作为结构体输入到入口函数中。这将避免依赖 CLI 生成 `PackageMetadata` 的字节，这容易混淆和出错。
4. 支持增量上传代码片段并最终发布整个软件包的功能。这种方法可以绕过每笔交易64KB的体积上限的限制，从而部署超出这个大小的较大软件包。
5. 在虚拟机（VM）的变更中，一个突出的功能是能够在不预设代码地址的情况下直接部署代码。这项能力可以和动态生成新对象相结合，后者用于容纳这些新部署的代码。



## 五、参考实现
请参考[参考实现](https://github.com/aptos-labs/aptos-core/pull/11748)。

### 1. 新的 object_code_deployment 模块
```rust
module aptos_framework::object_code_deployment {
  #[resource_group_member(group = aptos_framework::object::ObjectGroup)]
  struct PublisherRef has key {
    // 代码对象的扩展引用。
    extend_ref: ExtendRef,
  }

  /// 创建一个新的对象来托管代码，并且如果代码可升级，则发送一个 PublisherRef 对象给发布者。
  public entry fun publish(publisher: &signer, metadata_serialized: vector<u8>, code: vector<vector<u8>>);

  /// 升级现有的代码对象中的代码
  /// 需要发布者有一个 PublisherRef 对象。
  public entry fun upgrade(
    publisher: &signer,
    metadata_serialized: vector<u8>,
    code: vector<vector<u8>>,
    code_object: Object<PublisherRef>,
  );

  /// 使现有的可升级包变为不可变。
  /// 需要发布者有一个 PublisherRef 对象。
  public entry fun freeze(publisher: &signer, code_object: Object<PackageRegistry>);
}
```

### 2. CLI 改进
```
> aptos move publish --profile testnet
> aptos move create-object-and-publish-package --address-name <address_name> --profile testnet
--------------------------------------
部署代码到 0x1234
PublisherRef 发送给 0xpublisher 以供将来升级

> aptos move upgrade-object-package --object-address 0x1234 --profile testnet
--------------------------------------
将代码升级到最新版本，位于 0x1234
```

## 六、参考实现

待完成



## 七、未来潜力

对象是无限可扩展的，因此在将来，对象代码部署可以允许发布者添加自定义资源 / 数据 / 功能。



## 八、时间表

本 AIP 的预期时间表是发布 1.9 版。
