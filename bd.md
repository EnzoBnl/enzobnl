<!--NOTE HEAD START-->
<link rel="icon" type="image/png" href="./imgs/favicon_db.png" />
<script src="https://cdnjs.cloudflare.com/ajax/libs/mermaid/8.0.0/mermaid.min.js"></script>
<script type="text/x-mathjax-config">MathJax.Hub.Config({tex2jax: {skipTags: ['script', 'noscript','style', 'textarea', 'pre'],inlineMath: [['$','$']]}});</script>
<script src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML" type="text/javascript"></script>
<script>document.body.style.background = "#f2f2f2";</script>
<!--NOTE HEAD END-->

# Hadoop's MapReduce

- On mapper node:
  1. *RECORD READER*
  2. *MAPPER*
  3. *PARTITIONER*
  4. *SORT*
  5. *COMBINER*
- Shuffle
- On reduce nodes:
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
eyJoaXN0b3J5IjpbLTExMjcyODkyOTEsMTc1MjQ4NjA0NywtNj
E0OTQ2MjUsMTAyMjU4MTYwNCwxODM0NTAwNzEzLDE0MTY3NDAy
MTEsMTExOTI4NjcwNiwtNzU1MTEzMzUxLC0xNzYyNTMwNDU1XX
0=
-->