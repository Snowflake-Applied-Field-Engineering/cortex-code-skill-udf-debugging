# UDF Debugging Cortex Code Skill

Debug Snowpark User-Defined Functions using Snowflake Event Tables. This skill provides guided workflows for:

- Setting up event tables and configuring telemetry levels (LOG_LEVEL, TRACE_LEVEL, METRIC_LEVEL)
- Querying UDF logs, metrics, and traces from event tables
- Analyzing common Snowpark error patterns and suggesting fixes
- Adding logging and OpenTelemetry instrumentation to Python UDFs

The skill routes between two sub-workflows based on user intent:

| Sub-Skill | Purpose |
|-----------|---------|
| `setup/` | Event table detection and telemetry configuration |
| `debug/` | Log querying, error analysis, and fix suggestions |

Reference documents in `references/` cover event table schema, observability parameters, error patterns, and Python logging examples.

## Skill Structure

Each skill follows a standard layout:

```tree
<skill-name>/
  SKILL.md              # Entry point: intent detection, routing, key concepts
  <sub-skill>/
    SKILL.md            # Sub-workflow with step-by-step guidance
  references/
    *.md                # Supporting reference material
```

`SKILL.md` files use YAML frontmatter (`name`, `description`, `parent_skill`) and define structured workflows with mandatory stopping points where user confirmation is required before executing mutations.

## License

Apache License 2.0 -- see [LICENSE.txt](LICENSE.txt) for details.

## Support

This repository and its contents are NOT supported by Snowflake.

The resources, scripts, and documentation provided here are created and maintained independently. They are intended for internal use, educational purposes, or to streamline specific workflow tasks at your own risk.

For questions, issues, or contributions, please open an issue on GitHub. Support is provided on a best-effort basis, and Snowflake Support is unable to assist with content in this repository.