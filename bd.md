<!--NOTE HEAD START-->
<link rel="icon" type="image/png" href="./imgs/favicon_db.png" />
<script src="https://cdnjs.cloudflare.com/ajax/libs/mermaid/8.0.0/mermaid.min.js"></script>
<script type="text/x-mathjax-config">MathJax.Hub.Config({tex2jax: {skipTags: ['script', 'noscript','style', 'textarea', 'pre'],inlineMath: [['$','$']]}});</script>
<script src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML" type="text/javascript"></script>
<script>document.body.style.background = "#f2f2f2";</script>
<!--NOTE HEAD END-->

# Spark
## Architecture vocabulary

1 **cluster node** $\rightarrow$ is composed of several worker nodes

1 **worker node** $\rightarrow$ 
- launches 1 or several executors
- is dedicated exclusively to its parent application

1 **executor** $\rightarrow$
- is a java application (=1 JVM)
- may start several threads

1 **executor's thread** $\rightarrow$
- runs several tasks

1 **applications** $\rightarrow$
- runs several spark jobs
- launch 1 worker by node by cluster node

1 **job** $\rightarrow$ is a DAG of stages

1 **stage** $\rightarrow$ is a DAG of steps ending with a shuffle/exchange

1 **step** $\rightarrow$ runs as a collection of parallel tasks

1 **task** $\rightarrow$ acts on 1 RDD's partition

*DAG = Directed Acyclic Graph. They are used by spark to represent Jobs' stages or Stages' steps*


## Usefull confs:
```scala
SparkSession.builder.config("spark.default.parallelism", "12") // default = 200
```

## Vector Type
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
## DataFrame

- Spark SQL first realease: Spark 1.0.0 (May 30, 2014) (see [Spark SQL's paper](https://dl.acm.org/citation.cfm?id=2742797) by Michael Armbrust)
 
### Memory format 
#### during processing
https://spoddutur.github.io/spark-notes/deep_dive_into_storage_formats.html

- 1.0.0: There was no "DataFrame" but only `SchemaRDD`. It was a `RDD` of Java Objects on-heap (see spark [Spark's RDDs paper](http://people.csail.mit.edu/matei/papers/2012/nsdi_spark.pdf) by Matei Zaharia, 2011).
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

#### is a DataFrame sorted ?
use `df.queryExecution.sparkPlan.outputOrdering` that returns a sequence of `org.apache.spark.sql.catalyst.expressions.SortOrder`s.
```scala
val dfIsSorted = !df.sort().queryExecution.sparkPlan.outputOrdering.isEmpty
```

#### contoguousity (TODO validate)
There is only a contiguousity of the `UnsafeRow`s' memory because an `RDD[UnsafeRow]` is a collection of `UnsafeRow`s' referencies that lives somewhere on-heap. This causes many CPU's caches defaults, each new record to process causing one new default.

#### Caching

- When a dataset is cached using `def cache(): this.type = persist()` it is basically `persit`ed with default storageLevel which is `MEMORY_AND_DISK`:

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
  def cache(): this.type = persist()
```

- `sparkSession.sharedState.cacheManager.cacheQuery` stores plan and other information of the df to cache in an `IndexedSeq` of instances of: 
```scala
/** Holds a cached logical plan and its data */
case class CachedData(plan: LogicalPlan, cachedRepresentation: InMemoryRelation)
```

This index don't holds any data directly.
It help the `SparkSession` to remember not to clear the resulting `RDD[InternalRow]` of plans registered as "to cache" after their next execution.

- Caching is shared among `LogicalPlan`s with same result, in `CacheManager`:
```scala
def lookupCachedData(plan: LogicalPlan): Option[CachedData] = readLock {  
  cachedData.asScala.find(cd => plan.sameResult(cd.plan))  
}
```

- `unpersist()` is not lazy, it directly remove the dataframe cached data.

- Caching operations are not purely functional:
```scala
df2 = df.cache()
```
is equivalent to
```scala
df.cache()
df2 = df
```
#### `DataFrame` vs other `Dataset[<not Row>]` steps of rows processing
Short: DataFrame less secure but a bit faster.

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

[SEE SQL DAG](https://raw.githubusercontent.com/EnzoBnl/enzobnl.github.io/master/figs/DFSQLdiag.png)

Steps (~80 lines of generated code):
- Get input `Iterator` during init: `public void init(int index, scala.collection.Iterator[] inputs)`. An `UnsafeRowWriter` is also instanciated with 4 fields:

```scala
filter_mutableStateArray_0[1] = new org.apache.spark.sql.catalyst.expressions.codegen.UnsafeRowWriter(4, 96)
```

- `protected void processNext()` method is used to launch processing. It iterates through the input iterator, casting each element into an`InternalRow`.

```scala
InternalRow scan_row_0 = (InternalRow) scan_mutableStateArray_0[0].next();
```

- It instanciates Java objects from the needed fields. There is no deserialization thanks to `UnsafeRow` in-place accessors implementation. Here it gets pseudo:

```scala
boolean scan_isNull_1 = scan_row_0.isNullAt(1);
UTF8String scan_value_1 = scan_isNull_1 ? null : (scan_row_0.getUTF8String(1));
```

- If the pseudo column is not null, it computes the new feature using substring

```scala
if (!(!scan_isNull_1)) continue;
UTF8String filter_value_3 = scan_value_1.substringSQL(2, 2147483647);
```

- Apply the filter with a direct call to `.startsWith` and if the result is negative, it skips the current record. `references` is an `Object[]` that holds udfs, and constants parameterizing the processing. `references[2]` holds the string `"a"`.

```scala
// references[2] holds the string "a"
boolean filter_value_2 = filter_value_3.startsWith(((UTF8String) references[2]));
if (!filter_value_2) continue;
```

- Even if the new feature has already been computed for filtering purpose, it is computed another time for the new column itself. The other needed fields for output (id and name) are scanned.

```scala
boolean scan_isNull_0 = scan_row_0.isNullAt(0);
int scan_value_0 = scan_isNull_0 ? -1 : (scan_row_0.getInt(0));

boolean scan_isNull_2 = scan_row_0.isNullAt(2);
UTF8String scan_value_2 = scan_isNull_2 ? null : (scan_row_0.getUTF8String(2));

UTF8String project_value_3 = null;
project_value_3 = scan_value_1.substringSQL(2, 2147483647);
```

- The final fields are written with the pattern:

```scala
if (false) {
  filter_mutableStateArray_0[1].setNullAt(3);
} else {
  filter_mutableStateArray_0[1].write(3, project_value_3);
}
```

- The `UnsafeRow` is builded and appended to a `LinkedList<InternalRow>` (attribute defined in the subclass`BufferedRowIterator`)

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

[SEE SQL DAG](https://raw.githubusercontent.com/EnzoBnl/enzobnl.github.io/master/figs/DSSQLdiag.png)

Steps (~300 lines of generated code):
- Get input `Iterator` during init: `public void init(int index, scala.collection.Iterator[] inputs)`. An `UnsafeRowWriter` is also instanciated with 4 fields:

```scala
project_mutableStateArray_0[1] = new org.apache.spark.sql.catalyst.expressions.codegen.UnsafeRowWriter(4, 96)
```

- `protected void processNext()` method is used to launch processing. It iterates through the input iterator, casting each element into an`InternalRow`.

```scala
InternalRow scan_row_0 = (InternalRow) scan_mutableStateArray_0[0].next();
```

- It instanciates Java objects from the needed fields. There is no deserialization thanks to `UnsafeRow` in-place accessors implementation. Here it gets pseudo:

```scala
boolean scan_isNull_1 = scan_row_0.isNullAt(1);
UTF8String scan_value_1 = scan_isNull_1 ? null : (scan_row_0.getUTF8String(1));
```

- It passes all the scanned fields to `deserializetoobject_doConsume_0(<fieldX type> fieldXName, boolean fieldXIsNull, ...)`. This method will build a `User(id: Int, pseudo: String, name: String)` instance. In our case it requires to convert the scanned values from `UTF8String`s to `String`, the cast between `int` and `Int` being automatic.

```scala
[...]
deserializetoobject_funcResult_1 = deserializetoobject_expr_2_0.toString();
[...]
final com.enzobnl.sparkscalaexpe.playground.User deserializetoobject_value_3 = false ? 
  null : 
  new com.enzobnl.sparkscalaexpe.playground.User(
    deserializetoobject_argValue_0, 
    deserializetoobject_mutableStateArray_0[0], 
    deserializetoobject_mutableStateArray_0[1]
  );
```

- It now calls `mapelements_doConsume_0(deserializetoobject_value_3, false);`, `deserializetoobject_value_3` being the `User` instance lately build.



- `mapelements_doConsume_0(com.enzobnl.sparkscalaexpe.playground.User mapelements_expr_0_0, boolean mapelements_exprIsNull_0_0)` applies the anonymous function provided to the `map`: apply the substring and return a Tuple4

```scala
mapelements_funcResult_0 = ((scala.Function1) references[2]).apply(mapelements_mutableStateArray_0[0]);
if (mapelements_funcResult_0 != null) {
  mapelements_value_1 = (scala.Tuple4) mapelements_funcResult_0;
} else {
  mapelements_isNull_1 = true;
}
```

- It also apply the filter and continues to next record if condition not met

```scala
filter_funcResult_0 = ((scala.Function1) references[4]).apply(filter_mutableStateArray_0[0]);

                        if (filter_funcResult_0 != null) {
                            filter_value_0 = (Boolean) filter_funcResult_0;
                        } else {
                            filter_isNull_0 = true;
                        }
[...]
if (filter_isNull_0 || !filter_value_0) continue;
```

- It then call `serializefromobject_doConsume_0(mapelements_value_1, mapelements_isNull_1)` that will do the necessary conversions (back from `UTF8String` to `Dtring`...) to build an `UnsafeRow` from the `Tuple4[Int, String, String, String]` using `UnsafeRowWriter`.

```scala
boolean serializefromobject_isNull_12 = serializefromobject_resultIsNull_2;
UTF8String serializefromobject_value_12 = null;
if (!serializefromobject_resultIsNull_2) {
  serializefromobject_value_12 = org.apache.spark.unsafe.types.UTF8String.fromString(deserializetoobject_mutableStateArray_0[4]);
}
[...]
if (serializefromobject_isNull_12) {
  project_mutableStateArray_0[7].setNullAt(3);
} else {
  project_mutableStateArray_0[7].write(3, serializefromobject_value_12);
}
append((project_mutableStateArray_0[7].getRow()));
```

[Full code available here](https://gist.github.com/EnzoBnl/37e07e9cf7bce440734c7d33304257f0)

#### Conversion to RDD: `df.rdd` vs `df.queryExecution.toRdd()`
[Jacek Laskowski's post on SO](https://stackoverflow.com/questions/44708629/is-dataset-rdd-an-action-or-transformation)

1. `.rdd`
It deserializes `InternalRow`s and put back data on-heap. It's still lazy: the need of deserialization is recorded but not triggered.
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
usage:
```scala
df.rdd
.map((row: Row) => Row.fromSeq(Seq(row.getAs[Long]("i")+10, row.getAs[Long]("i")-10)))  
.collect()
```

2. `.queryExecution.toRdd()`

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
usage:
```scala
df.queryExecution.toRdd
.map((row: InternalRow) => InternalRow.fromSeq(Seq(row.getLong(0)+10, row.getLong(0)-10)))  
```

#### OOP design
`Dataset` can be viewed as a **functional builder** for a `LogicalPlan`, implemented as a **fluent API** friendly to SQL users.
```scala
val df2 = df1.join(...).select(...).where(...).orderBy(...).show()
```

`Dataset` class makes use of **Delegation Pattern** in many places, to delegate work to its underlying `RDD`, e.g. `Dataset.reduce` :
```scala
def reduce(func: (T, T) => T): T = withNewRDDExecutionId {  
  rdd.reduce(func)  
}
```

### SQL window function
``` SQL
SELECT 
  some_col,
  __func__ OVER (
    PARTITION BY partitionCol 
    ORDER BY orderCol __frame_type__ 
    BETWEEN start AND end
  )
  FROM ...
```

**\_\_func\_\_**: Raking/Analytic/Aggregation function

**\_\_frame_type\_\_**:  
- **ROW** (*start* and *end* are then index and offsets: `ROWS BETWEEN UNBOUNDED PRECEDING AND 10 FOLLOWING`,  the frame contains every records from the begining of the partition to the ten next records after current one) 
- **RANGE** (*start* and *end* are then values in *orderCol* unit : `RANGE BETWEEN 13000 PRECEDING AND CURRENT ROW FOLLOWING`, given that *ORDER BY* has been performed on column **price** and that *current_p* is the price of the current record, the frame contains all the records that have a value of **price** *p* that is between *current_p -13000* and *current_p*)

## Closures
The following will compile and run fine but it will only do what is expected if you run Spark in local mode:
```scala
val rdd: RDD[String] = ???  
val startingWithA = mutable.Set[String]()  
rdd.foreach((a: String) => if (a.toLowerCase().startsWith("a")) startingWithA += a)
println(s"rdd contains ${startingWithA.size} records starting with 'a'")
```
because `startingWithA` will not be shared among JVMs in cluster mode. 

Use [accumulators](https://spark.apache.org/docs/latest/rdd-programming-guide.html#accumulators) instead.

## Include a dependency from spark-package in maven's pom.xml

- add the repository
```xml
<project>
    [...]
    <repositories>  
        <repository>
            <id>spark-packages</id>
            <url>http://dl.bintray.com/spark-packages/maven/</url>  
        </repository>
    </repositories>  
    [...]
</project>
```

# Delta Lake
# ElasticSearch
## Parallels with distributed relationnal databases
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

# GCP
- install gcloud SDK
`curl https://dl.google.com/dl/cloudsdk/release/install_google_cloud_sdk.bash | bash`

## BigQuery vs BigTable
*BigQuery* excels for OLAP (OnLine Analytical Processing): scalable and efficient analytic querying on unchanging data (or just appending data).
*BigTable* excels for OLTP (OnLine Transaction Processing): scalable and efficient read and write


<!--stackedit_data:
eyJoaXN0b3J5IjpbLTc5NzM5Mjk5MCwtNjE0OTQ2MjUsMTAyMj
U4MTYwNCwxODM0NTAwNzEzLDE0MTY3NDAyMTEsMTExOTI4Njcw
NiwtNzU1MTEzMzUxLC0xNzYyNTMwNDU1XX0=
-->