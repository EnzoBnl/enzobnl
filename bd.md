<!--NOTE HEAD START-->
<link rel="icon" type="image/png" href="./imgs/favicon_db.png" />
<script src="https://cdnjs.cloudflare.com/ajax/libs/mermaid/8.0.0/mermaid.min.js"></script>
<script type="text/x-mathjax-config">MathJax.Hub.Config({tex2jax: {skipTags: ['script', 'noscript','style', 'textarea', 'pre'],inlineMath: [['$','$']]}});</script>
<script src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML" type="text/javascript"></script>
<script>document.body.style.background = "#f2f2f2";</script>
<!--NOTE HEAD END-->

# Hadoop's MapReduce
ORDER PARTITIONER -> COMBINER

In Hadoop- The definitive guide 3rd edition, page 209, we have below words:

Before it writes to disk, the thread first divides the data into partitions

corresponding to the reducers that they will ultimately be sent to. Within each

partition, the background thread performs an in-memory sort by key, and if

there is a combiner function, it is run on the output of the sort. Running the combiner function makes for a more compact map output, so there is less data to write to local disk and to transfer to the reducer.
Each time the memory buffer reaches the spill threshold, a new spill file is created, so after the map task has written its last output record, there could be several spill files. Before the task is finished, the spill files are merged into a single partitioned and sorted output file. The configuration property io.sort.factor controls the maximum number of streams to merge at once; the default is 10.
If there are at least three spill files (set by the min.num.spills.for.combine property), the combiner is run again before the output file is written. Recall that combiners may be run repeatedly over th einput without affecting the final result. If there are only one or two spills, the potential reduction in map output size is not worth the overhead in invoking the combiner, so it is not run again for this map output.So combiner is run during merge spilled file.

- On mapper node:
  1. *RECORD READER*
  2. *MAPPER*
  3. *PARTITIONER*
  4. *SORT* 
  5. *COMBINER*
- SHUFFLE
- On reduce nodes 
  1. *FETCH*
  2. *MERGE/SORT*
  3. *REDUCE*


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
eyJoaXN0b3J5IjpbLTEwODAwNDA0OTcsMTc1MjQ4NjA0NywtNj
E0OTQ2MjUsMTAyMjU4MTYwNCwxODM0NTAwNzEzLDE0MTY3NDAy
MTEsMTExOTI4NjcwNiwtNzU1MTEzMzUxLC0xNzYyNTMwNDU1XX
0=
-->