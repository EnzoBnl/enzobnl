<!--NOTE HEAD START-->
<link rel="icon" type="image/png" href="./imgs/favicon_db.png" />
<script src="https://cdnjs.cloudflare.com/ajax/libs/mermaid/8.0.0/mermaid.min.js"></script>
<script type="text/x-mathjax-config">MathJax.Hub.Config({tex2jax: {skipTags: ['script', 'noscript','style', 'textarea', 'pre'],inlineMath: [['$','$']]}});</script>
<script src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML" type="text/javascript"></script>
<script>document.body.style.background = "#f2f2f2";</script>
<!--NOTE HEAD END-->

## Spark
### Architecture insights
1 cluster Node $\rightarrow$ is composed of several worker nodes

1 worker node $\rightarrow$ launches 1 or several executors

1 executor $\rightarrow$
- is a java application (=1 JVM)
- may start several threads

1 executor's thread $\rightarrow$
- runs several tasks

1 task $\rightarrow$ acts on 1 RDD's partition

1 job $\rightarrow$ is composed of multiple stages

1 stage $\rightarrow$ is a collection of tasks

1 applications $\rightarrow$
- runs several spark jobs
- launch 1 dedicated worker by node


### Usefull confs:
```scala
spark.conf.set("spark.sql.shuffle.partitions", "5") // default = 200
```
### Vector Type
`org.apache.spark.ml.linalg.Vector`
has the following spark sql type (note that values are in `ArrayType`):
```scala
private[this] val _sqlType = {  
// type: 0 = sparse, 1 = dense  
// We only use "values" for dense vectors, and "size", "indices", and "values" for sparse  
// vectors. The "values" field is nullable because we might want to add binary vectors later,  
// which uses "size" and "indices", but not "values".  
StructType(Seq(  
StructField("type", ByteType, nullable = false),  
StructField("size", IntegerType, nullable = true),  
StructField("indices", ArrayType(IntegerType, containsNull = false), nullable = true),  
StructField("values", ArrayType(DoubleType, containsNull = false), nullable = true)))  
}
```
### DataFrame

- Spark SQL first realease: Spark 1.0.0 (May 30, 2014) (see [Spark SQL's paper](https://dl.acm.org/citation.cfm?id=2742797) by Michael Armbrust)
 
#### Memory format during processing
https://spoddutur.github.io/spark-notes/deep_dive_into_storage_formats.html

- 1.0.0: There was no "DataFrame" but only `SchemaRDD`. It was a `RDD` of Java Objects on-heap (see spark [fundamental paper](http://people.csail.mit.edu/matei/papers/2012/nsdi_spark.pdf) by Matei Zaharia).
- 1.3.0: `DataFrame` is born and is still and RDD of on-heap objects. `SchemaRDD` became an alias for smooth deprecation purpose.
```scala
@deprecated("use DataFrame", "1.3.0")
  type SchemaRDD = DataFrame
```
- Since 1.4.0 (June 11, 2015) it is `RDD` of `InternalRow`s that are **Binary Row-Based Format** known as **Tungsten Row Format**. `InternalRow`s:
  - allows **in-place elements access** that avoid serialization/deserialization --> just a little little bit slower than `RDD`s for element access but very very faster when it comes to shuffles.
  - store their data **off-heap** --> divide by 4 memory footprint compared to RDDs of Java objects.
  - [UnsafeRow](https://jaceklaskowski.gitbooks.io/mastering-spark-sql/spark-sql-UnsafeRow.html) is the basic implementation of [InternalRow](https://jaceklaskowski.gitbooks.io/mastering-spark-sql/spark-sql-InternalRow.html) (see descriptions of Jacek Laskowski's *Mastering Spark SQL* links for each)
  
- 1.6.0: `Dataset` is created as a separated class. There is conversions between `Dataset`s and `DataFrame`s. 
- Since 2.0.0, `DataFrame` is merged with `Dataset` and remains just an alias `type DataFrame = Dataset[Row]`.

#### Memory contoguousity (TODO validate)
There is only a contiguousity of the `UnsafeRow`s' memory because an `RDD[UnsafeRow]` is a collection of `UnsafeRow`s' referencies that lives somewhere on-heap. This causes many CPU's caches defaults, each new record to process causing one new default.

#### Memory format when cached
When a dataset is cached using `def cache(): this.type = persist()` it is basically `persit`ed with default storageLevel which is `MEMORY_AND_DISK`:

```scala
/**
   * Persist this Dataset with the default storage level (`MEMORY_AND_DISK`).
   *
   * @group basic
   * @since 1.6.0
   */
  def persist(): this.type = {
    sparkSession.sharedState.cacheManager.cacheQuery(this)
    this
  }

  /**
   * Persist this Dataset with the default storage level (`MEMORY_AND_DISK`).
   *
   * @group basic
   * @since 1.6.0
   */
```

`sparkSession.sharedState.cacheManager.cacheQuery` stores plan and other information of the df to cache in an `IndexedSeq` of instances of:
```scala
/** Holds a cached logical plan and its data */
case class CachedData(plan: LogicalPlan, cachedRepresentation: InMemoryRelation)
```

This index don't holds any data directly.
It help the `SparkSession` to remember not to clear the resulting `RDD[InternalRow]` of plans registered as "to cache" after their next execution.

#### `DataFrame` vs other `Dataset[<not Row>]` steps of rows processing
Let's compare processing steps of the `GeneratedIteratorForCodegenStage1` class that you can view by calling `.queryExecution.debug.codegen()` on a `DataFrame`

The semantic is:
1. **load a csv** with three column: an Integer id, a String pseudo and a String name.
2. create a new feature containing a substring of the pseudo
3. apply a filter on the new feature

##### DataFrame
```scala
val df = spark.read
      .format("csv")
      .option("header", "false")
      .option("delimiter", ",")
      .schema(StructType(Seq(
        StructField("id", IntegerType, true),
        StructField("pseudo", StringType, true),
        StructField("name", StringType, true))))
      .load("/bla/bla/bla.csv")
      .toDF("id", "pseudo", "name")
      .selectExpr("*", "substr(pseudo, 2) AS sub")
      .filter("sub LIKE 'a%' ")
```

Steps:
1. Get input `Iterator` during init: `public void init(int index, scala.collection.Iterator[] inputs)`. An `UnsafeRowWriter` is also instanciated with 4 fields:
```scala
filter_mutableStateArray_0[1] = new org.apache.spark.sql.catalyst.expressions.codegen.UnsafeRowWriter(4, 96)
```
2. `protected void processNext()` method is used to launch processing. It iterates through the input iterator, casting each element into an`InternalRow`.
```scala
InternalRow scan_row_0 = (InternalRow) scan_mutableStateArray_0[0].next();
```
3. It instanciates Java objects from the needed fields. There is no deserialization thanks to `UnsafeRow` in-place accessors implementation. Here it gets pseudo:
```scala
boolean scan_isNull_1 = scan_row_0.isNullAt(1);
UTF8String scan_value_1 = scan_isNull_1 ? null : (scan_row_0.getUTF8String(1));
```
4. If the pseudo column is not null, it computes the new feature using substring
```scala
if (!(!scan_isNull_1)) continue;
UTF8String filter_value_3 = scan_value_1.substringSQL(2, 2147483647);
```
5. Apply the filter with a direct call to `.startsWith` and if the result is negative, it skips the current record. `references` is an `Object[]` that holds udfs, and constants parameterizing the processing. `references[2]` holds the string `"a"`.
```scala
// references[2] holds the string "a"
boolean filter_value_2 = filter_value_3.startsWith(((UTF8String) references[2]));
if (!filter_value_2) continue;
```
6. Even if the new feature has already been computed for filtering purpose, it is computed another time for the new column itself. The other needed fields for output (id and name) are scanned.
```scala
boolean scan_isNull_0 = scan_row_0.isNullAt(0);
int scan_value_0 = scan_isNull_0 ? -1 : (scan_row_0.getInt(0));

boolean scan_isNull_2 = scan_row_0.isNullAt(2);
UTF8String scan_value_2 = scan_isNull_2 ? null : (scan_row_0.getUTF8String(2));

UTF8String project_value_3 = null;
project_value_3 = scan_value_1.substringSQL(2, 2147483647);
```
7. The final fields are written with the pattern:
```scala
if (false) {
  filter_mutableStateArray_0[1].setNullAt(3);
} else {
  filter_mutableStateArray_0[1].write(3, project_value_3);
}
```
8. The `UnsafeRow` is builded and appended to a `LinkedList<InternalRow>` (attribute defined in the subclass`BufferedRowIterator`)
```scala
append((filter_mutableStateArray_0[1].getRow()));
```

##### Dataset
```scala
val ds = spark.read
      .format("csv")
      .option("header", "false")
      .option("delimiter", ",")
      .schema(StructType(Seq(
        StructField("id", IntegerType, true),
        StructField("pseudo", StringType, true),
        StructField("name", StringType, true))))
      .load("/home/enzo/Data/sofia-air-quality-dataset/2019-05_bme280sof.csv")
      .toDF("id", "pseudo", "name")
      .as[User]
      .map((user: User) => if(user.name != null)(user.id, user.name, user.pseudo, user.name.substring(1)) else (user.id, user.name, user.pseudo, ""))
      .filter((extendedUser: (Int, String, String, String)) => extendedUser._4.startsWith("a"))
```

Steps:
1. Get input `Iterator` during init: `public void init(int index, scala.collection.Iterator[] inputs)`. An `UnsafeRowWriter` is also instanciated with 4 fields:

```scala
filter_mutableStateArray_0[1] = new org.apache.spark.sql.catalyst.expressions.codegen.UnsafeRowWriter(4, 96)
```

2. `protected void processNext()` method is used to launch processing. It iterates through the input iterator, casting each element into an`InternalRow`.

```scala
InternalRow scan_row_0 = (InternalRow) scan_mutableStateArray_0[0].next();
```

3. It instanciates Java objects from the needed fields. There is no deserialization thanks to `UnsafeRow` in-place accessors implementation. Here it gets pseudo:

```scala
boolean scan_isNull_1 = scan_row_0.isNullAt(1);
UTF8String scan_value_1 = scan_isNull_1 ? null : (scan_row_0.getUTF8String(1));
```

4. If the pseudo column is not null, it computes the new feature using substring

```scala
if (!(!scan_isNull_1)) continue;
UTF8String filter_value_3 = scan_value_1.substringSQL(2, 2147483647);
```

5. Apply the filter with a direct call to `.startsWith` and if the result is negative, it skips the current record. `references` is an `Object[]` that holds udfs, and constants parameterizing the processing. `references[2]` holds the string `"a"`.

```scala
// references[2] holds the string "a"
boolean filter_value_2 = filter_value_3.startsWith(((UTF8String) references[2]));
if (!filter_value_2) continue;
```

6. Even if the new feature has already been computed for filtering purpose, it is computed another time for the new column itself. The other needed fields for output (id and name) are scanned.

```scala
boolean scan_isNull_0 = scan_row_0.isNullAt(0);
int scan_value_0 = scan_isNull_0 ? -1 : (scan_row_0.getInt(0));

boolean scan_isNull_2 = scan_row_0.isNullAt(2);
UTF8String scan_value_2 = scan_isNull_2 ? null : (scan_row_0.getUTF8String(2));

UTF8String project_value_3 = null;
project_value_3 = scan_value_1.substringSQL(2, 2147483647);
```

7. The final fields are written with the pattern:

```scala
if (false) {
  filter_mutableStateArray_0[1].setNullAt(3);
} else {
  filter_mutableStateArray_0[1].write(3, project_value_3);
}
```

8. The `UnsafeRow` is builded and appended to a `LinkedList<InternalRow>` (attribute defined in the subclass`BufferedRowIterator`)

```scala
append((filter_mutableStateArray_0[1].getRow()));
```

#### `df.rdd` vs `df.queryExecution.toRdd()`
[Jacek Laskowski's post on SO](https://stackoverflow.com/questions/44708629/is-dataset-rdd-an-action-or-transformation)

##### `.rdd`
It deserialize `InternalRow`s and put back data on-heap 
It's a transformation that returns `RDD[T]`.
If it's called on a `DataFrame = Dataset[Row]`, it returns `RDD[Row]`.
```scala
class Dataset[T] private[sql](  
    @transient val sparkSession: SparkSession,  
    @DeveloperApi @Unstable @transient val queryExecution: QueryExecution,  
    encoder: Encoder[T]) extends Serializable{
        [...]
	lazy val rdd: RDD[T] = {  
	  val objectType = exprEnc.deserializer.dataType  
	  rddQueryExecution.toRdd.mapPartitions { rows =>  
	    rows.map(_.get(0, objectType).asInstanceOf[T])  
	  }  
	}
```
use:
```scala
df.rdd
.map((row: Row) => Row.fromSeq(Seq(row.getAs[Long]("i")+10, row.getAs[Long]("i")-10)))  
.collect()
```

##### `.queryExecution.toRdd()`
It is used by `.rdd`.
If you stuck to this step, you keep your rows `InternalRow`s off-heap.
```scala
class QueryExecution(  
    val sparkSession: SparkSession,  
    val logical: LogicalPlan,  
    val tracker: QueryPlanningTracker = new QueryPlanningTracker) {  
    [...]
        lazy val toRdd: RDD[InternalRow] = executedPlan.execute()
```
use:
```scala
df.queryExecution.toRdd()
.map((row: InternalRow) => InternalRow.fromSeq(Seq(row.getLong(0)+10, row.getLong(0)-10)))  
```
- *Seems to be faster than dataframe for simple map ???*



#### OOP design
It's an unclear design
For me Dataset is a **builder** for a *SparkPlan* (decoupling of parameterization needed for building from created SparkPlan)  using an **immutable** (because a new sparkplan is created to go with a new dataframe at each modification: never "`return this;`") **Fluent Interface** :chaining methods possibility WITHOUT returning `this.type`,returning DataFrame for "select" for example & therefor violating OCP from SOLID principles$^{(1)}$, + meaning in the use of this chaining, not only keystrokes saving, also create pseudo sentences)
```scala
val df2 = df1.select(...).withColumn(...).where(...)
```
The ready to be built *SparkPlan* will never be accessed by user and it will be build when it's execution is ran !

$^{(1)}$: This violation can be avoided for single Builders by using generics/type parameterized builder class, each methods of the class returning parameterized type instead of simply base class with `return this`.

### SQL window function
``` SQL
SELECT __func__ OVER (PARTITION BY partitionCol ORDER BY orderCol __frame_type__ BETWEEN start AND end)`
```

**\_\_func\_\_**: Raking/Analytic/Aggregation function

**\_\_frame_type\_\_**:  **ROW** (*start* and *end* are then index and offsets: `ROWS BETWEEN UNBOUNDED PRECEDING AND 10 FOLLOWING`,  the frame contains every records from the begining of the partition to the ten next records after current one) or **RANGE** (*start* and *end* are then values in *orderCol* unit : `RANGE BETWEEN 13000 PRECEDING AND CURRENT ROW FOLLOWING`, given that *ORDER BY* has been performed on column **price** and that *current_p* is the price of the current record, the frame contains all the records that have a value of **price** *p* that is between *current_p -13000* and *current_p*)

### Closures and spark:
Impossible to make it work because referencies copied are living in driver and unreachable by executors in cluster mode (can work on localmode). Use [accumulators](https://spark.apache.org/docs/latest/rdd-programming-guide.html#accumulators) for this use cases to be rubust to deployment.

## ElasticSearch
### Parallels with distributed relationnal databases
1h # Elasticsearch Tutorial & Getting Started (course preview) https://www.youtube.com/watch?v=ksTTlXNLick

|E| RDB |
|--|--|
| Cluster | DataBase Engine |
|Index|Database|
|Type|Table|
|Document|Record|
|Properties|Columns|
|Node|Node|
|Index Shards|DB Partitions|
|Shards' Replicas|Partitions' Replicas|

## GCP
### BigQuery vs BigTable
*BigQuery* excels for OLAP (OnLine Analytical Processing): scalable and efficient analytic querying on unchanging data (or just appending data).
*BigTable* excels for OLTP (OnLine Transaction Processing): scalable and efficient read and write
