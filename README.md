# AI301 Open Source Capstone — Contribution Log

**Contributor:** Nguyen Nhat Quang Truong
**Cohort:** AI301 | AI Open Source Capstone — Summer 2026, Section 1C
**Status:** Phase I Complete

> Living engineering journal for my CodePath AI301 capstone contribution.
> Updated every phase. Mentors and staff read this to know where I am and how to help.

---

## At a Glance

| Field | Value |
|---|---|
| **Project** | `dbt-labs/dbt-utils` |
| **Issue** | [#1026 — expression_is_true tests for not-false: should test for true](https://github.com/dbt-labs/dbt-utils/issues/1026) |
| **Type** | Bug |
| **Language** | Jinja + SQL (dbt macros) |
| **My fork** | https://github.com/nnqtruong/dbt-utils |
| **Scope** | One macro file (`macros/generic_tests/expression_is_true.sql`) + integration tests |
| **Fallback** | Another small dbt-utils macro bug, or the Apache Doris SQL-function pool (off-list, mentor pending approval) |

---

## Phase I — Issue Selection

### Issue
https://github.com/dbt-labs/dbt-utils/issues/1026

### Problem Summary
`dbt_utils.expression_is_true` is meant to assert that a SQL expression holds for every row. The macro finds failing rows with `where not({{ expression }})` in both of its branches (with and without `column_name`). Because SQL three-valued logic makes `not(NULL)` evaluate to NULL — which a WHERE clause drops — any row where the expression evaluates to NULL **passes silently** instead of failing. A test asserting "this must be true" should not be satisfied by an unknown value, so NULL rows should fail. "Fixed" means the test also fails NULL results, in both branches, with integration tests proving it.

### Why I Chose This Issue
Why I Chose This Issue

1. On-domain for my target role. My career direction is data analyst -> data engineering / analytics engineering -> quantitative dev. dbt is effectively synonymous with the analytics-engineering role, and dbt-utils is its core test/macro library — a merged change here maps directly to the resume story I want to tell.
2. Chosen on a deliberate risk basis. The pre-vetted issue list turned out high-contention: every strong candidate I evaluated in larger projects (Trino, Meltano) was already claimed, had an open PR closing it, or had stalled.
3. Verified the lane is clear. dbt-utils is a small repo (~60 open issues) where I could confirm availability: #1026 has no assignee, no linked PR, and its reporter is a drive-by account unlikely to build it themselves.
4. Fits my strengths, no language gap. It's SQL/Jinja, not Java — while still requiring real judgment. The fix is a NULL-semantics correctness question, the kind of edge-case reasoning that's genuinely mine to own, not something to generate.
5. It's a bug, not a feature. That avoids the design-debate limbo that stalls feature requests in this repo's triage queue.
6. A nuance I surfaced while reading the code: the change is a behavior change, not a pure fix — tests that currently pass on NULLs would begin to fail. So my opening comment asks the maintainers a design question (NULL-fails as default, or opt-in behind a config flag) rather than assuming a direction.

### Feasibility Guardrails
- **Phase II environment gate:** By end of Week 2, have `dbt-utils` running locally against a dev warehouse and the NULL-passes-silently behavior reproduced. If not there by then, fall back to a smaller dbt-utils macro bug or the Doris function pool.
- **Phase IV review gate:** This repo's Stale-bot is active (it has flagged similar issues at 180 days), which signals thin maintainer attention and real review latency. If my PR/comment is untouched ~7–10 days after posting, ping the thread once; if still cold, start Cycle 2 in parallel rather than wait.

---

## Phase II — Reproduce & Plan
_(To be completed Week 2)_

### Understanding the Issue
Current macro (both branches use `where not(...)`):

```sql
{% if column_name is none %}
where not({{ expression }})
{%- else %}
where not({{ column_name }} {{ expression }})
{%- endif %}
```

On a NULL expression result, `not(NULL)` → NULL → row excluded → row passes. Fix must cover both branches.

### Reproduction Process
_[Local dbt-utils setup; a seed/model with NULL-producing expressions; show the test passing when it should fail.]_

### Solution Approach
Add a null check to both branches:

```sql
{% if column_name is none %}
where not({{ expression }}) or ({{ expression }}) is null
{%- else %}
where not({{ column_name }} {{ expression }}) or ({{ column_name }} {{ expression }}) is null
{%- endif %}
```

Pending maintainer direction on default-vs-opt-in.

---

## Phase III — Build
_(To be completed Week 3+)_

### Testing Strategy
_[Integration tests covering: not-true rows fail, NULL rows fail, true rows pass — across both the `column_name` and no-`column_name` branches, and both `should_store_failures()` modes.]_

### Implementation Notes
_[Decisions, how it matches existing dbt-utils test conventions.]_

---

## Phase IV — Pull Request & Review
_(To be completed Week 4+)_

### PR Link
_[url]_

### PR Summary
_[What I changed and why, in my own words.]_

### Maintainer Feedback Log
_[Each round: what was asked, how I responded, what I revised.]_

---

## AI Usage Log

> Where I used AI, what I asked, and how I verified. I own every line; "the AI suggested it" is not an answer to a reviewer.

**Tools:** Claude (web) for issue-selection strategy and drafting; reading + verification done by me against source.

| Date | Task | What AI did | How I verified | Reflection |
|---|---|---|---|---|
| _2026-06-07_ | Issue selection | Helped filter candidates by contention/scope/domain; reverse-engineered the curated-list filter (label scrape) to find low-contention issues | Read each candidate's live issue page myself; confirmed #1026 unassigned, no linked PR | The contention analysis was the real value; final pick was my call |
| _2026-06-07_ | Claim comment | Drafted a comment framing the NULL-semantics design question | Read `expression_is_true.sql` myself; rewrote the comment in my own voice before posting | Owning the NULL reasoning matters — maintainer may push back |
| _2026-06-07_ | Code reading | Flagged that the bug affects *both* WHERE branches (the reporter's example only covered one) and the `should_store_failures()` wrinkle | Confirmed both branches in source; verified the store-failures path | Caught a gap in the original issue report |
| | | | | |


# AI301 Contribution README — Phase II: Reproduce and Plan

**Contributor:** Wayne Truong  
**Course:** AI301 — AI Open Source Capstone (CodePath)  
**Date:** June 14, 2026  
**Upstream project:** [dbt-labs/dbt-utils](https://github.com/dbt-labs/dbt-utils)  
**Issue:** [#1026 — `expression_is_true` tests for not-false: should test for true](https://github.com/dbt-labs/dbt-utils/issues/1026)

---

## Issue Summary

The `dbt_utils.expression_is_true` generic test is intended to assert that a SQL expression evaluates to `TRUE` for every row. In practice, the macro compiles to `WHERE NOT(expression)`, which only catches rows where the expression is explicitly `FALSE`. Rows where the expression evaluates to `NULL` are silently treated as passing.

This is a classic SQL three-valued logic bug:

| Expression result | `NOT(expression)` | Included in `WHERE`? | Current test behavior |
|---|---|---|---|
| `TRUE` | `FALSE` | No | Pass (correct) |
| `FALSE` | `TRUE` | Yes | Fail (correct) |
| `NULL` | `NULL` | No | **Pass (bug)** |

The test therefore checks **"not false"** rather than **"is true"**.

---

## Maintainer Direction

Before opening a PR, I asked on the issue whether the fix should change default behavior or be opt-in. Maintainer [@msh210 confirmed an opt-in approach](https://github.com/dbt-labs/dbt-utils/issues/1026#issuecomment-4645503157):

> "Good point, @nnqtruong, that people may be relying on it. So I suppose an opt-in is the way to go."

**Plan implication:** add a new argument (proposed name: `fail_on_null`, default `false`) rather than changing existing behavior.

---

## Local Development Environment

### Tools and versions

| Component | Version / detail |
|---|---|
| OS | macOS (darwin 25.5.0, arm64) |
| Python | 3.12.4 |
| dbt-core | 1.13.0-a1 |
| dbt-postgres | 1.11.0-b1 |
| Database | Postgres 17.5 (`cimg/postgres:17.5`) via Docker |
| Repo setup | Forked `dbt-labs/dbt-utils`, local package linked via `integration_tests/packages.yml` (`local: ../`) |

### Setup steps

```bash
# Clone fork
git clone https://github.com/<your-username>/dbt-utils.git
cd dbt-utils

# Start Postgres (port 5433 used because 5432 was occupied by another container)
docker run -d --name dbt-utils-postgres-repro \
  -e POSTGRES_USER=root \
  -e POSTGRES_PASSWORD=password \
  -e POSTGRES_DB=dbt_utils_test \
  -p 5433:5432 cimg/postgres:17.5

# Virtual environment + dependencies
python3 -m venv env
source env/bin/activate
pip install --upgrade pip setuptools
pip install --pre dbt-postgres -r dev-requirements.txt

# Load credentials (override port for local container)
export POSTGRES_HOST=localhost
export POSTGRES_USER=root
export DBT_ENV_SECRET_POSTGRES_PASS=password
export POSTGRES_PORT=5433
export POSTGRES_DATABASE=dbt_utils_test
export POSTGRES_SCHEMA=dbt_utils_integration_tests_postgres
```

### Environment verification

```bash
cd integration_tests
dbt deps --target postgres
dbt debug --target postgres
```

**Result:** `All checks passed!`

```
Connection:
  host: localhost
  port: 5433
  user: root
  database: dbt_utils_test
  schema: dbt_utils_integration_tests_postgres
  Connection test: [OK connection ok]
```

---

## Bug Reproduction

### Root cause in source code

File: `macros/generic_tests/expression_is_true.sql`

Both branches of the macro use `WHERE NOT(...)`:

```sql
{% if column_name is none %}
where not({{ expression }})
{%- else %}
where not({{ column_name }} {{ expression }})
{%- endif %}
```

Generic tests in dbt return rows that **fail** the assertion. With `WHERE NOT(expr)`, only explicitly `FALSE` rows are returned. `NULL` rows are excluded because `NOT NULL` evaluates to `NULL`, and `WHERE NULL` filters the row out.

### Reproduction data

Created seed file: `integration_tests/data/schema_tests/data_test_expression_is_true_null.csv`

```csv
col_a,col_b
1,0
0,1
,
```

Row 1: `1 + 0 = 1` → `TRUE` (should pass)  
Row 2: `0 + 1 = 1` → `TRUE` (should pass)  
Row 3: `NULL + NULL = NULL` → `NULL` (should fail, but currently passes)

Added test in `integration_tests/models/generic_tests/schema.yml`:

```yaml
- name: data_test_expression_is_true_null
  data_tests:
    - dbt_utils.expression_is_true:
        expression: col_a + col_b = 1
```

### Reproduction commands

```bash
cd integration_tests
dbt seed --target postgres --select data_test_expression_is_true_null
dbt test --target postgres --select data_test_expression_is_true_null
```

### Reproduction result — test incorrectly PASSES

```
1 of 1 START test dbt_utils_expression_is_true_data_test_expression_is_true_null_col_a_col_b_1
1 of 1 PASS dbt_utils_expression_is_true_data_test_expression_is_true_null_col_a_col_b_1
Done. PASS=1 WARN=0 ERROR=0 SKIP=0
```

**Expected behavior:** test should fail because row 3 has a `NULL` expression result.  
**Actual behavior:** test passes. Bug confirmed.

### Compiled SQL (generated by dbt)

```sql
select
    1
from "dbt_utils_test"."dbt_utils_integration_tests_postgres"."data_test_expression_is_true_null"
where not(col_a + col_b = 1)
```

This query returns **0 rows** on the reproduction dataset, so dbt reports the test as passing.

### Pure SQL confirmation (Postgres)

```sql
CREATE TEMP TABLE repro AS
SELECT 1 AS col_a, 0 AS col_b
UNION ALL SELECT 0, 1
UNION ALL SELECT NULL::int, NULL::int;

-- Current (buggy) behavior: 0 failing rows
SELECT * FROM repro WHERE NOT(col_a + col_b = 1);
-- Result: 0 rows

-- Correct behavior: NULL row is a failure
SELECT * FROM repro
WHERE NOT(col_a + col_b = 1) OR (col_a + col_b = 1) IS NULL;
-- Result: 1 row (the NULL row)
```

| Query | Failing rows returned |
|---|---|
| `WHERE NOT(col_a + col_b = 1)` | 0 (NULL silently passes) |
| `WHERE NOT(...) OR (... IS NULL)` | 1 (NULL correctly fails) |

### Seeded table contents (verified in Postgres)

```
 col_a | col_b
-------+-------
     1 |     0
     0 |     1
       |
(3 rows)
```

---

## Phase III Implementation Plan

| Step | Task | File(s) |
|---|---|---|
| 1 | Add `fail_on_null=False` argument to test signature | `macros/generic_tests/expression_is_true.sql` |
| 2 | Build `test_expression` once for both model-level and column-level branches | same |
| 3 | When `fail_on_null=true`, append `OR (test_expression) IS NULL` to the `WHERE` clause | same |
| 4 | Add integration seed with NULL rows | `integration_tests/data/schema_tests/data_test_expression_is_true_null.csv` |
| 5 | Add tests: default still passes; `fail_on_null: true` catches NULL rows (use existing `error_if: "<1"` pattern) | `integration_tests/models/generic_tests/schema.yml` |
| 6 | Document new argument in README | `README.md` |
| 7 | Run full Postgres integration suite: `make test target=postgres` | — |
| 8 | Open PR against `main`, link #1026, request review | GitHub |

### Design constraints

- **No breaking change:** default behavior stays the same (`fail_on_null=false`).
- **Both branches:** fix applies to model-level (`where not({{ expression }})`) and column-level (`where not({{ column_name }} {{ expression }})`) paths.
- **store_failures:** only the `SELECT` list differs when storing failures; the `WHERE` logic is the same.
- **CHANGELOG:** do not edit `CHANGELOG.md` directly — maintainers use auto-generated release notes per `CONTRIBUTING.md`.

### Proposed usage (Phase III)

```yaml
- dbt_utils.expression_is_true:
    expression: col_a + col_b = total
    fail_on_null: true
```

---

## Risks and Open Questions

| Item | Notes |
|---|---|
| Backward compatibility | Addressed by opt-in default; no existing tests should break |
| Column-level tests | Must verify `fail_on_null` on column tests (`expression: + col_b = 1`) |
| Cross-adapter behavior | `IS NULL` is standard SQL; should work on all adapters dbt-utils supports |
| Naming | `fail_on_null` chosen for clarity; open to maintainer feedback on PR |

---

## Phase II Completion Checklist

- [x] Fork and clone `dbt-labs/dbt-utils`
- [x] Stand up local dev environment (Docker Postgres + dbt)
- [x] Verify environment with `dbt debug`
- [x] Reproduce bug with seed data containing NULL rows
- [x] Document root cause (SQL three-valued logic + `WHERE NOT`)
- [x] Confirm maintainer direction (opt-in flag)
- [x] Write Phase III implementation plan
- [ ] Open PR (Phase III)
- [ ] Respond to maintainer review (Phase IV)

---

## References

- Issue: https://github.com/dbt-labs/dbt-utils/issues/1026
- Maintainer opt-in comment: https://github.com/dbt-labs/dbt-utils/issues/1026#issuecomment-4645503157
- Macro source: https://github.com/dbt-labs/dbt-utils/blob/main/macros/generic_tests/expression_is_true.sql
- Contributing guide: https://github.com/dbt-labs/dbt-utils/blob/main/CONTRIBUTING.md
- Integration test guide: https://github.com/dbt-labs/dbt-utils/blob/main/integration_tests/README.md
