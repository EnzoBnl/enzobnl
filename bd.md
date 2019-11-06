<!--NOTE HEAD START-->
<link rel="icon" type="image/png" href="./imgs/favicon_db.png" />
<script src="https://cdnjs.cloudflare.com/ajax/libs/mermaid/8.0.0/mermaid.min.js"></script>
<script type="text/x-mathjax-config">MathJax.Hub.Config({tex2jax: {skipTags: ['script', 'noscript','style', 'textarea', 'pre'],inlineMath: [['$','$']]}});</script>
<script src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML" type="text/javascript"></script>
<script>document.body.style.background = "#f2f2f2";</script>
<!--NOTE HEAD END-->

# Theory
## PACELC theorem
Abadi's *"Consistency Tradeoffs in Modern Distributed Database System Design"*, 2012.

P = (network) Partitioning = A sub-part of the nodes become unreachable.

A = Availability = Requests are newer answered with an error.

C = Consistency = Requests get the latest version of what they ask for or an error.

L = Latency = Requests get fast their result in its currently available state.

**Theorem**: In case of *network Partition*, the system remains *Available* **OR** *Consistent*, else it ensures *low Latency* **OR** *Consistency*



||PC|PA|
|--|--|--|
|**EC**|[MongoDB](https://en.wikipedia.org/wiki/MongoDB), BigTable/HBase and fully ACID$^{[1]}$ systems: [VoltDB](https://en.wikipedia.org/wiki/VoltDB "VoltDB")/H-Store, Megastore, [MySQL Cluster](https://en.wikipedia.org/wiki/MySQL_Cluster "MySQL Cluster")|Most in-memory datagrids (Apache Ignite, Hazelcast IMDG)|
|**EL**| PNUTS |[DynamoDB](https://en.wikipedia.org/wiki/Amazon_DynamoDB "Amazon DynamoDB"), [Cassandra](https://en.wikipedia.org/wiki/Apache_Cassandra "Apache Cassandra"), [Riak](https://en.wikipedia.org/wiki/Riak "Riak"), [Cosmos DB](https://en.wikipedia.org/wiki/Cosmos_DB "Cosmos DB")|

# ACID
- Atomicity
- Consistency
- Isolation
- Durability

# Hadoop's MapReduce

Steps of a job containing a Mapper and a Reducer

- On mapper node:
  1. *RECORD READER*
  2. *MAPPER*
  3. *PARTITIONER*
  4. *Sort*
  5. **COMBINER*: 
- On reduce nodes:
  1. *Fetch*
  2. *Merge/Sort*
  3. *REDUCER*

*: optional
*UPPER CASE*: steps relying on a "hook class"


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
eyJoaXN0b3J5IjpbLTUzOTgzNjUzOCwtMTg1OTU0MjE2MywxNz
QzMTY5MDA0LC03Mzk4NTI5MzUsMjAxOTMwNDg5NywtMTg3MTQ1
Njg3OSwxNzUyNDg2MDQ3LC02MTQ5NDYyNSwxMDIyNTgxNjA0LD
E4MzQ1MDA3MTMsMTQxNjc0MDIxMSwxMTE5Mjg2NzA2LC03NTUx
MTMzNTEsLTE3NjI1MzA0NTVdfQ==
-->