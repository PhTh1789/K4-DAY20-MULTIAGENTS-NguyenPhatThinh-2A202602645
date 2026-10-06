---
name: repository-bug-fix-workflow
description: Use when fixing bugs in an existing package that has repository rules for tests, typing, or changelog entries.
---
- Read the project instructions and inspect the affected package, tests, and changelog before editing.
- Preserve all original test files; add or update only tests explicitly permitted by the project rules.
- Add a regression test for each distinct bug fixed, and run the required test suite.
- Annotate every parameter and return value of each public function when the project requires type hints.
- Record each fix under the required changelog heading, following the project’s specified bullet format.
- Before finishing, check the diff to confirm protected files are unchanged and all required tests and changelog entries are present.
