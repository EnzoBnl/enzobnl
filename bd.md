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
[See dedicated notes](https://bonnal-enzo.github.io/spark.html)
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

## Hadoop
### `JAVA_HOME` said to be missing but is not
Getting `ERROR: JAVA_HOME is not set and could not be found.` but the variable is exported in your `.bashrc` and/or `.profile` ? -> You need to also export it by uncommenting the proper line in `<hadoop home>/etc/hadoop/hadoop-env.sh`.

## Google Cloud Platform

### Submit and monitor a Dataproc spark job (YARN)
#### Submit
1. **Submit your spark app** in your terminal using `gcloud` cli
```bash
gcloud dataproc jobs submit spark --cluster=<dataproc-cluster-name>  --region=<e.g. europe-west1> --jars gs://<path-to-jarname.jar --class com.package.name.to.MainClassName --properties 'spark.executor.cores=2,[...]' -- <arg1 for MainClass> <arg2 for MainClass> [...]
```
2. **Visit the page describing your master node** VM in Google Cloud Console at: 
```
https://console.cloud.google.com/compute/instancesDetail/zones/<region, e.g.: europe-west1-c>/instances/<dataproc cluster name>-m
```
5.  **Copy the ephemeral external ip address of the master** node VM, we will take the example address `104.155.87.75` as of now
6.  **Open a SSH connection to that master VM**, forwarding its 1080 port using:
```bash
ssh -A -D 1080 104.155.87.75
```
7. Launch a Chrome/**Chromium with a proxy on 1080** port (the one passing through our ssh connection)
```bash
chromium-browser --user-data-dir --proxy-server="socks5://127.0.0.1:1080"
```
8. Use this browser to **access the YARN cluster UI** at `http://<dataproc cluster name>-m:8088/cluster/` or the **Spark History Server** at `http://<dataproc cluster name>-m:18080`

### BigQuery vs BigTable
- *BigQuery* excels for OLAP (OnLine Analytical Processing): scalable and efficient analytic querying on unchanging data (or just appending data).
- *BigTable* excels for OLTP (OnLine Transaction Processing): scalable and efficient read and write

### BigQuery
- Storage Level: 
  - distributed file system: Google Colossus (on hard drives)
  - columnar storage format: [Google Capacitor](https://cloud.google.com/blog/products/bigquery/inside-capacitor-bigquerys-next-generation-columnar-storage-format)
- Intermediate state storage: *in-memory shuffler* component (separated from compute ressources)
- Compute Engine: Dremel X (Successor to Dremel)

See:
Google's [*BigQuery Under the Hood* podcast](https://www.youtube.com/watch?v=2jDGAl4Ef-Y) and [blog article](https://cloud.google.com/blog/products/bigquery/bigquery-under-the-hood)

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

# File formats
## Parquet
Sources:
- [Blog post by Mridul Verma](https://miuv.blog/2018/08/21/handling-large-amounts-of-data-with-parquet-part-1/)
- [Twitter engineering blog post](https://blog.twitter.com/engineering/en_us/a/2013/announcing-parquet-10-columnar-storage-for-hadoop.html)
- [SO post by Artem Ignatiev](https://stackoverflow.com/a/45541741/6580080)
- [Blog post by Bartosz Konieczny](https://www.waitingforcode.com/apache-parquet/encodings-apache-parquet/read)

### Hierarchy
- Root Folder
- File
- Row group
- Column chunk
- Pages: a column chunk is composed of 1 **or more** data page(s) + optional pages like a dictionary page (only 1 dictionary page per column chunk)

### Compression
A compression algorithm is applied by parquet, the compression unit being the **page**.

Here are the supported algorithms, when set through *Spark* configuration `"spark.sql.parquet.compression.codec"`:
- snappy (default)
- gzip
- lzo
- brotli
- lz4
- zstd

### Encodings
https://github.com/apache/parquet-format/blob/master/Encodings.md
Parquet framework **automatically find out which encoding to use** for each column depending on the **data type** (must be supported by the encoding) and the **values characteristics** (some rules decide if it is worth to use this or this encoding, depending on the data values). *Dictionary encoding* is a default that parquet will try on each column: It is considered to be the most efficient encoding if its condition of trigger are met (small number of distinct values).

#### Dictionary encoding
If *dictionary encoding* is enabled, parquet's *row groups* looks like:

![](https://i2.wp.com/miuv.blog/wp-content/uploads/2018/08/blank-diagram-17-e1534819920877.png?resize=594%2C226&ssl=1)
*Columns chucks*'s *data pages* consist of a bunch of references relative to the *dictionary page* that follows them.

This *dictionary pages* are keys and values encoded using plain encoding.

There is a fallback to plain encoding if it turns out that there is too much unique values to leverage this encoding.

In Spark, one can desactivate dictionary encoding with the config: `"parquet.enable.dictionary" -> "false"`, that can be useful as it may take the place of a more efficient encoding. (*dictionary encoding* taking the place of *delta string encoding* on `BYTE_ARRAY`s for example)

#### Delta encoding
Supported types are `INT32` and `INT64`. 
This encoding leverage the **similarity between successive values**, for instance when integers are representing *timestamps*.

#### Delta strings encoding
Suppoted type is `BYTE_ARRAY`.
Known as [Incremental encoding](https://en.wikipedia.org/wiki/Incremental_encoding), it is another delta encoding that elude the writing of common prefix and suffix between: may lead to nice compression for close HTML pages.



<!--stackedit_data:
eyJoaXN0b3J5IjpbMTQ5NjIyNTA5LDI5MDA0OTA3NSw0MjQ0Mj
MyNzMsLTE3MTEwNTMwMCwtMTc2ODIxMjgwNywtMTA5Mzc5MTUz
NSwxMTk5MDQ3OTk1LC0xMzQyMTM2NjksMjA4MzMyOTY5MCwxMT
AzMzIxNjYsLTIxMzI1NDQ0MjUsNTQyNzY3NTU4LC0xOTQ1NzIx
MTE2LC0yMDIwNTIzNjU5LDkyMjczMTQwNywxMzQ0OTQxMTcxLD
E5MzgzOTM4OTAsLTIxNDA4MzE3NDksMTU1NzE3Mzk1LDgwNDc1
ODMzN119
-->