# Snowflake Observability Parameters

This reference covers the Snowflake parameters that control logging, metrics, and tracing for UDFs and stored procedures.

## Parameter Overview

| Parameter | Purpose | Type | Default |
|-----------|---------|------|---------|
| [EVENT_TABLE](#event_table) | Specifies the event table for logging messages | Object | None |
| [LOG_LEVEL](#log_level) | Controls which log messages are captured | Object | `OFF` |
| [METRIC_LEVEL](#metric_level) | Controls metrics collection | Object | `NONE` |
| [TRACE_LEVEL](#trace_level) | Controls trace/span collection | Object | `OFF` |
| [SQL_TRACE_QUERY_TEXT](#sql_trace_query_text) | Whether to capture SQL text in traces | Account | `OFF` |

---

## EVENT_TABLE

Specifies the name of the event table for logging messages from stored procedures and UDFs contained by the object with which the event table is associated.

> **Note**: Associating an event table with a database is available in Enterprise Edition or higher.

**Documentation**: [EVENT_TABLE Parameter](https://docs.snowflake.com/en/sql-reference/parameters#event-table)

### Values

Any existing event table created by executing the `CREATE EVENT TABLE` command.

**Default**: None

> **Note**: While not set as the default value for this parameter, every Snowflake account contains an event table at `SNOWFLAKE.TELEMETRY.EVENTS`. This table should be considered the de-facto default table.

### Can Be Set At

- Account
- Database

### Setting EVENT_TABLE

```sql
-- Account level (all objects in the account log to this event table)
ALTER ACCOUNT SET EVENT_TABLE = 'my_db.my_schema.my_event_table';

-- Database level (Enterprise Edition or higher)
ALTER DATABASE my_db SET EVENT_TABLE = 'my_db.my_schema.my_event_table';
```

### Checking Current Value

```sql
SHOW PARAMETERS LIKE 'EVENT_TABLE' IN ACCOUNT;
SHOW PARAMETERS LIKE 'EVENT_TABLE' IN DATABASE my_db;
```

### Required Privileges

| Scope | Required Privilege |
|-------|-------------------|
| Account | `ACCOUNTADMIN` role or `MODIFY` on account |
| Database | `MODIFY` on database |

### Prerequisites

If using a custom event table instead of the pre-existing `SNOWFLAKE.TELEMETRY.EVENTS` table, the custom event table must be created before setting the EVENT_TABLE parameter:

```sql
CREATE EVENT TABLE IF NOT EXISTS my_db.my_schema.my_event_table;
```

---

## LOG_LEVEL

Controls the severity level of log messages captured from handler code.

**Documentation**: [LOG_LEVEL Parameter](https://docs.snowflake.com/en/sql-reference/parameters#log-level)

### Values (Most to Least Verbose)

| Value | Captures | Use Case |
|-------|----------|----------|
| `TRACE` | All messages | Deep debugging |
| `DEBUG` | DEBUG, INFO, WARN, ERROR, FATAL | Development debugging |
| `INFO` | INFO, WARN, ERROR, FATAL | **Recommended for production** |
| `WARN` | WARN, ERROR, FATAL | Reduced noise |
| `ERROR` | ERROR, FATAL | Errors only |
| `FATAL` | FATAL only | Critical errors only |
| `OFF` | Nothing | Disabled (default) |

### Can Be Set At

- Account (default for all objects)
- Database
- Schema
- Stored Procedure
- Function (UDF/UDTF)
- Dynamic Table
- Iceberg Table
- Task
- Service

### Precedence

When set at multiple levels, the **more verbose** level wins.

Example: If account is `ERROR` but function is `DEBUG`, `DEBUG` is used.

### Setting LOG_LEVEL

```sql
-- Account level (recommended - applies to all objects)
ALTER ACCOUNT SET LOG_LEVEL = 'INFO';

-- Database level
ALTER DATABASE my_db SET LOG_LEVEL = 'INFO';

-- Schema level
ALTER SCHEMA my_db.my_schema SET LOG_LEVEL = 'INFO';

-- Function level
ALTER FUNCTION my_db.my_schema.my_udf(VARCHAR) SET LOG_LEVEL = 'DEBUG';

-- Stored procedure level
ALTER PROCEDURE my_db.my_schema.my_proc() SET LOG_LEVEL = 'DEBUG';
```

### Checking Current Value

```sql
SHOW PARAMETERS LIKE 'LOG_LEVEL' IN ACCOUNT;
SHOW PARAMETERS LIKE 'LOG_LEVEL' IN DATABASE my_db;
SHOW PARAMETERS LIKE 'LOG_LEVEL' IN FUNCTION my_db.my_schema.my_udf(VARCHAR);
```

### Required Privileges

| Scope | Required Privilege |
|-------|-------------------|
| Account | `MODIFY LOG LEVEL ON ACCOUNT` |
| Database | `MODIFY` on database |
| Schema | `MODIFY` on schema |
| Function/Procedure | `MODIFY` on object |

---

## METRIC_LEVEL

Controls whether metrics are collected from handler code execution.

**Documentation**: [METRIC_LEVEL Parameter](https://docs.snowflake.com/en/sql-reference/parameters#label-metric-level)

### Values

| Value | Description |
|-------|-------------|
| `ALL` | Collect all metrics |
| `NONE` | No metrics collected (default) |

### Can Be Set At

- Account
- Database
- Schema
- Stored Procedure
- Function (UDF/UDTF)

### Setting METRIC_LEVEL

```sql
-- Account level (recommended)
ALTER ACCOUNT SET METRIC_LEVEL = 'ALL';

-- Function level
ALTER FUNCTION my_db.my_schema.my_udf(VARCHAR) SET METRIC_LEVEL = 'ALL';
```

### Required Privileges

| Scope | Required Privilege |
|-------|-------------------|
| Account | `MODIFY METRIC LEVEL ON ACCOUNT` |
| Database/Schema/Object | `MODIFY` on object |

---

## TRACE_LEVEL

Controls whether trace events (spans) are captured from handler code.

> ⚠️ **Important**: When tracing events, you must also set the LOG_LEVEL parameter to one of its supported values. Traces will not be captured if LOG_LEVEL is `OFF`.

**Documentation**: [TRACE_LEVEL Parameter](https://docs.snowflake.com/en/sql-reference/parameters#trace-level)

### Values

| Value | Description |
|-------|-------------|
| `ALWAYS` | Always capture traces |
| `ON_EVENT` | Capture traces only when events are added |
| `OFF` | No tracing (default) |

### Can Be Set At

- Account
- Database
- Schema
- Stored Procedure
- Function (UDF/UDTF)

### Setting TRACE_LEVEL

```sql
-- Account level (recommended) - must also set LOG_LEVEL
ALTER ACCOUNT SET LOG_LEVEL = 'INFO';
ALTER ACCOUNT SET TRACE_LEVEL = 'ALWAYS';

-- Function level - must also set LOG_LEVEL
ALTER FUNCTION my_db.my_schema.my_udf(VARCHAR) SET LOG_LEVEL = 'INFO';
ALTER FUNCTION my_db.my_schema.my_udf(VARCHAR) SET TRACE_LEVEL = 'ALWAYS';
```

### Required Privileges

| Scope | Required Privilege |
|-------|-------------------|
| Account | `MODIFY TRACE LEVEL ON ACCOUNT` |
| Database/Schema/Object | `MODIFY` on object |

### What Traces Capture

When `TRACE_LEVEL = 'ALWAYS'`:
- Snowflake creates an auto-instrumented span for each execution
- Start and end timestamps
- Execution status
- Query context (query ID, database, schema)
- Custom span attributes and events (if added in code)

---

## SQL_TRACE_QUERY_TEXT

Controls whether the SQL text of traced statements is captured in the event table.

**Documentation**: [SQL_TRACE_QUERY_TEXT Parameter](https://docs.snowflake.com/en/sql-reference/parameters#sql-trace-query-text)

### Values

| Value | Description |
|-------|-------------|
| `ON` | Capture SQL text (up to 1024 characters) |
| `OFF` | Do not capture SQL text (default) |

### Can Be Set At

- Account only

### Setting SQL_TRACE_QUERY_TEXT

```sql
-- Requires ACCOUNTADMIN role
ALTER ACCOUNT SET SQL_TRACE_QUERY_TEXT = 'ON';
```

### When to Enable

Enable when you need to:
- See the exact SQL that triggered a trace
- Debug SQL executed within stored procedures
- Correlate traces with specific queries

**Caution**: Disable if SQL may contain sensitive information.

### Required Privileges

- `ACCOUNTADMIN` role required

---

## Recommended Configuration

### Account-Level Settings

These should be set once per account and left in place.

**Production accounts:**
```sql
ALTER ACCOUNT SET LOG_LEVEL = 'INFO';
ALTER ACCOUNT SET METRIC_LEVEL = 'ALL';
ALTER ACCOUNT SET TRACE_LEVEL = 'ALWAYS';
ALTER ACCOUNT SET SQL_TRACE_QUERY_TEXT = 'OFF'; -- To prevent possibly sensitive data in SQL queries from being logged
```

**Development accounts:**
```sql
ALTER ACCOUNT SET LOG_LEVEL = 'DEBUG';  -- Or 'INFO' to reduce noise in favor of frequently needing to raise the level at the session/object for debugging 
ALTER ACCOUNT SET METRIC_LEVEL = 'ALL';
ALTER ACCOUNT SET TRACE_LEVEL = 'ALWAYS';
ALTER ACCOUNT SET SQL_TRACE_QUERY_TEXT = 'ON';
```

### Session/Function-Level Overrides (For Debugging)

When additional verbosity is needed (e.g., debugging a specific UDF on a production account), override at the session or function level:

```sql
-- Option 1 (preferred): Session-level override — auto-expires when session ends
ALTER SESSION SET LOG_LEVEL = 'DEBUG';

-- Option 2: Function-level override — must be reverted when debugging is complete
ALTER FUNCTION my_db.my_schema.my_udf(VARCHAR) SET LOG_LEVEL = 'DEBUG';
-- After debugging:
-- ALTER FUNCTION my_db.my_schema.my_udf(VARCHAR) UNSET LOG_LEVEL;
```

These overrides take effect immediately and the more verbose level wins. Session-level is preferred because it requires no cleanup.

---

## Checking All Telemetry Parameters

```sql
-- Check account-level settings
SHOW PARAMETERS LIKE 'EVENT_TABLE' IN ACCOUNT;
SHOW PARAMETERS LIKE 'LOG_LEVEL' IN ACCOUNT;
SHOW PARAMETERS LIKE 'METRIC_LEVEL' IN ACCOUNT;
SHOW PARAMETERS LIKE 'TRACE_LEVEL' IN ACCOUNT;
SHOW PARAMETERS LIKE 'SQL_TRACE_QUERY_TEXT' IN ACCOUNT;

-- Check function-level settings
SHOW PARAMETERS LIKE 'LOG_LEVEL' IN FUNCTION my_db.my_schema.my_udf(VARCHAR);
SHOW PARAMETERS LIKE 'TRACE_LEVEL' IN FUNCTION my_db.my_schema.my_udf(VARCHAR);
```

---

## Troubleshooting

### No Logs Appearing

1. Check an event table is configured:
   ```sql
   SHOW PARAMETERS LIKE 'EVENT_TABLE' IN ACCOUNT;
   ```

2. Check LOG_LEVEL is not `OFF`:
   ```sql
   SHOW PARAMETERS LIKE 'LOG_LEVEL' IN ACCOUNT;
   SHOW PARAMETERS LIKE 'LOG_LEVEL' IN FUNCTION <fn>;
   ```

3. Verify the UDF has logging statements in code

4. Check event table has data:
   ```sql
   SELECT COUNT(*) FROM SNOWFLAKE.TELEMETRY.EVENTS
   WHERE TIMESTAMP > DATEADD('hour', -1, CURRENT_TIMESTAMP());
   ```

### No Traces Appearing

1. Check TRACE_LEVEL is `ALWAYS` or `ON_EVENT`:
   ```sql
   SHOW PARAMETERS LIKE 'TRACE_LEVEL' IN ACCOUNT;
   ```

2. **Check LOG_LEVEL is not `OFF`** - traces require LOG_LEVEL to be set:
   ```sql
   SHOW PARAMETERS LIKE 'LOG_LEVEL' IN ACCOUNT;
   ```

3. Verify the UDF includes `snowflake-telemetry-python` package and import

4. Query for SPAN records:
   ```sql
   SELECT * FROM SNOWFLAKE.TELEMETRY.EVENTS
   WHERE RECORD_TYPE = 'SPAN'
     AND TIMESTAMP > DATEADD('hour', -1, CURRENT_TIMESTAMP());
   ```

### Insufficient Privileges

If you get permission errors:
```sql
-- Check your current role
SELECT CURRENT_ROLE();

-- Request account-level privileges from admin
-- Or use function-level settings which require only MODIFY on the function
```

## References

- [Parameters | SQL Reference | Snowflake Documentation](https://docs.snowflake.com/en/sql-reference/parameter)
