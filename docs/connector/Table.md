# Table

`Table` is an [abstraction](#contract) of [logical structured data set](#implementations):

* a directory or files on a file system
* a topic of Apache Kafka
* a table in a catalog

`Table` can be loaded using [TableCatalog](catalog/TableCatalog.md#loadTable).

## Contract

### <span id="capabilities"> Capabilities

```java
Set<TableCapability> capabilities()
```

[TableCapabilities](TableCapability.md) of the table

Used when `Table` is asked whether or not it [supports a given capability](TableHelper.md#supports)

### <span id="columns"> Columns

```java
Column[] columns()
```

[Column](catalog/Column.md)s of this table

Default: [structTypeToV2Columns](catalog/CatalogV2Util.md#structTypeToV2Columns) based on the [schema](#schema)

Used when `Table` is asked whether or not it [supports a given capability](TableHelper.md#supports)

### Name

```java
String name()
```

Name of the table

### Partitioning

```java
Transform[] partitioning()
```

Partitions of the table (as [Transform](Transform.md)s)

Default: (empty)

Used when:

* [ResolveInsertInto](../logical-analysis-rules/ResolveInsertInto.md) logical analysis rule is executed
* `DataFrameWriter` is requested to [insertInto](../DataFrameWriter.md#insertInto) and [save](../DataFrameWriter.md#save)
* [DescribeTableExec](../physical-operators/DescribeTableExec.md) physical operator is executed

### Properties

```java
Map<String, String> properties()
```

Table properties

Default: (empty)

Used when:

* [DescribeTableExec](../physical-operators/DescribeTableExec.md) and [ShowTablePropertiesExec](../physical-operators/ShowTablePropertiesExec.md) physical operators are executed

### Schema

```java
StructType schema()
```

!!! note "Deprecated"
    Use [columns](#columns) instead.

[StructType](../types/StructType.md) of the table

Used when:

* `DataSourceV2Relation` utility is used to [create a DataSourceV2Relation logical operator](../logical-operators/DataSourceV2Relation.md#create)
* `SimpleTableProvider` is requested to [inferSchema](SimpleTableProvider.md#inferSchema)
* [DataSourceV2Strategy](../execution-planning-strategies/DataSourceV2Strategy.md) execution planning strategy is executed
* [DescribeTableExec](../physical-operators/DescribeTableExec.md) physical operator is executed
* `FileDataSourceV2` is requested to [inferSchema](../connectors/FileDataSourceV2.md#inferSchema)
* (Spark Structured Streaming) `TextSocketTable` is requested for a `ScanBuilder` with a read schema
* (Spark Structured Streaming) `DataStreamReader` is requested to load data

## Implementations

* [FileTable](../connectors/FileTable.md)
* [KafkaTable](../kafka/KafkaTable.md)
* [NoopTable](../noop/NoopTable.md)
* [StagedTable](StagedTable.md)
* [SupportsRead](SupportsRead.md)
* [SupportsWrite](SupportsWrite.md)
* [V1Table](V1Table.md)
* _others_
