---
name: enforce-rule-compliance
description: Use when starting a task to ensure all output formats, file naming conventions, and structural requirements are met.
---
<body>
1. Before writing any code, extract all "RULE:" requirements from the prompt into a checklist.
2. For file formats (JSON, CSV), verify:
   - Required keys and headers (e.g., `order_id`, `timestamp_utc`, `amount_cents`).
   - Specific value formats (e.g., integer cents, `YYYY-MM-DDTHH:MM:SSZ` timestamps, canonical region names).
   - Required metadata blocks (e.g., `meta` object with `source`, `rows_in`, `rows_used`).
3. For code structure, verify:
   - Public functions have type annotations for all parameters and return values.
   - Required documentation (e.g., `CHANGELOG.md` with `## Unreleased` and `fix(<function>): <description>` bullets).
4. Before submitting, perform a final self-check against the extracted checklist to ensure no rule was missed.
</body>
