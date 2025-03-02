# LogicalQueryStageStrategy Execution Planning Strategy

`LogicalQueryStageStrategy` is an [execution planning strategy](SparkStrategy.md) that [plans the following logical operators](#apply):

* [LogicalQueryStage](../logical-operators/LogicalQueryStage.md) with [BroadcastQueryStageExec](../physical-operators/BroadcastQueryStageExec.md) physical operator
* [Join](../logical-operators/Join.md) thereof

`LogicalQueryStageStrategy` is part of the [strategies](../SparkPlanner.md#strategies) of the [SparkPlanner](../SparkPlanner.md).

## <span id="apply"> Executing Strategy

```scala
apply(
  plan: LogicalPlan): Seq[SparkPlan]
```

For [Join](../logical-operators/Join.md) operators with an [equi-join condition](../ExtractEquiJoinKeys.md) and the left or right side being [broadcast stages](#isBroadcastStage), `apply` gives a [BroadcastHashJoinExec](../physical-operators/BroadcastHashJoinExec.md) physical operator.

For other `Join` operators and the left or right side being [broadcast stages](#isBroadcastStage), `apply` gives a [BroadcastNestedLoopJoinExec](../physical-operators/BroadcastNestedLoopJoinExec.md) physical operator.

For [LogicalQueryStage](../logical-operators/LogicalQueryStage.md) operators, `apply` simply gives the associated [physical plan](../logical-operators/LogicalQueryStage.md#physicalPlan).

`apply` is part of the [GenericStrategy](../catalyst/GenericStrategy.md#apply) abstraction.

### <span id="isBroadcastStage"> isBroadcastStage

```scala
isBroadcastStage(
  plan: LogicalPlan): Boolean
```

`isBroadcastStage` is `true` when the given [LogicalPlan](../logical-operators/LogicalPlan.md) is a [LogicalQueryStage](../logical-operators/LogicalQueryStage.md) leaf logical operator with a [BroadcastQueryStageExec](../physical-operators/BroadcastQueryStageExec.md) physical operator. Otherwise, `isBroadcastStage` is `false`.
