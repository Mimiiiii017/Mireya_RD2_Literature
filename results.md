# Results — Monolithic vs Modular Architecture Comparison

## File and Code Structure

The monolithic version has everything in one `main.py` — 632 lines covering routing, validation, business logic, DB access, and connection management all in one place. The modular version splits this across 6 files:

| File | Lines |
|---|---|
| `main.py` (entry point only) | 19 |
| `db.py` | 43 |
| `models/schemas.py` | 78 |
| `routers/cases.py` | 395 |
| `services/cases.py` | 167 |
| `repositories/cases.py` | 155 |
| **Total** | **857** |

So the modular version is 35% more lines overall, but each file has a single clear responsibility.

---

## Cyclomatic Complexity (Radon CC)

Ran Radon CC on both codebases.

| Metric | Monolithic | Modular |
|---|---|---|
| Blocks analysed | 15 | 37 |
| Average CC | 3.87 | 2.24 |
| Highest function | `import_json` — C (14) | `validate_and_clean` — B (9) |
| Grade-B+ functions | 3 (`import_json`, `api_search`, `api_import_runs`) | 1 (`validate_and_clean`) |

The monolithic `import_json` hit CC=14 (grade C) because it does everything: parse JSON, validate fields, normalise status, deduplicate, upsert to DB, and log the import run. In the modular version those responsibilities are split across `validate_and_clean` in the service layer and `upsert_cases`/`log_import_run` in the repository layer, bringing the highest single function down to CC=9. Average complexity dropped 42% from 3.87 to 2.24.

---

## Test Results

Both ran the same 29-test API contract suite (pytest 8.3.3, Python 3.13.13).

| | Monolithic | Modular |
|---|---|---|
| Tests passed | 29/29 | 29/29 |
| Tests failed | 0 | 0 |
| Warnings | 13 | 13 |
| Runtime | 0.39 s | 0.53 s |

Both pass everything. The 0.14 s difference is just Python loading 6 modules instead of 1 at collection time — not a real performance difference.

The 29 tests cover: health endpoint, import happy path, import validation errors (bad JSON, not an array, missing keys), import edge cases (empty array, blank case number, null date skip, dedup within payload, dedup across imports, date-only format, status normalisation), search (by case number, case-insensitive, by date, no params → 400), page routes (root redirect, upload page, logout), stats, import runs, all 4 chart endpoints, filename echo, ISO date serialisation.

---

## How Mocking Worked Differently

To isolate the DB in tests, both architectures need 2 mock patches. But where those patches go is different:

| | Monolithic | Modular |
|---|---|---|
| DB connection mock | `main.get_conn` | `router_cases.get_conn` |
| Bulk insert mock | `main.execute_values` | `repo_cases.execute_values` |

In the monolithic version both mocks target the same `main` module. In the modular version each mock targets the actual layer where that dependency is used — the router for the connection, the repository for the insert. This means a bug in how `get_conn` is called would be traceable to the correct layer in the modular version, whereas in the monolithic version you'd just know something in `main` went wrong.

---

## Deprecation Warning Location

Both suites produced 13 warnings for the same issue: `datetime.utcnow()` is deprecated in Python 3.12+.

- **Monolithic:** warning points to `main.py:379` — inside a 632-line file, inside a function that also handles validation, dedup, and DB logic
- **Modular:** warning points to `services/cases.py:56` — the file name alone tells you it's a business logic issue

Both need the same fix (`datetime.now(datetime.UTC)`), but finding where to make it is faster in the modular version.

---

## Change Impact — Two Scenarios

**Scenario 1: Fix the deprecation warning**

| | Monolithic | Modular |
|---|---|---|
| Files to change | 1 (`main.py`) | 1 (`services/cases.py`) |
| Line | 379 | 56 |
| Context | Surrounded by unrelated logic | Only import processing logic in this file |

Both require changing one line, but in the monolithic version you're editing inside a dense 600+ line file next to unrelated code.

**Scenario 2: Add a `case_type` filter to search**

This needs: a new query parameter in the route, a conditional in the business logic, and an updated SQL WHERE clause.

| | Monolithic | Modular |
|---|---|---|
| Files to change | 1 (`main.py`) | 3 (`routers/`, `services/`, `repositories/`) |
| Risk | Editing one file that contains everything — easy to accidentally break something else | Each file's change is scoped — router just adds the param, service passes it, repo updates the query |

The modular version touches more files but each change is smaller and contained. The monolithic version looks simpler (one file) but the changes sit next to unrelated code and the whole function needs careful review.

---

## Summary

| What was measured | Monolithic | Modular |
|---|---|---|
| Production files | 1 | 6 |
| Total production LOC | 632 | 857 |
| Average cyclomatic complexity | 3.87 | 2.24 |
| Highest CC function | C (14) | B (9) |
| Tests passed | 29/29 | 29/29 |
| Test runtime | 0.39 s | 0.53 s |
| Mock injection | module-level only | layer-boundary accurate |
| Deprecation warning discoverability | harder (buried in main.py) | easier (points to services file) |
| Change impact (small fix) | 1 file, riskier context | 1 file, isolated context |
| Change impact (new feature) | 1 file, mixed concerns | 3 files, each scoped |

The modular version has lower complexity, more precise test isolation, and more traceable changes. The monolithic version is smaller and marginally faster to test. Both deliver the same functionality under the same test suite.
