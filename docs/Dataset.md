# Dataset

`Dataset[T]` is a strongly-typed data structure that represents a structured query over rows of `T` type.

`Dataset` is created using [SQL](sql/index.md) or [Dataset](spark-sql-dataset-operators.md) high-level declarative "languages".

![Dataset's Internals](images/spark-sql-Dataset.png)

It is fair to say that `Dataset` is a Spark SQL developer-friendly layer over the following two low-level entities:

1. [QueryExecution](QueryExecution.md) (with the parsed yet unanalyzed [LogicalPlan](logical-operators/LogicalPlan.md) of a structured query)

1. [Encoder](Encoder.md) (of the type of the records for fast serialization and deserialization to and from [InternalRow](InternalRow.md))

## Creating Instance

`Dataset` takes the following when created:

* <span id="sparkSession"> [SparkSession](SparkSession.md)
* <span id="queryExecution"> [QueryExecution](QueryExecution.md)
* <span id="encoder"> [Encoder](Encoder.md) of `T`

!!! note
    `Dataset` can be created using [LogicalPlan](logical-operators/LogicalPlan.md) when [executed using SessionState](SessionState.md#executePlan).

When created, `Dataset` requests [QueryExecution](#queryExecution) to [assert analyzed phase is successful](QueryExecution.md#assertAnalyzed).

`Dataset` is created when:

* [Dataset.apply](#apply) (for a [LogicalPlan](logical-operators/LogicalPlan.md) and a [SparkSession](SparkSession.md) with the [Encoder](Encoder.md) in a Scala implicit scope)

* [Dataset.ofRows](#ofRows) (for a [LogicalPlan](logical-operators/LogicalPlan.md) and a [SparkSession](SparkSession.md))

* [Dataset.toDF](Dataset-untyped-transformations.md#toDF) untyped transformation is used

* [Dataset.select](spark-sql-Dataset-typed-transformations.md#select), [Dataset.randomSplit](spark-sql-Dataset-typed-transformations.md#randomSplit) and [Dataset.mapPartitions](spark-sql-Dataset-typed-transformations.md#mapPartitions) typed transformations are used

* [KeyValueGroupedDataset.agg](KeyValueGroupedDataset.md#agg) operator is used (that requests `KeyValueGroupedDataset` to [aggUntyped](KeyValueGroupedDataset.md#aggUntyped))

* [SparkSession.emptyDataset](SparkSession.md#emptyDataset) and [SparkSession.range](SparkSession.md#range) operators are used

* `CatalogImpl` is requested to
[makeDataset](CatalogImpl.md#makeDataset) (when requested to [list databases](CatalogImpl.md#listDatabases), [tables](CatalogImpl.md#listTables), [functions](CatalogImpl.md#listFunctions) and [columns](CatalogImpl.md#listColumns))

## <span id="observe"> observe

```scala
observe(
  observation: Observation,
  expr: Column,
  exprs: Column*): Dataset[T]
observe(
  name: String,
  expr: Column,
  exprs: Column*): Dataset[T]
```

When executed with `name` argument, `observe` creates a typed `Dataset` with a [CollectMetrics](logical-operators/CollectMetrics.md) logical operator.

For an [Observation](Observation.md) argument, `observe` requests the `Observation` to [observe](Observation.md#on) this `Dataset` (that in turn uses the `name`-argument `observe`).

!!! note "Requirements"
    1. An `Observation` can be used with a `Dataset` only once
    1. `Observation` does not support streaming `Dataset`s.

## <span id="repartition"> repartition

```scala
repartition(
  partitionExprs: Column*): Dataset[T] // (1)!
repartition(
  numPartitions: Int): Dataset[T] // (2)!
repartition(
  numPartitions: Int,
  partitionExprs: Column*): Dataset[T]  // (3)!
```

1. An alias of [repartitionByExpression](#repartitionByExpression) with undefined `numPartitions`
1. Creates a [Repartition](logical-operators/Repartition.md) logical operator with `shuffle` enabled
1. An alias of [repartitionByExpression](#repartitionByExpression)

### <span id="repartitionByExpression"> repartitionByExpression

```scala
repartitionByExpression(
  numPartitions: Option[Int],
  partitionExprs: Seq[Column]): Dataset[T]
```

`repartitionByExpression` [withTypedPlan](#withTypedPlan) with a new [RepartitionByExpression](logical-operators/RepartitionByExpression.md) (with the given `numPartitions` and `partitionExprs`, and the [logicalPlan](#logicalPlan)).

### <span id="repartition-example"> Example: Number of Partitions Only

```scala
val numsRepd = nums.repartition(numPartitions = 4)
```

```scala
assert(numsRepd.rdd.getNumPartitions == 4, "Number of partitions should be 4")
```

```text
scala> numsRepd.explain(extended = true)
== Parsed Logical Plan ==
Repartition 4, true
+- RepartitionByExpression [id#4L ASC NULLS FIRST]
   +- Range (0, 10, step=1, splits=Some(16))

== Analyzed Logical Plan ==
id: bigint
Repartition 4, true
+- RepartitionByExpression [id#4L ASC NULLS FIRST]
   +- Range (0, 10, step=1, splits=Some(16))

== Optimized Logical Plan ==
Repartition 4, true
+- Range (0, 10, step=1, splits=Some(16))

== Physical Plan ==
AdaptiveSparkPlan isFinalPlan=false
+- Exchange RoundRobinPartitioning(4), REPARTITION_BY_NUM, [id=#75]
   +- Range (0, 10, step=1, splits=16)
```

## <span id="repartitionByRange"> repartitionByRange

```scala
repartitionByRange(
  partitionExprs: Column*): Dataset[T]
repartitionByRange(
  numPartitions: Int,
  partitionExprs: Column*): Dataset[T]
repartitionByRange(
    numPartitions: Option[Int],
    partitionExprs: Seq[Column]): Dataset[T] // (1)!
```

1. A private method

`repartitionByRange` creates a [SortOrder](expressions/SortOrder.md)s with [Ascending](expressions/SortOrder.md#Ascending) sorting direction (_ascending nulls first_) for the given `partitionExprs` with no sorting specified.

In the end, `repartitionByRange` creates a [Dataset](#apply) with a [RepartitionByExpression](logical-operators/RepartitionByExpression.md) (with the `SortOrder`s, the [logicalPlan](#logicalPlan) and the given `numPartitions`).

### <span id="repartitionByRange-example"> Example: Partition Expressions Only

```scala
val nums = spark.range(10).repartitionByRange($"id".asc)
```

```text
scala> println(nums.queryExecution.logical.numberedTreeString)
00 'RepartitionByExpression ['id ASC NULLS FIRST]
01 +- Range (0, 10, step=1, splits=Some(16))
```

!!! note
    Adaptive Query Execution is enabled.

```text
scala> println(nums.queryExecution.toRdd.getNumPartitions)
1
```

```text
scala> println(nums.queryExecution.toRdd.toDebugString)
(1) SQLExecutionRDD[17] at toRdd at <console>:24 []
 |  ShuffledRowRDD[16] at toRdd at <console>:24 []
 +-(16) MapPartitionsRDD[15] at toRdd at <console>:24 []
    |   MapPartitionsRDD[11] at toRdd at <console>:24 []
    |   MapPartitionsRDD[10] at toRdd at <console>:24 []
    |   ParallelCollectionRDD[9] at toRdd at <console>:24 []
```

### <span id="repartitionByRange-example-numPartitions"> Example: Number of Partitions and Partition Expressions

```text
val q = spark.range(10).repartitionByRange(numPartitions = 5, $"id")
```

```text
scala> println(q.queryExecution.logical.numberedTreeString)
00 'RepartitionByExpression ['id ASC NULLS FIRST], 5
01 +- Range (0, 10, step=1, splits=Some(16))
```

```text
scala> println(q.queryExecution.toRdd.getNumPartitions)
5
```

```text
scala> println(q.queryExecution.toRdd.toDebugString)
(5) SQLExecutionRDD[8] at toRdd at <console>:24 []
 |  ShuffledRowRDD[7] at toRdd at <console>:24 []
 +-(16) MapPartitionsRDD[6] at toRdd at <console>:24 []
    |   MapPartitionsRDD[2] at toRdd at <console>:24 []
    |   MapPartitionsRDD[1] at toRdd at <console>:24 []
    |   ParallelCollectionRDD[0] at toRdd at <console>:24 []
```

## <span id="logicalPlan"> LogicalPlan

```scala
logicalPlan: LogicalPlan
```

`Dataset` initializes an internal [LogicalPlan](logical-operators/LogicalPlan.md) when [created](#creating-instance).

`logicalPlan` requests the [QueryExecution](#queryExecution) for a [LogicalPlan](#commandExecuted) (with commands executed per the [command execution mode](QueryExecution.md#mode)).

With [SQLConf.FAIL_AMBIGUOUS_SELF_JOIN_ENABLED](SQLConf.md#FAIL_AMBIGUOUS_SELF_JOIN_ENABLED) enabled, `logicalPlan`...FIXME. Otherwise, `logicalPlan` returns the `LogicalPlan` intact.

## Lazy Values

The following are Scala **lazy values** of `Dataset`. Using lazy values guarantees that the code to initialize them is executed once only (when accessed for the first time) and the computed value never changes afterwards.

Learn more in the [Scala Language Specification]({{ scala.spec }}/05-classes-and-objects.html#lazy).

### <span id="rdd"> RDD

```scala
rdd: RDD[T]
```

`rdd` ...FIXME

`rdd` is used when:

* `Dataset` is requested for an [RDD](#rdd) and [withNewRDDExecutionId](#withNewRDDExecutionId)

### <span id="rddQueryExecution"> QueryExecution

```scala
rddQueryExecution: QueryExecution
```

`rddQueryExecution` [creates a deserializer](CatalystSerde.md#deserialize) for the `T` type and the [logical query plan](#logicalPlan) of this `Dataset`.

In other words, `rddQueryExecution` simply adds a new [DeserializeToObject](logical-operators/DeserializeToObject.md) unary logical operator as the parent of the current [logical query plan](#logicalPlan) (that in turn becomes a child operator).

In the end, `rddQueryExecution` requests the [SparkSession](#sparkSession) for the [SessionState](SparkSession.md#sessionState) to [create a QueryExecution](SessionState.md#executePlan) for the query plan with the top-most `DeserializeToObject`.

`rddQueryExecution` is used when:

* `Dataset` is requested for an [RDD](#rdd) and [withNewRDDExecutionId](#withNewRDDExecutionId)

## <span id="withNewRDDExecutionId"> withNewRDDExecutionId

```scala
withNewRDDExecutionId[U](
  body: => U): U
```

`withNewRDDExecutionId` executes the input `body` action (that produces the result of type `U`) under a [new execution id](SQLExecution.md#withNewExecutionId) (with the [QueryExecution](#rddQueryExecution)).

`withNewRDDExecutionId` requests the [QueryExecution](#rddQueryExecution) for the [SparkPlan](#executedPlan) to [resetMetrics](physical-operators/SparkPlan.md#resetMetrics).

`withNewRDDExecutionId` is used when the following `Dataset` operators are used:

* [reduce](#reduce), [foreach](#foreach) and [foreachPartition](#foreachPartition)

## <span id="writeTo"> writeTo

```scala
writeTo(
  table: String): DataFrameWriterV2[T]
```

`writeTo` creates a [DataFrameWriterV2](DataFrameWriterV2.md) for the input table and this `Dataset`.

## <span id="write"> write

```scala
write: DataFrameWriter[T]
```

`write` creates a [DataFrameWriter](DataFrameWriter.md) for this `Dataset`.

## <span id="withTypedPlan"> withTypedPlan

```scala
withTypedPlan[U : Encoder](
  logicalPlan: LogicalPlan): Dataset[U]
```

??? note "Final Method"
    `withTypedPlan` is annotated with [@inline](https://www.scala-lang.org/api/current/scala/inline.html) annotation that requests the Scala compiler to try especially hard to inline it

`withTypedPlan`...FIXME

## <span id="withSetOperator"> withSetOperator

```scala
withSetOperator[U: Encoder](
  logicalPlan: LogicalPlan): Dataset[U]
```

??? note "Final Method"
    `withSetOperator` is annotated with [@inline](https://www.scala-lang.org/api/current/scala/inline.html) annotation that requests the Scala compiler to try especially hard to inline it

`withSetOperator`...FIXME

## <span id="apply"> apply

```scala
apply[T: Encoder](
  sparkSession: SparkSession,
  logicalPlan: LogicalPlan): Dataset[T]
```

`apply`...FIXME

`apply` is used when:

* `Dataset` is requested to [withTypedPlan](#withTypedPlan) and [withSetOperator](#withSetOperator)

## <span id="withAction"> Executing Action Under New Execution ID

```scala
withAction[U](
  name: String,
  qe: QueryExecution)(
  action: SparkPlan => U)
```

`withAction` [creates a new execution ID](SQLExecution.md#withNewExecutionId) to execute the given `action` with the [optimized physical query plan](QueryExecution.md#executedPlan) (of the given [QueryExecution](QueryExecution.md)).

`withAction` [resets the performance metrics](physical-operators/SparkPlan.md#resetMetrics).

---

`withAction` is used to execute the following `Dataset` actions:

Action | Name
-------|-----
 [isEmpty](#isEmpty) | `isEmpty`
 [checkpoint](#checkpoint) | `checkpoint` or `localCheckpoint`
 [head](#head) | `head`
 [tail](#tail) | `tail`
 [collect](#collect) | `collect`
 [collectAsList](#collectAsList) | `collectAsList`
 [toLocalIterator](#toLocalIterator) | `toLocalIterator`
 [count](#count) | `count`
 [collectToPython](#collectToPython) | `collectToPython`
 [tailToPython](#tailToPython) | `tailToPython`
 [collectAsArrowToR](#collectAsArrowToR) | `collectAsArrowToR`
 [collectAsArrowToPython](#collectAsArrowToPython) | `collectAsArrowToPython`

## <span id="collectAsArrowToPython"> collectAsArrowToPython

```scala
collectAsArrowToPython: Array[Any]
```

`collectAsArrowToPython`...FIXME

---

`collectAsArrowToPython` is used when:

* `PandasConversionMixin` ([PySpark]({{ book.pyspark }}/sql/PandasConversionMixin#_collect_as_arrow)) is requested to `_collect_as_arrow`

## <span id="collectToPython"> collectToPython

```scala
collectToPython(): Array[Any]
```

`collectToPython`...FIXME

---

`collectToPython` is used when:

* `DataFrame` ([PySpark]({{ book.pyspark }}/sql/DataFrame#collect)) is requested to `collect`

<!---
## Review Me

Datasets are _lazy_ and structured query operators and expressions are only triggered when an action is invoked.

```text
import org.apache.spark.sql.SparkSession
val spark: SparkSession = ...

scala> val dataset = spark.range(5)
dataset: org.apache.spark.sql.Dataset[Long] = [id: bigint]

// Variant 1: filter operator accepts a Scala function
dataset.filter(n => n % 2 == 0).count

// Variant 2: filter operator accepts a Column-based SQL expression
dataset.filter('value % 2 === 0).count

// Variant 3: filter operator accepts a SQL query
dataset.filter("value % 2 = 0").count
```

The <<spark-sql-dataset-operators.md#, Dataset API>> offers declarative and type-safe operators that makes for an improved experience for data processing (comparing to [DataFrames](DataFrame.md) that were a set of index- or column name-based [Row](Row.md)s).

`Dataset` offers convenience of RDDs with the performance optimizations of DataFrames and the strong static type-safety of Scala. The last feature of bringing the strong type-safety to [DataFrame](DataFrame.md) makes Dataset so appealing. All the features together give you a more functional programming interface to work with structured data.

```text
scala> spark.range(1).filter('id === 0).explain(true)
== Parsed Logical Plan ==
'Filter ('id = 0)
+- Range (0, 1, splits=8)

== Analyzed Logical Plan ==
id: bigint
Filter (id#51L = cast(0 as bigint))
+- Range (0, 1, splits=8)

== Optimized Logical Plan ==
Filter (id#51L = 0)
+- Range (0, 1, splits=8)

== Physical Plan ==
*Filter (id#51L = 0)
+- *Range (0, 1, splits=8)

scala> spark.range(1).filter(_ == 0).explain(true)
== Parsed Logical Plan ==
'TypedFilter <function1>, class java.lang.Long, [StructField(value,LongType,true)], unresolveddeserializer(newInstance(class java.lang.Long))
+- Range (0, 1, splits=8)

== Analyzed Logical Plan ==
id: bigint
TypedFilter <function1>, class java.lang.Long, [StructField(value,LongType,true)], newInstance(class java.lang.Long)
+- Range (0, 1, splits=8)

== Optimized Logical Plan ==
TypedFilter <function1>, class java.lang.Long, [StructField(value,LongType,true)], newInstance(class java.lang.Long)
+- Range (0, 1, splits=8)

== Physical Plan ==
*Filter <function1>.apply
+- *Range (0, 1, splits=8)
```

It is only with Datasets to have syntax and analysis checks at compile time (that was not possible using [DataFrame](DataFrame.md), regular SQL queries or even RDDs).

Using `Dataset` objects turns `DataFrames` of [Row](Row.md) instances into a `DataFrames` of case classes with proper names and types (following their equivalents in the case classes). Instead of using indices to access respective fields in a DataFrame and cast it to a type, all this is automatically handled by Datasets and checked by the Scala compiler.

If however a logical-operators/LogicalPlan.md[LogicalPlan] is used to <<creating-instance, create a `Dataset`>>, the logical plan is first SessionState.md#executePlan[executed] (using the current SessionState.md#executePlan[SessionState] in the `SparkSession`) that yields the [QueryExecution](QueryExecution.md) plan.

A `Dataset` is <<Queryable, Queryable>> and `Serializable`, i.e. can be saved to a persistent storage.

NOTE: SparkSession.md[SparkSession] and [QueryExecution](QueryExecution.md) are transient attributes of a `Dataset` and therefore do not participate in Dataset serialization. The only _firmly-tied_ feature of a `Dataset` is the [Encoder](Encoder.md).

You can request the ["untyped" view](spark-sql-dataset-operators.md#toDF) of a Dataset or access the spark-sql-dataset-operators.md#rdd[RDD] that is generated after executing the query. It is supposed to give you a more pleasant experience while transitioning from the legacy RDD-based or DataFrame-based APIs you may have used in the earlier versions of Spark SQL or encourage migrating from Spark Core's RDD API to Spark SQL's Dataset API.

The default storage level for `Datasets` is spark-rdd-caching.md[MEMORY_AND_DISK] because recomputing the in-memory columnar representation of the underlying table is expensive. You can however [persist a `Dataset`](caching-and-persistence.md#persist).

NOTE: Spark 2.0 has introduced a new query model called spark-structured-streaming.md[Structured Streaming] for continuous incremental execution of structured queries. That made possible to consider Datasets a static and bounded as well as streaming and unbounded data sets with a single unified API for different execution models.

A `Dataset` is spark-sql-dataset-operators.md#isLocal[local] if it was created from local collections using SparkSession.md#emptyDataset[SparkSession.emptyDataset] or SparkSession.md#createDataset[SparkSession.createDataset] methods and their derivatives like <<toDF,toDF>>. If so, the queries on the Dataset can be optimized and run locally, i.e. without using Spark executors.

NOTE: `Dataset` makes sure that the underlying `QueryExecution` is [analyzed](QueryExecution.md#analyzed) and CheckAnalysis.md#checkAnalysis[checked].

[[properties]]
[[attributes]]
.Dataset's Properties
[cols="1,2",options="header",width="100%",separator="!"]
!===
! Name
! Description

! [[deserializer]] `deserializer`
a! Deserializer expressions/Expression.md[expression] to convert internal rows to objects of type `T`

Created lazily by requesting the <<exprEnc, ExpressionEncoder>> to [resolveAndBind](ExpressionEncoder.md#resolveAndBind)

Used when:

* `Dataset` is <<apply, created>> (for a logical plan in a given `SparkSession`)

* spark-sql-dataset-operators.md#spark-sql-dataset-operators.md[Dataset.toLocalIterator] operator is used (to create a Java `Iterator` of objects of type `T`)

* `Dataset` is requested to <<collectFromPlan, collect all rows from a spark plan>>

! `logicalPlan`
a! [[logicalPlan]] Analyzed <<logical-operators/LogicalPlan.md#, logical plan>> with all <<Command.md#, logical commands>> executed and turned into a <<LocalRelation.md#creating-instance, LocalRelation>>.

[source, scala]
----
logicalPlan: LogicalPlan
----

When initialized, `logicalPlan` requests the <<queryExecution, QueryExecution>> for [analyzed logical plan](QueryExecution.md#analyzed). If the plan is a <<Command.md#, logical command>> or a union thereof, `logicalPlan` <<withAction, executes the QueryExecution>> (using <<SparkPlan.md#executeCollect, executeCollect>>).

! `planWithBarrier`
a! [[planWithBarrier]]

[source, scala]
----
planWithBarrier: AnalysisBarrier
----

! [[rdd]] `rdd`
a! (lazily-created) spark-rdd.md[RDD] of JVM objects of type `T` (as converted from rows in `Dataset` in the [internal binary row format](InternalRow.md)).

[source, scala]
----
rdd: RDD[T]
----

NOTE: `rdd` gives `RDD` with the extra execution step to convert rows from their internal binary row format to JVM objects that will impact the JVM memory as the objects are inside JVM (while were outside before). You should not use `rdd` directly.

Internally, `rdd` [creates a deserializer](CatalystSerde.md#deserialize) the Dataset's [logical plan](#logicalPlan).

[source, scala]
----
val dataset = spark.range(5).withColumn("group", 'id % 2)
scala> dataset.rdd.toDebugString
res1: String =
(8) MapPartitionsRDD[8] at rdd at <console>:26 [] // <-- extra deserialization step
 |  MapPartitionsRDD[7] at rdd at <console>:26 []
 |  MapPartitionsRDD[6] at rdd at <console>:26 []
 |  MapPartitionsRDD[5] at rdd at <console>:26 []
 |  ParallelCollectionRDD[4] at rdd at <console>:26 []

// Compare with a more memory-optimized alternative
// Avoids copies and has no schema
scala> dataset.queryExecution.toRdd.toDebugString
res2: String =
(8) MapPartitionsRDD[11] at toRdd at <console>:26 []
 |  MapPartitionsRDD[10] at toRdd at <console>:26 []
 |  ParallelCollectionRDD[9] at toRdd at <console>:26 []
----

`rdd` then requests `SessionState` to SessionState.md#executePlan[execute the logical plan] to get the corresponding [RDD of binary rows](QueryExecution.md#toRdd).

NOTE: `rdd` uses <<sparkSession, SparkSession>> to SparkSession.md#sessionState[access `SessionState`].

`rdd` then requests the Dataset's <<exprEnc, ExpressionEncoder>> for the expressions/Expression.md#dataType[data type] of the rows (using [deserializer](ExpressionEncoder.md#deserializer) expression) and spark-rdd-transformations.md#mapPartitions[maps over them (per partition)] to create records of the expected type `T`.

NOTE: `rdd` is at the "boundary" between the internal binary row format and the JVM type of the dataset. Avoid the extra deserialization step to lower JVM memory requirements of your Spark application.

!===

=== [[inputFiles]] Getting Input Files of Relations (in Structured Query) -- `inputFiles` Method

[source, scala]
----
inputFiles: Array[String]
----

`inputFiles` requests <<queryExecution, QueryExecution>> for [optimized logical plan](QueryExecution.md#optimizedPlan) and collects the following logical operators:

* [LogicalRelation](logical-operators/LogicalRelation.md) with [FileRelation](FileRelation.md) (as the [BaseRelation](logical-operators/LogicalRelation.md#relation))

* [FileRelation](FileRelation.md)

* [HiveTableRelation](hive/HiveTableRelation.md)

`inputFiles` then requests the logical operators for their underlying files:

* [inputFiles](FileRelation.md#inputFiles) of the `FileRelations`

* [locationUri](CatalogStorageFormat.md#locationUri) of the `HiveTableRelation`

=== [[resolve]] `resolve` Internal Method

[source, scala]
----
resolve(colName: String): NamedExpression
----

CAUTION: FIXME

=== [[isLocal]] Is Dataset Local? -- `isLocal` Method

[source, scala]
----
isLocal: Boolean
----

`isLocal` flag is enabled (i.e. `true`) when operators like `collect` or `take` could be run locally, i.e. without using executors.

Internally, `isLocal` checks whether the logical query plan of a `Dataset` is LocalRelation.md[LocalRelation].

=== [[isStreaming]] Is Dataset Streaming? -- `isStreaming` method

[source, scala]
----
isStreaming: Boolean
----

`isStreaming` is enabled (i.e. `true`) when the logical plan logical-operators/LogicalPlan.md#isStreaming[is streaming].

Internally, `isStreaming` takes the Dataset's logical-operators/LogicalPlan.md[logical plan] and gives logical-operators/LogicalPlan.md#isStreaming[whether the plan is streaming or not].

=== [[Queryable]] Queryable

CAUTION: FIXME

## <span id="ofRows"> Creating DataFrame (For Logical Query Plan and SparkSession)

```scala
ofRows(
  sparkSession: SparkSession,
  logicalPlan: LogicalPlan): DataFrame
```

!!! note
    `ofRows` is part of `Dataset` Scala object that is marked as a `private[sql]` and so can only be accessed from code in `org.apache.spark.sql` package.

`ofRows` returns [DataFrame](DataFrame.md) (which is the type alias for `Dataset[Row]`). `ofRows` uses [RowEncoder](RowEncoder.md) to convert the schema (based on the input `logicalPlan` logical plan).

Internally, `ofRows` SessionState.md#executePlan[prepares the input `logicalPlan` for execution] and creates a `Dataset[Row]` with the current SparkSession.md[SparkSession], the [QueryExecution](QueryExecution.md) and [RowEncoder](RowEncoder.md).

`ofRows` is used when:

* `DataFrameReader` is requested to [load data from a data source](DataFrameReader.md#load)

* `Dataset` is requested to execute <<checkpoint, checkpoint>>, `mapPartitionsInR`, <<withPlan, untyped transformations>> and <<withSetOperator, set-based typed transformations>>

* `RelationalGroupedDataset` is requested to [create a DataFrame from aggregate expressions](RelationalGroupedDataset.md#toDF), `flatMapGroupsInR` and `flatMapGroupsInPandas`

* `SparkSession` is requested to <<SparkSession.md#baseRelationToDataFrame, create a DataFrame from a BaseRelation>>, <<SparkSession.md#createDataFrame, createDataFrame>>, <<SparkSession.md#internalCreateDataFrame, internalCreateDataFrame>>, <<SparkSession.md#sql, sql>> and <<SparkSession.md#table, table>>

* `CacheTableCommand`, <<CreateTempViewUsing.md#run, CreateTempViewUsing>>, <<InsertIntoDataSourceCommand.md#run, InsertIntoDataSourceCommand>> and `SaveIntoDataSourceCommand` logical commands are executed (run)

* `DataSource` is requested to [writeAndRead](DataSource.md#writeAndRead) (for a [CreatableRelationProvider](CreatableRelationProvider.md))

* `FrequentItems` is requested to `singlePassFreqItems`

* `StatFunctions` is requested to `crossTabulate` and `summary`

* Spark Structured Streaming's `DataStreamReader` is requested to `load`

* Spark Structured Streaming's `DataStreamWriter` is requested to `start`

* Spark Structured Streaming's `FileStreamSource` is requested to `getBatch`

* Spark Structured Streaming's `MemoryStream` is requested to `toDF`

=== [[withNewExecutionId]] Tracking Multi-Job Structured Query Execution (PySpark) -- `withNewExecutionId` Internal Method

[source, scala]
----
withNewExecutionId[U](body: => U): U
----

`withNewExecutionId` executes the input `body` action under [new execution id](SQLExecution.md#withNewExecutionId).

NOTE: `withNewExecutionId` sets a unique execution id so that all Spark jobs belong to the `Dataset` action execution.

=== [[collectFromPlan]] Collecting All Rows From Spark Plan -- `collectFromPlan` Internal Method

[source, scala]
----
collectFromPlan(plan: SparkPlan): Array[T]
----

`collectFromPlan`...FIXME

NOTE: `collectFromPlan` is used for spark-sql-dataset-operators.md#head[Dataset.head], spark-sql-dataset-operators.md#collect[Dataset.collect] and spark-sql-dataset-operators.md#collectAsList[Dataset.collectAsList] operators.

=== [[selectUntyped]] `selectUntyped` Internal Method

[source, scala]
----
selectUntyped(columns: TypedColumn[_, _]*): Dataset[_]
----

`selectUntyped`...FIXME

NOTE: `selectUntyped` is used exclusively when <<spark-sql-Dataset-typed-transformations.md#select, Dataset.select>> typed transformation is used.

=== [[sortInternal]] `sortInternal` Internal Method

[source, scala]
----
sortInternal(global: Boolean, sortExprs: Seq[Column]): Dataset[T]
----

`sortInternal` <<withTypedPlan, creates a Dataset>> with <<Sort.md#, Sort>> unary logical operator (and the <<logicalPlan, logicalPlan>> as the <<Sort.md#child, child logical plan>>).

[source, scala]
----
val nums = Seq((0, "zero"), (1, "one")).toDF("id", "name")
// Creates a Sort logical operator:
// - descending sort direction for id column (specified explicitly)
// - name column is wrapped with ascending sort direction
val numsSorted = nums.sort('id.desc, 'name)
val logicalPlan = numsSorted.queryExecution.logical
scala> println(logicalPlan.numberedTreeString)
00 'Sort ['id DESC NULLS LAST, 'name ASC NULLS FIRST], true
01 +- Project [_1#11 AS id#14, _2#12 AS name#15]
02    +- LocalRelation [_1#11, _2#12]
----

Internally, `sortInternal` firstly builds ordering expressions for the given `sortExprs` columns, i.e. takes the `sortExprs` columns and makes sure that they are <<spark-sql-Expression-SortOrder.md#, SortOrder>> expressions already (and leaves them untouched) or wraps them into <<spark-sql-Expression-SortOrder.md#, SortOrder>> expressions with <<spark-sql-Expression-SortOrder.md#Ascending, Ascending>> sort direction.

In the end, `sortInternal` <<withTypedPlan, creates a Dataset>> with <<Sort.md#, Sort>> unary logical operator (with the ordering expressions, the given `global` flag, and the <<logicalPlan, logicalPlan>> as the <<Sort.md#child, child logical plan>>).

NOTE: `sortInternal` is used for the <<spark-sql-dataset-operators.md#sort, sort>> and <<spark-sql-dataset-operators.md#sortWithinPartitions, sortWithinPartitions>> typed transformations in the Dataset API (with the only change of the `global` flag being enabled and disabled, respectively).

=== [[withPlan]] Helper Method for Untyped Transformations and Basic Actions -- `withPlan` Internal Method

[source, scala]
----
withPlan(logicalPlan: LogicalPlan): DataFrame
----

`withPlan` simply uses <<ofRows, ofRows>> internal factory method to create a `DataFrame` for the input <<logical-operators/LogicalPlan.md#, LogicalPlan>> and the current <<sparkSession, SparkSession>>.

NOTE: `withPlan` is annotated with Scala's https://www.scala-lang.org/api/current/scala/inline.html[@inline] annotation that requests the Scala compiler to try especially hard to inline it.

`withPlan` is used in [untyped transformations](Dataset-untyped-transformations.md)

=== [[i-want-more]] Further Reading and Watching

* (video) https://youtu.be/i7l3JQRx7Qw[Structuring Spark: DataFrames, Datasets, and Streaming]
-->
