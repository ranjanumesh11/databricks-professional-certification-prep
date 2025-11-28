# Section 6 — Data Governance

Quick-reference tables for data privacy, access control, and regulatory compliance.

---

## Delta Lake Time Travel & Versioning

??? note "Time Travel Operations"

    | Operation | SQL/Code | Usage | Notes |
    |-----------|----------|-------|-------|
    | **Query Version** | `SELECT * FROM table@v10` | Read specific version | Use for auditing |
    | **Query Timestamp** | `SELECT * FROM table TIMESTAMP AS OF '2023-01-01'` | Read at point in time | More user-friendly than version |
    | **Describe History** | `DESCRIBE HISTORY table` | View version history | Shows operations, timestamps, versions |
    | **Restore Version** | `RESTORE TABLE table TO VERSION AS OF 10` | Rollback to version | Useful for recovering from bad updates |
    | **Restore Timestamp** | `RESTORE TABLE table TO TIMESTAMP AS OF '2023-01-01'` | Rollback to time | Alternative to version-based restore |
    | **Version Comparison** | `SELECT * FROM table@v1 EXCEPT SELECT * FROM table@v2` | Find differences | Use for auditing changes |

??? note "Change Data Feed (CDF)"

    | Operation | Code/Command | Usage | Notes |
    |-----------|--------------|-------|-------|
    | **Enable CDF** | `ALTER TABLE t SET TBLPROPERTIES (delta.enableChangeDataFeed = true)` | Track changes | Must enable before capturing changes |
    | **Query Changes** | `SELECT * FROM table_changes('table', start_version)` | Get change records | Returns _change_type column |
    | **Stream CDF** | `.option("readChangeFeed", "true").option("startingVersion", n)` | Stream changes | Use with readStream |
    | **Change Types** | `_change_type IN ('insert', 'update_preimage', 'update_postimage', 'delete')` | Filter operations | Understand change semantics |
    | **Version Range** | `table_changes('table', start_version, end_version)` | Changes between versions | Useful for incremental processing |

---

## Data Privacy & Security

??? note "Dynamic Views & Row-Level Security"

    | Pattern | SQL Code | Usage | Notes |
    |---------|----------|-------|-------|
    | **Redaction** | `CASE WHEN is_member('admins') THEN email ELSE 'REDACTED' END` | Hide sensitive data | Based on user group membership |
    | **Role-Based Views** | `CREATE VIEW pii_view AS SELECT * WHERE is_member('pii_access')` | Restrict row access | Grant on view, not base table |
    | **Conditional Filtering** | `WHERE CASE WHEN is_member('admins') THEN TRUE ELSE country = 'US' END` | Dynamic row filtering | Different users see different rows |
    | **Column Masking** | Multiple CASE statements per column | Mask multiple columns | Use for PII fields |
    | **Audit Logging** | Track view access via query history | Compliance requirement | Monitor sensitive data access |

??? note "Access Control Functions"

    | Function | SQL | Usage | Notes |
    |----------|-----|-------|-------|
    | **is_member()** | `is_member('group_name')` | Check group membership | Returns boolean |
    | **current_user()** | `current_user()` | Get current username | Use for user-specific filtering |
    | **session_user()** | `session_user()` | Get session user | Similar to current_user() |

---

## Propagating Deletes

??? note "Delete Propagation Pattern"

    | Step | Code/Command | Usage | Notes |
    |------|--------------|-------|-------|
    | **Capture Delete Requests** | Stream delete events to `delete_requests` table | Track deletion timeline | Include deadline and status |
    | **Process Deletes** | `DELETE FROM table WHERE id IN (SELECT id FROM delete_requests)` | Execute deletion | Usually after grace period |
    | **Enable CDF on Source** | `ALTER TABLE source SET TBLPROPERTIES (delta.enableChangeDataFeed = true)` | Track source deletes | Required for downstream propagation |
    | **Stream CDF Deletes** | `.option("readChangeFeed", "true").filter("_change_type = 'delete'")` | Capture delete operations | Read from source CDF |
    | **Cascade to Related Tables** | `DELETE FROM child WHERE parent_id IN (SELECT id FROM deletes_view)` | Referential integrity | Use foreachBatch pattern |
    | **Update Delete Status** | `MERGE INTO delete_requests ... WHEN MATCHED THEN UPDATE SET status = 'deleted'` | Track completion | Audit trail |

??? note "Delete Request Management"

    | Operation | Code/Pattern | Usage | Notes |
    |-----------|--------------|-------|-------|
    | **Add Deadline** | `date_add(request_timestamp, 30)` | Compliance grace period | E.g., 30-day GDPR requirement |
    | **Status Tracking** | Status: 'requested', 'deleted', 'expired' | Lifecycle management | Track delete progression |
    | **Verification** | Compare versions with EXCEPT | Confirm deletions | `table@v_before EXCEPT table@v_after` |
    | **Batch Delete Stream** | `foreachBatch(process_deletes)` | Custom deletion logic | Access full SQL in micro-batch |

---

## Compliance & Auditing

??? note "Compliance Patterns"

    | Pattern | Implementation | Usage | Notes |
    |---------|---------------|-------|-------|
    | **GDPR Right to Erasure** | Delete propagation with CDF | 30-day deletion requirement | Track request and completion dates |
    | **Data Retention** | Time travel + vacuum policy | Keep versions for audit period | Balance storage cost vs. retention needs |
    | **Audit Trail** | DESCRIBE HISTORY + CDF | Track all data changes | Immutable change log |
    | **Access Logs** | Query history + dynamic views | Monitor PII access | Compliance reporting |
    | **Data Lineage** | Unity Catalog lineage | Track data flow | Understand upstream/downstream impact |
    | **Column-Level Security** | Dynamic views with masking | Restrict column access | Different views for different roles |

??? note "Table Properties for Governance"

    | Property | Configuration | Usage | Notes |
    |----------|--------------|-------|-------|
    | **Change Data Feed** | `delta.enableChangeDataFeed = true` | Track all changes | Required for CDC patterns |
    | **Data Retention** | `delta.deletedFileRetentionDuration = "interval 30 days"` | Time travel window | Default is 7 days |
    | **Log Retention** | `delta.logRetentionDuration = "interval 30 days"` | Transaction log history | Separate from data retention |
    | **Column Mapping** | `delta.columnMapping.mode = 'name'` | Support column renames | Enable before renaming |

---

## Key Concepts

1. **Dynamic Views**: Use `is_member()` to create role-based data access without duplicating tables
2. **Delete Propagation**: Capture deletes via CDF and cascade to related tables using foreachBatch
3. **Time Travel**: Query historical versions for auditing and rollback; configure retention periods
4. **Change Data Feed**: Track all table changes (insert/update/delete) for compliance and downstream propagation
5. **Row-Level Security**: Filter rows dynamically based on user attributes and group membership
6. **Compliance Windows**: Use deadline columns to track regulatory timelines (e.g., 30-day GDPR deletion)
7. **Audit Trail**: Leverage DESCRIBE HISTORY and CDF for immutable change logs
