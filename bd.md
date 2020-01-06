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

### BigQuery vs BigTable
- *BigQuery* excels for OLAP (OnLine Analytical Processing): scalable and efficient analytic querying on unchanging data (or just appending data).
- *BigTable* excels for OLTP (OnLine Transaction Processing): scalable and efficient read and write

## Delta Lake
### DeltaLog & ACID guarantees

#### 0) the DeltaLog
**Deltalog** = *Delta Lake*'s transaction log. 

The deltalog is a collection of ordered json files. It acts as a single source of truth giving to users access to the last version of a `DeltaTable`'s state.

#### 1) Atomicity
- *Delta Lake* breaks down every operation performed by an user into *commits*, themselves composed of *actions*.
- A commit is recorded in the deltalog only once each of its actions has successfully completed (else it is reverted and restarted or an error is thrown), ensuring its **atomicity**.

#### 2) Consistency
The **consistency** of a `DeltaTable` is guaranteed by their strong schema checking.

#### 3) Isolation
Concurrency of commits is managed to ensure their **isolation**. An optimistic concurrency control is applied:

- When a commit execution starts, the thread snapshots the current deltalog.
- When the commit actions have completed, the thread checks if the *Deltalog* has been updated by another one in the meantime:
  - If not it records the commit in the deltalog
  - Else it updates its `DeltaTable` view and attempts again to register the commit, after a step of reprocessing if needed.
  
#### 4) Durability
Commits containing actions that mutate the `DeltaTable`'s data need to finish their writes/deletions on underlying *Parquet* files (stored on the filesystem) to be considered as successfully completed, making them **durable**.
___
Further readings: 

[*Diving Into Delta Lake: Unpacking The Transaction Log*](https://databricks.com/blog/2019/08/21/diving-into-delta-lake-unpacking-the-transaction-log.html)

[ACID properties](https://en.wikipedia.org/wiki/ACID)

# Useful ports
|service|port|
|--|--|
|YARN jobs UI|8088|
|Spark UI|4040|
|Spark History Server (Applications level)|18080|
|Jupyter notebook|8888|

<!--stackedit_data:
eyJoaXN0b3J5IjpbODA0NzU4MzM3LDE0ODA0NTg3NzYsMTkwND
Y3NjgwOCwxMTkxNjcyODg1LC0xMjU3MDA1MzAsLTE5NzA3MzU0
MDQsLTEyNzQ5NjYzNCwtMTQzNzYxMjM5NywtMTA2NjY4MDA4OC
wyMDkzMjM1NTg4LDE4MTEzMTExOTYsLTUzOTgzNjUzOCwtMTg1
OTU0MjE2MywxNzQzMTY5MDA0LC03Mzk4NTI5MzUsMjAxOTMwND
g5NywtMTg3MTQ1Njg3OSwxNzUyNDg2MDQ3LC02MTQ5NDYyNSwx
MDIyNTgxNjA0XX0=
-->