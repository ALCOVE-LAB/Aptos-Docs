---
AIP-47
标题：聚合器 V2
作者：georgemitenkov (https://github.com/georgemitenkov)，vusirikala (https://github.com/vusirikala)，gelash (https://github.com/gelash)，igor-aptos (https://github.com/igor-aptos)
讨论地址（可选）：https://github.com/aptos-foundation/AIPs/issues/226
状态：已接受
最后呼叫结束日期（可选）：<mm/dd/yyyy 最后留下反馈和评论的日期>
类型：标准（核心/框架）
创建日期：08/09/2023
更新日期（可选）： 
要求（可选）：
---

[TOC]

# AIP-47 - 聚合器 V2

## 一、概述

本 AIP 改善并丰富了现有的聚合器（Aggregators）概念，聚合器本身是用于高效并行计数的工具。在改进之后，此类操作就不必再依赖于串行执行的方式了，尤其在以下场景中：

- 使用聚合器进行控制流程的操作（例如，如果递增计数器不超过限制，则执行事务操作）
- 在其他地方存储聚合器的值（通过聚合器快照的概念）

这些操作将不再需要顺序执行。

## 二、动机

如果我们看下面的示例代码：

```
let a = borrow_global_mut<A>(shared_addr1);
a.value += 1;                                  <- 经常同时更改
move_to(signer, B {value: a.value});           <-  仅用于写入
```

如果我们从多个事务调用该代码，它们都需要一个接一个地执行，因为上面对 a.value 进行了读取和写入，因此每两个事务都会产生读/写冲突。
这基本上限制了使用上述代码的事务的单线程性能。

聚合器和聚合器快照利用了这样一个观察结果，即一些经常同时更改的变量通常不会影响其余的计算，并且可以在事务执行期间由 VM 使用占位值计算，一旦计数器值已知，就可以在后处理阶段用实际值直接修改。

聚合器表示一个经常更改的值，聚合器快照表示一个依赖于特定时间的聚合器值的值。
上面的代码将被转换为可以并行执行的代码：

```
let a = borrow_global_mut<A>(shared_addr1);
aggregator_v2::add(a.value, 1);                                   <- 经常同时更改
move_to(signer, B {value: aggregator_v2::snapshot(a.value)});     <-  仅用于写入
```

如果需要值用于执行流程或更复杂的操作，我们可以选择：
- 立即执行计算（并放弃该事务的并行性优势）。
    - 这通过 `aggregator_v2::read` 和 `aggregator_v2::read_snapshot` 函数提供
- 或者提供期望值并进行推测性执行，稍后重新执行（使后续事务无效），如果实现看到应该使用不同的值
    - 因为我们不太可能估计出实际的 `Aggregator.value`，所以我们只支持这一点来获取“布尔”结果 - 即聚合器修改是否可以应用或超出限制。这通过 `aggregator_v2::try_add` 函数提供


## 三、影响

>  谁将受到这一变化的影响？受众需要采取什么样的行动？



## 四、理由

>  解释为什么您提交了这个提案，而不是选择其他解决方案。为什么这是最好的可能结果？



## 五、规范

### 1. 高级实现

模块结构和函数签名：
```
module aptos_framework::aggregator_v2 {
   struct Aggregator<IntElement> has store {
      value: IntElement,
      max_value: IntElement,
   }
   struct AggregatorSnapshot<IntElement> has store {
      value: IntElement,
   }
   struct DerivedStringSnapshot has store, drop {
      value: String,
      padding: vector<u8>,
   }

   public native fun try_add(aggregator: &mut Aggregator<IntElement>, value: IntElement): bool;

   public native fun read(aggregator: &Aggregator<IntElement>): IntElement;

   public native fun snapshot(aggregator: &Aggregator<IntElement>): AggregatorSnapshot<IntElement>;
   
   public native fun read_snapshot<IntElement>(aggregator_snapshot: &AggregatorSnapshot<IntElement>): IntElement;

   public native fun derive_string_concat<IntElement>(before: String, snapshot: &AggregatorSnapshot<IntElement>, after: String): DerivedStringSnapshot;

   public native fun read_derived_string(snapshot: &DerivedStringSnapshot): String;
}
```

修改或写入的操作路径效率很高，而读取操作的路径则相对耗费资源。这就表明 `try_add`、`snapshot` 和 `derive_string_concat` 函数通常执行得很快（只有在 `try_add` 返回的值与它上一次调用时不同时例外）。相反，如果在距离数据被修改的时间点很近的情况下调用 `read` 或 `read_snapshot` 函数，它们会让工作负载变得必须按顺序执行。

不过，请注意，如果 `read_snapshot` 或 `read_derived_string` 函数的调用距离它们所依托的 `snapshot` 或 `derive_string_concat` 函数创建的时间点不是特别近的话（也就是说，不在同一笔交易内执行），那么这些读取操作通常还是成本较低的。

#### 1.1 流程
- 我们一次执行一组事务（在共识中一次执行一个块，在状态同步期间一次执行一个块）。我们为每次执行创建一个新的 BlockSTM/MVHashMap，并且 Aggregator 处理就在该范围内进行。
- 每当我们从存储中读取数据时（即 MVHashMap 没有数据，需要从存储中获取数据时），我们会获取资源中的所有 Aggregators/AggregatorSnapshots，将它们的值从存储中获取并作为它们的初始值，为它们提供唯一的 ID，并用该唯一 ID 替换它们的值字段。在 VM 中，`value` 字段现在变成了 `ValueImpl::DelayedFieldID`，它就像一个引用 - 它是一个标识符，实际的值被提取出来。
- 对于 Aggregator 的修改会作为一个单独的键 - EphemeralId，而不是代表包含它的资源的 StateKey，因此对实际值的修改不会产生冲突。
- BlockSTM 会假设执行事务，并跟踪它明确提供给每个事务的值（无论是通过 `aggregator::read` 还是通过 `aggregator::try_add`  提供的布尔值）。如果它提供的值与最终值不匹配 - BlockSTM 将使执行无效，并要求重新运行它。
- 在最后一步，当 BlockSTM 创建事务输出时，它会用实际值替换 `value` 字段中的唯一 ID。
- 为了确保正确计算 Gas 费用，每个 DelayedField 都有固定的宽度（序列化时的字节表示长度） - 当它是实际值时，以及当它是 `ValueImpl::DelayedFieldID` 时。为此，变长的字符串字段被放置在 `DerivedStringSnapshot` 中，它具有额外的填充字段 - 以保证固定的宽度。`DelayedFieldID` 由 `unique_index` 和其固定的 `width` 组成。

### 2. 实现细节

#### 2.1 标识符

我们希望 Aggregators 被内联存储在使用它们的资源中，但在执行期间和写冲突时希望它们具有间接性。因此，我们希望将 Aggregators 的 `value` 字段视为对临时地址的引用。

我们通过对 `value` 字段进行双重处理来实现这一点 - 在存储和链上事务输出中，它表示实际值。在执行中 - 在 MVHashMap 和 VM 中，它始终表示一个 EphemeralIDs（u64），它在区块/块（block/chunk）执行期间是唯一的。

生成 EphemeralIDs：我们需要 EphemeralIDs 在区块/块（block/chunk）执行期间是唯一的。但 EphemeralIDs 确实是短暂的 - 它们不需要被确定性地分配 - 也就是说，每个验证者可以为同一个块中的同一个 Aggregator 分配不同的短暂 ID，它们仍然保证会生成事务输出。我们通过 `generate_id()` 函数在需要时生成它们，通过单个 `AtomicU32` 计数器访问 - 在 Aggregator 从存储中读取时，以及在事务内创建新的 Aggregator 时。

用 EphemeralID 替换值：对于从存储中读取的所有数据，我们需要将其替换为临时 ID，然后传递给 VM，并存储原始值 - 以便在执行期间访问。我们将在 MVHashMap 中执行此操作，当它发现需要从存储中获取数据时，将其替换，并在将其缓存到 MVHashMap 或将其返回给 VM 之前执行此操作。

用值替换 EphemeralID：当 BlockSTM 为事务实现并在生成事务输出之前进行材料化时，我们将在写集中替换所有这些内容，以及从读集中替换 - 如果此事务修改了该 Aggregator。

替换的机制：我们此时有一个 `vec<u8>`，我们需要在其中进行内联替换。因此，我们需要进行反序列化、遍历和替换，然后再次序列化。由于反序列化和序列化会进行遍历，我们添加了一种方法来钩入（hook ）它们，并在其遍历期间执行交换。我们在 MoveTypeLayout 中添加了一个新的枚举值 - Marked，它包装了原始类型，并提供了一种在标记值上调用的方法来传递给序列化/反序列化。我们实现了一个交换函数，它存储原始值，并将替换值放入其中。对于用 EphemeralIDs 替换值的流程，我们执行 deserialize_with_marked() → serialize()，对于将 EphemeralID 替换为值，我们执行 deserialize() → serialize_with_marked()。

注意事项：我们需要能够对所有 Aggregators 进行替换，无论它们存储在何处 - 包括表和资源组。对于特定的 ResourceGroups，我们需要能够在双重嵌套的序列化中进行替换。我们计划通过将拆分从适配器移动到 MVHashMap 中来实现这一点，以便 ResourceGroup 的每个元素都表示为单独的项，并且可以独立地进行转换。作为额外的好处，这将消除 ResourceGroup 中不同资源的字段之间的读/写冲突，因此 VMChangeSet 产生的 blob 更小（即只有在 ResourceGroup 中接触到的资源和最终事务输出中的完整 blob）。

#### 2.2 聚合器写操作和冲突定义

聚合器值的访问和更改将在单独的路径上进行跟踪 - 即 EphemeralID（与 StateKey 区分开来 - 因为它们永远不会存储在存储中的该路径下）。它们将具有略有不同的逻辑吞吐量，类似于模块将具有略有不同的逻辑，因此我们将修改 VMChangeSet 和 MVHashMap 来单独跟踪数据、模块和聚合器信息。

聚合器写操作与常规写操作不同 - 在发生冲突和需要重新执行时，它们具有不同的逻辑。聚合器的修改以两种方式影响事务 - 需要在最后插入的值，以及从 try_add/try_sub 的结果的控制流中。我们不需要担心前者，对于后者，我们需要知道执行是否正确 - 或者我们需要重新执行。

我们打算使用一个成本低且具有投机性质的聚合器值，用于事务执行，用于决定要返回给 try_add/try_sub 的值，并且我们将跟踪哪个输入值范围会返回相同的结果，然后一旦我们知道了正确的聚合器值 - 我们就知道是否需要重新执行事务，还是只需修补值即可。

对聚合器的写操作可以是 Set(value) 或 Delta(value)。读取约束将是：

```
struct AggregatorReadConstraints {
  read: u128,
  is_explicit_read: bool,
  min_overflow_delta: u128,
  max_achieved_delta: u128,
  min_achieved_neg_delta: u128, 
  max_underflow_neg_delta: u128,
}
```

如果给定一个与 `read` 不同的新值对聚合器进行更新，我们可以通过以下方式验证事务是否仍然有效：

```
new_read + min_overflow > aggregator.limit &&
new_read + max_achieved <= aggregator.limit &&
new_read >= min_achieved_neg_delta &&
new_read < max_underflow_neg_delta 
```

基本上，只有在读取的值足够不同时，才会发生读/写冲突 - 使得上述检查返回 false。

写操作和读取约束是可聚合和可关联的，因此在 MVHashMap 中，我们不需要使用新值进行更新，然后在其后的所有其他版本上重新更新所有其他值，我们只需保留第一个值，并为其他版本记录写操作和读取约束，并在调用 `aggregator_v2::read` 时动态聚合它们。

有了这个想法，我们可以实现以下聚合器：

- 在执行期间 - 访问聚合器会计算该版本的值
- 使用上述规则，验证冲突是否与该版本的计算值相匹配

并且在 BlockSTM 中不需要额外的更改。但这可能效率低下 - 动态计算值和聚合增量是昂贵的，因此我们将首先执行以下操作（但将测试并迭代其他选项）：

- 在执行期间，对于显式读取之外的值访问不会聚合增量，而是将最近完全计算的值取到我们要查找的版本（这将是最近提交的版本）
- 在验证阶段，我们还会采取一种成本较低的方法来计算数值，这样一来，即使验证结果为通过，也不意味着就不需要重新执行计算。
- 为了执行最终验证，我们计划新增或再利用一个"滚动提交"阶段，并且在该阶段计算聚合器的最新"完整实体化值"，让之前提到的低成本计算方式可以利用这个值进行操作。

### 3. AggregatorSnapshot

AggregatorSnapshot 需要链接到聚合器的特定点，因此 `deferred_read()` 的内部实现将是 ：克隆一个聚合器（创建一个只存在于内存中的聚合器，但具有增量历史记录等所有内容，但不能进一步修改），并创建一个链接到该聚合器的 AggregatorSnapshot。

目前，我们不会暴露聚合器的克隆，因此计算值和跟踪历史记录的链接只能有 1 层深度。

## 六、参考实现

- API 在 [aggregator_v2.move](https://github.com/aptos-labs/aptos-core/blob/main/aptos-move/framework/aptos-framework/sources/aggregator_v2/aggregator_v2.move)
- 原生函数的实现在 [aggregator_v2.rs](https://github.com/aptos-labs/aptos-core/blob/main/aptos-move/framework/src/natives/aggregator_natives/aggregator_v2.rs) 中，扩展/上下文的实现在 [delayed_field_extension.rs](https://github.com/aptos-labs/aptos-core/blob/main/aptos-move/aptos-aggregator/src/delayed_field_extension.rs) 中。聚合器 V2 的回退（非并发）实现完全位于第一个文件中。
- [delayed_field_id.rs](https://github.com/aptos-labs/aptos-core/blob/main/third_party/move/move-vm/types/src/delayed_values/delayed_field_id.rs) 中的 DelayedFieldID
- [view.rs](https://github.com/aptos-labs/aptos-core/blob/main/aptos-move/block-executor/src/view.rs) 中 TDelayedFieldView 接口的实现，跟踪 [captured_reads.rs](https://github.com/aptos-labs/aptos-core/blob/main/aptos-move/block-executor/src/captured_reads.rs) 中的 `CapturedReads.delayed_field_reads`
- BlockSTM 在 [executor.rs](https://github.com/aptos-labs/aptos-core/blob/main/aptos-move/block-executor/src/executor.rs) 中处理验证和物化，并在 [versioned_delayed_fields.rs](https://github.com/aptos-labs/aptos-core/blob/main/aptos-move/mvhashmap/src/versioned_delayed_fields.rs) 中存储版本化的值
- 在 [resource_group_adapter.rs](https://github.com/aptos-labs/aptos-core/blob/7b5cf257d6546e394ddd761f332df7b712bacd19/aptos-move/aptos-vm-types/src/resource_group_adapter.rs) 中处理 VMChangeSet 中的资源组拆分的逻辑
- 端到端测试在 [aggregator_v2.rs](https://github.com/aptos-labs/aptos-core/blob/7b5cf257d6546e394ddd761f332df7b712bacd19/aptos-move/e2e-move-tests/src/tests/aggregator_v2.rs) 中

以下是实现不同功能的开关标志： 

- 使用 API 需要启用 `AGGREGATOR_V2_API` 开关。 
- 资源组的分割操作需要启用 `RESOURCE_GROUPS_SPLIT_IN_VM_CHANGE_SET` 开关。 
- 采取延迟及并行处理 AggregatorsV2 的功能需要启用 `AGGREGATOR_V2_DELAYED_FIELDS` 开关。

## 七、风险和缺点

此变动导致系统复杂度上升，因此需要持续维护。其中大部分复杂性被隐藏在 AggregatorContext 和 BlockSTM/MVHashMap 的设计内部，而其他一些复杂的地方则在具体的实施细节中有所体现。



## 八、未来潜力

这为我们提供了一个新的途径：通过观察到有些变量虽然频繁同时发生变化，但它们通常不会影响其他计算过程，我们可以将此原理应用到更复杂数据类型上，比如字符串格式化时的快照和各种集合（例如集合等）。这样做有望提高真实场景下复杂任务的并行处理能力。



## 九、时间表

初步目标定为 aptos-release-v1.10



### 1. 建议的实施时间表

完整的实施已经完成，包括一套测试。



### 2. 建议的开发者平台支持时间表

这简化了聚合器值的外部支持（与 V1 相比），因为值是直接存储在资源本身中的。

依赖聚合器的值现在有了一个间接层 - `AggregatorSnapshot`。我们将适当地处理聚合器的用法，即在 AIP-43 等中。

我们还将添加更多关于如何正确使用聚合器的文档/学习资源。



### 3. 建议的部署时间表

初步计划在 aptos-release-v1.10 时部署



## 十、安全考虑

这里的主要安全考虑是执行的正确性和确定性。设计组件已经进行了审查，并且我们正在进行广泛的测试。



## 十一、测试（可选）

包括大量的单元测试和端到端测试，以及基准测试来验证性能改进，并与朴素方法进行比较。性能测试包含在 `single_node_performance.py` 回归基准测试套件中。
