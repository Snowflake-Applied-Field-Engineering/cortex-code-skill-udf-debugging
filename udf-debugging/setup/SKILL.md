---
name: udf-debugging-setup
description: "Set up event table and telemetry levels for UDF debugging"
parent_skill: udf-debugging
---

# Event Table Setup for UDF Debugging

This workflow helps detect existing event table configuration, set up telemetry collection if needed, and ensure UDFs are properly configured for logging.

## When to Load

Main skill routes here when user wants to:
- Enable event table logging
- Configure telemetry levels
- Check if logging is set up correctly
- Troubleshoot missing logs

---

## Workflow

### Step 1: Detect Current Event Table Configuration

**Goal:** Understand what event table configuration exists

**Actions:**

1. **Check account-level event table**:
   ```sql
   SHOW PARAMETERS LIKE 'EVENT_TABLE' IN ACCOUNT;
   ```

2. **Explain to user**:
   > All Snowflake accounts include `SNOWFLAKE.TELEMETRY.EVENTS` by default.
   > This is the recommended event table for most use cases.

**Present findings**:
- Account event table: `<value or "Not configured">`
- Effective event table for target UDF: `<determined value>`

---

### Step 2: Check Telemetry Levels

**Goal:** Verify that telemetry collection is enabled at appropriate levels

**Load reference:** [../references/observability-parameters.md](../references/observability-parameters.md)

**Actions:**

1. **Check account-level telemetry settings**:
   ```sql
   SHOW PARAMETERS LIKE 'LOG_LEVEL' IN ACCOUNT;
   SHOW PARAMETERS LIKE 'METRIC_LEVEL' IN ACCOUNT;
   SHOW PARAMETERS LIKE 'TRACE_LEVEL' IN ACCOUNT;
   ```

2. **If user has a target UDF, check its specific settings**:
   ```sql
   -- For a function (include parameter types)
   SHOW PARAMETERS LIKE 'LOG_LEVEL' IN FUNCTION <db.schema.fn_name>(<param_types>);
   SHOW PARAMETERS LIKE 'TRACE_LEVEL' IN FUNCTION <db.schema.fn_name>(<param_types>);
   ```

3. **Summarize effective levels**:

   | Level Type | Account | Object (if set) | Effective |
   |------------|---------|-----------------|-----------|
   | LOG_LEVEL | `<val>` | `<val>` | `<most verbose>` |
   | METRIC_LEVEL | `<val>` | `<val>` | `<effective>` |
   | TRACE_LEVEL | `<val>` | `<val>` | `<effective>` |

**Understanding LOG_LEVEL values** (most to least verbose):
- `TRACE` - All messages
- `DEBUG` - Debug and above
- `INFO` - Info and above (recommended default)
- `WARN` - Warnings and errors only
- `ERROR` - Errors only
- `FATAL` - Fatal errors only
- `OFF` - No logging

**Understanding TRACE_LEVEL values:**
- `ALWAYS` - Always capture traces (recommended)
- `ON_EVENT` - Capture only when events are added
- `OFF` - No tracing (default)

---

### Step 3: Determine Setup Needs and Present Options

**Goal:** Decide what configuration changes are needed and present options to user

Based on Steps 1-2, determine the situation:

| Situation | Action |
|-----------|--------|
| No event table configured | Go to Step 4A |
| Event table exists but telemetry levels disabled at account | Go to Step 4B (present options) |
| Account levels enabled but need more verbose level for debugging | Go to Step 4C |
| Everything configured correctly | Go to Step 5 (verification) |

---

### Step 4A: Enable Account-Level Event Collection

**Goal:** Set up telemetry collection using the default event table

**Recommendation:**
> The `SNOWFLAKE.TELEMETRY.EVENTS` event table exists by default in all accounts.
> For most use cases, enabling account-level telemetry with this table is the simplest approach.

**Proposed SQL:**
```sql
-- Enable logging at INFO level (captures INFO, WARN, ERROR, FATAL)
ALTER ACCOUNT SET LOG_LEVEL = 'INFO';

-- Enable all metrics collection
ALTER ACCOUNT SET METRIC_LEVEL = 'ALL';

-- Enable tracing for all executions
ALTER ACCOUNT SET TRACE_LEVEL = 'ALWAYS';
```

**Required privileges:**
- `MODIFY LOG LEVEL ON ACCOUNT`
- `MODIFY METRIC LEVEL ON ACCOUNT`
- `MODIFY TRACE LEVEL ON ACCOUNT`

(Typically requires ACCOUNTADMIN or delegated privileges)

**⚠️ MANDATORY STOPPING POINT**: Present the SQL and ask user to confirm before executing.

**If user lacks permissions or prefers not to change account settings**, go to Step 4B-Fallback.

---

### Step 4B: Telemetry Levels Disabled - Present Options

**Goal:** When telemetry collection is disabled at account level, enable it at account level

**⚠️ MANDATORY STOPPING POINT**: Present this to the user:

---

**Telemetry collection is currently disabled at the account level.**

I recommend enabling telemetry at the **account level**:
- Applies to all UDFs/procedures automatically
- No need to configure each object individually
- Centralized management of telemetry settings

**Enable at Account Level:**

```sql
ALTER ACCOUNT SET LOG_LEVEL = 'INFO';
ALTER ACCOUNT SET METRIC_LEVEL = 'ALL';
ALTER ACCOUNT SET TRACE_LEVEL = 'ALWAYS';
```

Required privileges: `MODIFY LOG LEVEL ON ACCOUNT`, `MODIFY METRIC LEVEL ON ACCOUNT`, `MODIFY TRACE LEVEL ON ACCOUNT` (typically ACCOUNTADMIN)

**Then, to raise verbosity for this debugging session:**

```sql
-- Option 1 (preferred): Session-level override — auto-expires when session ends
ALTER SESSION SET LOG_LEVEL = 'DEBUG';

-- Option 2: Function-level override — must be reverted when debugging is complete
ALTER FUNCTION <db.schema.fn_name>(<param_types>) SET LOG_LEVEL = 'DEBUG';
```

---

**If user lacks account-level privileges:**
- Go to Step 4D to generate an admin permission request
- In the meantime, function-level settings can be used as a workaround (go to Step 4B-Fallback)

---

### Step 4B-Fallback: Enable Telemetry at Function Level (Workaround)

**Goal:** Enable telemetry on the specific UDF when user lacks account-level privileges

**When to use:**
- User lacks account-level permissions and cannot get them
- Temporary workaround until admin enables account-level telemetry

**Proposed SQL:**
```sql
-- Enable logging on specific function (DEBUG captures all messages)
ALTER FUNCTION <db.schema.fn_name>(<param_types>) SET LOG_LEVEL = 'DEBUG';

-- Enable tracing on specific function
ALTER FUNCTION <db.schema.fn_name>(<param_types>) SET TRACE_LEVEL = 'ALWAYS';
```

**Required privileges:**
- `MODIFY` on the function
- `USAGE` on the database and schema

**⚠️ MANDATORY STOPPING POINT**: Ask user to confirm before executing.

**Inform user of limitations:**
> With function-level only, you'll need to set these parameters on each UDF you want to debug, and **you must revert them when debugging is complete** (e.g., `ALTER FUNCTION ... UNSET LOG_LEVEL;`). Request account-level enablement from an admin for a permanent solution.

---

### Step 4C: Raise Verbosity for Debugging

**Goal:** Override the account-level log level with a more verbose level for debugging

If the account level is set (e.g., INFO) but the user needs more verbose logging (e.g., DEBUG) for debugging:

**Option 1 (Preferred): Session-level override**
```sql
-- Automatically expires when the session ends — no cleanup needed
ALTER SESSION SET LOG_LEVEL = 'DEBUG';
```

**Option 2: Function-level override**
```sql
-- Set on specific function — must be reverted when debugging is complete
ALTER FUNCTION <db.schema.fn_name>(<param_types>) SET LOG_LEVEL = 'DEBUG';
```

**Required privileges (Option 2):**
- `MODIFY` on the function
- `USAGE` on the database and schema

**Note:** When set at multiple levels, the more verbose level wins. Session-level is preferred because it auto-expires. If using function-level, remind the user to revert after debugging:
```sql
ALTER FUNCTION <db.schema.fn_name>(<param_types>) UNSET LOG_LEVEL;
```

**⚠️ MANDATORY STOPPING POINT**: Ask user to confirm before executing.

---

### Step 4D: Generate Admin Permission Request

**Goal:** Create a message the user can send to an admin

If user lacks privileges to enable event collection:

**Generate this message:**

---

**Subject: Request to Enable UDF Telemetry Collection**

Hi,

I need to debug UDFs in our Snowflake account and require event table telemetry to be enabled. This will allow me to view logs, traces, and metrics from UDF executions.

**Why this is needed:**
- Debug failing or misbehaving UDFs
- View execution logs and error messages
- Monitor UDF performance with traces and metrics

**Recommended SQL to run (requires ACCOUNTADMIN or delegated privileges):**

```sql
-- Enable telemetry collection at account level
-- This uses the default SNOWFLAKE.TELEMETRY.EVENTS table
ALTER ACCOUNT SET LOG_LEVEL = 'INFO';
ALTER ACCOUNT SET METRIC_LEVEL = 'ALL';
ALTER ACCOUNT SET TRACE_LEVEL = 'ALWAYS';
```

**Alternative: Grant me privileges to manage telemetry levels:**
```sql
GRANT MODIFY LOG LEVEL ON ACCOUNT TO ROLE <my_role>;
GRANT MODIFY METRIC LEVEL ON ACCOUNT TO ROLE <my_role>;
GRANT MODIFY TRACE LEVEL ON ACCOUNT TO ROLE <my_role>;
```

**Documentation:**
- [Event Table Overview](https://docs.snowflake.com/en/developer-guide/logging-tracing/event-table-setting-up)
- [Setting Telemetry Levels](https://docs.snowflake.com/en/developer-guide/logging-tracing/telemetry-levels)

Thank you!

---

**⚠️ STOPPING POINT**: Present this message template to user.

---

### Step 4E: Database-Level Event Table (Only If Explicitly Requested)

**⚠️ WARNING**: Only proceed here if user explicitly requests a database-level event table.

**Explain limitations:**

> **Database-level event tables have important limitations:**
>
> 1. **Replication not supported** - Event tables cannot be replicated. Databases with event tables are skipped during replication.
>
> 2. **Non-database telemetry lost** - Telemetry from actions not owned by a database (e.g., anonymous SQL blocks) can only go to account-level event tables. Without an account-level table, this data is lost.
>
> 3. **Enterprise Edition required** - Database-level event tables require Snowflake Enterprise Edition.
>
> 4. **Increased management overhead** - Must manage event tables per database.
>
> **Recommendation:** Use a single account-level event table unless you have specific governance, isolation, or compliance requirements that cannot be met through views or row-level access policies.

**If user still wants to proceed:**

```sql
-- Create event table in specific database
CREATE EVENT TABLE <db>.<schema>.<event_table_name>;

-- Associate with database
ALTER DATABASE <db> SET EVENT_TABLE = '<db>.<schema>.<event_table_name>';

-- Set telemetry levels for that database
ALTER DATABASE <db> SET LOG_LEVEL = 'INFO';
```

**⚠️ MANDATORY STOPPING POINT**: Confirm user understands limitations before executing.

---

### Step 5: Verify Configuration

**Goal:** Confirm that telemetry collection is working

**Actions:**

1. **Verify settings applied:**
   ```sql
   SHOW PARAMETERS LIKE 'LOG_LEVEL' IN ACCOUNT;
   SHOW PARAMETERS LIKE 'METRIC_LEVEL' IN ACCOUNT;
   SHOW PARAMETERS LIKE 'TRACE_LEVEL' IN ACCOUNT;
   ```

2. **Test with a simple UDF call:**
   ```sql
   -- Execute a UDF that should produce logs
   SELECT <udf_name>(<test_params>);
   ```

3. **Check for logs with retry logic:**

   **⚠️ Important: Telemetry Ingestion Latency**
   
   Telemetry data may take several seconds to appear in the event table after UDF execution. **You MUST retry before concluding that telemetry collection is not working.**

   **Retry procedure:**
   1. **First query** → Wait 5 seconds after UDF execution, then query
   2. **No results** → Wait 10 more seconds, then query again
   3. **Still no results** → Wait 10 more seconds, then query a third time
   4. **After 3 retries with no results** → Investigate configuration issues

   ```sql
   SELECT *
   FROM SNOWFLAKE.TELEMETRY.EVENTS
   WHERE RESOURCE_ATTRIBUTES['snow.executable.name'] ILIKE '%<udf_name>%'
     AND TIMESTAMP > DATEADD('minute', -5, CURRENT_TIMESTAMP())
   ORDER BY TIMESTAMP DESC
   LIMIT 10;
   ```

   **Example verification workflow:**
   ```
   Execute UDF
     ↓
   Wait 5 seconds → Query event table → No results?
     ↓
   Wait 10 seconds → Query again → No results?
     ↓
   Wait 10 seconds → Query again → No results?
     ↓
   NOW investigate configuration issues
   ```

**If no logs appear after retries:**
- Verify the UDF contains logging statements (see [python-logging.md](../references/python-logging.md))
- Verify the UDF includes `snowflake-telemetry-python` package and `from snowflake import telemetry` import
- Check that UDF's LOG_LEVEL is not more restrictive than collection level
- Verify TRACE_LEVEL is set to 'ALWAYS' for span data

---

## Stopping Points Summary

1. ✋ Before executing any `ALTER ACCOUNT SET` command
2. ✋ When telemetry is disabled at account level - present account-level enablement
3. ✋ Before executing any `ALTER FUNCTION SET` command
4. ✋ Before executing any `ALTER DATABASE SET` command
5. ✋ When generating admin permission request
6. ✋ When user requests database-level event table (warn about limitations)

---

## Output

- Clear summary of current event table configuration
- Telemetry level status at account and function levels
- SQL scripts for enabling telemetry (with user approval)
- Admin request message if permissions are lacking
- Verification that configuration is working
