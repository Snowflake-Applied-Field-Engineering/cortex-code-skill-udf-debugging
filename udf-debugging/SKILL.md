---
name: udf-debugging
description: "Debug Snowpark UDFs using Snowflake Event Tables. Use for: UDF errors, adding logging to UDFs, querying UDF logs, debugging Python/Java/Scala UDFs, event table setup, custom traces, span attributes, trace events, OpenTelemetry, snowflake-telemetry-python. Triggers: debug UDF, UDF logs, event table, UDF error, snowpark logging, function not working, UDF not returning, trace UDF, add telemetry, custom span, trace event."
---

# UDF Debugging with Event Tables

Expert guidance for debugging Snowpark User-Defined Functions (UDFs) using Snowflake Event Tables. This skill helps you set up telemetry collection, query logs from UDFs, identify common error patterns, and add logging to your code.

## When to Use

Use this skill when users need to:
- Debug a failing or misbehaving UDF
- Set up event table logging for UDFs
- Query logs, metrics, or traces from UDF executions
- Add logging statements to Python/Java/Scala UDFs
- Add custom span attributes or trace events (OpenTelemetry)
- Understand why a UDF is returning unexpected results
- Troubleshoot Snowpark dependency or memory issues

## Prerequisites

- Access to the Snowflake account where the UDF runs
- Appropriate privileges to query event tables or set telemetry levels

---

## Intent Detection

When a user makes a request, detect their intent and route to the appropriate sub-skill:

### SETUP Intent

**Trigger phrases**: "enable event table", "set up logging", "configure telemetry", "enable tracing", "no logs showing", "event table not working", "how to enable logging"

**-> Load**: [setup/SKILL.md](setup/SKILL.md)

### DEBUG Intent

**Trigger phrases**: "debug UDF", "check logs", "UDF failing", "view errors", "query event table", "UDF logs", "why is my UDF", "trace UDF execution", "UDF not working"

**-> Load**: [debug/SKILL.md](debug/SKILL.md)

---

## Workflow Decision Tree

```
User Request
    |
    v
+-----------------------------+
| Run Quick Diagnostic Queries|
| (check telemetry levels)    |
+-----------------------------+
    |
    v
+-----------------------------+
| Account-level telemetry OFF?|
+-----------------------------+
    |               |
   YES              NO
    |               |
    v               v
  ⚠️ STOP       Detect Intent
  Present          |
  account-level    +---> SETUP → Load setup/SKILL.md
  enablement       |
  FIRST.           +---> DEBUG → Load debug/SKILL.md
  Wait for user.
  THEN route
  to intent.
```

### ⚠️ MANDATORY: Account-Level Telemetry Gate

**Before routing to ANY sub-skill (SETUP or DEBUG), you MUST check account-level telemetry.**

Run the Quick Diagnostic Queries below. If **any** of `LOG_LEVEL = OFF`, `METRIC_LEVEL = NONE`, or `TRACE_LEVEL = OFF` at the account level:

1. **STOP** — Do NOT proceed to any sub-skill workflow
2. **Present account-level enablement** to the user with the SQL below
3. **Wait for user response** before continuing
4. **NEVER use session-level or function-level telemetry as a substitute** for account-level enablement

```sql
-- Present this to the user for approval:
ALTER ACCOUNT SET LOG_LEVEL = 'INFO';
ALTER ACCOUNT SET METRIC_LEVEL = 'ALL';
ALTER ACCOUNT SET TRACE_LEVEL = 'ALWAYS';
```

Only after the user has addressed account-level telemetry (enabled it, or explicitly declined due to lack of privileges) should you route to the appropriate sub-skill.

---

## Quick Diagnostic Queries

For immediate assessment before routing:

```sql
-- Check if account has event table configured
SHOW PARAMETERS LIKE 'EVENT_TABLE' IN ACCOUNT;

-- Check current telemetry levels
SHOW PARAMETERS LIKE 'LOG_LEVEL' IN ACCOUNT;
SHOW PARAMETERS LIKE 'TRACE_LEVEL' IN ACCOUNT;
SHOW PARAMETERS LIKE 'METRIC_LEVEL' IN ACCOUNT;

-- Quick check for recent UDF logs (if event table exists)
SELECT COUNT(*) as log_count
FROM SNOWFLAKE.TELEMETRY.EVENTS
WHERE RESOURCE_ATTRIBUTES['snow.executable.type'] IN ('FUNCTION', 'PROCEDURE')
  AND TIMESTAMP > DATEADD('hour', -1, CURRENT_TIMESTAMP());
```

### ⚠️ Telemetry Ingestion Latency

**Important:** Telemetry data may take a few seconds to appear in the event table after UDF execution. If you query the event table immediately after executing a UDF and see no results:

1. **Wait 5-10 seconds** before querying again
2. **Retry the query** up to 3 times with delays between attempts
3. Only conclude that telemetry is missing after retries fail

This latency applies to:
- Initial telemetry collection after enabling LOG_LEVEL/TRACE_LEVEL
- Iterative debugging cycles (modify UDF → execute → check logs → repeat)
- Any scenario where you expect new telemetry data

---

## ⚠️ Telemetry-First Debugging Principle

**Always check for telemetry data before proceeding with any debugging.**

Even if the error source seems obvious from the code:
- **DO NOT skip telemetry collection** - runtime data reveals bugs not visible in code
- **ALWAYS prompt the user to enable collection** if no data is found
- **Require re-execution** of the UDF to generate telemetry before analyzing

This ensures:
1. Runtime errors are captured (may differ from code inspection)
2. Multiple bugs are caught (not just the obvious one)
3. Users learn effective debugging workflows
4. Future debugging is easier with telemetry baselines

---

## Sub-Skills

| Sub-Skill | Purpose | When to Load |
|-----------|---------|--------------|
| [setup/SKILL.md](setup/SKILL.md) | Event table detection & telemetry setup | SETUP intent |
| [debug/SKILL.md](debug/SKILL.md) | Query logs, analyze errors, suggest fixes | DEBUG intent |

---

## Reference Documents

| Reference | Content |
|-----------|---------|
| [references/observability-parameters.md](references/observability-parameters.md) | Snowflake Parameters: EVENT_TABLE, LOG_LEVEL, METRIC_LEVEL, TRACE_LEVEL, SQL_TRACE_QUERY_TEXT |
| [references/event-table-columns.md](references/event-table-columns.md) | Event table schema and column descriptions |
| [references/python-logging.md](references/python-logging.md) | Logging, custom traces, span attributes, and events |
| [references/error-patterns.md](references/error-patterns.md) | Common Snowpark errors and fixes |

---

## ⚠️ Telemetry Scope Policy

**⚠️ MANDATORY STOPPING POINT: If any of LOG_LEVEL = OFF, METRIC_LEVEL = NONE, and/or TRACE_LEVEL = OFF at the account level, you MUST present account-level enablement to the user and wait for approval BEFORE doing anything else with telemetry.**

**NEVER skip account-level enablement.** Session-level and function-level telemetry are **supplements** for raising verbosity during debugging — they are NOT replacements for account-level baselines. NEVER suggest session-level or function-level as an alternative to account-level enablement.

**NEVER suggest database-level telemetry parameter changes.**

**Required two-step pattern:**

```sql
-- Step 1 (REQUIRED): Account level baseline — MUST be presented first
ALTER ACCOUNT SET LOG_LEVEL = 'INFO';
ALTER ACCOUNT SET METRIC_LEVEL = 'ALL';
ALTER ACCOUNT SET TRACE_LEVEL = 'ALWAYS';

-- Step 2 (optional): Raise verbosity for this debugging session
-- Option A (preferred): Session-level override — automatically expires when session ends
ALTER SESSION SET LOG_LEVEL = 'DEBUG';

-- Option B: Function-level override — must be reverted when debugging is complete
ALTER FUNCTION db.schema.my_udf(VARCHAR) SET LOG_LEVEL = 'DEBUG';
-- After debugging:
ALTER FUNCTION db.schema.my_udf(VARCHAR) UNSET LOG_LEVEL;
```

This pattern ensures:
- All UDFs/procedures are covered by default at the account level
- Debugging sessions get more verbose output without affecting the account-wide setting permanently
- No per-database telemetry configuration drift
- Session-level is preferred because it auto-expires; function-level overrides persist and must be manually reverted

**If the user lacks account-level privileges:** Direct them to request privileges from an admin (see setup/SKILL.md Step 4D). Only then fall back to function-level as a temporary workaround.

---

## Key Concepts

### Event Table Precedence

An order of precedence determines which event table collects telemetry:

```
Database Event Table > Account Event Table
```

- If a database has an associated event table, objects in that database write to the database's event table
- Other databases without their own event table write to the account-level event table
- **Recommendation**: Use the default account-level event table (`SNOWFLAKE.TELEMETRY.EVENTS`)

### Telemetry Level Hierarchy

Levels can be set at multiple scopes and override each other:

**Session parameters**: Account > User > Session
**Object parameters**: Account > Database > Schema > Object (UDF/procedure)

When set at multiple levels, the **more verbose** level wins. This means a `DEBUG` session-level setting will capture more than an `INFO` account-level setting.

See [references/observability-parameters.md](references/observability-parameters.md) for detailed parameter documentation.

### Default Event Table

All Snowflake accounts include `SNOWFLAKE.TELEMETRY.EVENTS` by default. This is recommended for most use cases.

### Telemetry Package for Auto-Instrumentation

**Always prompt users to add** the `snowflake-telemetry-python` package and import. This enables **auto-instrumentation**:

```sql
CREATE OR REPLACE FUNCTION my_udf(...)
  RETURNS ...
  LANGUAGE PYTHON
  PACKAGES = ('snowflake-telemetry-python')  -- Enables auto-instrumentation
  ...
AS $$
from snowflake import telemetry  # ALWAYS include this import
...
$$;
```

**What auto-instrumentation provides (no extra code needed):**
- Snowflake creates a span (`snow.auto_instrumented`) for each execution
- Captures start/end timestamps, execution status, and query context
- Links traces to query IDs for correlation

### Custom Span Attributes and Events (Only When Needed)

**Only add custom traces when auto-instrumentation isn't sufficient:**
- `telemetry.set_span_attribute(key, value)` - Add searchable metadata
- `telemetry.add_event(name, attributes)` - Add timestamped milestones

**Use cases for custom traces:**
- Breaking down execution into smaller portions for analysis
- Adding searchable metadata (customer ID, order ID) to query later
- Marking specific milestones in complex operations
- Isolating computation-heavy actions (e.g., ML model training)

See [references/python-logging.md](references/python-logging.md) for complete examples.

---

## Stopping Points Summary

All sub-skills follow this philosophy: **NO changes without explicit user approval.**

- **READ-ONLY queries**: Can run freely (diagnostics, log queries)
- **⚠️ BEFORE ANY telemetry changes**: If any of `LOG_LEVEL = OFF`, `METRIC_LEVEL = NONE`, and/or `TRACE_LEVEL = OFF` at the account-level, you MUST present account-level enablement first. NEVER jump to session-level or function-level as a shortcut.
- **ANY mutation**: Requires stopping point and user approval
  - Enabling telemetry collection
  - Modifying LOG_LEVEL/METRIC_LEVEL/TRACE_LEVEL
  - Altering UDF parameters

---

## Output

- Clear diagnosis of event table configuration
- Actionable debugging suggestions based on log analysis
- Code snippets for adding logging to UDFs
- SQL scripts for enabling telemetry (with user approval)
