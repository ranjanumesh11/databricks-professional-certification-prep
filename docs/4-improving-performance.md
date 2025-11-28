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

## Key Concepts

1. **Standard UDFs**: Row-by-row processing with high serialization overhead; avoid when possible
2. **Pandas UDFs**: Vectorized processing using Apache Arrow for fast data transfer; 10-100x faster than standard UDFs
3. **Built-in Functions**: Always prefer native Spark functions (`pyspark.sql.functions`) over UDFs
4. **Type Safety**: Pandas UDFs require explicit type hints and return type specifications
5. **Registration**: UDFs must be registered with `spark.udf.register()` to be used in SQL
6. **Performance**: Minimize data movement, use broadcast joins, filter early, and select only needed columns
