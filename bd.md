<!--NOTE HEAD START-->
<link rel="icon" type="image/png" href="./imgs/favicon_db.png" />
<script src="https://cdnjs.cloudflare.com/ajax/libs/mermaid/8.0.0/mermaid.min.js"></script>
<script type="text/x-mathjax-config">MathJax.Hub.Config({tex2jax: {skipTags: ['script', 'noscript','style', 'textarea', 'pre'],inlineMath: [['$','$']]}});</script>
<script src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML" type="text/javascript"></script>
<script>document.body.style.background = "#f2f2f2";</script>
<!--NOTE HEAD END-->

# Theory
## PACeLC theorem
Abadi's *"Consistency Tradeoffs in Modern Distributed Database System Design"*, 2012.

**Theorem**: In case of network **P**artition, the system remains **A**vailable *or* **C**onsistent, **e**lse it ensures low **L**atency *or* **C**onsistency

- P = (network) Partitioning = A sub-part of the nodes become unreachable.
- A = Availability = Requests are newer answered with an error.
- C = Consistency = Requests get the latest version of what they ask for or an error.
- E = Else
- L = Latency = Requests get fast their result in its currently available state.

||PC|PA|
|--|--|--|
|**EC**|[MongoDB](https://en.wikipedia.org/wiki/MongoDB), BigTable/HBase and fully ACID$^{[1]}$ systems: [VoltDB](https://en.wikipedia.org/wiki/VoltDB "VoltDB")/H-Store, Megastore, [MySQL Cluster](https://en.wikipedia.org/wiki/MySQL_Cluster "MySQL Cluster")|Most in-memory datagrids (Apache Ignite, Hazelcast IMDG)|
|**EL**| PNUTS |[DynamoDB](https://en.wikipedia.org/wiki/Amazon_DynamoDB "Amazon DynamoDB"), [Cassandra](https://en.wikipedia.org/wiki/Apache_Cassandra "Apache Cassandra"), [Riak](https://en.wikipedia.org/wiki/Riak "Riak"), [Cosmos DB](https://en.wikipedia.org/wiki/Cosmos_DB "Cosmos DB")|

# ACID DataBase properties in short
A transaction is a sequence of database operations that satisfies the following rules:
- **[Atomicity]** A transaction is managed as a ***atomic*** unit that can be fully roll-backed if it fails.
- **[Consistency]** A transaction is ***consistent*** in that it cannot be committed if it violates database system rules.
- **[Isolation]** Independent transactions are ***isolated*** in that running them concurrently or sequentially leads to the same database system state.
- **[Durability]** A transaction is ***durable*** in that once it it fully committed, it will remain committed even in case of system failures (power off, network partitioning...).

# Hadoop's MapReduce

Execution steps of a MapReduce job containing 1 Mapper and 1 Reducer

- Map phase on mapper nodes:
  1. **RecordReader**: 
     - reads the input file's blocks from HDFS
     - divides them in *key->value* units called records
     - each block's records will be passed to a mapper
  2. **Mapper**: There is by default as many mappers as input file's blocks.
     - maps through records
     - produces 0 to *n* output record(s) for each input record
  3. **Partitioner**: 
     - organizes mapper output's records into partitions
     - each partition is intended to be fetched by a single and different reducer node.
  4. *Sort*: Sort records on *key* within each partitions.
  5. (optional) ***"Combiner"*** **Reducer**: an intermediate reducer called on each mapper node. 
     - within each sorted partition, for each different key, combiner produces a combiner output record having the same type as Mapper ones
     - combiner reduces the size of partitions and save network bandwidth
- Reduce phase on reduce nodes:
  1. *Fetch*: Access (mainly through network connection) to Map phase output written on HDFS.
  2. *Sort-Merge*: Merge sorted files
  3. **Reducer**: Leverage the previous sortings (on mappers and during merge on reducer) to efficiently output a record for each different key


**bold** steps relying on a "hook class" that are exposed to the user for extension.

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
eyJoaXN0b3J5IjpbMjA5MzIzNTU4OCwxODExMzExMTk2LC01Mz
k4MzY1MzgsLTE4NTk1NDIxNjMsMTc0MzE2OTAwNCwtNzM5ODUy
OTM1LDIwMTkzMDQ4OTcsLTE4NzE0NTY4NzksMTc1MjQ4NjA0Ny
wtNjE0OTQ2MjUsMTAyMjU4MTYwNCwxODM0NTAwNzEzLDE0MTY3
NDAyMTEsMTExOTI4NjcwNiwtNzU1MTEzMzUxLC0xNzYyNTMwND
U1XX0=
-->