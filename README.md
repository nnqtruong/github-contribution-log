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