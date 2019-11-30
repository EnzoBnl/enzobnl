
<!--NOTE HEAD START-->
<link rel="icon" type="image/png" href="./imgs/favicon_db.png" />
<script src="https://cdnjs.cloudflare.com/ajax/libs/mermaid/8.0.0/mermaid.min.js"></script>
<script type="text/x-mathjax-config">MathJax.Hub.Config({tex2jax: {skipTags: ['script', 'noscript','style', 'textarea', 'pre'],inlineMath: [['$','$']]}});</script>
<script src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML" type="text/javascript"></script>
<script>document.body.style.background = "#f2f2f2";</script>
<!--NOTE HEAD END-->

# Spark
## Architecture vocabulary

|Component|Desc|
|--|--|
| **cluster node** | is composed of several worker nodes |
| **worker node** | launches 1 or several executors & is dedicated exclusively to its parent application |
|**executor**| is a java application & may start several threads |
| **executor's thread** | runs several job's tasks sequentially|
| **applications** | runs several spark jobs & launches 1 worker by cluster node |
| **job** | is a collection of stages organized in DAG |
| **stage** | is a DAG of steps whose roots and leafs are shuffles or I/Os|
| **step** | runs as a collection of tasks|
| **task** | operate on 1 RDD's partition |

*DAG = Directed Acyclic Graph. They are used by spark to represent Jobs' stages or Stages' steps*

 
## Unified Memory Management (1.6+)
Useful sources:
-  pull request document [Unified Memory Management in Spark 1.6](https://www.linuxprobe.com/wp-content/uploads/2017/04/unified-memory-management-spark-10000.pdf) by Andrew Or and Josh Rosen
-  blog article [Apache Spark and off-heap memory](https://www.waitingforcode.com/apache-spark/apache-spark-off-heap-memory/read) by Bartosz Konieczny.

### Repartition of a machine's memory allocation to a Spark executor

<div class="mermaid">
graph TB
-4[executor memory]
-3[off-heap overheads:<br/>-VM overheads<br/>-interned strings<br/>-other native overheads]
-2[For other executors or unused]
-1[Worker node's memory]
0[on-heap execution & storage region]
1[on-heap executor space]
2[off-heap executor space]
3[on-heap execution region]
4[on-heap storage region]
33[off-heap execution region]
44[off-heap storage region]
5[Highly unmanaged:<br/>-on-heap internal metadata<br/>-user data structures<br/>-and imprecise size estimation<br/>-in the case of unusually largerecords]
-1 --> -2
-1--spark.memory.offHeap.size/total bytes-->2
-1--spark.executor.memory/total JVM format-->-4
-4 --1 - spark.executor.memoryOverhead -->1
-4 -- spark.executor.memoryOverhead -->-3
1 --spark.memory.fraction-->0
1 --1-spark.memory.fraction-->5
0 -- spark.memory.storageFraction--> 4
0 --1 - spark.memory.storageFraction--> 3
2 -- spark.memory.storageFraction--> 44
2 --1 - spark.memory.storageFraction--> 33
</div>

### On-heap executor space
The **On-heap executor space** is divided in 2 regions:
- **Execution region**: 
buffering intermediate data when performing shuffles, joins, sorts and aggregations

- **Storage region**: 
  - caching data blocks to optimize for future accesses 
  - torrent broadcasts
  - sending large task results

### Execution and storage regions behaviors in *unified memory management*

1. if execution needs to use some space:
   - if its region space is not filled: it uses it
   - else if there is available unused space in storage region: it borrows it and uses it
   - else: excess data is spilled to disk and it uses freed space
2. if storage needs to use some space:
   - if its region space is not filled: it uses it
   - else if some of its region space has been borrowed by execution: it takes it back by triggering a spill to disk of some execution data, and uses it.
   - else: excess cached blocks are evicted (Least Recently Used policy) and it uses freed space


## Memory format (during processing) evolution  (SQL)
https://spoddutur.github.io/spark-notes/deep_dive_into_storage_formats.html

- 1.0.0 (May 26, 2014): There was no "DataFrame" but only `SchemaRDD`. It was a `RDD` of fully deserialized Java Objects.
- 1.3.0 (Mar 6, 2015): `DataFrame` is born and is still and RDD of  objects. `SchemaRDD` became an alias for smooth deprecation purpose.

```scala
@deprecated("use DataFrame", "1.3.0")
  type SchemaRDD = DataFrame
```

- Since 1.4.0 (June 11, 2015) it is `RDD` of `InternalRow`s that are **Binary Row-Based Format** known as **Tungsten Row Format**. `InternalRow`s:
  - allows **in-place elements access** that avoid serialization/deserialization --> just a little little bit slower than `RDD`s for element access but very very faster when it comes to shuffles.
  - store their data **off-heap** --> divide by 4 memory footprint compared to RDDs of Java objects.
  - [UnsafeRow](https://jaceklaskowski.gitbooks.io/mastering-spark-sql/spark-sql-UnsafeRow.html) is the basic implementation of [InternalRow](https://jaceklaskowski.gitbooks.io/mastering-spark-sql/spark-sql-InternalRow.html) (see descriptions of Jacek Laskowski's *Mastering Spark SQL* links for each)
  
- 1.6.0 (Dec 22, 2015): `Dataset` is created as a separated class. There is conversions between `Dataset`s and `DataFrame`s. 
- Since 2.0.0 (Jul 19, 2016), `DataFrame` is merged with `Dataset` and remains just an alias `type DataFrame = Dataset[Row]`.


### contoguousity (TODO validate) (SQL)
There is only a contiguousity of the `UnsafeRow`s' memory because an `RDD[UnsafeRow]` is a collection of `UnsafeRow`s' referencies that lives somewhere on-heap. This causes many CPU's caches defaults, each new record to process causing one new default.

## Caching (SQL)

|  |default storage level|
|--|--|
| `RDD.persist` |`MEMORY_ONLY`|
| `Dataset.persist` |`MEMORY_AND_DISK`|


- When a dataset is cached using `def cache(): this.type = persist()` it is basically persisted with default `storageLevel` which is `MEMORY_AND_DISK`:

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

- `InMemoryRelation` holds a memory optimized `RDD[CachedBatch]` whose memory format is columnar and optionally compressed.

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

## is a DataFrame sorted ?
One can use `df.queryExecution.sparkPlan.outputOrdering` that returns a sequence of `org.apache.spark.sql.catalyst.expressions.SortOrder`s to retrieve this information:

```scala
def isSorted(df: Dataset[_]): Boolean =
  !df.sort().queryExecution.sparkPlan.outputOrdering.isEmpty
```

## `DataFrame` vs `Dataset[<not Row>]` row processing steps
Short: DataFrame less secure but a bit faster.

Let's compare processing steps of the `GeneratedIteratorForCodegenStage1` class that you can view by calling `.queryExecution.debug.codegen()` on a `DataFrame`

The semantic is:
1. **load a csv** with three column: an Integer id, a String pseudo and a String name.
2. create a new feature containing a substring of the pseudo
3. apply a filter on the new feature

### DataFrame's WholeStageCodegen execution...
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

### ... vs Dataset's WholeStageCodegen execution
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

## Conversion to RDD: `df.rdd` vs `df.queryExecution.toRdd()`
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

## Dataset's OOP design
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

## SQL window function syntax 
(not Spark specific)
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


## Closures
The following will compile and run fine but it will only do what is expected if you run Spark in local mode:
```scala
val rdd: RDD[String] = ???  
val startingWithA = mutable.Set[String]()  
rdd.foreach((a: String) => if (a.toLowerCase().startsWith("a")) startingWithA += a)
println(s"rdd contains ${startingWithA.size} records starting with 'a'")
```

because `startingWithA` will not be shared among JVMs in cluster mode.
Actually in non local modes (both client and master), it will print `rdd contains 0 records starting with 'a'` because the `mutable.Set[String]()` instance called for its size information lives inside the driver process JVM heap and is not populated by executors threads that live in other JVMs (the executors ones).

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
Partitioning & graphs in Spark




# Partitions in Spark

## Partitioning (SQL & Core)
### Partitioner
SQL:
Main partitioning
- `HashPartitioning`: on `keys` uses *Murmur Hash 3* (fast non-cryptographic hashing, easy to reverse) (while `PairRDD`s ' `partitionBy(hashPartitioner)` uses `.hashCode()`)

- `RangePartitioning`: Estimate key ditribution with sampling -> find split values -> each partition is splitted according -> result in new partitioning quite even and with **no overlapping**

- `RoundRobinPartitioning`: distribute evenly (based on a circular distribution of a quantum/batch of row to each partition). Used when `repartition(n)`.

Always partition on Long or Int, hash/encrypt string key if necessary.

### Materialize partitions
// FIXME
RDD: 
```scala
val rdd: RDD[T] = ...
rdd.foreachPartition(p: Iterator[T] => ())
```

Dataset: 
avoid (even for dataframes where T=Row):
```scala
val ds: Dataset[T] = ...
ds.foreachPartition(p: Iterator[T] => p.size)
```
which just calls:
`def foreachPartition(f: Iterator[T] => Unit): Unit = withNewRDDExecutionId(rdd.foreachPartition(f)) `
equivalent to:
```scala
val ds: Dataset[T] = ...
val rddRow: RDD[T] = ds.rdd
rddRow.foreachPartition(p: Iterator[T] => p.size)
```

but prefer:
```scala
val ds: Dataset[T] = ...
ds.count()
```

which skips two steps: DeserializeToObjects & MapPartitions
```scala
spark.range(100000).queryExecution.toRdd.foreachPartition(p => ())
```
is up to 30% faster than
```scala
spark.range(100000).foreachPartition(p => ())}  
```

## Repartitioning  (SQL & Core)
### coalesce
Try to minimize the data that need to be shuffled by merging partitions on the same executors in priority -> fast but may lead inequal partition 
### repartition
If column not given: round robin
else: hashpartitioning

if numpartition not given: reads  `"spark.sql.shuffle.partitions"`

### repartitionByRange
repartition by range, forced to pass columns.

### N°partitions heuristic
Really optimized runtime with n°partitions (= number max of parallel tasks) = 4 or 5 times n°available threads
```scala
val spark = SparkSession  
  .builder  
  .config("spark.default.parallelism", (<n°CPUs> * 4).toString)  // RDDs
  .config("spark.sql.shuffle.partitions", <n°CPUs> * 4)  // Datasets
  [...]
  .getOrCreate
```

### Pushing the repartitioning to HDFS source
```scala
sc.textFile("hdfs://.../file.txt", x)
```
can lead to a faster execution than
```scala
sc.textFile("hdfs://.../file.txt").repartition(x)
```
because the former will delegate the repartitioning to Hadoop's `TextInputFormat`.

## Data structures

SQL & RDD:
join are materialized as `ZippedPartitionsRDD2` whose memory format depends on the provided `var f: (Iterator[A], Iterator[B]) => Iterator[V]`


narrow transformations are materialized as `MapPartitionsRDD`, whose memory format depends on the provided `f: (TaskContext, Int, Iterator[T]) => Iterator[U]  // (TaskContext, partition index, iterator)`


both have a class attribute `preservesPartitioning` tagging: " Whether the input function preserves the partitioner, which should be `false` unless `prev` [previous RDD] is a pair RDD and the input function doesn't modify the keys."

Unions results in a `UnionRDD`, big picture: Each virtual partition is a list of parent partitions. 
A repartition by key called on a UnionRDD where all the parents are already hashpartitioned will only trigger a coalesce partitioning.

```scala

edges.repartition(5).union(edges.repartition(3)).mapPartitions(p => Iterator(p.size)).show()
+-------+
|  value|
+-------+
| 877438|
| 874330|
| 888988|
| 883017|
| 873434|
|1424375|
|1500710|
|1472122|
+-------+
```
With RDDs, better to use `sc.union` instead of chaining unions. In SQL it's automatically optimized.


SQL:
INSERT Tungsten UnsafeRow off-heap format

`WholestageCodegen.doExecute` (overriding of `SparkPlan.doExecute`) `mapPartitions` of parent RDD associating to them a `Iterator[InternalRow]` based on a generated class extending `BufferedRowIterator` (named `GeneratedIteratorForCodegenStageX`) which fill a `LinkedList[InternalRow]` (`append` method inputing a written `UnsafeRowWriter`'s `getRow` output)

`RDD[UnsafeRow]` partitions are sometimes materialized as contiguous byte arrays, CPUs cache hit-ration effective on iteration:
- `Dataset`s' `collect` (which is used by broadcast thread on driver among others) uses `collectFromPlan` which uses `SparkPlan.executeCollect`which uses `SparkPlan.getByteArrayRdd: RDD[(Long, Array[Byte])]` : " Packing the UnsafeRows into byte array for faster serialization. The byte arrays are in the following format: [size] [bytes of UnsafeRow] [size] [bytes of UnsafeRow] ...". RDDs does not. On this space optimized RDD it calls RDDs' concat:
```scala
def collect(): Array[T] = withScope {  
  val results = sc.runJob(this, (iter: Iterator[T]) => iter.toArray)
  Array.concat(results: _*)  
}
```

*Note*: the `toArray` is free because after `getByteArrayRdd`, the partition is an iterator of only one element which is a tuple `(number_of_rows, optimized_byte_array)`.
- Hash joins are relying on `BytesToBytesMap`, "An append-only hash map where keys and values are contiguous regions of bytes."

## Join
### Join algorithms families

- **Nested loop**: For each row in table A, loop on table B's rows to find matches
$O($\vert edges \vert$ * $\vert vertices \vert$)$
```python
for e in edges:
    for v in vertices:
        if join_condition(e, v):
            yield e + v
```

- **Hash join**: Create a join_key -> row hashmap for the smallest table. Loop on the biggest table and search for matches in the hashmap.
$O($\vert vertices \vert$ + $\vert edges \vert$)$, only equi joins, additional O($\vert vertices \vert$) space complexity

```python
vertices_map = {v.join_key: v for v in vertices}  # O($\vert vertices \vert$)

for e in edges:  # O($\vert edges \vert$)
    v = vertices_map.get(e.join_key, None)  # considered O(1)
    if v is not None:
        yield e + v
```

- **Sort-merge join**: Sort tables and iterate on both of them in the same move in one loop

$O($\vert vertices \vert$*log($\vert vertices \vert$) + $\vert edges \vert$*log($\vert edges \vert$))$ , adaptable to handle not only equi joins

```python
vertices.sort(lambda v: v.join_key)  # O($\vert vertices \vert$*log($\vert vertices \vert$)
edges.sort(lambda e: e.join_key)  # O($\vert edges \vert$*log($\vert edges \vert$)
i, j = 0, 0
while(i < len(vertices) and j < len(edges)):  # O($\vert vertices \vert$ + $\vert edges \vert$)
    if vertices[i].join_key < edges[i].join_key and i < len(vertices):
        i += 1
    elif vertices[i].join_key == edges[i].join_key:
        yield vertices[i] + edges[i]
        i += 1
        j += 1
    else:
        j += 1
```

### Joins in Spark  (SQL)
https://github.com/vaquarkhan/Apache-Kafka-poc-and-notes/wiki/Apache-Spark-Join-guidelines-and-Performance-tuning
https://databricks.com/session/optimizing-apache-spark-sql-joins

Following DAGs are extracts from Jacek Laskowski's [Internals of Apache Spark](https://jaceklaskowski.gitbooks.io/mastering-spark-sql/))

Main types:
- **Broadcasts**: base paper: *"cost-based optimization is only used to select join algorithms: for relations that are known to be small, Spark SQL uses a broadcast join, using a peer-to-peer broadcast facility available in Spark. (Table sizes are estimated if the table is cached in memory or comes from an external file, or if it is the result of a subquery with a LIMIT)"*
  (see `org.apache.spark.sql.execution.exchange.BroadcastExchangeExec`)
  - `RDD[InternalRows]` is `collect`ed **onto the driver node** as an `Array[(Long, Array[Byte]]`. Can cause this message though: "Not enough memory to build and broadcast the table to all worker nodes. As a workaround, you can either disable broadcast by setting `spark.sql.autoBroadcastJoinThreshold` to -1 or **increase the spark driver memory** by setting spark.driver.memory to a higher value".
  - It is transformed by the `BroadcastMode` in use
  - The partitions resulting data structures are then send to all the other partitions.

- **Shuffle**: Exchange tasks based on HashPartitioner (`:- Exchange hashpartitioning(id#37, 200)`)

Implems:
- **BroadcastHashJoin**: Only if one table is tiny, avoid shuffling the big table
<img src="https://jaceklaskowski.gitbooks.io/mastering-spark-sql/images/spark-sql-BroadcastHashJoinExec-webui-query-details.png" width="350"/>
uses `+- BroadcastExchange HashedRelationBroadcastMode` : build a `HashedRelation` ready to be broadcasted

`spark.sql.autoBroadcastJoinThreshold`
"Maximum size (in bytes) for a table that will be broadcast to all worker nodes when performing a join."
Default: `10L * 1024 * 1024` (10M)

The size of a dataframe is deduced from the `sizeInBytes` of the `LogicalPlan`'s `Statistics` 


- **ShuffledHashJoin**: Only if one is enough small to handle partitioned hashmap creation overhead
<img src="https://jaceklaskowski.gitbooks.io/mastering-spark-sql/images/spark-sql-ShuffledHashJoinExec-webui-query-details.png" width="350"/>
```scala
spark.conf.get("spark.sql.join.preferSortMergeJoin")
```
uses `+- Exchange hashpartitioning`
in each partition a `HashedRelation` is build (relies on `BytesToBytesMap`).
- **SortMergeJoin**: Fits for 2 even table big sizes, require keys to be orderable
<img src="https://jaceklaskowski.gitbooks.io/mastering-spark-sql/images/spark-sql-SortMergeJoinExec-webui-query-details.png" width="350"/>
uses `+- Exchange hashpartitioning`
- **BroadcastNestedLoopJoin**: Only if one table is tiny, and it's not possible to use hashing based **BroadcastHashJoin** (because key is array which is hashed by address, ...)
<img src="https://jaceklaskowski.gitbooks.io/mastering-spark-sql/images/spark-sql-BroadcastNestedLoopJoinExec-webui-details-for-query.png" width="350"/>
uses `+- BroadcastExchange IdentityBroadcastMode` pass the partitions as they are: optimized array of `UnsafeRow`s' bytes obtained with `getByteArrayRdd().collect()`

[spark.sql.join.preferSortMergeJoin](https://jaceklaskowski.gitbooks.io/mastering-spark-sql/spark-sql-properties.html#spark.sql.join.preferSortMergeJoin) is an internal configuration property and is enabled by default.

That means that [JoinSelection](https://jaceklaskowski.gitbooks.io/mastering-spark-sql/spark-sql-SparkStrategy-JoinSelection.html) execution planning strategy (and so Spark Planner) prefers sort merge join over [shuffled hash join](https://jaceklaskowski.gitbooks.io/mastering-spark-sql/spark-sql-SparkPlan-ShuffledHashJoinExec.html).

### Deal with skewed data  (SQL & Core)

When loaded, data is evenly partitioned by default. The danger comes when queries involves `HashPartioner`.

```scala
val df = spark.read  
  .format("csv")  
  .option("delimiter", ",")  
  .load("/home/enzo/Data/celineeng.edges")  
  .toDF("src", "dst", "origin")  
  
df.show(false)  
  
val partitionSizes = df.repartition(1000, col("dst")).mapPartitions(p => Iterator(p.size)).collect()  
println(partitionSizes.max, partitionSizes.min, partitionSizes.size)  // (36291,116,1000)
```

Cause: PowerLaw graph web, applied to particular website ? 


We always have a website with:
- n°pages >> n°partitions: hashes fill the full range(nPartition))
- n°links_by_page is evenly distributed (~200 by page): no risk that a few partitions containing the most outlinking pages causes a skewing.
- avg_n°links_by_page << n°pages: this is another condition for reduction of the partitions' size variance.

The following query won't cause skew problems, partition receiving quite evenly $\vert edges\vert$/n°partitions records each,

```sql
SELECT * FROM edges JOIN vertices ON edges.src = vertices.id
```

But, as every page points back to the home, the hash partitioning on `edges.dst` may lead to a big skewing: the partition containing the home key will at least contains $\vert vertices \vert$ records.

```sql
SELECT * FROM edges JOIN vertices ON edges.dst = vertices.id
```

The avg n°record_by_partition = $\vert edges \vert$/n°partitions 
We have actually a skew problem if 
$\vert vertices \vert$>>$\vert edges \vert$/n°partitions  
i.e. n°partitions >> $\vert edges \vert$/$\vert vertices \vert$ 
i.e. **n°partitions >> avg_n°links_by_page**
- This limits to the horizontal scaling potential

**Skew is problematic because a partition is processed only by one task at a time: The few tasks assigned to process the huge partitions will delay the end of the step, majority of the CPUs waiting for these few thread to end**
#### convert to Broadcast join
If the skewed big table is joined with a relatively, try to repartition evenly (RoundRobinPartitioning, `repartition(n)`) and use a broadcast join if spark does not managed to do it itself (tune threshold `"spark.sql.autoBroadcastJoinThreshold"` to the desired size in Bytes. 
#### 2-steps join
Trick: (`<hubs>` = `(home_id)` but can contains other hubs)

```sql
SELECT *
FROM edges JOIN vertices ON edges.dst = vertices.id 
WHERE edges.dst NOT IN (<hubs>);
```

```scala
val hashJoin = edges
  .join(
    vertices, 
    edges("dst") === vertices("id")
  )
  .where(not(edges("dst").isin(<hubs>)))
```

So second query will be converted to a broadcast join (=replicated join=Map-side join):

```sql
SELECT *
FROM edges JOIN vertices ON edges.dst = vertices.id 
WHERE edges.dst IN (<hubs>);
```

```scala
val broadcastJoin = edges
  .join(
    broadcast(vertices), 
    edges("dst") === vertices("id")
  )
  .where(edges("dst").isin(<hubs>))
```

The partial results of the two queries can then be merged to get the final results.

```scala
val df = hashJoin.union(broadcastJoin)  // by position
val df = hashJoin.unionByName(broadcastJoin)  // by name 
```

####  Duplicating little table
implem here for RDDs: https://github.com/tresata/spark-skewjoin

#### nulls skew
- dispatching
https://stackoverflow.com/a/43394695/6580080
Skew may happen when key column contains many null values.
Null values in key column are not participating in the join, but still, as they are needed in output for outer joins, they are hashpartitioned as the other key and we end up with a huge partition of null key:
- minor?: This slow down the join (sorting, many nulls to skip)
- major: this forces one task to fetch many record over network to build the partition.

[Fix](https://stackoverflow.com/a/43394695/6580080): 

```scala
df.withColumn("key", expr("ifnull(key, int(-round(rand()*100000)))"))
```

(simplified in the case there is no key < 0)

- filter: 

```scala
val data = ...
val notJoinable = data.filter('keyToJoin.isNull)
val joinable = data.filter('keyToJoin.isNotNull)
joinable.join(...) union notJoinable
```

### multi-join prepartitioning trick @ LinkedIn
TODO https://fr.slideshare.net/databricks/improving-spark-sql-at-linkedin
TODO: Matrix multiplicityhttps://engineering.linkedin.com/blog/2017/06/managing--exploding--big-data

### Range Joins
TODO

## GroupBy  (SQL & Core)
For `groupBy` on `edges.dst`, all's right because only the pre-aggregates (`partial_count(1)`, 1 row per distinct page id in each partition) are exchanged through cluster: This is equivalent to the `rdd.reduce`, `rdd.reduceByKey`, `rdd.aggregateByKey`, `combineByKey`  and not like `rdd.groupByKey` or `rdd.groupBy` which does not perform pre-aggregation and send everything over the network...

```sql
SELECT src, count(*) FROM edges GROUP BY src
```

```
== Physical Plan ==
*(2) HashAggregate(keys=[src#4L], functions=[count(1)])
+- Exchange hashpartitioning(src#4L, 48)
   +- *(1) HashAggregate(keys=[src#4L], functions=[partial_count(1)])
      +- *(1) InMemoryTableScan [src#4L]
```

(* means WholeStageCodegen used)

## Sort  (SQL & Core)

An `orderBy` starts with a step of exchange relying on `RangePartitioner` that "partitions sortable records by range into **roughly equal ranges**. The ranges are determined by sampling the content of the RDD passed in." (scaladoc)

This might not cause skew because the algorithm is **based on a sampling of partitions** that gives an insight on the distribution of the keys in the entire RDD, **BUT** if there is a specific value of the key that is very very frequent, its weight will not be divided and we will have one partition quite big containing only this skewed key.

See `RangePartitioner.sketch` (determines smooth weighted boundaries candidates) and `RangePartitioner.determineBounds` (determine boundaries from candidates based on wanted partitions, ...)

The result of sort step is sorted partitions that does not overlap.

```sql
SELECT src, count(*) as c FROM edges GROUP BY src ORDER BY c
```

```
== Physical Plan ==
*(3) Sort [c#18L DESC NULLS LAST], true, 0
+- Exchange rangepartitioning(c#18L DESC NULLS LAST, 48)
   +- *(2) HashAggregate(keys=[src#4L], functions=[count(1)])
      +- Exchange hashpartitioning(src#4L, 48)
         +- *(1) HashAggregate(keys=[src#4L], functions=[partial_count(1)])
            +- *(1) InMemoryTableScan [src#4L]
```

## Exchange/Shuffle (SQL & Core)

http://hydronitrogen.com/apache-spark-shuffles-explained-in-depth.html
https://0x0fff.com/spark-architecture-shuffle/

Shuffle in short: When exchange is needed, local partitions output are written to disk [**local file system**], and a shuffle manager is notified that the chunk is ready to be fetched by other executors.

Spill in short: Spill means that RDD's data is serialized and written to disk when it does not fit anymore in memory. Not linked directly to shuffle (? FIXME)

### Actors involved in shuffle (FIXME)
- `ShuffleManager` is trait that is instantiated on driver (register shuffles) and executors (ask to write or read data over connections with other executors). 
The default current implementation is [`SortShuffleManager`](https://github.com/apache/spark/blob/master/core/src/main/scala/org/apache/spark/shuffle/sort/SortShuffleManager.scala). Memory-based shuffle managers proposed with first graphx release has been abandoned because it lacks in reliability and do not gain much in speed because disk based shuffles leverage hyper fast data transfers with `java.nio.channels.FileChannel.transferTo(...)`. SSDs storage usage adds also a great speed up.

- The `ShuffleManager.getReader: ShuffleReader` allows to fetch `org.apache.spark.sql.execution.ShuffledRowRDD extends RDD[InternalRow]` which *"is a specialized version of `org.apache.spark.rdd.ShuffledRDD` that is optimized for shuffling rows instead of Java key-value pairs"*.
See `BypassMergeSortShuffleWriter` which relies on `DiskBlockObjectWriter` & `BlockManager`

### Exchanges planning (SQL)
Exchange are carefully optimized by Catalyst and are ordered to be as cheap as possible.

For example:

```scala
spark.conf.set("spark.sql.autoBroadcastJoinThreshold", 1)  
val sch = StructType(Seq(StructField("id", LongType), StructField("v", LongType)))

val df1: DataFrame = spark.read.schema(sch).load(...)  
val df2: DataFrame = spark.read.schema(sch).load(...)  
df1.join(df2.repartition(col("id")).groupBy("id").count(), df1("id") === df2("id")).explain()  


*(5) SortMergeJoin [id#0L], [id#4L], Inner
:- *(2) Sort [id#0L ASC NULLS FIRST], false, 0  
: +- Exchange hashpartitioning(id#0L, 200)  
:   +- *(1) Project [id#0L, v#1L]  
:     +- *(1) Filter isnotnull(id#0L)  
:       +- *(1) FileScan parquet [id#0L,v#1L] ...
+- *(4) Sort [id#4L ASC NULLS FIRST], false, 0  
  +- *(4) HashAggregate(keys=[id#4L], functions=[count(1)])  
    +- *(4) HashAggregate(keys=[id#4L], functions=[partial_count(1)])  
      +- Exchange hashpartitioning(id#4L, 200)  
        +- *(3) Project [id#4L]  
          +- *(3) Filter isnotnull(id#4L)  
            +- *(3) FileScan parquet [id#4L] ...
```

**/!\\** Unecessary exchange is triggered when renaming columns:

```scala
edges.repartition(10, col("src")).groupBy("src").count().explain()  
edges.repartition(10, col("src")).withColumnRenamed("src", "id").groupBy("id").count().explain()  

== Physical Plan ==
*(2) HashAggregate(keys=[src#98L], functions=[count(1)])
+- *(2) HashAggregate(keys=[src#98L], functions=[partial_count(1)])
   +- Exchange hashpartitioning(src#98L, 10)
      +- *(1) FileScan csv [src#98L] ...
      
== Physical Plan ==
*(3) HashAggregate(keys=[id#115L], functions=[count(1)])
+- Exchange hashpartitioning(id#115L, 48)
   +- *(2) HashAggregate(keys=[id#115L], functions=[partial_count(1)])
      +- *(2) Project [src#98L AS id#115L]
         +- Exchange hashpartitioning(src#98L, 10)
            +- *(1) FileScan csv [src#98L] ...
```

## Coming soon
###  Adaptative Execution (AE) in 3.0.0
[JIRA](https://issues.apache.org/jira/browse/SPARK-9850?jql=text%20~%20%22adaptative%20execution%22)
[JIRA issue's Google Doc](https://docs.google.com/document/d/1mpVjvQZRAkD-Ggy6-hcjXtBPiQoVbZGe3dLnAKgtJ4k/edit)
[Intel doc](https://software.intel.com/en-us/articles/spark-sql-adaptive-execution-at-100-tb)
apache/spark master branch is about to be released in the next months as Spark 3.0.0.
AE open since 1.6 has been merged 15/Jun/19. (Lead by # Carson Wang from Intel)

> a. dynamic parallelism  
I believe [Carson Wang](https://issues.apache.org/jira/secure/ViewProfile.jspa?name=carsonwang) is working on it. He will create a new ticket when the PR is ready.

> b. sort merge join to broadcast hash join or shuffle hash join  
It's included in the current framework

> c. skew handling.  
I don't think this one is started. The design doc is not out yet.

# Powerful external projects
- Uppon Spark ML:
  - [JohnSnowLabs](https://github.com/JohnSnowLabs)/**[spark-nlp](https://github.com/JohnSnowLabs/spark-nlp)**: Natural language processing complete toolbox from lemmatization and POS tagging to BERT embedding.
  - [salesforce](https://github.com/salesforce)/**[TransmogrifAI](https://github.com/salesforce/TransmogrifAI)**: Auto ML  (no Deep Learning) integrating following algorithms under the hood:  
    - classification -> `GBTCLassifier`, `LinearSVC`, `LogisticRegression`, `DecisionTrees`, `RandomForest` and `NaiveBayes`
    - regression -> `GeneralizedLinearRegression`, `LinearRegression`, `DecisionTrees`, `RandomForest` and `GBTreeRegressor`


# References
### Papers
- [2011 *Resilient Distributed Datasets: A Fault-Tolerant Abstraction for In-Memory Cluster Computing*](http://people.csail.mit.edu/matei/papers/2012/nsdi_spark.pdf), Zaharia Chowdhury Das Dave Ma McCauley Franklin Shenker Stoica
- 2013 *GraphX: A Resilient Distributed Graph System on Spark*, Reynold S Xin Joseph E Gonzalez Michael J Franklin Ion Stoica
- 2013 *Shark: SQL and Rich Analytics at Scale*, Reynold S. Xin, Josh Rosen, Matei Zaharia Michael J. Franklin Scott Shenker Ion Stoica
- 2014 *GraphX: Graph Processing in a  Distributed Dataflow Framework*, J E. Gonzalez R S. Xin A Dave, D Crankshaw  M J. Franklin I Stoica
- 2015 *Spark SQL: Relational Data Processing in Spark*, Michael Armbrust Reynold S. Xin

### Other sources
- [Map-side join in Hive](https://cwiki.apache.org/confluence/display/Hive/Skewed+Join+Optimization)
- [Skewed dataset join in Spark](https://stackoverflow.com/questions/40373577/skewed-dataset-join-in-spark)
- [Mastering Spark SQL](https://jaceklaskowski.gitbooks.io/mastering-spark-sql/)
- [Partitioning with Spark GraphFrames](https://stackoverflow.com/questions/41351802/partitioning-with-spark-graphframes/41353889#41353889)
- [Big Data analysis Coursera](https://www.coursera.org/lecture/big-data-analysis/joins-Nz9XW)
- [HashPartitioner explained](https://stackoverflow.com/questions/31424396/how-does-hashpartitioner-work)
- [Spark's configuration (latest)](https://spark.apache.org/docs/lastest/configuration.html)

<!--stackedit_data:
eyJoaXN0b3J5IjpbMTI0MDcwNzI4OCw2MDI3MDc4NzUsOTgxOD
AyNTI0LDEyMDMwNTQ4MDEsMTEwMTk5OTAxNSwxNDQxMjQ1OSwt
MTgzNDU1NzIwNSwxNjYwMDI1NjYsMTM4NTQ5NDg5MSwyNDE2OT
Q1NDAsODg2ODY0OTc2LC0zMjY0MDUyMiwxODAxMjgwODc4LDEx
OTM1ODk5NTAsMTkxMTE0NTU2NSw4MTE1OTg2NTAsOTQwOTk1MT
YzLDEwMzA3MDA4Myw1NzIyNDQ2MTAsMTA3NTk2MDU5N119
-->