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
  1. **RecordReader**: Read the file from HDFS and divide it in units called records.
  2. **Mapper**: app
  3. **Partitioner**: Organize mapper output's records into partitions, each partition will be fetched by a single and different reducer node.
  4. *Sort*: Sort records within partitions
  5. (optional) **Combiner**: 
- Reduce phase on reduce nodes:
  1. *Fetch*: Access (mainly through network connection) to Map phase output written on HDFS.
  2. *Sort-Merge*: Merge sorted files
  3. **Reducer**


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
eyJoaXN0b3J5IjpbLTQyNjMwODY0OCwtNTM5ODM2NTM4LC0xOD
U5NTQyMTYzLDE3NDMxNjkwMDQsLTczOTg1MjkzNSwyMDE5MzA0
ODk3LC0xODcxNDU2ODc5LDE3NTI0ODYwNDcsLTYxNDk0NjI1LD
EwMjI1ODE2MDQsMTgzNDUwMDcxMywxNDE2NzQwMjExLDExMTky
ODY3MDYsLTc1NTExMzM1MSwtMTc2MjUzMDQ1NV19
-->