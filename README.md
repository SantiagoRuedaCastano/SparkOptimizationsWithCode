# Spark Optimizations WithCode

## Partitioning
Partitioning refers to diving the data into smaller, manageable chunks(partitions) across the cluster's nodes. Proper partitioning ensures parallel processing and avoids data skew, leading to balanced workloads and improved performance.


```scala
dfRepartitioned = df.repartition(10, "columnName")
```

## Reduce shuffle partitions
By default, Spark has a high number of shuffle partitions(200). Reducing this number can improve performance, especially for small datasets.
```scala
spark.conf.set("spark.sql.shuffle.partitions", "20")
```

## Coalesce
Coalesce reduces the number of partitions in a Dataframe, which is more efficient than repartitioning when decreasing the number of partitions
```scala
dfCoalesced = df.coalesce(8)
```

## Caching and persistence

Caching and persistence are used to store intermediate results in memoy or disk, reducing the need for recomputation. This is particularly useful when the same Dataframe is accessed multimples times in a Spark Job.
```scala
df.cache()
df.show()

df.persist(StorageLevel.MEMORY_AND_DISK)
df.show()
```

## Reduce task serialization overhead
Using `Kyro` serialization can reduce the overhead associated with task deserealization, improving performance compared to the default java serialization.
```scala
spark.conf.set("spark.serializer", "org.apache.spark.serializer.KyroSerializer")
```

## Broadcast variables
Broadcast variables allow the distribution of read-only variable to all nodes in the cluster, which can be more efficient than shipping the variable with every task. It's particularly useful for small lookup tables
```scala
//Broadcast variable
val broadcastVar = sc.broadcast([1, 2, 3])
```

## Columnar format
Using columnar format like Parquet or ORC can improve read performance by allowing Spark to read only the neccessary columns. The formats also support efficient compressions and encoding schemas
```scala
df.write.parquet("path/to/parquet/file")
```

## Predicate Pushdown
Predicate pushdown allows Spark to filter data at the data source level before loading it into memory, reducing the amount of data transferred and improving the performance.
```scala
df = spark.read.parquet("path/to/parquet/file").filter("columnName > 100")
```


## Vectorized UDFs(Pandas UDFs)
Vectorized UDFs, utilize apache Arrow to process batches of rows, improving performance compared row-by-row processing in standard UDFs.
```python
from pyspark.sql.functions
import pandas_udf, PandasUDFType

@pandas_udf("double", PandasUDFType.SCALAR)
def vectorized_udf(x):
    return x+1

df.withColumn("new_column", vectorized_udf(df["existing_column"])).show()
```

## Avoid using Explode
Explode is an expensive operation to flat arrays into multiple rows. Using it should be minimized, or optimized by reducing the size of the Dataframe before flattening(exploding) it.

```python
from pyspark.sql.functions import explode
df_exploded = df.withColumn("exploded_column", explode(df["array_colum"]))
```

## Tungsten Execution Engine
Tungsten is Spark memory computation engine that optimizes the execution plan for Dataframes and Dataset, utilizing memory and CPU more efficiently.
Tungsten is enabled by default in Spark, no specific code is needed. However, using Dataset and Dataframe ensures you leverage tungsten's optimization.

## Using Dataframes/Datasets API
The Dataframes/datasets API provides high-level abstractions and optimizations compare to RDDs, including Catalyst Optimizer for query planning and execution.
```scala
val df = spark.read.parquet("path/to/file")
val dfResult = df.groupBy("columnName").agg(sum("valueColumn"))
```

## Catalyst Optimizer

## Join Optimization
Broadcast join are the more efficient than shuffle join when one of the Dataframes is small, as the small Dataframe is brocasted to all nodes, avoiding shuffles.
```scala
val dfResult = df1.join(broadcast(df2), condition, "inner")
```

## Join Strategies


| **Join Strategy**                       | **Equality / Non-Equality** | **Relative Cost**                                                                                               | **When It’s Used**                                                                            | **Hint to Force** |
| --------------------------------------- | ----------------------------------------------- | --------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------- | ----------------- |
| **Broadcast Hash Join (BHJ)**           | Equality only                                   | Cheap (very fast, avoids shuffle) if one side is small enough                                                   | One dataset is small enough to broadcast to all executors (default ≤ 10MB, configurable)      | `broadcast(df)`   |
| **Shuffle Hash Join (SHJ)**             | Equality only                                   | Expensive if skewed; otherwise moderate. Requires shuffle of both sides, but hash build side must fit in memory | Both sides are large but one is still small enough to build a hash table after shuffle        | `SHUFFLE_HASH`    |
| **Sort-Merge Join (SMJ)**               | Equality only                                   | More expensive than BHJ, but scalable for very large datasets; shuffle + sort both sides                        | Default for large joins when no broadcast fits, works well if data already sorted/partitioned | `MERGE`           |
| **Broadcast Nested Loop Join (BNLJ)**   | Equality + Non-Equality                         | Expensive (scans broadcasted dataset for every row in large side)                                               | When one side is small but join condition is not equality (e.g. `<`, `>`, `!=`, range joins)  | `BROADCAST_NL`    |
| **Cartesian Product Join (Cross Join)** | No condition (pure cross product)               | Extremely expensive (O(n × m)), last resort                                                                     | When explicitly requested (cross join) or non-equality join without broadcastable side        | `CROSS`           |



## Resource Allocation
Properly allocating resources such as executor memory and cores ensures optimal performance by matching the resource requirements of your Spark jobs.
```bash
spark-submit --executor-memory 4g --executor-cores 2 sparkjob.jar
```

## Skew Optimization
Handling skewed data can improve performance. Techniques like salting(adding random key) can help distribute skewed data more evenly across partitions
```python
pyspark.sql.functions import rand

df_salted = df.withColumn("salt", (rand()*10).cast("int"))
df_salted_repartitioned = df_salted.repartition("salt")
```

## Speculative Execution
Speculative execution re-runs slow tasks in parallel and uses the result of the first completed task, helping to mitigate the impact of straggler tasks.
```scala
spark.conf.set("spark.speculation", "true")
```


## Adaptive  Query Execution (AQE)
AQE optimizes query execution plans dyhnamically based on runtime statistics, such as the actual size of data processed, leading to more efficientquery execution.
```scala
spark.conf.set("spark.sql.adaptive.enabled", "true")
```

## Dynamic Partition Pruning
Dynamic Partition Pruning improves the performance of join queries by dynamically prunning partitions at runtime, reducing the amount of data read.
```scala
spark.conf.set("spark.sql.dynamicPartitionPruning.enabled", "true")
```

## Using Data Locality
Ensuring that data processing happens as close to the data as possible reduces network I/O, leading to faster data processing.

## Leveraging Built-in Functions
Built-in functions are optimized for performance and should be preferred over custom UDFs, which can introduce significant overhead.
```scala
df.select(col("*"), lit("a").alias("newColumn"))
```

## Bucketing

Bucketing is a technique in Apache Spark (and Hive) to physically organize data into a fixed number of buckets (files) based on the value of a column. It’s similar to hash partitioning, but unlike partitioning:

- Partitioning creates a directory for each partition column value.

- Bucketing creates a fixed number of files per table, regardless of distinct values.

- Bucketing is typically used to optimize joins and groupBy operations on large tables.

- Each row is assigned to a bucket by applying a hash function on the bucketing column and then taking has (column) % numBuckets.

```scala
spark.conf.set("spark.sql.bucketing.enabled", "true")

df.write
      .format("parquet")
      .bucketBy(4, "id")    // 4 buckets by hash of id
      .sortBy("id")         // optional: sorting inside each bucket
      .saveAsTable("people_bucketed")

    // Read back the bucketed table
    val bucketedDF = spark.table("people_bucketed")

    bucketedDF.show()
```



















