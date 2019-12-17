# Spark
- Vectorized Parquet Reader OOM 
- NodeManager do the shuffle if Yarn Shuffle Manager
- Yarn shuffle manager does not save shuffle writes on HDFS, need a replication to prevent node failure 
- In spark SQL, scan parallelism (number of scan tasks) is the input block size (parquet on CS or HDF files are 128MB)
<!--stackedit_data:
eyJoaXN0b3J5IjpbMTYxMjgwMTMzMiwtMTA2NjA3MTc4NV19
-->