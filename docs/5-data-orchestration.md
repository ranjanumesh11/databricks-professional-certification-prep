# Section 5 — Data Orchestration

Quick-reference tables for multi-task jobs, task dependencies, and workflow orchestration.

---

## Databricks Jobs & Workflows

??? note "Multi-Task Job Configuration"

    | Component | Configuration | Usage | Notes |
    |-----------|--------------|-------|-------|
    | **Task Types** | Notebook, Python script, JAR, SQL, dbt | Different execution types | Mix in single job |
    | **Task Dependencies** | `depends_on: [task1, task2]` | Control execution order | Tasks run in parallel when possible |
    | **Job Parameters** | `base_parameters` at job level | Pass to all tasks | Task parameters override job parameters |
    | **Task Parameters** | Individual task parameters | Override job defaults | Access via `dbutils.widgets.get()` |
    | **Cluster Configuration** | Per-task or shared cluster | Resource allocation | Shared clusters reduce startup time |
    | **Retry Policy** | `max_retries: 3`, `timeout_seconds: 3600` | Handle failures | Set per task |

??? note "Task Orchestration Patterns"

    | Pattern | Configuration | Usage | Notes |
    |---------|--------------|-------|-------|
    | **Sequential Tasks** | Linear `depends_on` chain | ETL pipeline stages | Task2 depends on Task1, Task3 on Task2, etc. |
    | **Parallel Tasks** | No dependencies between tasks | Independent processing | Execute simultaneously |
    | **Fan-out/Fan-in** | Multiple tasks depend on one, then converge | Parallel processing + aggregation | Common in data pipelines |
    | **Conditional Execution** | Use `if-else` task type | Dynamic workflows | Skip tasks based on conditions |
    | **Trigger Types** | Scheduled, file arrival, manual | When to run | Use file arrival for event-driven pipelines |

??? note "Parameter Passing & Widgets"

    | Operation | Code/Command | Usage | Notes |
    |-----------|--------------|-------|-------|
    | **Create Widget** | `dbutils.widgets.text("name", "default")` | Define input parameter | Available in notebook UI |
    | **Get Widget Value** | `dbutils.widgets.get("name")` | Access parameter value | Returns string |
    | **Remove Widget** | `dbutils.widgets.remove("name")` | Clean up | Good practice at end of notebook |
    | **Pass to Next Task** | `dbutils.notebook.exit(json.dumps(result))` | Return value from notebook | Use `taskValues` in downstream tasks |
    | **Access Task Value** | `dbutils.jobs.taskValues.get("task_name", "key")` | Get value from previous task | Only in multi-task jobs |

---

## Workflow Monitoring & Control

??? note "Job Monitoring"

    | Operation | API/UI | Usage | Notes |
    |-----------|--------|-------|-------|
    | **Job Runs** | Jobs UI > Run History | View execution history | Filter by status, date |
    | **Task Output** | Task details > Output | See returned values | From `dbutils.notebook.exit()` |
    | **Logs** | Task > View Logs | Troubleshoot failures | Stdout, stderr, driver logs |
    | **Metrics** | Task > Metrics | Performance monitoring | Duration, data processed |
    | **Email Alerts** | Job settings > Alerts | Failure notifications | Can configure per task or job |
    | **Webhooks** | Job settings > Webhooks | External integrations | Trigger external systems |

??? note "Job Control & Management"

    | Operation | Code/Command | Usage | Notes |
    |-----------|--------------|-------|-------|
    | **Run Now** | UI or `runs submit` API | Manual trigger | Can override parameters |
    | **Cancel Run** | UI or `runs cancel` API | Stop execution | Terminates all running tasks |
    | **Repair Run** | UI > Repair | Re-run failed tasks | Preserves successful task results |
    | **Jobs API** | `databricks jobs create/update` | Programmatic management | Use for CI/CD |
    | **dbutils.notebook.run** | `dbutils.notebook.run("path", timeout, params)` | Run nested notebook | Synchronous execution |
    | **%run** | `%run ./helper_notebook` | Include notebook | Shares namespace with calling notebook |

---

## Scheduling & Triggers

??? note "Job Triggers"

    | Trigger Type | Configuration | Usage | Notes |
    |-------------|--------------|-------|-------|
    | **Scheduled** | Cron expression or simple schedule | Time-based execution | Supports timezone configuration |
    | **File Arrival** | Cloud storage path monitoring | Event-driven workflows | Use with Auto Loader |
    | **Manual** | No schedule | On-demand execution | Via UI or API |
    | **Continuous** | Continuous execution | Real-time processing | Use with streaming jobs |
    | **Triggered** | Via API call | External orchestration | Integrate with Airflow, etc. |

---

## Key Concepts

1. **Multi-Task Jobs**: Orchestrate complex workflows with task dependencies and parallel execution
2. **Task Dependencies**: Define execution order; Databricks automatically parallelizes independent tasks
3. **Parameter Passing**: Use widgets for inputs and `taskValues` to pass data between tasks
4. **Cluster Strategies**: Shared clusters for faster startup vs. isolated clusters for resource control
5. **Error Handling**: Configure retries, timeouts, and alerts for robust pipelines
6. **Job Repair**: Re-run only failed tasks without re-executing successful ones
7. **Integration**: Use Jobs API for CI/CD and external orchestration tools

---

## Official Documentation

- [Databricks Jobs and Workflows](https://docs.databricks.com/workflows/jobs/jobs.html)
- [Task Orchestration](https://docs.databricks.com/workflows/jobs/create-run-jobs.html)
- [Jobs API Reference](https://docs.databricks.com/api/workspace/jobs)
- [dbutils.widgets](https://docs.databricks.com/dev-tools/databricks-utils.html#widgets-utility-dbutilswidgets)
- [Databricks Notebooks](https://docs.databricks.com/notebooks/index.html)
