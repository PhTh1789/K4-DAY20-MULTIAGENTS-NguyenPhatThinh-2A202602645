---
name: structured-data-deliverables
description: Use when transforming tabular input into JSON or CSV deliverables with specified schemas, deduplication, or money and timestamp rules.
---
- Translate every output requirement into an explicit schema and validation checklist before processing data.
- Track input-row counts separately from counts after deduplication or exclusion; calculate metadata from the correct population.
- Define the deduplication key and missing-value policy, and apply them consistently to every output and summary.
- Represent monetary values in the required units; use decimal-safe arithmetic for currency and convert to integer minor units when required.
- Normalize timestamps to the required timezone and exact format, and map categories to canonical spellings.
- Write CSV headers and column order exactly as specified; ensure each output row satisfies the required inclusion rules.
- Reopen and validate generated files: check types, required keys, row counts, uniqueness, formats, and consistency between outputs.
