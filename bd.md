<!--NOTE HEAD START-->
<link rel="icon" type="image/png" href="./imgs/favicon_db.png" />
<script src="https://cdnjs.cloudflare.com/ajax/libs/mermaid/8.0.0/mermaid.min.js"></script>
<script type="text/x-mathjax-config">MathJax.Hub.Config({tex2jax: {skipTags: ['script', 'noscript','style', 'textarea', 'pre'],inlineMath: [['$','$']]}});</script>
<script src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML" type="text/javascript"></script>
<script>document.body.style.background = "#f2f2f2";</script>
<!--NOTE HEAD END-->

# Theorems and definitions
## PACeLC theorem
Abadi's *"Consistency Tradeoffs in Modern Distributed Database System Design"*, 2012.

**Theorem**: In case of network **P**artition, the system remains **A**vailable *or* **C**onsistent, **e**lse it ensures low **L**atency *or* **C**onsistency

- P = (network) Partitioning = A sub-part of the nodes become unreachable.
- A = Availability = Requests try to return the more recent available state of the result.
- C = Consistency = A request result is either up-to-date or an error.
- E = Else
- L = Latency = A request get its result in its more fastly accessible state.

||PC|PA|
|--|--|--|
|**EC**|[MongoDB](https://en.wikipedia.org/wiki/MongoDB), BigTable/HBase and fully ACID$^{[1]}$ systems: [VoltDB](https://en.wikipedia.org/wiki/VoltDB "VoltDB")/H-Store, Megastore, [MySQL Cluster](https://en.wikipedia.org/wiki/MySQL_Cluster "MySQL Cluster")|Most in-memory datagrids (Apache Ignite, Hazelcast IMDG)|
|**EL**| PNUTS |[DynamoDB](https://en.wikipedia.org/wiki/Amazon_DynamoDB "Amazon DynamoDB"), [Cassandra](https://en.wikipedia.org/wiki/Apache_Cassandra "Apache Cassandra"), [Riak](https://en.wikipedia.org/wiki/Riak "Riak"), [Cosmos DB](https://en.wikipedia.org/wiki/Cosmos_DB "Cosmos DB")|

PACeLC is an extension of the CAP theorem.

## ACID DataBase properties (in short)
A transaction is a sequence of database operations that satisfies the following rules:
- **[Atomicity]** A transaction is managed as a ***atomic*** unit that can be fully roll-backed if it fails.
- **[Consistency]** A transaction is ***consistent*** in that it cannot be committed if it violates database system rules.
- **[Isolation]** Independent transactions are ***isolated*** in that running them concurrently or sequentially leads to the same database system state.
- **[Durability]** A transaction is ***durable*** in that once it it fully committed, it will remain committed even in case of system failures (power off, network partitioning...).

# Frameworks' notes
## Spark
[See dedicated notes](https://enzobnl.github.io/spark.html)
## Hadoop's MapReduce
Execution steps of a MapReduce job containing 1 Mapper and 1 Reducer (steps **in bold** rely on *hook classes* exposed to the user for extension):

- Map phase on mapper nodes:
  1. **RecordReader**: 
     - reads the input file's blocks from HDFS
     - divides them in *key->value* units called records
     - each block's records will be passed to the same mapper
  2. **Mapper**: There is by default as many mappers as input file's blocks.
     - maps through records
     - produces 0 to *n* output record(s) for each input record
  3. **Partitioner**: 
     - organizes mapper output's records into partitions, based on their *keys*.
     - each partition is intended to be fetched by a single and different reducer node.
  4. *Sort*: Sort records on *key* within each partitions
  5. (optional) ***"Combiner"*** **Reducer**: an intermediate reducer called on each mapper node. 
     - within each sorted partition, for each different key, combiner produces an output record having the same type as Mapper ones
     - combiner reduces the size of partitions and save network bandwidth
- Reduce phase on reduce nodes:
  1. *Fetch*: Each reducer node gather its partitions (written by map phase to HDFS), mainly through network connections.
  2. *Sort-Merge*: Merge pre-sorted partition chunks into a final sorted reducer input partition.
  3. **Reducer**: Produce reduce output record for each *key*. It leverages that its input is sorted.

## ElasticSearch
[1h Elasticsearch Tutorial & Getting Started](https://www.youtube.com/watch?v=ksTTlXNLick)
### Parallels with distributed relationnal databases


|Elastic Search| Relational DataBase |
|--|--|
| Cluster | DataBase engine & servers |
|Index|Database|
|Type|Table|
|Document|Record|
|Property|Column|
|Node|Node|
|Index Shard|DB Partition|
|Shard's Replica|Partition's Replica|

## Google Cloud Platform's tools
- install gcloud SDK
`curl https://dl.google.com/dl/cloudsdk/release/install_google_cloud_sdk.bash | bash`

### BigQuery vs BigTable
- *BigQuery* excels for OLAP (OnLine Analytical Processing): scalable and efficient analytic querying on unchanging data (or just appending data).
- *BigTable* excels for OLTP (OnLine Transaction Processing): scalable and efficient read and write

## Delta Lake
### DeltaLog & ACID guarantees
Deltalog = *Delta Lake* transaction log.

[DataBricks' blog, Diving Into Delta Lake: Unpacking The Transaction Log](https://databricks.com/blog/2019/08/21/diving-into-delta-lake-unpacking-the-transaction-log.html)
#### Atomicity
- *Delta Lake* breaks down every operation performed by auser into *commits*, themselves composed of *actions*.
- **Atomicity of commits** is guaranteed by the fact that a commit is recorded in the deltalog **only once each action composing it have successfully completed**, else it is reverted.
#### Consistency
- The consistency of the `DeltaTable`s is guaranteed by their strong schema checking.
#### Isolation
Optimistic concurrency control ensures the isolation of the commit
- When a commit execution starts, it snapshot the current delta 
#### Durability
- Durability is guaranteed by the fact that commits involving `DeltaTable`s' data mutation need to finish to write or delete the underlying parquet files on the filesystem to be considered completed.

<!--stackedit_data:
eyJoaXN0b3J5IjpbLTEwODk1MDU5NjIsLTE5NzA3MzU0MDQsLT
EyNzQ5NjYzNCwtMTQzNzYxMjM5NywtMTA2NjY4MDA4OCwyMDkz
MjM1NTg4LDE4MTEzMTExOTYsLTUzOTgzNjUzOCwtMTg1OTU0Mj
E2MywxNzQzMTY5MDA0LC03Mzk4NTI5MzUsMjAxOTMwNDg5Nywt
MTg3MTQ1Njg3OSwxNzUyNDg2MDQ3LC02MTQ5NDYyNSwxMDIyNT
gxNjA0LDE4MzQ1MDA3MTMsMTQxNjc0MDIxMSwxMTE5Mjg2NzA2
LC03NTUxMTMzNTFdfQ==
-->