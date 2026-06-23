# AI301 Open Source Capstone — Contribution Log

**Contributor:** Nguyen Nhat Quang Truong (`nnqtruong`)  
**Cohort:** AI301 | AI Open Source Capstone — Summer 2026, Section 1C  
**Status:** Phases I–IV Complete — PR open, awaiting maintainer review

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
