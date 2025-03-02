# AggregationIterators

`AggregationIterator` is an [abstraction](#contract) of [aggregation iterators](#implementations) of [UnsafeRow](../UnsafeRow.md)s.

```scala
abstract class AggregationIterator(...)
extends Iterator[UnsafeRow]
```

From [scala.collection.Iterator]({{ scala.api }}/scala/collection/Iterator.html):

> Iterators are data structures that allow to iterate over a sequence of elements. They have a `hasNext` method for checking if there is a next element available, and a `next` method which returns the next element and discards it from the iterator.

## Implementations

* [ObjectAggregationIterator](ObjectAggregationIterator.md)
* [SortBasedAggregationIterator](SortBasedAggregationIterator.md)
* [TungstenAggregationIterator](TungstenAggregationIterator.md)

## Creating Instance

`AggregationIterator` takes the following to be created:

* <span id="partIndex"> Partition ID
* <span id="groupingExpressions"> Grouping [NamedExpression](../expressions/NamedExpression.md)s
* <span id="inputAttributes"> Input [Attribute](../expressions/Attribute.md)s
* <span id="aggregateExpressions"> [AggregateExpression](../expressions/AggregateExpression.md)s
* <span id="aggregateAttributes"> Aggregate [Attribute](../expressions/Attribute.md)s
* <span id="initialInputBufferOffset"> Initial input buffer offset
* <span id="resultExpressions"> Result [NamedExpression](../expressions/NamedExpression.md)s
* <span id="newMutableProjection"> Function to create a new `MutableProjection` given expressions and attributes (`(Seq[Expression], Seq[Attribute]) => MutableProjection`)

??? note "Abstract Class"
    `AggregationIterator` is an abstract class and cannot be created directly. It is created indirectly for the [concrete AggregationIterators](#implementations).

## <span id="AggregateModes"> AggregateModes

When [created](#creating-instance), `AggregationIterator` makes sure that there are at most 2 distinct `AggregateMode`s of the [AggregateExpression](#aggregateExpressions)s.

The `AggregateMode`s have to be a subset of the following mode pairs:

* `Partial` and `PartialMerge`
* `Final` and `Complete`

## <span id="aggregateFunctions"> AggregateFunctions

```scala
aggregateFunctions: Array[AggregateFunction]
```

When [created](#creating-instance), `AggregationIterator` [initializes AggregateFunctions](#initializeAggregateFunctions) in the [aggregateExpressions](#aggregateExpressions) (with [initialInputBufferOffset](#initialInputBufferOffset)).

## <span id="initializeAggregateFunctions"> initializeAggregateFunctions

```scala
initializeAggregateFunctions(
  expressions: Seq[AggregateExpression],
  startingInputBufferOffset: Int): Array[AggregateFunction]
```

`initializeAggregateFunctions`...FIXME

`initializeAggregateFunctions` is used when:

* `AggregationIterator` is requested for the [aggregateFunctions](#aggregateFunctions)
* `ObjectAggregationIterator` is requested for the [mergeAggregationBuffers](ObjectAggregationIterator.md#mergeAggregationBuffers)
* `TungstenAggregationIterator` is requested to [switchToSortBasedAggregation](TungstenAggregationIterator.md#switchToSortBasedAggregation)

## <span id="generateProcessRow"> Generating Process Row Function

```scala
generateProcessRow(
  expressions: Seq[AggregateExpression],
  functions: Seq[AggregateFunction],
  inputAttributes: Seq[Attribute]): (InternalRow, InternalRow) => Unit
```

!!! note "`generateProcessRow` is a procedure"
    `generateProcessRow` creates a Scala function (_procedure_) that takes two [InternalRow](../InternalRow.md)s and produces no output.

    ```scala
    def generateProcessRow(currentBuffer: InternalRow, row: InternalRow): Unit = {
      ...
    }
    ```

`generateProcessRow` creates a mutable `JoinedRow` (of two [InternalRow](../InternalRow.md)s).

`generateProcessRow` branches off based on the given [AggregateExpression](../expressions/AggregateExpression.md)s (`expressions`).

With no [AggregateExpression](../expressions/AggregateExpression.md)s (`expressions`), `generateProcessRow` creates a function that does nothing (and "swallows" the input).

!!! note "`functions` Argument"
    `generateProcessRow` works differently based on the type of the given [AggregateFunction](../expressions/AggregateFunction.md)s:

    * [DeclarativeAggregate](../expressions/DeclarativeAggregate.md)
    * [AggregateFunction](../expressions/AggregateFunction.md)
    * [ImperativeAggregate](../expressions/ImperativeAggregate.md)

Otherwise, with some [AggregateExpression](../expressions/AggregateExpression.md)s (`expressions`), `generateProcessRow`...FIXME

---

`generateProcessRow` is used when:

* `AggregationIterator` is requested for the [processRow function](#processRow)
* `ObjectAggregationIterator` is requested for the [mergeAggregationBuffers function](ObjectAggregationIterator.md#mergeAggregationBuffers)
* `TungstenAggregationIterator` is requested to [switchToSortBasedAggregation](TungstenAggregationIterator.md#switchToSortBasedAggregation)

## <span id="generateOutput"> generateOutput

```scala
generateOutput: (UnsafeRow, InternalRow) => UnsafeRow
```

When [created](#creating-instance), `AggregationIterator` [creates a ResultProjection function](#generateResultProjection).

`generateOutput` is used when:

* `ObjectAggregationIterator` is requested for the [next element](ObjectAggregationIterator.md#next) and to [outputForEmptyGroupingKeyWithoutInput](ObjectAggregationIterator.md#outputForEmptyGroupingKeyWithoutInput)
* `SortBasedAggregationIterator` is requested for the [next element](SortBasedAggregationIterator.md#next) and to [outputForEmptyGroupingKeyWithoutInput](SortBasedAggregationIterator.md#outputForEmptyGroupingKeyWithoutInput)
* `TungstenAggregationIterator` is requested for the [next element](TungstenAggregationIterator.md#next) and to [outputForEmptyGroupingKeyWithoutInput](TungstenAggregationIterator.md#outputForEmptyGroupingKeyWithoutInput)

### <span id="generateResultProjection"> generateResultProjection

```scala
generateResultProjection(): (UnsafeRow, InternalRow) => UnsafeRow
```

`generateResultProjection`...FIXME

## <span id="initializeBuffer"> initializeBuffer

```scala
initializeBuffer(
  buffer: InternalRow): Unit
```

`initializeBuffer` requests the [expressionAggInitialProjection](#expressionAggInitialProjection) to [store an execution result](../expressions/MutableProjection.md#target) of an empty row in the given [InternalRow](../InternalRow.md) (`buffer`).

`initializeBuffer` requests [all the ImperativeAggregate functions](#allImperativeAggregateFunctions) to [initialize](../expressions/ImperativeAggregate.md#initialize) with the `buffer` internal row.

---

`initializeBuffer` is used when:

* `MergingSessionsIterator` is requested to `newBuffer`, `initialize`, `next`, `outputForEmptyGroupingKeyWithoutInput`
* `SortBasedAggregationIterator` is requested to [newBuffer](SortBasedAggregationIterator.md#newBuffer), [initialize](SortBasedAggregationIterator.md#initialize), [next](SortBasedAggregationIterator.md#next) and [outputForEmptyGroupingKeyWithoutInput](SortBasedAggregationIterator.md#outputForEmptyGroupingKeyWithoutInput)
