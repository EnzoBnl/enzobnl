<!--NOTE HEAD START-->
<link rel="icon" type="image/png" href="./imgs/favicon_db.png" />
<script src="https://cdnjs.cloudflare.com/ajax/libs/mermaid/8.0.0/mermaid.min.js"></script>
<script type="text/x-mathjax-config">MathJax.Hub.Config({tex2jax: {skipTags: ['script', 'noscript','style', 'textarea', 'pre'],inlineMath: [['$','$']]}});</script>
<script src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML" type="text/javascript"></script>
<script>document.body.style.background = "#f2f2f2";</script>
<!--NOTE HEAD END-->
## Spark
### Architectures
1 cluster Node = multiple worker nodes

1 worker node = multiple executors

1 executor = 
- java application (=1 JVM)
- multiple threads

1 job = multiple stages

1 applications =
- multiple spark jobs
- 1 worker by node

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
### Dataset 
- DataFrame is now (2.4.0) alias for Dataset[Row].
- Spark SQL first realease: Spark 1.0.0 (May 30, 2014)
 
#### Format
https://spoddutur.github.io/spark-notes/deep_dive_into_storage_formats.html

- 1.0.0 to 1.3.X: it was RDD (see spark [fundamental paper](http://people.csail.mit.edu/matei/papers/2012/nsdi_spark.pdf) by Matei Zaharia)  of Java Objects
-  Since 1.4.0 (June 11, 2015) it is Datasets (see [Spark SQL's paper](https://dl.acm.org/citation.cfm?id=2742797) by Michael Armbrust) : RDD of `InternalRow`s that are **Binary Row-Based Format** known as **Tungsten Row Format** stored **OFF-HEAP**. **In place elements access** avoid deserialization ! So *faster and lighter*. [UnsafeRow](https://jaceklaskowski.gitbooks.io/mastering-spark-sql/spark-sql-UnsafeRow.html) is the basic implementation of [InternalRow](https://jaceklaskowski.gitbooks.io/mastering-spark-sql/spark-sql-InternalRow.html) (see jaceklaskowski links for each)
in spark.sql.execution.QueryExecution:

```scala
/** Internal version of the RDD. Avoids copies and has no schema */  
lazy val toRdd: RDD[InternalRow] = executedPlan.execute()
```
#### `df.rdd`/`df.queryExecution.toRdd()`
[Jacek post](https://stackoverflow.com/questions/44708629/is-dataset-rdd-an-action-or-transformation)
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

#### in memory contoguousity

#### OOP design
It's an unclear design
For me Dataset is a **builder** for a *SparkPlan* (decoupling of parameterization needed for building from created SparkPlan)  using an **immutable** (because a new sparkplan is created to go with a new dataframe at each modification: never "`return this;`") **Fluent Interface** :chaining methods possibility WITHOUT returning `this.type`,returning DataFrame for "select" for example & therefor violating OCP from SOLID principles$^{(1)}$, + meaning in the use of this chaining, not only keystrokes saving, also create pseudo sentences)
```scala
val df2 = df1.select(...).withColumn(...).where(...)
```
The ready to be built *SparkPlan* will never be accessed by user and it will be build when it's execution is ran !

$^{(1)}$: This violation can be avoided for single Builders by using generics/type parameterized builder class, each methods of the class returning parameterized type instead of simply base class with `return this`.

#### Caching
Caching can be partial ! 
Here a subpart of the plan of df will be cached
```python
df = spark.createDataFrame([(1,2), (1,2)]).toDF("a", "b").cache().selectExpr("f(a)").show()
```

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
