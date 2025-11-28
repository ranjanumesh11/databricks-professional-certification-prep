# Section 4 — Improving Performance

Quick-reference tables for Python UDFs, vectorization, and performance optimization.

---

## Python UDFs

??? note "UDF Types and Registration"

    | UDF Type | Code/Command | Usage | Notes |
    |----------|--------------|-------|-------|
    | **Standard UDF** | `udf(func)` or `spark.udf.register("name", func)` | Python UDF (row-by-row) | Slow due to serialization overhead |
    | **Decorator UDF** | `@udf("returnType")` | Define UDF with decorator | Type must be specified as string |
    | **Pandas UDF (Vectorized)** | `@pandas_udf("returnType")` | Vectorized operations | Much faster than standard UDFs |
    | **Register for SQL** | `spark.udf.register("sql_name", udf_func)` | Use UDF in SQL queries | Available in `%sql` cells |

??? note "UDF Performance Patterns"

    | Pattern | Code/Command | Usage | Notes |
    |---------|--------------|-------|-------|
    | **Standard Python UDF** | `def func(x): return x * 2` then `udf(func)` | Simple row-by-row logic | Serializes data to Python process |
    | **Pandas UDF Series** | `def func(s: pd.Series) -> pd.Series: return s * 2` | Vectorized column operations | Processes batches with Pandas |
    | **Apply UDF** | `df.select(my_udf(col("price"), lit(50)))` | Use in DataFrame API | Can combine with `lit()` for constants |
    | **SQL UDF Usage** | `SELECT my_udf(price, 50) FROM table` | Use in SQL context | Must register with `spark.udf.register()` |

??? note "UDF Best Practices"

    | Practice | Code/Command | Usage | Notes |
    |----------|--------------|-------|-------|
    | **Prefer Built-in Functions** | Use `F.when()`, `F.coalesce()`, etc. | Native Spark functions | Always faster than UDFs |
    | **Use Pandas UDF** | `@pandas_udf("double")` | When UDF is necessary | 10-100x faster than standard UDF |
    | **Type Hints** | `def func(x: pd.Series) -> pd.Series` | Pandas UDF type safety | Required for Pandas UDFs |
    | **Avoid Complex Logic** | Keep UDF simple | Minimize computation | Consider breaking into steps |
    | **Test Performance** | Compare with `.explain()` and timing | Measure impact | Standard UDFs create extra stages |

---

## Vectorization with Pandas

??? note "Pandas UDF Patterns"

    | Pattern | Code/Command | Usage | Notes |
    |---------|--------------|-------|-------|
    | **Scalar Pandas UDF** | `@pandas_udf("double") def f(s: pd.Series) -> pd.Series` | Column transformation | Most common pattern |
    | **Grouped Map** | `@pandas_udf(..., functionType=PandasUDFType.GROUPED_MAP)` | Group-level operations | Deprecated in Spark 3.0+ |
    | **Apply in Pandas** | `df.groupBy("key").applyInPandas(func, schema)` | Custom group processing | Modern alternative to GROUPED_MAP |
    | **Map Iterator** | `df.mapInPandas(func, schema)` | Process batches | Low memory overhead for large datasets |

??? note "Performance Optimization"

    | Technique | Code/Command | Usage | Notes |
    |-----------|--------------|-------|-------|
    | **Cache** | `df.cache()` or `df.persist()` | Reuse DataFrame | Use for iterative operations |
    | **Broadcast** | `F.broadcast(small_df)` | Small table joins | Avoid shuffle for small tables |
    | **Repartition** | `df.repartition(n, "key")` | Control parallelism | Use before expensive operations |
    | **Coalesce** | `df.coalesce(n)` | Reduce partitions | Minimize output files |
    | **Predicate Pushdown** | `.filter()` early in chain | Filter before joins | Let Delta Lake skip files |
    | **Column Pruning** | `.select()` only needed columns | Reduce data volume | Especially important with wide tables |

---

## Delta Lake Optimization

??? note "Table Partitioning"

    | Concept | Syntax/Command | Usage | Notes |
    |---------|----------------|-------|-------|
    | **Create Partitioned Table** | `CREATE TABLE table PARTITIONED BY (col1, col2)` | Physical separation into subdirectories | Use for very large tables with low-cardinality columns |
    | **Partition Columns** | Common: date/time, region, category | Choose based on query filters | Avoid high-cardinality columns (user_id, transaction_id) |
    | **Directory Structure** | `table_location/col1=value1/col2=value2/` | Hierarchical folder layout | Enables partition-level access control |
    | **Partition Pruning** | `WHERE date = '2023-01-01'` | Skip entire partitions | Query optimizer uses partition filters |
    | **Trade-offs** | Small files per partition, rigid schema | Not ideal for small/medium tables | OPTIMIZE only runs per partition |

??? note "Z-Order Indexing"

    | Concept | Syntax/Command | Usage | Notes |
    |---------|----------------|-------|-------|
    | **Enable Z-Ordering** | `OPTIMIZE table ZORDER BY (col1, col2)` | Co-locate column data in files | No subdirectories created |
    | **Data Co-location** | Groups similar values together | Example: ID 1-50 in file1, 51-100 in file2 | Improves data skipping efficiency |
    | **Multi-column** | `ZORDER BY (customer_id, order_date)` | Optimize for multiple filters | Order matters; choose most selective first |
    | **Non-incremental** | Must rerun on new data | Full table reorganization | Can be expensive for frequent updates |
    | **File Statistics** | Min/max values per file | Query planner skips irrelevant files | Leveraged automatically by Delta Lake |

??? note "Liquid Clustering"

    | Concept | Syntax/Command | Usage | Notes |
    |---------|----------------|-------|-------|
    | **Create Clustered Table** | `CREATE TABLE table CLUSTER BY (col1, col2)` | Define clustering at table level | Modern replacement for Z-order |
    | **Alter Existing Table** | `ALTER TABLE table CLUSTER BY (col1, col2)` | Add clustering to existing table | No data rewrite required |
    | **Trigger Clustering** | `OPTIMIZE table` | Compact and cluster files | Incremental operation; only processes new data |
    | **Redefine Keys** | `ALTER TABLE table CLUSTER BY (new_cols)` | Change clustering columns | Adapts to evolving query patterns |
    | **Automatic Clustering** | `CREATE TABLE table CLUSTER BY AUTO` | Databricks chooses keys | Requires Predictive Optimization enabled |
    | **Advantages** | Incremental, flexible, efficient | Smart optimization | Only rewrites unoptimized files |
    | **Key Selection** | Based on query filters | Analyze `WHERE` clause patterns | Use frequently filtered columns |

??? note "Transaction Log & Checkpoints"

    | Concept | Details | Usage | Notes |
    |---------|---------|-------|-------|
    | **Transaction Log** | `_delta_log/` directory with JSON files | Records all table operations | Each commit = one JSON file |
    | **Checkpoint Files** | Parquet files every 10 commits | Snapshot of entire table state | Faster than reading many JSONs |
    | **File Statistics** | Min, max, nullCount per column | Captured per data file | Limited to first 32 columns |
    | **Log Retention** | Default: 30 days | Controls time travel window | Set via `delta.logRetentionDuration` |
    | **Statistics Columns** | `numRecords`, `minValues`, `maxValues` | Per-file metadata | Used for data skipping |
    | **Data Skipping** | Automatic file pruning | Query optimizer uses statistics | No configuration needed |
    | **Nested Fields** | Count toward 32-column limit | Struct with 8 fields = 8 columns | Order schema for important columns first |

??? note "Auto Optimize"

    | Feature | Configuration | Usage | Notes |
    |---------|--------------|-------|-------|
    | **Optimized Writes** | `delta.autoOptimize.optimizeWrite = true` | Write 128MB files per partition | Happens during write operation |
    | **Auto Compaction** | `delta.autoOptimize.autoCompact = true` | Compact after write completes | 128MB target (not 1GB like manual OPTIMIZE) |
    | **Table Properties** | Set on `CREATE TABLE` or `ALTER TABLE` | Enable per table | Can override per session |
    | **Session Config** | `spark.databricks.delta.optimizeWrite.enabled` | Enable for session | Takes precedence over table properties |
    | **Auto Tuning** | Adjusts file size for merge patterns | Smaller files for frequent merges | Reduces merge operation duration |
    | **Z-Order Limitation** | Auto compaction doesn't support Z-order | Use manual OPTIMIZE for Z-order | Z-ordering is more expensive |
    | **When to Use** | Streaming writes, frequent small batches | Reduces small file problem | Trade-off: slower writes for better reads |

??? note "Manual Optimization Commands"

    | Command | Syntax | Usage | Notes |
    |---------|--------|-------|-------|
    | **Basic OPTIMIZE** | `OPTIMIZE table` | Compact small files | Target: 1GB files |
    | **OPTIMIZE with Z-Order** | `OPTIMIZE table ZORDER BY (col1, col2)` | Compact + co-locate data | Computationally expensive |
    | **VACUUM** | `VACUUM table [RETAIN hours]` | Delete old data files | Default: 7 days retention |
    | **DESCRIBE HISTORY** | `DESCRIBE HISTORY table` | View transaction log | Shows commits, operations, versions |
    | **DESCRIBE DETAIL** | `DESCRIBE DETAIL table` | Table metadata | Location, format, partitioning |
    | **ANALYZE TABLE** | `ANALYZE TABLE table COMPUTE STATISTICS` | Update table statistics | Usually automatic in Delta Lake |

---

## Key Concepts

1. **Standard UDFs**: Row-by-row processing with high serialization overhead; avoid when possible
2. **Pandas UDFs**: Vectorized processing using Apache Arrow for fast data transfer; 10-100x faster than standard UDFs
3. **Built-in Functions**: Always prefer native Spark functions (`pyspark.sql.functions`) over UDFs
4. **Type Safety**: Pandas UDFs require explicit type hints and return type specifications
5. **Registration**: UDFs must be registered with `spark.udf.register()` to be used in SQL
6. **Performance**: Minimize data movement, use broadcast joins, filter early, and select only needed columns
7. **Partitioning**: Use for very large tables with low-cardinality partition columns; creates subdirectories
8. **Z-Order Indexing**: Co-locates data without subdirectories; non-incremental, requires full rerun
9. **Liquid Clustering**: Modern, incremental, flexible clustering; adapts to query patterns
10. **Transaction Log**: JSON files record operations; checkpoint every 10 commits for fast state resolution
11. **File Statistics**: Min/max/nullCount per column enables automatic data skipping
12. **Auto Optimize**: Optimized writes (128MB files) + auto compaction; reduces small file problem

---

## Official Documentation

### UDFs & Vectorization
- [Pandas UDFs (Vectorized UDFs)](https://spark.apache.org/docs/latest/api/python/user_guide/sql/arrow_pandas.html)
- [PySpark SQL Functions](https://spark.apache.org/docs/latest/api/python/reference/pyspark.sql/functions.html)
- [Apache Arrow and Spark](https://arrow.apache.org/docs/python/apache_spark.html)
- [Spark Performance Tuning Guide](https://spark.apache.org/docs/latest/sql-performance-tuning.html)
- [UDF Best Practices](https://docs.databricks.com/udf/python.html)

### Delta Lake Optimization
- [Delta Lake Optimize](https://docs.delta.io/latest/optimizations-oss.html)
- [Z-Order Clustering](https://docs.databricks.com/delta/data-skipping.html)
- [Liquid Clustering](https://docs.databricks.com/delta/clustering.html)
- [Auto Optimize](https://docs.databricks.com/delta/tune-file-size.html)
- [Delta Lake Transaction Log](https://docs.databricks.com/delta/delta-intro.html#transaction-log)
- [Predictive Optimization](https://docs.databricks.com/optimizations/predictive-optimization.html)
- [Table Partitioning](https://docs.databricks.com/tables/partitions.html)
