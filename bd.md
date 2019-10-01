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

Steps (~80 lines of generated code):
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
Steps (~300 lines of generated code):
1. Get input `Iterator` during init: `public void init(int index, scala.collection.Iterator[] inputs)`. An `UnsafeRowWriter` is also instanciated with 4 fields:

```scala
project_mutableStateArray_0[1] = new org.apache.spark.sql.catalyst.expressions.codegen.UnsafeRowWriter(4, 96)
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

31. It passes all the scanned fields to `deserializetoobject_doConsume_0(<fieldX type> fieldXName, boolean fieldXIsNull, ...)`. This method will build a `User(id: Int, pseudo: String, name: String)` instance. In our case it requires to convert the scanned values from `UTF8String`s to `String`, the cast between `int` and `Int` being automatic.

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

32. It now calls `mapelements_doConsume_0(deserializetoobject_value_3, false);`, `deserializetoobject_value_3` being the `User` instance lately build.



4. `mapelements_doConsume_0(com.enzobnl.sparkscalaexpe.playground.User mapelements_expr_0_0, boolean mapelements_exprIsNull_0_0)` applies the anonymous function provided to the `map`: apply the substring and return a Tuple4

```scala
mapelements_funcResult_0 = ((scala.Function1) references[2]).apply(mapelements_mutableStateArray_0[0]);
if (mapelements_funcResult_0 != null) {
  mapelements_value_1 = (scala.Tuple4) mapelements_funcResult_0;
} else {
  mapelements_isNull_1 = true;
}
```

5. It also apply the filter and continues to next record if condition not met

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

6. It then call `serializefromobject_doConsume_0(mapelements_value_1, mapelements_isNull_1)` that will do the necessary conversions (back from `UTF8String` to `Dtring`...) to build an `UnsafeRow` from the `Tuple4[Int, String, String, String]` using `UnsafeRowWriter`.

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

Full code at the end of the page.

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



# Appendix
## DataFrame vs Dataset codegen snippet
```scala
import org.apache.spark.SparkException;
import org.apache.spark.sql.catalyst.InternalRow;
import org.apache.spark.unsafe.types.UTF8String;

//Found 1 WholeStageCodegen subtrees.
//        == Subtree 1 / 1 ==
//        *(1) SerializeFromObject [assertnotnull(input[0, scala.Tuple4, true])._1 AS _1#75, staticinvoke(class org.apache.spark.unsafe.types.UTF8String, StringType, fromString, assertnotnull(input[0, scala.Tuple4, true])._2, true, false) AS _2#76, staticinvoke(class org.apache.spark.unsafe.types.UTF8String, StringType, fromString, assertnotnull(input[0, scala.Tuple4, true])._3, true, false) AS _3#77, staticinvoke(class org.apache.spark.unsafe.types.UTF8String, StringType, fromString, assertnotnull(input[0, scala.Tuple4, true])._4, true, false) AS _4#78]
//        +- *(1) Filter <function1>.apply
//        +- *(1) MapElements <function1>, obj#74: scala.Tuple4
//        +- *(1) DeserializeToObject newInstance(class com.enzobnl.sparkscalaexpe.playground.User), obj#73: com.enzobnl.sparkscalaexpe.playground.User
//        +- *(1) Project [_c0#53 AS id#59, _c1#54 AS pseudo#60, _c2#55 AS name#61]
//        +- *(1) FileScan csv [_c0#53,_c1#54,_c2#55] Batched: false, Format: CSV, Location: InMemoryFileIndex[file:/home/enzo/Prog/spark/data/graphx/users.txt], PartitionFilters: [], PushedFilters: [], ReadSchema: struct<_c0:int,_c1:string,_c2:string>
//
//        Generated code:

/**
 * Dataset new col with substring and filter on it
 * val ds = spark.read
 *       .format("csv")
 *       .option("header", "false")
 *       .option("delimiter", ",")
 *       .schema(StructType(Seq(
 *         StructField("id", IntegerType, true),
 *         StructField("pseudo", StringType, true),
 *         StructField("name", StringType, true))))
 *       .load("/home/enzo/Data/sofia-air-quality-dataset/2019-05_bme280sof.csv")
 *       .toDF("id", "pseudo", "name")
 *       .as[User]
 *       .map((user: User) => if(user.name != null)(user.id, user.name, user.pseudo, user.name.substring(1)) else (user.id, user.name, user.pseudo, ""))
 *       .filter((extendedUser: (Int, String, String, String)) => extendedUser._4.startsWith("a"))
 */
public class DFvsDS {
    public Object generate(Object[] references) {
        return new GeneratedIteratorForCodegenStage1(references);
    }

    // codegenStageId=1
    final class GeneratedIteratorForCodegenStage1 extends org.apache.spark.sql.execution.BufferedRowIterator {
        private Object[] references;
        private scala.collection.Iterator[] inputs;
        private int deserializetoobject_argValue_0;
        private boolean mapelements_resultIsNull_0;
        private boolean filter_resultIsNull_0;
        private boolean serializefromobject_resultIsNull_0;
        private boolean serializefromobject_resultIsNull_1;
        private boolean serializefromobject_resultIsNull_2;
        private java.lang.String[] deserializetoobject_mutableStateArray_0 = new java.lang.String[5];
        private org.apache.spark.sql.catalyst.expressions.codegen.UnsafeRowWriter[] project_mutableStateArray_0 = new org.apache.spark.sql.catalyst.expressions.codegen.UnsafeRowWriter[8];
        private scala.collection.Iterator[] scan_mutableStateArray_0 = new scala.collection.Iterator[1];
        private scala.Tuple4[] filter_mutableStateArray_0 = new scala.Tuple4[1];
        private com.enzobnl.sparkscalaexpe.playground.User[] mapelements_mutableStateArray_0 = new com.enzobnl.sparkscalaexpe.playground.User[1];

        public GeneratedIteratorForCodegenStage1(Object[] references) {
            this.references = references;
        }

        public void init(int index, scala.collection.Iterator[] inputs) {
            partitionIndex = index;
            this.inputs = inputs;
            scan_mutableStateArray_0[0] = inputs[0];
            project_mutableStateArray_0[0] = new org.apache.spark.sql.catalyst.expressions.codegen.UnsafeRowWriter(3, 64);
            project_mutableStateArray_0[1] = new org.apache.spark.sql.catalyst.expressions.codegen.UnsafeRowWriter(3, 64);

            project_mutableStateArray_0[2] = new org.apache.spark.sql.catalyst.expressions.codegen.UnsafeRowWriter(1, 32);
            project_mutableStateArray_0[3] = new org.apache.spark.sql.catalyst.expressions.codegen.UnsafeRowWriter(1, 32);
            project_mutableStateArray_0[4] = new org.apache.spark.sql.catalyst.expressions.codegen.UnsafeRowWriter(1, 32);
            project_mutableStateArray_0[5] = new org.apache.spark.sql.catalyst.expressions.codegen.UnsafeRowWriter(1, 32);
            project_mutableStateArray_0[6] = new org.apache.spark.sql.catalyst.expressions.codegen.UnsafeRowWriter(1, 32);
            // only this one is used to write final Tuple4 rows with append((project_mutableStateArray_0[7].getRow()));
            // append is inherited from BufferedRowIterator.
            project_mutableStateArray_0[7] = new org.apache.spark.sql.catalyst.expressions.codegen.UnsafeRowWriter(4, 96);

        }
        private void mapelements_doConsume_0(com.enzobnl.sparkscalaexpe.playground.User mapelements_expr_0_0, boolean mapelements_exprIsNull_0_0) throws java.io.IOException {
            do {
                boolean mapelements_isNull_1 = true;
                scala.Tuple4 mapelements_value_1 = null;
                if (!false) {
                    mapelements_resultIsNull_0 = false;

                    if (!mapelements_resultIsNull_0) {
                        mapelements_resultIsNull_0 = mapelements_exprIsNull_0_0;
                        mapelements_mutableStateArray_0[0] = mapelements_expr_0_0;
                    }

                    mapelements_isNull_1 = mapelements_resultIsNull_0;
                    if (!mapelements_isNull_1) {
                        Object mapelements_funcResult_0 = null;
                        // apply map anonymous function stored in (scala.Function1) references[2]
                        // on custom case class User stored in mapelements_mutableStateArray_0[0]
                        mapelements_funcResult_0 = ((scala.Function1) references[2]).apply(mapelements_mutableStateArray_0[0]);

                        if (mapelements_funcResult_0 != null) {
                            mapelements_value_1 = (scala.Tuple4) mapelements_funcResult_0;
                        } else {
                            mapelements_isNull_1 = true;
                        }

                    }
                }

                boolean filter_isNull_0 = true;
                boolean filter_value_0 = false;
                if (!false) {
                    filter_resultIsNull_0 = false;

                    if (!filter_resultIsNull_0) {
                        filter_resultIsNull_0 = mapelements_isNull_1;
                        filter_mutableStateArray_0[0] = mapelements_value_1;
                    }

                    filter_isNull_0 = filter_resultIsNull_0;
                    if (!filter_isNull_0) {
                        Object filter_funcResult_0 = null;
                        filter_funcResult_0 = ((scala.Function1) references[4]).apply(filter_mutableStateArray_0[0]);

                        if (filter_funcResult_0 != null) {
                            filter_value_0 = (Boolean) filter_funcResult_0;
                        } else {
                            filter_isNull_0 = true;
                        }

                    }
                }
                // if filter failed, skip line by not calling
                // serializefromobject_doConsume_0(scala.Tuple4 mapelements_value_1, boolean mapelements_isNull_1)
                if (filter_isNull_0 || !filter_value_0) continue;

                ((org.apache.spark.sql.execution.metric.SQLMetric) references[3]).add(1);

                serializefromobject_doConsume_0(mapelements_value_1, mapelements_isNull_1);

            } while (false);

        }

        private void deserializetoobject_doConsume_0(int deserializetoobject_expr_0_0, boolean deserializetoobject_exprIsNull_0_0, UTF8String deserializetoobject_expr_1_0, boolean deserializetoobject_exprIsNull_1_0, UTF8String deserializetoobject_expr_2_0, boolean deserializetoobject_exprIsNull_2_0) throws java.io.IOException {
            if (deserializetoobject_exprIsNull_0_0) {
                throw new NullPointerException(((java.lang.String) references[1]));
            }
            deserializetoobject_argValue_0 = deserializetoobject_expr_0_0;

            boolean deserializetoobject_isNull_6 = true;
            java.lang.String deserializetoobject_value_6 = null;
            if (!deserializetoobject_exprIsNull_1_0) {
                deserializetoobject_isNull_6 = false;
                if (!deserializetoobject_isNull_6) {
                    Object deserializetoobject_funcResult_0 = null;
                    deserializetoobject_funcResult_0 = deserializetoobject_expr_1_0.toString();
                    deserializetoobject_value_6 = (java.lang.String) deserializetoobject_funcResult_0;

                }
            }
            deserializetoobject_mutableStateArray_0[0] = deserializetoobject_value_6;

            boolean deserializetoobject_isNull_8 = true;
            java.lang.String deserializetoobject_value_8 = null;
            if (!deserializetoobject_exprIsNull_2_0) {
                deserializetoobject_isNull_8 = false;
                if (!deserializetoobject_isNull_8) {
                    Object deserializetoobject_funcResult_1 = null;
                    deserializetoobject_funcResult_1 = deserializetoobject_expr_2_0.toString();
                    deserializetoobject_value_8 = (java.lang.String) deserializetoobject_funcResult_1;

                }
            }
            deserializetoobject_mutableStateArray_0[1] = deserializetoobject_value_8;

            final com.enzobnl.sparkscalaexpe.playground.User deserializetoobject_value_3 = false ?
                    null : new com.enzobnl.sparkscalaexpe.playground.User(deserializetoobject_argValue_0, deserializetoobject_mutableStateArray_0[0], deserializetoobject_mutableStateArray_0[1]);

            mapelements_doConsume_0(deserializetoobject_value_3, false);

        }

        private void serializefromobject_doConsume_0(scala.Tuple4 serializefromobject_expr_0_0, boolean serializefromobject_exprIsNull_0_0) throws java.io.IOException {
            if (serializefromobject_exprIsNull_0_0) {
                throw new NullPointerException(((java.lang.String) references[5]));
            }
            boolean serializefromobject_isNull_1 = true;
            int serializefromobject_value_1 = -1;
            if (!false) {
                serializefromobject_isNull_1 = false;
                if (!serializefromobject_isNull_1) {
                    Object serializefromobject_funcResult_0 = null;
                    serializefromobject_funcResult_0 = serializefromobject_expr_0_0._1();
                    serializefromobject_value_1 = (Integer) serializefromobject_funcResult_0;

                }
            }
            serializefromobject_resultIsNull_0 = false;

            if (!serializefromobject_resultIsNull_0) {
                if (serializefromobject_exprIsNull_0_0) {
                    throw new NullPointerException(((java.lang.String) references[6]));
                }
                boolean serializefromobject_isNull_5 = true;
                java.lang.String serializefromobject_value_5 = null;
                if (!false) {
                    serializefromobject_isNull_5 = false;
                    if (!serializefromobject_isNull_5) {
                        Object serializefromobject_funcResult_1 = null;
                        serializefromobject_funcResult_1 = serializefromobject_expr_0_0._2();

                        if (serializefromobject_funcResult_1 != null) {
                            serializefromobject_value_5 = (java.lang.String) serializefromobject_funcResult_1;
                        } else {
                            serializefromobject_isNull_5 = true;
                        }

                    }
                }
                serializefromobject_resultIsNull_0 = serializefromobject_isNull_5;
                deserializetoobject_mutableStateArray_0[2] = serializefromobject_value_5;
            }

            boolean serializefromobject_isNull_4 = serializefromobject_resultIsNull_0;
            UTF8String serializefromobject_value_4 = null;
            if (!serializefromobject_resultIsNull_0) {
                serializefromobject_value_4 = org.apache.spark.unsafe.types.UTF8String.fromString(deserializetoobject_mutableStateArray_0[2]);
            }
            serializefromobject_resultIsNull_1 = false;

            if (!serializefromobject_resultIsNull_1) {
                if (serializefromobject_exprIsNull_0_0) {
                    throw new NullPointerException(((java.lang.String) references[7]));
                }
                boolean serializefromobject_isNull_9 = true;
                java.lang.String serializefromobject_value_9 = null;
                if (!false) {
                    serializefromobject_isNull_9 = false;
                    if (!serializefromobject_isNull_9) {
                        Object serializefromobject_funcResult_2 = null;
                        serializefromobject_funcResult_2 = serializefromobject_expr_0_0._3();

                        if (serializefromobject_funcResult_2 != null) {
                            serializefromobject_value_9 = (java.lang.String) serializefromobject_funcResult_2;
                        } else {
                            serializefromobject_isNull_9 = true;
                        }

                    }
                }
                serializefromobject_resultIsNull_1 = serializefromobject_isNull_9;
                deserializetoobject_mutableStateArray_0[3] = serializefromobject_value_9;
            }

            boolean serializefromobject_isNull_8 = serializefromobject_resultIsNull_1;
            UTF8String serializefromobject_value_8 = null;
            if (!serializefromobject_resultIsNull_1) {
                serializefromobject_value_8 = org.apache.spark.unsafe.types.UTF8String.fromString(deserializetoobject_mutableStateArray_0[3]);
            }
            serializefromobject_resultIsNull_2 = false;

            if (!serializefromobject_resultIsNull_2) {
                if (serializefromobject_exprIsNull_0_0) {
                    throw new NullPointerException(((java.lang.String) references[8]));
                }
                boolean serializefromobject_isNull_13 = true;
                java.lang.String serializefromobject_value_13 = null;
                if (!false) {
                    serializefromobject_isNull_13 = false;
                    if (!serializefromobject_isNull_13) {
                        Object serializefromobject_funcResult_3 = null;
                        serializefromobject_funcResult_3 = serializefromobject_expr_0_0._4();

                        if (serializefromobject_funcResult_3 != null) {
                            serializefromobject_value_13 = (java.lang.String) serializefromobject_funcResult_3;
                        } else {
                            serializefromobject_isNull_13 = true;
                        }
                    }
                }
                serializefromobject_resultIsNull_2 = serializefromobject_isNull_13;
                deserializetoobject_mutableStateArray_0[4] = serializefromobject_value_13;
            }

            boolean serializefromobject_isNull_12 = serializefromobject_resultIsNull_2;
            UTF8String serializefromobject_value_12 = null;
            if (!serializefromobject_resultIsNull_2) {
                serializefromobject_value_12 = org.apache.spark.unsafe.types.UTF8String.fromString(deserializetoobject_mutableStateArray_0[4]);
            }
            project_mutableStateArray_0[7].reset();

            project_mutableStateArray_0[7].zeroOutNullBytes();

            project_mutableStateArray_0[7].write(0, serializefromobject_value_1);

            if (serializefromobject_isNull_4) {
                project_mutableStateArray_0[7].setNullAt(1);
            } else {
                project_mutableStateArray_0[7].write(1, serializefromobject_value_4);
            }

            if (serializefromobject_isNull_8) {
                project_mutableStateArray_0[7].setNullAt(2);
            } else {
                project_mutableStateArray_0[7].write(2, serializefromobject_value_8);
            }

            if (serializefromobject_isNull_12) {
                project_mutableStateArray_0[7].setNullAt(3);
            } else {
                project_mutableStateArray_0[7].write(3, serializefromobject_value_12);
            }
            append((project_mutableStateArray_0[7].getRow()));

        }

        protected void processNext() throws java.io.IOException {
            while (scan_mutableStateArray_0[0].hasNext()) {
                InternalRow scan_row_0 = (InternalRow) scan_mutableStateArray_0[0].next();
                ((org.apache.spark.sql.execution.metric.SQLMetric) references[0]).add(1);
                boolean scan_isNull_0 = scan_row_0.isNullAt(0);
                int scan_value_0 = scan_isNull_0 ?
                        -1 : (scan_row_0.getInt(0));
                boolean scan_isNull_1 = scan_row_0.isNullAt(1);
                UTF8String scan_value_1 = scan_isNull_1 ?
                        null : (scan_row_0.getUTF8String(1));
                boolean scan_isNull_2 = scan_row_0.isNullAt(2);
                UTF8String scan_value_2 = scan_isNull_2 ?
                        null : (scan_row_0.getUTF8String(2));

                deserializetoobject_doConsume_0(scan_value_0, scan_isNull_0, scan_value_1, scan_isNull_1, scan_value_2, scan_isNull_2);
                if (shouldStop()) return;
            }
        }

    }
}

/**
 * DataFrame new col with substring and filter on it
 * val df = spark.read
 *       .format("csv")
 *       .option("header", "false")
 *       .option("delimiter", ",")
 *       .schema(StructType(Seq(
 *         StructField("id", IntegerType, true),
 *         StructField("pseudo", StringType, true),
 *         StructField("name", StringType, true))))
 *       .load("/home/enzo/Data/sofia-air-quality-dataset/2019-05_bme280sof.csv")
 *       .toDF("id", "pseudo", "name")
 *       .selectExpr("*", "substr(pseudo, 2) AS sub")
 *       .filter("sub LIKE 'a%' ")
 */
class DFvsDS2 {

    //Found 1 WholeStageCodegen subtrees.
//        == Subtree 1 / 1 ==
//        *(1) Project [_c0#10 AS id#16, _c1#11 AS pseudo#17, _c2#12 AS name#18, substring(_c1#11, 2, 2147483647) AS sub#22]
//        +- *(1) Filter (isnotnull(_c1#11) && StartsWith(substring(_c1#11, 2, 2147483647), a))
//        +- *(1) FileScan csv [_c0#10,_c1#11,_c2#12] Batched: false, Format: CSV, Location: InMemoryFileIndex[file:/home/enzo/Prog/spark/data/graphx/users.txt], PartitionFilters: [], PushedFilters: [IsNotNull(_c1)], ReadSchema: struct<_c0:int,_c1:string,_c2:string>
//
//        Generated code:
    public Object generate(Object[] references) {
        return new GeneratedIteratorForCodegenStage1(references);
    }

    // codegenStageId=1
    final class GeneratedIteratorForCodegenStage1 extends org.apache.spark.sql.execution.BufferedRowIterator {
        private Object[] references;
        private scala.collection.Iterator[] inputs;
        private org.apache.spark.sql.catalyst.expressions.codegen.UnsafeRowWriter[] filter_mutableStateArray_0 = new org.apache.spark.sql.catalyst.expressions.codegen.UnsafeRowWriter[2];
        private scala.collection.Iterator[] scan_mutableStateArray_0 = new scala.collection.Iterator[1];

        public GeneratedIteratorForCodegenStage1(Object[] references) {
            this.references = references;
        }

        public void init(int index, scala.collection.Iterator[] inputs) {
            // inputs are Iterators of InternalRows: there is no data contiguousity inter rows -> not CPU caches optimized
            partitionIndex = index;
            this.inputs = inputs;
            scan_mutableStateArray_0[0] = inputs[0];
            filter_mutableStateArray_0[0] = new org.apache.spark.sql.catalyst.expressions.codegen.UnsafeRowWriter(3, 64);
            filter_mutableStateArray_0[1] = new org.apache.spark.sql.catalyst.expressions.codegen.UnsafeRowWriter(4, 96);
        }

        protected void processNext() throws java.io.IOException {
            // scan_mutableStateArray_0[0] filled with inputs iterator in init
            while (scan_mutableStateArray_0[0].hasNext()) {
                // get next input InternalRow
                InternalRow scan_row_0 = (InternalRow) scan_mutableStateArray_0[0].next();
                ((org.apache.spark.sql.execution.metric.SQLMetric) references[0]).add(1);
                do {
                    boolean scan_isNull_1 = scan_row_0.isNullAt(1);
                    // getUTF8String: direct access to the string without deserialization
                    UTF8String scan_value_1 = scan_isNull_1 ?
                            null : (scan_row_0.getUTF8String(1));

                    if (!(!scan_isNull_1)) continue;

                    UTF8String filter_value_3 = null;
                    filter_value_3 = scan_value_1.substringSQL(2, 2147483647);

                    boolean filter_value_2 = false;
                    filter_value_2 = (filter_value_3).startsWith(((UTF8String) references[2]));
                    // if not startwith: skip and continue to next record
                    if (!filter_value_2) continue;

                    ((org.apache.spark.sql.execution.metric.SQLMetric) references[1]).add(1);

                    boolean scan_isNull_0 = scan_row_0.isNullAt(0);
                    int scan_value_0 = scan_isNull_0 ?
                            -1 : (scan_row_0.getInt(0));
                    boolean scan_isNull_2 = scan_row_0.isNullAt(2);
                    UTF8String scan_value_2 = scan_isNull_2 ?
                            null : (scan_row_0.getUTF8String(2));
                    UTF8String project_value_3 = null;
                    project_value_3 = scan_value_1.substringSQL(2, 2147483647);
                    filter_mutableStateArray_0[1].reset();

                    filter_mutableStateArray_0[1].zeroOutNullBytes();

                    if (scan_isNull_0) {
                        filter_mutableStateArray_0[1].setNullAt(0);
                    } else {
                        filter_mutableStateArray_0[1].write(0, scan_value_0);
                    }

                    if (false) {
                        filter_mutableStateArray_0[1].setNullAt(1);
                    } else {
                        filter_mutableStateArray_0[1].write(1, scan_value_1);
                    }

                    if (scan_isNull_2) {
                        filter_mutableStateArray_0[1].setNullAt(2);
                    } else {
                        filter_mutableStateArray_0[1].write(2, scan_value_2);
                    }

                    if (false) {
                        filter_mutableStateArray_0[1].setNullAt(3);
                    } else {
                        filter_mutableStateArray_0[1].write(3, project_value_3);
                    }
                    append((filter_mutableStateArray_0[1].getRow()));

                } while (false);
                if (shouldStop()) return;
            }
        }

    }
}
```

