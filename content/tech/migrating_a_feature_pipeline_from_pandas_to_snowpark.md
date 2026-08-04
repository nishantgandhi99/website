---
title: "Migrating a Feature Pipeline from Pandas to Snowpark: Why Validation Discipline Matters More Than Code Generation"
date: 2026-04-17
draft: false
author: "Nishant M Gandhi"
summary: "Rewriting a feature extraction pipeline from Oracle/pandas to Snowpark in 2 weeks. 85%+ latency reduction. Complete data parity. The key: validation discipline beats code generation speed."
tags: ["snowpark", "migration", "data-engineering", "validation", "enterprise"]
---

## The Problem

We had to migrate our feature extraction pipeline from Oracle/pandas to Snowpark. I was doing it solo.

The current system was struggling. Every month it ran for 2+ hours, generating thousands of intermediate files and making ~700 SQL round trips just to process one batch. Tens of thousands of Python loops. This was the pipeline the entire business intelligence team depended on every week.

But the real reason we couldn't keep using it: we wanted to build a feature store. A centralized system where features get computed once, versioned, and reused across the ML organization. You can't build that on top of a 2-hour pipeline. You need speed. You need to iterate. You need to scale.

So we had to migrate. Solo. Complete parity required.

The constraint wasn't speed - Copilot made the code generation fast. The constraint was validation. Making sure every change was correct before shipping it.

---

## What Actually Worked

I didn't try to be perfect from the start. That would have killed us.

Instead I built one thing first: a comparison script. Before touching Snowpark. Before any AI. Just a Python script that could load both outputs and compare them row by row, column by column.

Then I spent the first week rewriting the pipeline with Copilot. It was rough. Copilot generated code that looked correct - code that would pass code review, pass syntax checks, generate output that looked reasonable. But it was wrong in ways that wouldn't surface until we compared it.

Week two was refinement. Every iteration, I'd run the comparison. It would show me exact rows where we diverged. Tell Copilot: "Row 204 is wrong here." Make a change. Run the comparison again. Iterate.

After the migration went live, I spent a month running both systems in parallel - shadow mode. New data every week. Both pipelines running. Both outputs compared. Looking for anything that diverged.

Migrations don't fail because the ML is wrong. They fail because you ship something that looks right but is subtly broken. The only defense against that is measurement.

---

## Where Copilot Faltered (And Why)

Copilot was useful for translating patterns. Show it pandas code, get Snowpark equivalent back. The problem: 80% correct is 0% safe.

The code looked right. It passed syntax checks. It generated output. None of it would fail a code review. But none of it was exactly correct. Here's what I found when I actually compared the outputs:

### Blind Spot 1: Missing Calculation Steps

```python
# Original pandas code
df_pivot = df.pivot_table(
    index='entity_id',
    columns='category',
    values='amount',
    aggfunc='first'    # CRITICAL: take FIRST value per group
)

# Copilot generated
df_sp = df_sp.groupby(['entity_id', 'category']).agg(
    F.max('amount')    # WRONG: takes MAX, not FIRST
)
```

Copilot skipped the row-numbering logic that selects the correct row.

### Blind Spot 2: Type Assumption Errors

Copilot assumed Snowpark types match pandas. They don't.
- Snowpark Int64 != pandas int32
- Snowpark Decimal != pandas float64
- Result: silent precision loss, silent data mutations

### Blind Spot 3: Cohort Filtering

Original pipeline: set intersection between two entity cohorts before aggregation.
Copilot: completely forgot. Generated extra rows.

### Blind Spot 4: The NaN Trap

```python
# Original pandas code - takes the LAST row always
df = pd.DataFrame({
    'account': ['A', 'A', 'A'],
    'date': ['2025-04-10', '2025-04-15', '2025-04-30'],
    'value': [0.10, 0.42, np.nan]
})

# What we want: the last row, even if it's NaN
df.iloc[-1]  # Returns: value = NaN (CORRECT)

# What Copilot gave us: skips the NaN, returns earlier value
df.groupby('account').last()  # Returns: value = 0.42 (WRONG)
```

`groupby().last()` returns the last non-NaN value, not the last row. If an entity's most recent observation is NaN, this silently returns stale data. Breaks any logic depending on data recency.

None of these would throw errors. None would fail code review. All would generate plausible-looking output. Validation caught every single one.

---

## The Validation Loop: Not Pretty, But Effective

Before writing any Snowpark code, I built one thing.

```python
# compare_outputs.py - load both runs, compare cell-by-cell
def compare_outputs(run1, run2):
    # Load output files from both systems
    # Compare every row, every column
    # Print row numbers + percentage of mismatches per column
    # Flag dtype mismatches (and whether data is actually identical)
    pass
```

That's it. Simple script. But it changed everything.

Every time Copilot generated code, I'd run this. It would show me exactly where we broke.

**Iteration 1 (Copilot's first shot):**
```
feature_calc_A: 791 mismatches (0.92%)
  Row 204: -0.07 vs -0.05
  Row 481: -0.36 vs -0.07
feature_calc_B: 762 mismatches (0.88%)
feature_calc_C: 2499 mismatches (2.89%)
```

Now I had something to work with. Trace row 204 through both pipelines. Figure out why it's different. Tell Copilot: "Fix feature_calc_C; it's wrong at row 204."

**Iteration 2:** Apply missing entity intersection filter. Mismatches drop to 800.

**Iteration 3:** Fix groupby().last() NaN bug. Down to 200 mismatches.

**Iteration 4:** Fix type system (keep Int64, Decimal native). Down to 2 mismatches (OK, just data freshness).

**Iteration 5:** Complete parity.

5-6 tight iteration cycles beat one perfect attempt every time. I was measuring every change. Every iteration told me if I was getting closer or farther away.

---

## Eight Things That Nearly Broke This Migration

Getting complete parity on tens of thousands of rows across many output files sounds simple. It wasn't. Here's what I discovered along the way - some of these nearly shipped to production broken.

### Column Name Casing

Snowflake auto-uppercases all unquoted SQL identifiers. Oracle returned `Num_Feature_A_Cat1`. Snowflake returned `NUM_FEATURE_A_CAT1`. Downstream code did `df['Num_Feature_A_Cat1']` and crashed with `KeyError`. I found this in the second iteration of the comparison.

Apply a column name normalization mapping after every Snowflake materialization:

```python
column_mapping = {
    "NUM_FEATURE_A_CATEGORY1":       "Num_Feature_A_Category1",
    "NUM_FEATURE_B_CATEGORY1":       "Num_Feature_B_Category1",
    "RATING_MAX_CATEGORY2":          "Rating_Max_Category2",
    # ... 20+ more mappings
}
df = df.rename(columns={k: v for k, v in column_mapping.items() if k in df.columns})
```

### Date Type Mismatches

Oracle returns dates as Python strings (`'2025-04-15'`). Snowflake materializes dates as `datetime64[ns]`.

Downstream code that did `df['business_date'].str.split('-')` threw `AttributeError`. I wasted two hours on this because the error only showed up when downstream code tried to use the dates.

Post-materialization normalization handles this:

```python
date_columns = ['business_date', 'calendar_date', ...]
for col in date_columns:
    if pd.api.types.is_datetime64_any_dtype(df[col]):
        df[col] = df[col].dt.strftime("%Y-%m-%d")
```

### Type Precision Loss During Materialization

This one haunted me. Numeric types with different precision requirements lose accuracy when materializing across systems - Decimal to float64 being the common case. The result is silent output format incompatibilities downstream.

I found it when comparing row 427 and noticed the decimal field was off by $0.02. Seemed like a rounding issue. It wasn't.

Normalize precision after materialization, selectively, only where exact representation is required:

```python
if requires_exact_precision(column):
    df[column] = df[column].apply(
        lambda x: normalize_precision(x) if pd.notna(x) else None
    )
```

### The NaN Trap

This one broke parity silently and I almost shipped it.

```python
# Original pandas behavior (what business logic expects):
Account 001, sorted by date:
  2025-04-15: Feature_X = 0.42
  2025-04-30: Feature_X = NaN    # most recent observation

iloc[-1] returns: NaN   # CORRECT - preserves the latest observation

# Snowpark groupby().last() behavior:
groupby().last() returns: 0.42   # WRONG - skips trailing NaN, returns earlier non-NaN
```

If an account's most recent date has NaN in a feature column, `groupby().last()` silently substitutes an earlier non-NaN value. This breaks any business logic that depends on the *recency* of data, not the *presence* of data.

Use `sort_values + drop_duplicates(keep='last')` instead:

```python
def _vectorized_last(df: pd.DataFrame) -> pd.DataFrame:
    return (
        df
        .sort_values([entity_id, timestamp_col])
        .drop_duplicates(subset=[entity_id], keep='last')  # takes last row, always
        .reset_index(drop=True)
    )
```

This always selects the final row, regardless of NaN content. 30x faster than per-account loops, parity-preserving.

### Entity Intersection Logic

The original pipeline applies a set intersection between two entity sets before aggregation. Copilot missed it entirely. Snowpark produced extra rows that shouldn't exist.

I found this when the output row counts didn't match. Spent 30 minutes tracing why. The filter was just missing entirely.

Make the filter explicit:

```python
set_cohort_a = set(df_cohort_a[entity_id].unique())
set_cohort_b = set(df_cohort_b[entity_id].unique())
set_inter = set_cohort_a.intersection(set_cohort_b)

df_cohort_a = df_cohort_a[df_cohort_a[entity_id].isin(set_inter)]
df_cohort_b = df_cohort_b[df_cohort_b[entity_id].isin(set_inter)]
```

### Dynamic Pivot Without Native PIVOT Support

Snowflake's native `PIVOT` syntax requires knowing column values at query design time. Here the category values are data-driven and change across time periods. Hardcoding them defeats the purpose.

I tried PIVOT first. Didn't work. Had to rethink the approach.

Write to a temp table, query distinct values dynamically, then generate `CASE WHEN` SQL:

```python
# Step 1: Write to temp table
df_sp.write.mode("overwrite").save_as_table(temp_table_name, table_type="temporary")

# Step 2: Discover values
unique_names = [row["NAME"] for row in
    session.sql(f"SELECT DISTINCT NAME FROM {temp_table_name}").collect()]

# Step 3: Generate dynamic SQL
case_statements = [
    f"MAX(CASE WHEN NAME = '{name}' THEN value END) AS {name}_col"
    for name in unique_names
]
pivot_sql = f"SELECT entity_id, {', '.join(case_statements)} FROM {temp_table_name} GROUP BY entity_id"
return session.sql(pivot_sql)  # still lazy
```

Cost: Two round trips instead of one. Worth it for correctness.

### HTTP 503 Transient Errors on Large Result Sets

Snowflake's result batch service occasionally returns HTTP 503 for large result sets. It's a transient infrastructure issue, not a query error - but the connector doesn't distinguish it from a real failure.

This happened to me at 2 AM during a validation run. Thought the code was broken. It wasn't - Snowflake was just overloaded.

Wrap `to_pandas()` in exponential backoff:

```python
def _fetch_pandas_chunked(df_sp, query_name, max_retries=5):
    backoff = 1.0
    for attempt in range(max_retries):
        try:
            return df_sp.to_pandas()
        except Exception as e:
            if "503" in str(e) or "Service Unavailable" in str(e):
                time.sleep(backoff)
                backoff = min(backoff * 2, 30.0)  # cap at 30 seconds
                attempt += 1
            else:
                raise  # non-transient error - fail immediately
```

### Type System Discipline - Don't Force Conversions

My first instinct was to "match Oracle types" - convert Snowpark's native Int64 to int32, Decimal to float64. Get everything compatible.

This was wrong. Dead wrong. I lost precision. Introduced silent NaN differences. Coupled the new pipeline to Oracle's schema.

```python
# WRONG: Forcing Oracle types
result = result.with_column(
    F.col("numeric_id_col").cast("int32"),  # Snowpark > Oracle type
    F.col("numeric_precision_col").cast("float32")
)
```

That's a category error. You're mutating data to fit a type system that has nothing to do with the target environment.

Keep Snowpark's native types flowing through. Normalize *only at boundaries* - comparison and serialization:

```python
# RIGHT: Preserve Snowpark types through processing
# Int64 stays Int64 (not int32)
# Decimal stays Decimal (not float64)
# Only normalize in _normalize_dataframe_for_parity() - called AFTER
# materialization to pandas, only for testing/comparison purposes.
```

This eliminates systematic precision drift.

---

## The Results

After 5-6 iteration cycles and the type system fix, validation passed. Complete data parity.

**Before (Oracle/pandas legacy system):**
- Latency: 2+ hours per run
- Data ingress: 20+ GB
- Network round trips: ~700+ SQL queries
- Intermediate files: thousands per run

**After (Snowflake/Snowpark):**
- Latency: 15-20 minutes per run
- Data ingress: ~8 GB
- Network round trips: ~20-30 SQL queries
- Intermediate files: ~20 per run

That's 85%+ latency reduction. Practitioners iterate significantly faster. Infrastructure costs dropped 60% in data ingress alone. 95% fewer intermediate files to debug. And - most importantly - zero production incidents.

We shipped it. It worked.

---

## What I Learned

The 2-week timeline wasn't a result of speed. It was a result of never shipping a change without measuring it.

Most of the hardest bugs in this migration weren't the ones that threw errors. They were the ones that produced output - output that looked right, that would pass a code review, that didn't surface until you compared it row by row against the original. `groupby().last()` returns a value. It's just not the right value.

Build the comparison script before you write any migration code. Let every iteration answer one question: is the output closer to correct than the last one? Everything else follows from that.

That's it. That's the whole playbook.