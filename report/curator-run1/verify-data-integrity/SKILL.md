---
name: verify-data-integrity
description: Use when processing data or logs to ensure calculations, sorting, and transformations match the required specifications.
---
<body>
1. Verify sorting requirements (e.g., "sorted by service, then by timestamp_utc, ascending") before finalizing the output structure.
2. Check for data-specific transformations:
   - Replace hyphens with underscores in service names.
   - Convert all monetary values to integer cents.
   - Ensure all timestamps are normalized to UTC.
3. Validate the output against the schema requirements (e.g., `schema_version: 2`, `generated_by: "log-triage"`).
4. Run a test script to verify the output file content matches the expected schema and constraints before declaring the task complete.
</body>
