# AggregateCodegenSupport Physical Operators

`AggregateCodegenSupport` is an [extension](#contract) of the [BaseAggregateExec](BaseAggregateExec.md) abstraction for [aggregate physical operators](#implementations) that support [Whole-Stage Java Code Generation](../whole-stage-code-generation/index.md) (with [produce](#doProduce) and [consume](#doConsume) code execution paths).

`AggregateCodegenSupport` is a `BlockingOperatorWithCodegen`.

## Contract

### <span id="doConsumeWithKeys"> doConsumeWithKeys

```scala
doConsumeWithKeys(
  ctx: CodegenContext,
  input: Seq[ExprCode]): String
```

See:

* [HashAggregateExec](HashAggregateExec.md#doConsumeWithKeys)
* [SortAggregateExec](SortAggregateExec.md#doConsumeWithKeys)

Used when:

* `AggregateCodegenSupport` is requested to [doConsume](#doConsume)

### <span id="doProduceWithKeys"> doProduceWithKeys

```scala
doProduceWithKeys(
  ctx: CodegenContext): String
```

See:

* [HashAggregateExec](HashAggregateExec.md#doProduceWithKeys)
* [SortAggregateExec](SortAggregateExec.md#doProduceWithKeys)

Used when:

* `AggregateCodegenSupport` is requested to [doProduce](#doProduce)

### <span id="needHashTable"> needHashTable

```scala
needHashTable: Boolean
```

Whether this aggregate operator needs to build a hash table

| Aggregate Physical Operator | needHashTable |
| :-------------------------: | :--------------: |
| [HashAggregateExec](HashAggregateExec.md) | [:white_check_mark:](HashAggregateExec.md#needHashTable) |
| [SortAggregateExec](SortAggregateExec.md) | [❌](HashAggregateExec.md#needHashTable) |

Used when:

* `AggregateCodegenSupport` is requested to [doProduceWithoutKeys](#doProduceWithoutKeys)

## Implementations

* [HashAggregateExec](HashAggregateExec.md)
* [SortAggregateExec](SortAggregateExec.md)

## <span id="supportCodegen"> supportCodegen Flag

```scala
supportCodegen: Boolean
```

`supportCodegen` is part of the [CodegenSupport](CodegenSupport.md#supportCodegen) abstraction.

---

`supportCodegen` is enabled (`true`) when all the following hold:

* All `AttributeReference`s in the [aggregateBufferAttributes](#aggregateBufferAttributes) use [mutable data types](../UnsafeRow.md#isMutable)
* There are no [ImperativeAggregate](../expressions/ImperativeAggregate.md)s (among the [AggregateFunction](../expressions/AggregateExpression.md#aggregateFunction)s of the [AggregateExpressions](BaseAggregateExec.md#aggregateExpressions))

## <span id="doProduce"> Generating Java Source Code for Produce Path

```scala
doProduce(
  ctx: CodegenContext): String
```

`doProduce` is part of the [CodegenSupport](CodegenSupport.md#doProduce) abstraction.

---

`doProduce` [doProduceWithoutKeys](#doProduceWithoutKeys) when this aggregate operator uses no [grouping keys](BaseAggregateExec.md#groupingExpressions). Otherwise, `doProduce` [doProduceWithKeys](#doProduceWithKeys).

### <span id="doProduceWithoutKeys"> doProduceWithoutKeys

```scala
doProduceWithoutKeys(
  ctx: CodegenContext): String
```

`doProduceWithoutKeys`...FIXME

## <span id="doConsume"> Generating Java Source Code for Consume Path

```scala
doConsume(
  ctx: CodegenContext,
  input: Seq[ExprCode],
  row: ExprCode): String
```

`doConsume` is part of the [CodegenSupport](CodegenSupport.md#doConsume) abstraction.

---

`doConsume` [doConsumeWithoutKeys](#doConsumeWithoutKeys) when this aggregate operator uses no [grouping keys](BaseAggregateExec.md#groupingExpressions). Otherwise, `doConsume` [doConsumeWithKeys](#doConsumeWithKeys).

### <span id="doConsumeWithoutKeys"> doConsumeWithoutKeys

```scala
doConsumeWithoutKeys(
  ctx: CodegenContext,
  input: Seq[ExprCode]): String
```

`doConsumeWithoutKeys`...FIXME
