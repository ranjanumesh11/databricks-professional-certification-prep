# Section 8 — Testing & Deployment

Quick-reference tables for code organization, testing, CI/CD, and deployment patterns.

---

## Code Organization & Imports

??? note "Notebook Imports"

    | Method | Code/Command | Usage | Notes |
    |--------|--------------|-------|-------|
    | **%run** | `%run ./path/to/notebook` | Include notebook | Shares namespace; relative or absolute path |
    | **%run Parent Directory** | `%run ../Includes/helper` | Run notebook up one level | Use `..` for parent directory |
    | **Python Import** | `from module import Class` | Import Python modules | Requires module in Python path |
    | **Sys Path Append** | `sys.path.append(os.path.abspath('../modules'))` | Add module directory | Makes modules importable |
    | **%pip install** | `%pip install package` or `%pip install ../path/to/wheel.whl` | Install package | Cluster-scoped install |

??? note "Module Structure"

    | Pattern | Structure | Usage | Notes |
    |---------|-----------|-------|-------|
    | **Package** | `module/__init__.py` | Define Python package | Empty or with imports |
    | **Subpackage** | `module/subpkg/__init__.py` | Nested packages | Organize related modules |
    | **Relative Imports** | `from .subpkg import Class` | Within package | Use dot notation |
    | **Absolute Imports** | `from module.subpkg import Class` | From anywhere | After adding to sys.path |
    | **Wheel Distribution** | `.whl` file | Distribute code | Build with `setup.py` |
    | **Workspace Files** | Repos or workspace files | Shared code | Use %run or add to path |

---

## Databricks CLI & Secrets

??? note "Databricks CLI Commands"

    | Command | Usage | Purpose | Notes |
    |---------|-------|---------|-------|
    | **configure** | `databricks configure --token` | Set up authentication | Creates `.databrickscfg` |
    | **workspace** | `databricks workspace ls /path` | List workspace items | View notebooks, folders |
    | **fs** | `databricks fs ls dbfs:/path` | List DBFS files | Browse file storage |
    | **jobs** | `databricks jobs list` | List jobs | View job configurations |
    | **clusters** | `databricks clusters list` | List clusters | Get cluster IDs |
    | **secrets** | `databricks secrets list-scopes` | List secret scopes | View available scopes |

??? note "Secrets Management"

    | Operation | Code/Command | Usage | Notes |
    |-----------|--------------|-------|-------|
    | **List Scopes** | `databricks secrets list-scopes` | View all scopes | CLI command |
    | **Create Scope** | `databricks secrets create-scope --scope <name>` | Create new scope | Backend: Azure Key Vault or Databricks-managed |
    | **Put Secret** | `databricks secrets put --scope <scope> --key <key>` | Add secret | Opens editor for value |
    | **Get Secret** | `dbutils.secrets.get("<scope>", "<key>")` | Retrieve secret in code | Returns string |
    | **List Secrets** | `databricks secrets list --scope <scope>` | View keys in scope | CLI command |
    | **ACL** | `databricks secrets put-acl --scope <scope> --principal <user>` | Control access | MANAGE, WRITE, READ |

---

## Testing Patterns

??? note "Unit Testing Approaches"

    | Pattern | Code/Implementation | Usage | Notes |
    |---------|---------------------|-------|-------|
    | **Assert** | `assert actual == expected, "Error message"` | Simple validation | Inline in notebooks |
    | **Count Validation** | `assert df.count() == expected_count` | Check row count | Common after transformations |
    | **Distinct Validation** | `assert df.select("id").distinct().count() == total` | Check uniqueness | Validate primary keys |
    | **Schema Validation** | `assert df.schema == expected_schema` | Check structure | Validate column names/types |
    | **Data Quality** | `assert df.filter("col IS NULL").count() == 0` | Check constraints | Validate business rules |
    | **pytest** | `def test_function(): assert ...` | Formal testing | Use with CI/CD |

??? note "Integration Testing"

    | Pattern | Approach | Usage | Notes |
    |---------|----------|-------|-------|
    | **Test Data** | Small synthetic dataset | Validate pipeline end-to-end | Store in DBFS or repo |
    | **Temporary Tables** | Create test tables in temp database | Isolated testing | Clean up after tests |
    | **Widget Parameters** | Pass test mode via widget | Switch between test/prod | `dbutils.widgets.get("mode")` |
    | **DLT Test Pipeline** | Separate development pipeline | Test DLT changes | Use development mode |
    | **Job Test Run** | Trigger job with test parameters | Validate orchestration | Use separate test database |

---

## CI/CD & Deployment

??? note "Deployment Strategies"

    | Strategy | Approach | Usage | Notes |
    |----------|----------|-------|-------|
    | **Repos Integration** | Git sync via Databricks Repos | Automatic sync with Git | Best for notebooks |
    | **Databricks CLI** | Script deployment via CLI | Automate with scripts | Good for jobs/clusters |
    | **Databricks Asset Bundles** | `databricks bundle deploy` | Modern deployment tool | Recommended for new projects |
    | **Terraform** | Infrastructure as code | Manage all resources | Enterprise standard |
    | **Azure DevOps / GitHub Actions** | CI/CD pipeline integration | Automated testing & deployment | Use with CLI or bundles |
    | **Workspace API** | REST API for resource management | Programmatic control | Most flexible |

??? note "Environment Management"

    | Pattern | Implementation | Usage | Notes |
    |---------|---------------|-------|-------|
    | **Separate Workspaces** | Dev, staging, prod workspaces | Complete isolation | Enterprise pattern |
    | **Separate Databases** | `dev_db`, `staging_db`, `prod_db` | Logical separation | Within same workspace |
    | **Environment Variables** | Cluster-level env vars or secrets | Configuration per environment | Use secrets for sensitive data |
    | **Widget Defaults** | Different defaults per environment | Runtime configuration | Set in job parameters |
    | **Conditional Logic** | `if env == "prod":` | Environment-specific code | Use sparingly |

---

## Job Parameterization

??? note "Parameter Patterns"

    | Pattern | Implementation | Usage | Notes |
    |---------|---------------|-------|-------|
    | **Widgets** | `dbutils.widgets.text("param", "default")` | Notebook inputs | Set in job or notebook UI |
    | **Job Parameters** | Set in job configuration | Apply to all tasks | Override task parameters |
    | **Task Parameters** | Set per task in job | Task-specific values | Override job parameters |
    | **Environment Detection** | `spark.conf.get("environment")` | Get current environment | Set in cluster config |
    | **Config Files** | JSON/YAML in DBFS or repos | External configuration | Load and parse in code |
    | **Unity Catalog Volumes** | Store config in volumes | Centralized config management | New recommended approach |

---

## Monitoring & Logging

??? note "Logging Best Practices"

    | Practice | Code/Implementation | Usage | Notes |
    |----------|---------------------|-------|-------|
    | **Print Statements** | `print(f"Processing {count} records")` | Simple logging | Shows in driver logs |
    | **Python Logging** | `import logging; logging.info("message")` | Structured logging | Better for production |
    | **Log Tables** | Write logs to Delta table | Persistent audit trail | Query with SQL |
    | **DLT Event Log** | System table `_event_log` | DLT-specific logging | Automatic for DLT pipelines |
    | **Job Run Output** | `dbutils.notebook.exit(json.dumps(result))` | Return metrics from tasks | Display in job run details |
    | **Custom Metrics** | Write metrics to table | Track KPIs over time | Dashboard with SQL |

??? note "Error Handling"

    | Pattern | Code | Usage | Notes |
    |---------|------|-------|-------|
    | **Try/Except** | `try: ... except Exception as e: ...` | Catch errors | Log and handle gracefully |
    | **Validation Before Processing** | Check conditions before execution | Fail fast | Better error messages |
    | **Dead Letter Queue** | Write failed records to separate table | Reprocess later | Don't lose data |
    | **Email Alerts** | Configure in job settings | Notify on failure | Include error details |
    | **Retry Logic** | Job-level retry configuration | Handle transient failures | Set max retries |
    | **Circuit Breaker** | Stop pipeline if too many errors | Prevent cascading failures | E.g., DLT expect_or_fail |

---

## Key Concepts

1. **Code Organization**: Use modules and packages for reusable code; %run for notebook includes
2. **Module Imports**: Add directories to `sys.path` or use `%pip install` for wheels
3. **Secrets Management**: Never hardcode credentials; use Databricks secrets with scopes
4. **Unit Testing**: Validate transformations with assertions; use pytest for formal testing
5. **CI/CD**: Use Databricks Repos + GitHub Actions or Azure DevOps for automated deployment
6. **Environment Separation**: Use separate workspaces or databases for dev/staging/prod
7. **Parameterization**: Use widgets and job parameters for configurable pipelines
8. **Logging**: Write to Delta tables for persistent audit trails; use structured logging
9. **Error Handling**: Fail fast with validation, catch exceptions, and alert on failures
10. **Deployment**: Databricks Asset Bundles or Terraform for infrastructure as code
