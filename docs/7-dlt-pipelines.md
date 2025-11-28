# Section 7 — Delta Live Tables (DLT)

Quick-reference tables for DLT pipelines, data quality, and incremental processing.

---

## DLT Table Definitions

??? note "Table Decorators & Types"

    | Decorator | Code/Usage | Purpose | Notes |
    |-----------|------------|---------|-------|
    | **@dp.table** | `@dp.table def table_name()` | Define DLT table | Creates managed Delta table |
    | **@dp.temporary_view** | `@dp.temporary_view def view_name()` | Define temporary view | Not persisted to storage |
    | **@dp.materialized_view** | `@dp.materialized_view def view_name()` | Define materialized view | Batch-only, fully refreshed |
    | **Streaming Table** | `dp.create_streaming_table("name")` | Explicitly create streaming table | For use with apply_changes |
    | **Named Table** | `@dp.table(name="custom_name")` | Specify table name | Override function name |

??? note "Table Properties"

    | Property | Configuration | Usage | Notes |
    |----------|--------------|-------|-------|
    | **Partitioning** | `@dp.table(partition_cols=["year", "month"])` | Optimize queries | Common for time-series data |
    | **Append-Only** | `table_properties={"delta.appendOnly": "true"}` | Prevent updates/deletes | Faster writes, better for logs |
    | **Reset Protection** | `table_properties={"pipelines.reset.allowed": "false"}` | Prevent data loss | Production safety |
    | **Quality Tag** | `@dp.table(table_properties={"quality": "silver"})` | Metadata tagging | Organize data quality tiers |
    | **Comment** | `@dp.table(comment="Description")` | Documentation | Appears in table metadata |

---

## Data Quality & Expectations

??? note "Expectation Types"

    | Expectation | Code | Behavior | Usage |
    |-------------|------|----------|-------|
    | **expect** | `@dp.expect("rule_name", "constraint")` | Record violation, continue | Track quality metrics without blocking |
    | **expect_or_drop** | `@dp.expect_or_drop("rule_name", "constraint")` | Drop invalid records | Filter out bad data |
    | **expect_or_fail** | `@dp.expect_or_fail("rule_name", "constraint")` | Stop pipeline on violation | Critical data quality requirements |
    | **expect_all** | `@dp.expect_all(rules_dict)` | Apply multiple rules | Pass dictionary of rule_name: constraint |
    | **expect_all_or_drop** | `@dp.expect_all_or_drop(rules_dict)` | Drop if any rule fails | Strict multi-rule filtering |
    | **expect_all_or_fail** | `@dp.expect_all_or_fail(rules_dict)` | Stop if any rule fails | Critical multi-rule validation |

??? note "Quarantine Pattern"

    | Step | Code | Usage | Notes |
    |------|------|-------|-------|
    | **Define Rules** | `rules = {"valid_price": "price > 0", "valid_id": "id IS NOT NULL"}` | Validation criteria | Dictionary of constraints |
    | **Valid Records** | `@dp.expect_all(rules) def silver_table()` | Pass all rules | Main data flow |
    | **Quarantine Table** | `@dp.expect_all_or_drop({"quarantine": "NOT(...)"}) def quarantine()` | Inverse logic | Capture failed records |
    | **Shared Source** | Both tables use same source function | Reuse transformation | Define once, apply twice |
    | **is_quarantined Column** | `.withColumn("is_quarantined", F.expr(quarantine_rules))` | Flag violations | Include in quarantine table |

---

## Auto Loader (Cloud Files)

??? note "Auto Loader Configuration"

    | Option | Code/Value | Usage | Notes |
    |--------|------------|-------|-------|
    | **Format** | `.format("cloudFiles")` | Enable Auto Loader | For incremental file ingestion |
    | **File Format** | `.option("cloudFiles.format", "json")` | Source file type | json, csv, parquet, etc. |
    | **Schema Inference** | `.option("cloudFiles.schemaLocation", path)` | Track schema evolution | Stores inferred schemas |
    | **Explicit Schema** | `.schema(schema_string)` | Provide schema | More efficient than inference |
    | **Schema Evolution** | `.option("cloudFiles.schemaHints", "col STRING")` | Guide inference | Override specific columns |
    | **Path Globbing** | `.load(f"{path}/year=*/month=*")` | Filter files by pattern | Partition filtering |

---

## Change Data Capture (CDC) in DLT

??? note "apply_changes Function"

    | Parameter | Configuration | Usage | Notes |
    |-----------|--------------|-------|-------|
    | **target** | `target="silver_table"` | Destination table name | Must be streaming table |
    | **source** | `source="bronze_cdc_view"` | Source view/table | Contains CDC records |
    | **keys** | `keys=["customer_id"]` | Primary key columns | Identifies unique records |
    | **sequence_by** | `sequence_by=F.col("timestamp")` | Ordering column | Determines which update is latest |
    | **except_column_list** | `except_column_list=["row_status"]` | Exclude columns | Don't write metadata columns |
    | **apply_as_deletes** | `apply_as_deletes=F.expr("row_status = 'delete'")` | Delete condition | When to remove records |
    | **stored_as_scd_type** | `stored_as_scd_type=2` | SCD Type 2 tracking | Keep history (default is Type 1) |

??? note "CDC Patterns"

    | Pattern | Code | Usage | Notes |
    |---------|------|-------|-------|
    | **Type 1 (Latest)** | Default `apply_changes` | Overwrite with latest | No history |
    | **Type 2 (History)** | `stored_as_scd_type=2` | Track changes over time | Adds `__START_AT`, `__END_AT` columns |
    | **Soft Deletes** | `apply_as_deletes=F.expr("row_status = 'delete'")` | Remove deleted records | Based on CDC flag |
    | **Filter Before CDC** | `.filter(F.col("row_status") != 'delete')` in source | Exclude deletes | If deletes not needed |

---

## Pipeline Configuration

??? note "Development vs Production"

    | Mode | Configuration | Usage | Notes |
    |------|--------------|-------|-------|
    | **Development** | Development mode in UI | Fast iteration | Schema inference, no optimization |
    | **Production** | Production mode | Performance optimized | Required for continuous updates |
    | **Triggered** | Triggered pipeline | Run once | Process available data then stop |
    | **Continuous** | Continuous pipeline | Always running | Low-latency streaming |
    | **Target Database** | Specify in pipeline settings | Output location | All tables created here |
    | **Storage Location** | Custom or default | Checkpoint and data storage | Separate from metastore |

??? note "DLT API & Control"

    | Operation | API/CLI | Usage | Notes |
    |-----------|---------|-------|-------|
    | **Create Pipeline** | `databricks pipelines create` | Define new pipeline | JSON configuration |
    | **Update Pipeline** | `databricks pipelines update` | Modify existing pipeline | Change libraries, settings |
    | **Start Pipeline** | `databricks pipelines start` | Trigger execution | Manual or scheduled |
    | **Stop Pipeline** | `databricks pipelines stop` | Halt continuous pipeline | Graceful shutdown |
    | **Reset Pipeline** | Reset in UI | Clear checkpoints & data | Start fresh |
    | **Get Status** | `databricks pipelines get` | Check pipeline state | Running, stopped, failed |

---

## Monitoring & Troubleshooting

??? note "Pipeline Observability"

    | Feature | Location | Usage | Notes |
    |---------|----------|-------|-------|
    | **Data Flow DAG** | DLT UI > Graph | Visualize dependencies | See table relationships |
    | **Data Quality Metrics** | DLT UI > Data Quality | View expectation results | Pass/fail counts per rule |
    | **Event Log** | System table: `<pipeline_name>_event_log` | Detailed pipeline events | Query for analysis |
    | **Lineage** | DLT UI > Lineage | Track data flow | Upstream/downstream tables |
    | **Update History** | DLT UI > Updates | Past execution results | Duration, records processed |

---

## Key Concepts

1. **Declarative ETL**: Define desired end state; DLT manages execution order and dependencies
2. **Data Quality**: Expectations track, filter, or block bad data with clear semantics
3. **Auto Loader**: Incrementally ingest files from cloud storage with schema evolution support
4. **apply_changes**: Automatic CDC processing with Type 1 or Type 2 SCD support
5. **Streaming Tables**: Append-only incremental processing; use for real-time pipelines
6. **Materialized Views**: Batch-processed, fully refreshed; use for aggregations
7. **Quarantine Pattern**: Separate tables for valid and invalid records using inverse expectations
8. **Development Mode**: Fast iteration with schema inference; switch to Production for performance
