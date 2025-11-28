# Section 3 — Data Processing

Quick-reference tables for streaming data processing, CDC, and joins.

---

## Databricks Streaming & CDC

??? note "Change Data Capture (CDC) Operations"

    | Operation | Code/Command | Usage | Notes |
    |-----------|--------------|-------|-------|
    | **Enable CDF on Table** | `ALTER TABLE <table> SET TBLPROPERTIES (delta.enableChangeDataFeed = true)` | Enable Change Data Feed tracking | Required for CDC operations |
    | **Query Table Changes** | `SELECT * FROM table_changes('<table>', <version>)` | Get changes since version | Returns insert/update/delete operations |
    | **Stream with CDF** | `.option("readChangeData", True).option("startingVersion", n)` | Read change data as stream | Use with readStream |
    | **Filter Row Status** | `.filter(F.col("row_status").isin(["insert", "update"]))` | Process specific operations | Common in CDC patterns |
    | **Window Ranking** | `Window.partitionBy("id").orderBy(F.col("time").desc())` | Get latest records | Use with `F.rank().over(window)` |
    | **Batch Upsert Pattern** | `foreachBatch(batch_upsert)` | Custom micro-batch logic | Combine window ops with MERGE |

??? note "Stream Processing Patterns"

    | Pattern | Code/Command | Usage | Notes |
    |---------|--------------|-------|-------|
    | **Stream-Stream Join** | `stream_df1.join(stream_df2, "key", "inner")` | Join two streaming DataFrames | Requires watermark for state management |
    | **Stream-Static Join** | `stream_df.join(F.broadcast(static_df), "key")` | Join stream with static lookup | Use broadcast for small static tables |
    | **Watermark** | `.withWatermark("timestamp_col", "10 minutes")` | Define late data threshold | Required for stateful operations |
    | **availableNow Trigger** | `.trigger(availableNow=True)` | Process all available data once | Micro-batch mode for incremental loads |
    | **Checkpoint Location** | `.option("checkpointLocation", path)` | Track stream progress | Required for fault tolerance |
    | **foreachBatch** | `.writeStream.foreachBatch(func)` | Custom batch processing | Access to full DataFrame API per batch |

??? note "MERGE Operations"

    | Operation | SQL Syntax | Usage | Notes |
    |-----------|------------|-------|-------|
    | **Basic MERGE** | `MERGE INTO target USING source ON key WHEN MATCHED THEN UPDATE SET * WHEN NOT MATCHED THEN INSERT *` | Upsert pattern | Standard CDC implementation |
    | **Conditional Update** | `WHEN MATCHED AND target.time < source.time THEN UPDATE SET *` | Update only newer records | Prevents overwriting with old data |
    | **Delete on Match** | `WHEN MATCHED THEN DELETE` | Remove matched records | Used for propagating deletes |
    | **Multiple Conditions** | Multiple `WHEN MATCHED` clauses | Complex merge logic | Evaluated in order |

---

## PySpark Data Processing

??? note "DataFrame Transformations"

    | Operation | Code/Command | Usage | Notes |
    |-----------|--------------|-------|-------|
    | **Parse JSON** | `F.from_json(F.col("value").cast("string"), schema)` | Extract nested JSON | Requires schema definition |
    | **Select Nested** | `.select("v.*")` | Flatten struct columns | After from_json |
    | **Window Function** | `.withColumn("rank", F.rank().over(window))` | Ranking within partitions | Not supported in streaming without foreachBatch |
    | **Broadcast Join** | `F.broadcast(small_df)` | Optimize small table joins | Use for lookup tables |
    | **Filter Multiple** | `.filter(F.col("status").isin(["a", "b"]))` | Multiple value filter | Alternative to OR conditions |
    | **Date Operations** | `F.date_add("timestamp", 30)` | Date arithmetic | Also date_sub, months_between, etc. |

??? note "Gold Table Aggregations"

    | Operation | Code/Command | Usage | Notes |
    |-----------|--------------|-------|-------|
    | **Group & Aggregate** | `.groupBy("country").agg(F.count("*").alias("total"))` | Basic aggregation | Use multiple agg functions in one call |
    | **Window Aggregate** | `.withColumn("running_total", F.sum("amount").over(window))` | Running calculations | Window without bounds = all rows in partition |
    | **Pivot** | `.groupBy("country").pivot("category").agg(F.sum("sales"))` | Wide format transformation | Creates column per pivot value |
    | **Explode Array** | `.select("*", F.explode("array_col").alias("item"))` | Array to rows | Creates new row for each array element |

---

## Key Concepts

1. **Change Data Capture (CDC)**: Track insert/update/delete operations on Delta tables using Change Data Feed
2. **Stream-Stream Joins**: Require watermarks and proper windowing for state management
3. **foreachBatch Pattern**: Enables full DataFrame API (including window functions) on streaming data by processing micro-batches
4. **MERGE Statement**: Standard pattern for upserts; supports conditional logic for complex CDC scenarios
5. **Broadcast Joins**: Optimize stream-static joins by broadcasting small lookup tables to all workers

---

## Official Documentation

- [Delta Lake Change Data Feed](https://docs.delta.io/latest/delta-change-data-feed.html)
- [Structured Streaming Programming Guide](https://spark.apache.org/docs/latest/structured-streaming-programming-guide.html)
- [Delta Lake MERGE Operations](https://docs.delta.io/latest/delta-update.html#merge)
- [Spark Structured Streaming + Kafka Integration](https://spark.apache.org/docs/latest/structured-streaming-kafka-integration.html)
- [PySpark Streaming Functions](https://spark.apache.org/docs/latest/api/python/reference/pyspark.sql/functions.html)
