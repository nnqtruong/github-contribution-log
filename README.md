# AI301 Open Source Capstone — Contribution Log

**Contributor:** Nguyen Nhat Quang Truong (`nnqtruong`)  
**Cohort:** AI301 | AI Open Source Capstone — Summer 2026, Section 1C  
**Status:** Cycle 1 Phases I–IV Complete (PR #1095 open) · Cycle 2 started — discovered & fixed a separate `star` regression (PR #1097 open)

> Living engineering journal for my CodePath AI301 capstone contribution.  
> Updated every phase. Mentors and staff read this to know where I am and how to help.

---

## At a Glance

| Field | Value |
|---|---|
| **Project** | [`dbt-labs/dbt-utils`](https://github.com/dbt-labs/dbt-utils) |
| **Issue** | [#1026 — expression_is_true tests for not-false: should test for true](https://github.com/dbt-labs/dbt-utils/issues/1026) |
| **Pull Request** | [#1095 — Add fail_on_null option to expression_is_true generic test](https://github.com/dbt-labs/dbt-utils/pull/1095) |
| **Type** | Bug (behavior correction, opt-in) |
| **Language** | Jinja + SQL (dbt macros) |
| **My fork** | https://github.com/nnqtruong/dbt-utils |
| **Branch** | `issue-1026-expression-is-true-fail-on-null` |
| **Scope** | 1 macro + integration tests + README (4 files, +52 / −7 lines) |
| **PR state** | Open — `resolves #1026`, CI pending |
| **Cycle 2** | [#1096](https://github.com/dbt-labs/dbt-utils/issues/1096) → [PR #1097](https://github.com/dbt-labs/dbt-utils/pull/1097) — `star()` `rename` case-folding fix (see [Cycle 2](#cycle-2--star-rename-case-folding-fix-issue-1096--pr-1097) below) |

---

## Phase I — Issue Selection

### Issue
https://github.com/dbt-labs/dbt-utils/issues/1026

### Problem Summary
`dbt_utils.expression_is_true` is meant to assert that a SQL expression holds for every row. The macro finds failing rows with `where not({{ expression }})` in both of its branches (with and without `column_name`). Because SQL three-valued logic makes `not(NULL)` evaluate to NULL — which a WHERE clause drops — any row where the expression evaluates to NULL **passes silently** instead of failing. A test asserting "this must be true" should not be satisfied by an unknown value, so NULL rows should fail. "Fixed" means the test can also fail NULL results, in both branches, with integration tests proving it.

| Expression result | `NOT(expression)` | Included in `WHERE`? | Current test behavior |
|---|---|---|---|
| `TRUE` | `FALSE` | No | Pass (correct) |
| `FALSE` | `TRUE` | Yes | Fail (correct) |
| `NULL` | `NULL` | No | **Pass (bug)** |

The test checks **"not false"** rather than **"is true"**.

### Why I Chose This Issue

1. **On-domain for my target role.** My career direction is data analyst → data engineering / analytics engineering → quantitative dev. dbt is effectively synonymous with the analytics-engineering role, and dbt-utils is its core test/macro library — a merged change here maps directly to the resume story I want to tell.
2. **Chosen on a deliberate risk basis.** The pre-vetted issue list turned out high-contention: every strong candidate I evaluated in larger projects (Trino, Meltano) was already claimed, had an open PR closing it, or had stalled.
3. **Verified the lane is clear.** dbt-utils is a small repo (~60 open issues) where I could confirm availability: #1026 has no assignee, no linked PR, and its reporter is a drive-by account unlikely to build it themselves.
4. **Fits my strengths, no language gap.** It's SQL/Jinja, not Java — while still requiring real judgment. The fix is a NULL-semantics correctness question, the kind of edge-case reasoning that's genuinely mine to own, not something to generate.
5. **It's a bug, not a feature.** That avoids the design-debate limbo that stalls feature requests in this repo's triage queue.
6. **A nuance I surfaced while reading the code:** the change is a behavior change, not a pure fix — tests that currently pass on NULLs would begin to fail. So my opening comment asks the maintainers a design question (NULL-fails as default, or opt-in behind a config flag) rather than assuming a direction.

### Feasibility Guardrails
- **Phase II environment gate:** ✅ Completed — dbt-utils running locally against Postgres; NULL-passes-silently behavior reproduced.
- **Phase IV review gate:** This repo's Stale-bot is active (it has flagged similar issues at 180 days), which signals thin maintainer attention and real review latency. If my PR/comment is untouched ~7–10 days after posting, ping the thread once; if still cold, start Cycle 2 in parallel rather than wait.

### Phase I Checklist
- [x] GitHub account with professional username (`nnqtruong`)
- [x] Contribution README repo public
- [x] Issue #1026 selected — open, unassigned, no linked PR
- [x] Commented on issue introducing myself and asking design question
- [x] Fork created: https://github.com/nnqtruong/dbt-utils

---

## Phase II — Reproduce & Plan

### Understanding the Issue

Current macro (`macros/generic_tests/expression_is_true.sql`) — both branches use `where not(...)`:

```jinja
{% if column_name is none %}
where not({{ expression }})
{%- else %}
where not({{ column_name }} {{ expression }})
{%- endif %}
```

On a NULL expression result, `not(NULL)` → NULL → row excluded → row passes. Fix must cover both branches.

Generic tests in dbt return rows that **fail** the assertion. With `WHERE NOT(expr)`, only explicitly `FALSE` rows are returned. `NULL` rows are excluded because `NOT NULL` evaluates to `NULL`, and `WHERE NULL` filters the row out.

### Maintainer Direction

Before opening a PR, I commented on the issue asking whether the fix should change default behavior or be opt-in. Maintainer [@msh210 confirmed an opt-in approach](https://github.com/dbt-labs/dbt-utils/issues/1026#issuecomment-4645503157):

> "Good point, @nnqtruong, that people may be relying on it. So I suppose an opt-in is the way to go."

**Plan implication:** add a new argument (`fail_on_null`, default `false`) rather than changing existing behavior.

### Local Development Environment

| Component | Version / detail |
|---|---|
| OS | macOS (darwin 25.5.0, arm64) |
| Python | 3.12.4 |
| dbt-core | 1.13.0-a1 |
| dbt-postgres | 1.11.0-b1 |
| Database | Postgres 17.5 (`cimg/postgres:17.5`) via Docker |
| Repo setup | Forked `dbt-labs/dbt-utils`, local package linked via `integration_tests/packages.yml` (`local: ../`) |

**Setup steps:**

```bash
git clone https://github.com/nnqtruong/dbt-utils.git
cd dbt-utils

# Port 5433 used because 5432 was occupied by another container
docker run -d --name dbt-utils-postgres-repro \
  -e POSTGRES_USER=root \
  -e POSTGRES_PASSWORD=password \
  -e POSTGRES_DB=dbt_utils_test \
  -p 5433:5432 cimg/postgres:17.5

python3 -m venv env && source env/bin/activate
pip install --upgrade pip setuptools
pip install --pre dbt-postgres -r dev-requirements.txt

export POSTGRES_HOST=localhost POSTGRES_USER=root
export DBT_ENV_SECRET_POSTGRES_PASS=password POSTGRES_PORT=5433
export POSTGRES_DATABASE=dbt_utils_test
export POSTGRES_SCHEMA=dbt_utils_integration_tests_postgres

cd integration_tests
dbt deps --target postgres
dbt debug --target postgres   # → All checks passed!
```

**Environment challenges encountered:**
1. **Port 5432 in use** — another Postgres container (`logistics_project-db-1`) occupied the default port. Resolved by running a dedicated container on port 5433.
2. **Docker not running initially** — had to start Docker Desktop before containers would launch.

### Reproduction Process

**1. Created seed** `integration_tests/data/schema_tests/data_test_expression_is_true_null.csv`:

```csv
col_a,col_b
1,0
0,1
,
```

Row 1: `1 + 0 = 1` → TRUE (should pass)  
Row 2: `0 + 1 = 1` → TRUE (should pass)  
Row 3: `NULL + NULL = NULL` → NULL (should fail, but currently passes)

**2. Added test** in `integration_tests/models/generic_tests/schema.yml`:

```yaml
- name: data_test_expression_is_true_null
  data_tests:
    - dbt_utils.expression_is_true:
        expression: col_a + col_b = 1
```

**3. Ran reproduction:**

```bash
dbt seed --target postgres --select data_test_expression_is_true_null
dbt test --target postgres --select data_test_expression_is_true_null
```

**4. Result — test incorrectly PASSES:**

```
1 of 1 PASS dbt_utils_expression_is_true_data_test_expression_is_true_null_col_a_col_b_1
Done. PASS=1 WARN=0 ERROR=0
```

**Expected:** test should fail because row 3 has a NULL expression result.  
**Actual:** test passes. Bug confirmed.

**5. Compiled SQL (generated by dbt):**

```sql
select 1
from "dbt_utils_test"."dbt_utils_integration_tests_postgres"."data_test_expression_is_true_null"
where not(col_a + col_b = 1)
-- Returns 0 rows → NULL row silently passes
```

**6. Pure SQL confirmation (Postgres):**

```sql
CREATE TEMP TABLE repro AS
SELECT 1 AS col_a, 0 AS col_b
UNION ALL SELECT 0, 1
UNION ALL SELECT NULL::int, NULL::int;

SELECT * FROM repro WHERE NOT(col_a + col_b = 1);
-- Result: 0 rows (NULL silently passes)

SELECT * FROM repro
WHERE NOT(col_a + col_b = 1) OR (col_a + col_b = 1) IS NULL;
-- Result: 1 row (NULL correctly fails)
```

### UMPIRE Solution Plan

| Phase | Content |
|---|---|
| **Understand** | Root cause is SQL three-valued logic in `expression_is_true.sql`; both branches affected; maintainer wants opt-in |
| **Match** | Reviewed `accepted_range.sql` for boolean arg pattern; existing `data_test_expression_is_true` for test structure; `error_if: "<1"` pattern from `data_test_equality` |
| **Plan** | Add `fail_on_null=False`; unify `test_expression` variable; conditional `OR ... IS NULL`; integration tests; README; no CHANGELOG edit |
| **Implement** | _(Phase III)_ |
| **Review** | _(Phase III)_ |
| **Evaluate** | _(Phase III — integration tests + CI)_ |

### Phase III Implementation Plan

| Step | Task | File(s) |
|---|---|---|
| 1 | Add `fail_on_null=False` argument to test signature | `macros/generic_tests/expression_is_true.sql` |
| 2 | Build `test_expression` once for both branches | same |
| 3 | When `fail_on_null=true`, append `OR (test_expression) IS NULL` | same |
| 4 | Add integration seed with NULL rows | `integration_tests/data/schema_tests/data_test_expression_is_true_null.csv` |
| 5 | Tests: default passes; `fail_on_null: true` catches NULL (`error_if: "<1"`) | `integration_tests/models/generic_tests/schema.yml` |
| 6 | Document new argument in README | `README.md` |
| 7 | Run `make test target=postgres` | — |
| 8 | Open PR against `main`, link #1026 | GitHub |

**Design constraints:**
- No breaking change — default `fail_on_null=false`
- Both model-level and column-level branches
- `store_failures` — only SELECT list differs; WHERE logic identical
- Do not edit `CHANGELOG.md` — auto-generated release notes per CONTRIBUTING.md

### Phase II Checklist
- [x] Working branch in fork (`issue-1026-expression-is-true-fail-on-null`)
- [x] Environment setup documented with challenges resolved
- [x] Numbered reproduction steps with expected vs actual behavior
- [x] Root cause identified (not symptom) with specific files named
- [x] UMPIRE plan written
- [x] Maintainer direction confirmed (opt-in)

---

## Phase III — Build

### Implementation Summary

Implemented an opt-in `fail_on_null` argument for the `expression_is_true` generic test. When `fail_on_null: true`, rows where the expression evaluates to `NULL` are returned as failing rows. Default behavior unchanged per maintainer direction.

### Commits

| Hash | Date | Message |
|---|---|---|
| `1dd9da8` | 2026-06-22 | Add fail_on_null argument to expression_is_true generic test |
| `89bae34` | 2026-06-22 | Add integration tests for expression_is_true fail_on_null behavior |
| `d527437` | 2026-06-22 | Document fail_on_null option for expression_is_true in README |

### Files Modified

| File | Change |
|---|---|
| `macros/generic_tests/expression_is_true.sql` | Added `fail_on_null` arg; unified `test_expression`; conditional `OR ... IS NULL` |
| `integration_tests/data/schema_tests/data_test_expression_is_true_null.csv` | New seed: 2 valid rows + 1 NULL row |
| `integration_tests/models/generic_tests/schema.yml` | Tests for default + `fail_on_null: true` (model + column level) |
| `README.md` | Documented NULL behavior and `fail_on_null` usage |

### Macro Change (Core Fix)

```jinja
{% test expression_is_true(model, expression, column_name=None, fail_on_null=False) %}
  {{ return(adapter.dispatch('test_expression_is_true', 'dbt_utils')(model, expression, column_name, fail_on_null)) }}
{% endtest %}

{% if column_name is none %}
  {% set test_expression = expression %}
{% else %}
  {% set test_expression = column_name ~ ' ' ~ expression %}
{% endif %}

where not({{ test_expression }})
{%- if fail_on_null %}
 or ({{ test_expression }}) is null
{%- endif %}
```

### Testing Strategy

**New integration tests** (`schema.yml`):

```yaml
- name: data_test_expression_is_true_null
  data_tests:
    - dbt_utils.expression_is_true:
        expression: col_a + col_b = 1                          # default → PASS
    - dbt_utils.expression_is_true:
        expression: col_a + col_b = 1
        fail_on_null: true
        config:
          error_if: "<1"                                          # confirms NULL detected
          warn_if: "<0"
  columns:
    - name: col_a
      data_tests:
        - dbt_utils.expression_is_true:
            expression: + col_b = 1                              # default → PASS
        - dbt_utils.expression_is_true:
            expression: + col_b = 1
            fail_on_null: true
            config:
              error_if: "<1"                                      # column branch
              warn_if: "<0"
```

**Coverage:**

| Test case | Branch | Expected | Result |
|---|---|---|---|
| Default, NULL row present | Model-level | PASS (backward compat) | ✅ |
| `fail_on_null: true` | Model-level | NULL row detected | ✅ |
| Default, NULL row present | Column-level | PASS (backward compat) | ✅ |
| `fail_on_null: true` | Column-level | NULL row detected | ✅ |
| Existing `data_test_expression_is_true` | Both | Unchanged (no NULL rows) | ✅ expected |

**Manual test commands:**

```bash
source env/bin/activate
export POSTGRES_HOST=localhost POSTGRES_USER=root DBT_ENV_SECRET_POSTGRES_PASS=password
export POSTGRES_PORT=5433 POSTGRES_DATABASE=dbt_utils_test
export POSTGRES_SCHEMA=dbt_utils_integration_tests_postgres

cd integration_tests
dbt seed --target postgres --select data_test_expression_is_true_null
dbt test --target postgres --select data_test_expression_is_true_null
```

### Challenges Faced

**1. Port 5432 already in use**  
`docker compose up` failed — another Postgres container occupied port 5432. Started dedicated container on port 5433 with `POSTGRES_PORT=5433`.

**2. Testing NULL failures without breaking CI**  
A test that genuinely fails would break the integration suite. Used the project's existing `error_if: "<1"` pattern — test passes when failing rows are found (count ≥ 1), verifying the fix without failing CI.

**3. Backward compatibility requirement**  
Changing default behavior would break existing users. Followed maintainer opt-in direction; default `fail_on_null=False` preserves all existing behavior.

### Edge Cases Considered

| Edge case | Handling |
|---|---|
| Model-level vs column-level tests | Unified `test_expression` variable for both branches |
| `store_failures` mode | Only SELECT list differs; WHERE logic identical |
| NULL from partial NULL inputs (`NULL + 1`) | Covered by seed row 3 |
| Cross-adapter `IS NULL` syntax | Standard SQL; no adapter-specific override needed |
| Breaking existing tests | Default `false`; existing seeds have no NULL rows |

### Phase III Checklist
- [x] Branch with meaningful commits (3 scoped commits)
- [x] Descriptive commit messages (not wip/fix/asdf)
- [x] Diff scoped to issue (4 files, no unrelated changes)
- [x] New integration tests exercise the fix
- [x] Tests follow existing project patterns
- [x] README updated
- [x] Challenges documented
- [x] Ready for PR submission

---

## Phase IV — Pull Request & Review

### PR Link
**https://github.com/dbt-labs/dbt-utils/pull/1095**

| Field | Detail |
|---|---|
| **Target** | `dbt-labs/dbt-utils` → `main` |
| **Head** | `nnqtruong:issue-1026-expression-is-true-fail-on-null` |
| **Issue link** | `resolves #1026` |
| **State** | Open — awaiting maintainer review |
| **Opened** | 2026-06-23 |
| **Files changed** | 4 (+52 / −7 lines) |
| **Issue comment** | https://github.com/dbt-labs/dbt-utils/issues/1026#issuecomment-4774861704 |

### PR Summary

**Why:** `expression_is_true` compiles to `WHERE NOT(expression)`. In SQL three-valued logic, `NOT NULL` → `NULL`, and rows where the WHERE clause is NULL are filtered out. NULL rows silently pass instead of failing. Maintainer confirmed opt-in approach to avoid breaking existing users.

**What:**
- Added optional `fail_on_null` argument (default `false`)
- When `true`, appends `OR (expression) IS NULL` to catch NULL rows as failures
- Applies to both model-level and column-level test branches
- Integration tests + README documentation

**Example usage:**

```yaml
- dbt_utils.expression_is_true:
    expression: col_a + col_b = total
    fail_on_null: true
```

### Before / After Evidence

**Before (bug — default behavior, NULL row present):**

```
$ dbt test --select data_test_expression_is_true_null
→ PASS (0 failing rows; NULL silently passes)

Compiled: where not(col_a + col_b = 1)
Pure SQL:  SELECT * FROM repro WHERE NOT(col_a + col_b = 1);  -- 0 rows
```

**After (fix — `fail_on_null: true`, same data):**

```
Compiled: where not(col_a + col_b = 1) or (col_a + col_b = 1) is null
Pure SQL:  SELECT * FROM repro WHERE NOT(...) OR (... IS NULL);  -- 1 row (NULL row)
Integration test with error_if: "<1" → PASS (confirms NULL detected)
```

### Plan vs. Built Alignment

| Phase II plan | Built in PR #1095 | Match? |
|---|---|---|
| Add `fail_on_null=False` argument | ✅ Test signature + macro | Yes |
| Update both WHERE branches | ✅ Unified `test_expression` | Yes |
| Opt-in, not breaking default | ✅ Default `false` | Yes |
| Integration seed with NULL rows | ✅ `data_test_expression_is_true_null.csv` | Yes |
| Tests for default + fail_on_null | ✅ Model + column level | Yes |
| README documentation | ✅ Added section with example | Yes |
| No CHANGELOG edit | ✅ Skipped per CONTRIBUTING.md | Yes |

### Maintainer Feedback Log

| Date | Reviewer | Feedback | Response | Commit |
|---|---|---|---|---|
| 2026-06-08 | @msh210 (issue) | Opt-in is the way to go — don't break default | Shaped entire implementation plan | — |
| 2026-06-23 | — | PR opened; awaiting first code review | Commented on issue with PR link | — |

> Update this table as maintainer feedback arrives. Phase IV credit does not require merge — a review-ready PR actively in review counts as completion.

### Learnings & Reflections

**What I learned about the codebase:**
- dbt-utils generic tests return **failing rows**; macro logic is inverted (`WHERE NOT`) to find violations.
- The `error_if: "<1"` pattern asserts a test *finds* failures without breaking CI.
- Local package development uses `integration_tests/packages.yml` with `local: ../`.

**What I learned about open source contribution:**
- Asking maintainers about breaking vs opt-in design **before** coding saved rework.
- Reading CONTRIBUTING.md matters — avoided editing CHANGELOG.md (auto-generated release notes).
- Scoping commits by concern (macro → tests → docs) makes review easier.

**What I would do differently:**
- Start Docker earlier and confirm port availability before debugging connection errors.
- Run full `make test target=postgres` locally before opening the PR.
- @mention a maintainer on the PR after CI results.

**Teachable insight:**  
SQL three-valued logic (`TRUE`, `FALSE`, `NULL`) is a common source of silent bugs in data tests. Any test using `WHERE NOT(condition)` inherits this behavior. When writing or reviewing dbt generic tests, always ask: **"What happens when the condition is NULL?"**

### Phase IV Checklist
- [x] PR open against upstream `main` (not fork-internal)
- [x] PR follows project template (Problem / Solution / Checklist / Test plan)
- [x] `resolves #1026` in description
- [x] Before/after evidence included
- [x] Issue comment with PR link posted
- [x] Plan aligns with implementation
- [x] Learnings & reflections written
- [ ] Maintainer code review received
- [ ] CI green
- [ ] PR merged

> **Second contribution cycle started** (rubric: "The Full Open Source Loop") — while monitoring this PR's CI I found, reported, and fixed a separate regression. Full write-up in [Cycle 2](#cycle-2--star-rename-case-folding-fix-issue-1096--pr-1097) below.

---

## Cycle 2 — `star()` rename case-folding fix (Issue #1096 → PR #1097)

> A full Phase I–IV loop on a second bug, opened while watching Cycle 1's CI: **discovered → reported → reproduced → fixed → PR**. Self-found regression, not a pre-vetted list issue.

### At a Glance (Cycle 2)

| Field | Value |
|---|---|
| **Issue** | [#1096 — `star()` `rename` ignored on case-folding warehouses](https://github.com/dbt-labs/dbt-utils/issues/1096) (I reported it) |
| **Pull Request** | [#1097 — fix(star): match `rename` keys case-insensitively](https://github.com/dbt-labs/dbt-utils/pull/1097) |
| **Type** | Bug (regression introduced in #1094) |
| **Branch** | `fix-star-rename-case-insensitivity` |
| **Scope** | 1 macro + 1 integration test + README (3 files, +18 / −3 lines) |
| **PR state** | Open — `resolves #1096` |

### Phase I — Discovery & Issue Selection

**How I found it.** Watching CI on my Cycle 1 PR (#1095), the **Snowflake** job was red — but on a test I never touched: `assert_equal_test_star_unquote_aliases_actual__expected`, part of the `star` macro. I traced it: all four of my `expression_is_true` tests passed; the failing test belongs to #1094 (`feature/star-unquote-alias-rename`), merged to `main` on 2026-06-18. Because PR CI runs against the branch *merged into* `main`, my PR inherited a failure that already exists on `main` — confirmed by #1094's own post-merge run showing the identical `FAIL 2`.

**Why I chose it.** A real, unowned regression (no existing issue or PR), in the same Jinja/SQL identifier-semantics lane as Cycle 1, small enough to own end-to-end. Fixing it restores clean CI signal for the whole repo, not just my PR.

**Issue filed:** [#1096](https://github.com/dbt-labs/dbt-utils/issues/1096) with root-cause analysis and a reproduction.

### Phase II — Reproduce & Plan

**Root cause.** `star()`'s `rename` (added in #1094) looks up aliases case-sensitively:

```jinja
{%- if col in rename %} as {{ rename[col] }}
```

`get_filtered_columns_in_relation()` returns column names as the warehouse stores them. Snowflake folds unquoted identifiers to UPPERCASE (`COLUMN_ONE`), but the test/user passes a lowercase key (`{"column_one": ...}`), so `col in rename` is always false → the rename is silently dropped. The two rename rows in `test_star_unquote_aliases` mismatch → `FAIL 2` on Snowflake. Lowercase-folding adapters (Postgres / Redshift / BigQuery) match by luck, so the bug is invisible there.

**Analogous pattern in the codebase.** The same macro family already solves this for `except`, in `get_filtered_columns_in_relation`:

```jinja
{%- set except = except | map("lower") | list %}
{%- if col.column|lower not in except -%}
```

So the fix mirrors it: normalize `rename` keys to lowercase and compare `col | lower`.

**Reproduction (local, Postgres).** I can't run Snowflake locally, so I reproduced the *same class* of failure on Postgres by supplying a rename key whose case differs from the stored column (`{"COLUMN_ONE": "renamed_col"}` against a `column_one` column). Against the buggy macro, `dbt test --select test_star_unquote_aliases` → `FAIL 1` ("Got 1 result"). This proves the bug is about case-matching, not Snowflake specifically.

**UMPIRE plan:**

| Phase | Content |
|---|---|
| **Understand** | Case-sensitive `rename` lookup vs adapter identifier folding; both `quote_identifiers` branches affected |
| **Match** | `except` normalization in `get_filtered_columns_in_relation` (`map("lower")`) |
| **Plan** | Build a lowercased `rename_lookup`; compare `col \| lower`; add a cross-adapter regression row; document in README |
| **Implement** | Phase III below |
| **Review** | Local Postgres run + PR against `main` |
| **Evaluate** | Regression row fails on buggy macro, passes on fix; full `star` suite `PASS=5` |

### Phase III — Build

**Commits:**

| Hash | Message |
|---|---|
| `c2afa53` | fix(star): match `rename` keys to column names case-insensitively |
| `f29cefe` | test(star): cover case-insensitive `rename` matching |
| `bb78cf2` | docs(star): note case-insensitive `rename` matching |

**Files modified:**

| File | Change |
|---|---|
| `macros/sql/star.sql` | Build a lowercased `rename_lookup`; match `col \| lower` in both branches |
| `integration_tests/models/sql/test_star_unquote_aliases.sql` | New row: uppercase rename key vs lowercase column |
| `README.md` | Note that `rename` keys match case-insensitively |

**Macro change (core fix):**

```jinja
{% set cols = dbt_utils.get_filtered_columns_in_relation(from, except) %}

{#-- Match `rename` keys case-insensitively, consistent with `except`. #}
{%- set rename_lookup = {} -%}
{%- for rename_key, rename_value in rename.items() -%}
    {%- do rename_lookup.update({rename_key | lower: rename_value}) -%}
{%- endfor -%}
...
{%- if col | lower in rename_lookup %} as {{ rename_lookup[col | lower] }}
```

**Testing:**

| Step | Result |
|---|---|
| New regression row vs **buggy** macro (Postgres) | `FAIL 1` — bug reproduced |
| New regression row vs **fixed** macro (Postgres) | `PASS` |
| Full `star` suite (5 tests) vs fixed macro (Postgres) | `PASS=5` |

**Challenge faced — CRLF line endings.** `star.sql` is the one file in the repo with Windows CRLF terminators (introduced by #1094); the rest uses LF. My first edit normalized it to LF, which made `git diff` show the *entire file* as changed — exactly the "unrelated formatting" noise reviewers reject. I restored the original CRLF blob and re-applied only the three logical changes with line-ending-preserving edits, so the diff is 8 lines on the macro, not 99.

**Edge cases considered:**

| Edge case | Handling |
|---|---|
| Both `quote_identifiers` branches | Fixed both `col in rename` sites |
| Lowercase-folding adapters | `col \| lower` against lowercased keys = no behavior change |
| Two keys differing only in case | Last-write-wins in `rename_lookup` (same as a plain dict) |
| `rename` precedence over `unquote_aliases` / prefix / suffix | Preserved — still the first `if` branch |

### Phase IV — Pull Request

**PR:** [#1097](https://github.com/dbt-labs/dbt-utils/pull/1097) — `resolves #1096`, open against `dbt-labs/dbt-utils:main`, follows the project template (Problem / Solution / Checklist).

**Before / after (Postgres, verified locally):**

```
key {"COLUMN_ONE": "renamed_col"} on a column_one column
Before:  FAIL 1  — compiled to "column_one"            (no alias, rename dropped)
After:   PASS    — compiled to "column_one" as renamed_col
Full star suite: PASS=5
```

**Plan vs. built:**

| Plan | Built | Match? |
|---|---|---|
| Lowercased `rename_lookup`, compare `col \| lower` | ✅ both branches | Yes |
| Mirror `except` convention | ✅ | Yes |
| Cross-adapter regression row | ✅ uppercase-key row | Yes |
| README note | ✅ | Yes |
| Minimal diff, no formatting churn | ✅ CRLF preserved, 3 files | Yes |

**Cross-PR note.** Left a [comment on Cycle 1's PR (#1095)](https://github.com/dbt-labs/dbt-utils/pull/1095#issuecomment-4775047314) explaining the Snowflake red is pre-existing and now tracked by #1096 / #1097, so it doesn't block that review.

**Teachable insight (Cycle 2).** Identifier case-folding differs by warehouse (Snowflake → upper, most others → lower). Any macro that matches a *user-supplied* column name against `get_columns_in_relation()` output must normalize case — `dbt-utils` already does this for `except`, and new column-name arguments should follow the same rule. A test that only runs on lowercase-folding adapters will hide the bug.

### Phase IV Checklist (Cycle 2)
- [x] Issue filed and triaged-ready (#1096) before the PR, per CONTRIBUTING.md
- [x] PR open against upstream `main` (not fork-internal), `resolves #1096`
- [x] PR follows project template; checklist filled
- [x] New regression test that fails on the bug and passes on the fix
- [x] Before/after evidence included (local Postgres)
- [x] README updated
- [x] Diff scoped to the fix (3 files, CRLF preserved)
- [ ] Maintainer code review received
- [ ] CI green
- [ ] PR merged

---

## AI Usage Log

> Where I used AI, what I asked, and how I verified. I own every line; "the AI suggested it" is not an answer to a reviewer.

**Tools:** Claude (web) for issue-selection strategy; Cursor for environment setup, reproduction, implementation scaffolding, and report drafting. Reading + verification done by me against source.

| Date | Task | What AI did | How I verified | Reflection |
|---|---|---|---|---|
| 2026-06-07 | Issue selection | Helped filter candidates by contention/scope/domain; reverse-engineered the curated-list filter to find low-contention issues | Read each candidate's live issue page myself; confirmed #1026 unassigned, no linked PR | The contention analysis was the real value; final pick was my call |
| 2026-06-07 | Claim comment | Drafted a comment framing the NULL-semantics design question | Read `expression_is_true.sql` myself; rewrote the comment in my own voice before posting | Owning the NULL reasoning matters — maintainer may push back |
| 2026-06-07 | Code reading | Flagged that the bug affects *both* WHERE branches and the `should_store_failures()` wrinkle | Confirmed both branches in source; verified the store-failures path | Caught a gap in the original issue report |
| 2026-06-14 | Environment setup | Guided Docker + dbt-postgres setup; diagnosed port 5432 conflict | Ran `dbt debug` myself; confirmed connection on port 5433 | Port conflict was environmental, not project-specific |
| 2026-06-14 | Bug reproduction | Drafted seed file and schema.yml test config; pure SQL repro script | Ran `dbt seed` + `dbt test` myself; verified 0 failing rows on NULL data | Reproduction output is my evidence — AI didn't run the commands |
| 2026-06-14 | Implementation plan | Proposed `fail_on_null` naming and `error_if` test pattern | Cross-checked against `accepted_range.sql` and existing integration tests | Pattern matching to codebase conventions was the key decision |
| 2026-06-22 | Implementation | Scaffolded macro change, integration tests, README section | Reviewed every line; confirmed both branches; checked diff scope (4 files only) | I own the Jinja logic and test design |
| 2026-06-22 | PR description | Drafted PR body following project template | Rewrote in my voice; verified `resolves #1026`; posted myself | PR description reflects my understanding of the tradeoff |
| 2026-06-22 | Contribution reports | Drafted Phase II–IV report sections | Cross-checked against actual terminal output, commit hashes, PR URL | Reports are portfolio artifacts — must match what actually happened |
| 2026-06-22 | Cycle 2 — CI triage | Helped read the failing Snowflake CI logs and isolate the failing test to the `star` macro | Pulled the logs myself (`gh run view --log-failed`); confirmed the same `FAIL 2` on `main`'s own post-#1094 run | The failure was *not* mine — verifying it on `main` before acting was the key step |
| 2026-06-22 | Cycle 2 — root cause | Pointed at the case-sensitive `col in rename` lookup and the `except` precedent | Read `star.sql` and `get_filtered_columns_in_relation.sql` myself; traced Snowflake upper-case folding | The `except` analogy made the correct fix obvious — match the existing convention |
| 2026-06-22 | Cycle 2 — implementation | Scaffolded the `rename_lookup` macro change, the regression test row, and README note | Ran the regression on Postgres against both the buggy and fixed macro myself (`FAIL 1` → `PASS`); kept the diff to 3 files | I own the Jinja and the test design; the CRLF-preservation call was mine to keep the diff clean |
| 2026-06-22 | Cycle 2 — issue/PR/log | Drafted #1096, #1097, the #1095 CI note, and this Cycle 2 write-up | Verified every link, commit hash, and the `PASS=5` / `FAIL 1` outputs against my own terminal before posting | These are public, graded artifacts — they must match what actually ran |

---

## References

- Issue: https://github.com/dbt-labs/dbt-utils/issues/1026
- PR: https://github.com/dbt-labs/dbt-utils/pull/1095
- Fork: https://github.com/nnqtruong/dbt-utils
- Branch: https://github.com/nnqtruong/dbt-utils/tree/issue-1026-expression-is-true-fail-on-null
- Maintainer opt-in comment: https://github.com/dbt-labs/dbt-utils/issues/1026#issuecomment-4645503157
- PR announcement comment: https://github.com/dbt-labs/dbt-utils/issues/1026#issuecomment-4774861704
- Macro source: https://github.com/dbt-labs/dbt-utils/blob/main/macros/generic_tests/expression_is_true.sql
- Contributing guide: https://github.com/dbt-labs/dbt-utils/blob/main/CONTRIBUTING.md
- Integration test guide: https://github.com/dbt-labs/dbt-utils/blob/main/integration_tests/README.md

### Cycle 2 — `star()` rename case-folding fix
- Issue: https://github.com/dbt-labs/dbt-utils/issues/1096
- PR: https://github.com/dbt-labs/dbt-utils/pull/1097
- Branch: https://github.com/nnqtruong/dbt-utils/tree/fix-star-rename-case-insensitivity
- Regression introduced by: https://github.com/dbt-labs/dbt-utils/pull/1094
- Cross-PR CI note on #1095: https://github.com/dbt-labs/dbt-utils/pull/1095#issuecomment-4775047314
- Macro source: https://github.com/dbt-labs/dbt-utils/blob/main/macros/sql/star.sql
- `except` precedent: https://github.com/dbt-labs/dbt-utils/blob/main/macros/sql/get_filtered_columns_in_relation.sql
