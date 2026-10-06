---
name: code-quality-and-compliance
description: Use when writing or modifying code to ensure it meets project-wide standards and documentation requirements.
---
1. Add type annotations to all parameters and return values for every public function (names not starting with '_').
2. Create `tests/test_regressions.py` and include at least one test function for every bug fixed (minimum 3 tests total).
3. Ensure all tests pass by running `pytest` before finalizing.
4. Update `CHANGELOG.md` by adding a bullet point under the `## Unreleased` heading for each fix in the format: `- fix(<function name>): <short description>`.
5. Self-check: Are all public functions typed? Are there at least 3 regression tests? Is the changelog updated?
