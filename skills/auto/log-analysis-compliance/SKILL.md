---
name: log-analysis-compliance
description: Use when parsing log files to ensure the output JSON structure and naming conventions are correct.
---
1. Ensure all service names in the output are lower-case and use underscores instead of hyphens (e.g., `payment_service`).
2. Sort the `errors` list by `service` (alphabetical), then by `timestamp_utc` (ascending).
3. Include the required top-level keys in the output JSON: `"schema_version": 2` and `"generated_by": "log-triage"`.
4. Self-check: Are service names formatted with underscores? Is the `errors` list sorted correctly? Are the schema version and generator fields included?
