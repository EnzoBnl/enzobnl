<!--NOTE HEAD START-->
<link rel="icon" type="image/png" href="./imgs/favicon_db.png" />
<script src="https://cdnjs.cloudflare.com/ajax/libs/mermaid/8.0.0/mermaid.min.js"></script>
<script type="text/x-mathjax-config">MathJax.Hub.Config({tex2jax: {skipTags: ['script', 'noscript','style', 'textarea', 'pre'],inlineMath: [['$','$']]}});</script>
<script src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML" type="text/javascript"></script>
<script>document.body.style.background = "#f2f2f2";</script>
<!--NOTE HEAD END-->



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
eyJoaXN0b3J5IjpbOTI5MTY5MDA1LC0xODcxNDU2ODc5LDE3NT
I0ODYwNDcsLTYxNDk0NjI1LDEwMjI1ODE2MDQsMTgzNDUwMDcx
MywxNDE2NzQwMjExLDExMTkyODY3MDYsLTc1NTExMzM1MSwtMT
c2MjUzMDQ1NV19
-->