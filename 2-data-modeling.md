# Section 2 — Data Modeling

Below are two concise reference sections followed by the detailed notebook explanations (theory and hands-on checklist). The first section is a high-level summary grouped by area (Databricks, Spark, Python, SQL). The second is a command/operator quick-reference table that shows syntax used in this course and a few useful extras marked with an asterisk (*).

---

## 1) High-level concepts (by area) — short bullets

### Databricks-specific
- Autoloader (cloudFiles): incremental file ingestion for cloud storage (used to stream JSON files into Bronze). Doc: https://docs.databricks.com/data-engineering/ingestion/auto-loader/index.html
- `availableNow` trigger: bounded processing of all currently-available files (demo/backfill). Doc: https://docs.databricks.com/data-engineering/ingestion/auto-loader/cloud-files-trigger.html
- Databricks Volumes / Unity Catalog: managed storage and cataloging for datasets. Doc: https://docs.databricks.com/data-governance/unity-catalog/index.html
- `dbutils.fs`: workspace filesystem utilities (ls, cp, rm). Doc: https://docs.databricks.com/dev-tools/databricks-utils.html

### Spark / PySpark
- Structured Streaming: `readStream` / `writeStream`, watermarking, triggers, `foreachBatch`. Docs: https://spark.apache.org/docs/latest/structured-streaming-programming-guide.html
- DataFrame APIs: `from_json`, `withColumn`, `cast`, `date_format`, `dropDuplicates`, `withWatermark`. Docs: https://spark.apache.org/docs/latest/api/python/
- Window functions and broadcast joins for dedup/enrichment. Docs: https://spark.apache.org/docs/latest/api/python/

### Python
- Core language constructs used: classes (`__init__`, `self`), lists/dicts/sets, f-strings, try/except, helper functions. Python stdlib reference: https://docs.python.org/3/library/stdtypes.html

### SQL / Delta
- Delta Lake `MERGE` for upserts and Type-2 SCD patterns. Docs: https://docs.delta.io/latest/delta-update.html#merge
- Managed Delta tables via `.table("name")` sink and `mergeSchema` option.

---

## 2) Quick commands & syntax reference (grouped by area)

Notes: To improve readability this section is split into small tables per area (Databricks, PySpark, SQL/Delta, Python). Items are numbered and options or sub-settings appear as indented sub-items (e.g. `1.1`).

<details>
<summary><strong>Databricks (1) — click to expand</strong></summary>

| Item | Command / Operator | Course usage (short) | Notes & extras |
|---:|---|---|---|
| 1 — Autoloader | `spark.readStream.format("cloudFiles").option("cloudFiles.format","json").load(path)` | used to ingest `kafka-raw` -> streaming DF | Extra options listed below (1.1, 1.2) |
| 1.1 — Autoloader option | `.option("cloudFiles.format","json")` | specify payload format | common: `json`, `parquet`, `csv` |
| 1.2 — Autoloader notifications* | `.option("cloudFiles.useNotifications","true")` | use S3 notifications to lower list cost | *Requires configuration of cloud events |
| 2 — Trigger | `.trigger(availableNow=True)` | used in `process_bronze()` to process available files and stop | Alternative: `.trigger(processingTime='10 seconds')` |
| 3 — Checkpointing | `.option("checkpointLocation", checkpoint_path)` | required on writeStream for progress tracking | ensure stable storage; use unique path per query |
| 4 — Partitioning (write) | `.partitionBy("topic","year_month")` | write partitioned Delta table | partitions improve read performance; avoid small partitions |
| 5 — Delta write options | `.option("mergeSchema", True).table("bronze")` | allow schema evolution on sink | Extra: `.mode("append")`, `format("delta")` |
| 6 — dbutils helpers | `dbutils.fs.ls(path)`, `dbutils.fs.cp(src,dst)`, `dbutils.fs.rm(path, True)` | copy and list dataset files (Copy-Datasets uses these) | DBFS vs `/Volumes/...` differences by catalog |

#### Short Autoloader example

```python
# Autoloader: read JSON files incrementally from cloud storage
autoloader_df = (spark.readStream
    .format("cloudFiles")
    .option("cloudFiles.format", "json")
    .schema("key BINARY, value BINARY, topic STRING, partition LONG, offset LONG, timestamp LONG")
    .load(f"{bookstore.dataset_path}/kafka-raw")
)

# Cast value to string and parse later with from_json in downstream processing
autoloader_df.select("topic", autoloader_df.value.cast("string").alias("value_text")).show(5)
```

</details>

<details>
<summary><strong>Spark / PySpark (2) — click to expand</strong></summary>

| Item | Command / Operator | Course usage (short) | Notes & extras |
|---:|---|---|---|
| 2.1 — Streaming read | `spark.readStream.table("bronze")` | read Bronze as streaming source | can also use `format("delta")` + `.load(path)` |
| 2.2 — JSON parsing | `F.from_json(F.col("value").cast("string"), json_schema)` | parse message payloads into struct | `from_json` returns Struct; use `.select("v.*")` to expand |
| 2.3 — Timestamp handling | `.withColumn("timestamp", (F.col("timestamp")/1000).cast("timestamp"))` | convert epoch ms -> timestamp | or use `to_timestamp` on string values |
| 2.4 — Partition key | `F.date_format("timestamp","yyyy-MM")` | derive `year_month` partition column | common for time-partitioning |
| 2.5 — Watermarking | `.withWatermark("order_timestamp","30 seconds")` | used for dedup state cleanup | choose watermark based on expected lateness |
| 2.6 — Deduplication | `.dropDuplicates(["order_id","order_timestamp"])` | remove duplicates in stream (with watermark) | `dropDuplicates` is stateful; watch memory |
| 2.7 — foreachBatch | `.writeStream.foreachBatch(func).option("checkpointLocation", ...).start()` | used to call upsert functions per micro-batch | `func(microBatchDF, batchId)` — use `MERGE` inside func |
| 2.8 — Window & rank | `Window.partitionBy(...).orderBy(F.col(...).desc())` and `F.rank()` | used in `upsert_customers_batch` for latest row selection | Window functions are powerful for dedupe & SCD logic |
| 2.9 — Broadcast join | `F.broadcast(df_small)` | used to enrich customers with country lookup | use for small static tables to avoid shuffle |
| 2.10 — Spark actions | `df.collect()` / `df.first()` / `df.take(n)` | used to read small results to driver | Prefer `first()` / `take(1)` over `collect()` for single-row reads |

</details>

<details>
<summary><strong>SQL / Delta (3) — click to expand</strong></summary>

| Item | Command / Operator | Course usage (short) | Notes & extras |
|---:|---|---|---|
| 3.1 — MERGE (Delta) | `MERGE INTO target USING source ON <cond> WHEN MATCHED THEN UPDATE ... WHEN NOT MATCHED THEN INSERT ...` | used for upserts & Type-2 SCD | Delta support for `MERGE` SQL; essential for exam |

</details>

<details>
<summary><strong>Python (4) — click to expand</strong></summary>

| Item | Command / Operator | Course usage (short) | Notes & extras |
|---:|---|---|---|
| 4.1 — Basics | `class`, `__init__`, `self`, `list.append()`, `dict` | `Copy-Datasets` creates `CourseDataset` instance and calls methods | `__init__` initializes attributes; avoid heavy I/O in constructor |

</details>

---

## 3) Where to read (official docs)

- Databricks Autoloader: https://docs.databricks.com/data-engineering/ingestion/auto-loader/index.html
- Databricks `dbutils`: https://docs.databricks.com/dev-tools/databricks-utils.html
- Databricks Unity Catalog & Volumes: https://docs.databricks.com/data-governance/unity-catalog/index.html
- Spark Structured Streaming guide: https://spark.apache.org/docs/latest/structured-streaming-programming-guide.html
- PySpark functions (from_json, date_format, etc): https://spark.apache.org/docs/latest/api/python/
- Delta Lake MERGE & update patterns: https://docs.delta.io/latest/delta-update.html#merge
- Python stdlib (core types): https://docs.python.org/3/library/stdtypes.html

---

<!-- Detailed notebook explanations follow (kept from original README) -->
