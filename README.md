# Polars Style Guide

Polars is a fast, open-source DataFrame library for Python. Its core is written in Rust and built on the Apache Arrow columnar memory format, so it is both quick and memory-efficient. This opinionated guide presents best practices for writing Polars expressions that are readable and efficient. It is inspired by the [PySpark Style Guide](https://github.com/palantir/pyspark-style-guide) developed by Palantir.

# Prefer `pl.col` syntax over direct access

Mixing plain column-name strings with `pl.col(...)` expressions in the same call leads to inconsistent code. Plain strings work fine in a simple `select`, but as soon as one column needs a transformation, like a cast, the rest of the list ends up in a different style. Wrapping every column in `pl.col` keeps the call uniform: every item is an expression, and adding a transformation later is a small change rather than a rewrite.

```python
# bad
df.select(
    'name',
    pl.col('age').cast(pl.Int64()),
    'address',
    'phone'
)

# better
df.select(
    pl.col('name'),
    pl.col('age').cast(pl.Int64()),
    pl.col('address'),
    pl.col('phone')
)
```

# Prefer functions for logical groups of expressions

When a set of expressions belongs together, such as cleaning a group of related columns or deriving a family of flags, wrap them in a named function that takes a DataFrame and returns one. The function name documents intent, the group can be tested on its own, and the calling pipeline reads like a list of steps. Keep each function focused on one logical step so it stays easy to reason about.

```python
# inline
orders_with_totals = orders_df.with_columns(
    (pl.col("price") * pl.col("quantity")).alias("total"),
    pl.col("name").str.strip_chars(),
)

# function
def add_order_total(df: pl.DataFrame) -> pl.DataFrame:
    return df.with_columns(
        (pl.col("price") * pl.col("quantity")).alias("total"),
        pl.col("name").str.strip_chars(),
    )

orders_with_totals = orders_df.pipe(add_order_total)
```

# Use `pipe` expressions to combine transformation functions

`DataFrame.pipe` passes a DataFrame into a function and returns the result, which lets you chain transformation functions from top to bottom, like a recipe. Each step stays a named unit, and you don't need intermediate variables to hold the state between them.

```python
def filter_active(df: pl.DataFrame) -> pl.DataFrame:
    return df.filter(pl.col("status") == "active")

# bad
result = add_order_total(filter_active(orders_df))

# good
result = (
    orders_df
    .pipe(filter_active)
    .pipe(add_order_total)
)
```

# Choose between `pipe` and direct function calls to illustrate data flow when joining dataframes

When joining DataFrames, consider using `pipe` when a minor table is joined onto a major table and the major table is not fundamentally changed by the join. Consider using the direct function invocation when both tables are of equal importance and the result is a qualitatively different table.

```python
# major/minor 
def add_names(transactions: pl.DataFrame, names: pl.DataFrame) -> pl.DataFrame:
    return transactions.join(names, on='customer_id', how='left')

transaction_df.pipe(add_names, names=name_df)

# equal
def merge_sources(internal: pl.DataFrame, vendor: pl.DataFrame) -> pl.DataFrame:
    return internal.join(vendor, on='customer_name', how='outer')

merge_sources(internal_df, external_df)
```

# Avoid reassigning variable names

Reusing a name like `df` means the value behind it keeps changing. When something goes wrong, you can't inspect an earlier stage without rerunning the code from the top, which makes debugging painful. Give each stage its own name so every intermediate result stays available for inspection.

```python
# bad
df = df.pipe(transform)

# good
transformed_df = df.pipe(transform)
```

# Use `select` statements to specify a schema contract

A `select` defines the exact set of columns that comes out of it, so it works as a contract for the output schema. Using it to add new columns blurs that contract. Instead, use `.with_columns()`, which says "add or modify these and keep everything else", and makes your intent clear to the reader.

```python
# bad
df.select(
    pl.all(),
    (pl.col("price") * 1.2).alias("price_with_tax"),
)

# good
df.select("name", "price")  # the output is exactly these two columns

df.with_columns(
    (pl.col("price") * 1.2).alias("price_with_tax"),
)
```

# Refactor complex logical operations

Long, nested boolean conditions are hard to read and even harder to verify. Give the individual pieces meaningful names, then combine them. The named version reads almost like a sentence, and each part can be checked on its own.

```python
# bad
df.with_columns(
    pl.when(
        (pl.col("valid_from").is_not_null() | pl.col("valid_to").is_not_null())
        & (pl.col("value") > 0)
    )
    .then(1)
    .otherwise(0)
    .alias("is_valid")
)

# better
is_time_bound = pl.col("valid_from").is_not_null() | pl.col("valid_to").is_not_null()
has_nonzero_value = pl.col("value") > 0

df.with_columns(
    pl.when(is_time_bound & has_nonzero_value).then(1).otherwise(0).alias("is_valid")
)
```

# Use `pl.lit(None)` for empty columns

Never use an empty string or another placeholder string that signals an empty value, such as `NA`. Those look like real data, so they get treated as values: an empty string is a legitimate string, and `"NA"` survives casts, joins and filters as text. `None` is a proper null that Polars understands in filters, aggregations and `is_null` checks, so missing data stays missing.

```python
# bad
df.with_columns(pl.lit("").alias("foo"))

# bad
df.with_columns(pl.lit("NA").alias("foo"))

# good
df.with_columns(pl.lit(None).alias("foo"))
```

# Using comments

Comments can provide useful insight into code, but it is often more valuable to refactor the code so it explains itself. If a block needs a comment to make sense, consider moving it into a function with a docstring instead. When you do write a comment, skip the ones that merely restate the logic you wrote. Aim for context that explains the *why* behind your decisions, since that is the part the code itself can't show.

```python
# bad: restates the code
# multiply price by quantity and subtract the discount
orders_with_totals = orders_df.with_columns(
    ((pl.col("price") * pl.col("quantity")) - pl.col("discount")).alias("total")
)

# good: explains the decision
# Finance reports totals net of discount, so the discount is subtracted here
# rather than in the report.
orders_with_totals = orders_df.with_columns(
    ((pl.col("price") * pl.col("quantity")) - pl.col("discount")).alias("total")
)
```

# User Defined Functions (UDF)

It is highly recommended to avoid UDFs, whether they come from `.map_elements()` or from iterating over rows. They are dramatically less performant than native Polars because they drop out of the optimized, multi-threaded engine and run Python for every value. In most situations, logic that seems to call for a UDF can be refactored to use only native Polars expressions, so check that option first.

```python
# bad
df.with_columns(
    pl.col("price").map_elements(lambda p: p * 1.2, return_dtype=pl.Float64).alias("price_with_tax")
)

# good
df.with_columns(
    (pl.col("price") * 1.2).alias("price_with_tax")
)
```

# Joins

Always specify the type of join explicitly, for example `how='inner'`. Relying on defaults makes it harder to see what the join is supposed to do.

Always verify the outcome of a join with Polars' built-in validation, for example `validate='1:1'`. This catches unexpected duplicates early and prevents a join explosion, where a many-to-many match silently multiplies your rows.

```python
orders_df.join(
    customers_df,
    on="customer_id",
    how="left",
    validate="m:1",  # many orders per customer, one customer row per id
)
```

Avoid `right` joins. Flipping the tables and using a `left` join expresses the same thing, and keeping the main table on the left makes the pipeline easier to follow.

# Multi-line expressions

Wrap the entire expression in a single set of parentheses instead of using a `\` at the end of each line. Parentheses let Polars chains span multiple lines cleanly, while backslashes are fragile: a single trailing space after one breaks the whole statement.

```python
# bad
df = df.filter(pl.col("status") == "running") \
    .filter(pl.col("is_monitored") == True) \
    .drop("is_monitored") 

# good
df = (
    df.filter(pl.col("status") == "running")
    .filter(pl.col("is_monitored") == True)
    .drop("is_monitored")
)
```

# Prefer LazyFrames over DataFrames

LazyFrames let Polars plan the whole query before running it, and that is what makes the streaming engine (`.collect(engine='streaming')`) possible. The streaming engine processes data in chunks, so datasets larger than memory can be handled safely. For this reason, prefer LazyFrames over DataFrames by default.

```python
# bad
df = pl.read_csv("transactions.csv")
result = df.filter(pl.col("amount") > 0).group_by("customer_id").agg(pl.col("amount").sum())

# good
result = (
    pl.scan_csv("transactions.csv")
    .filter(pl.col("amount") > 0)
    .group_by("customer_id")
    .agg(pl.col("amount").sum())
    .collect(engine='streaming')
)
```

Only use DataFrames for ad hoc analysis, where seeing the immediate result of each transformation is convenient and where you have confirmed that the data is small enough to be safe.

# Use selectors for batch transformations

Selectors let you pick columns by their properties, such as their dtype or name pattern, instead of listing them one by one. They have many uses and should be used in many cases. The `polars.selectors` module is conventionally imported as `cs`.

```python
import polars.selectors as cs

# bad
df.with_columns(
    pl.col("price_q1").cast(pl.Float64),
    pl.col("price_q2").cast(pl.Float64),
    pl.col("price_q3").cast(pl.Float64),
    pl.col("price_q4").cast(pl.Float64),
)

# good
df.with_columns(
    cs.starts_with("price_").cast(pl.Float64),
)

# good
df.with_columns(
    cs.numeric().cast(pl.Float64),
)

# good
df.select(
    cs.ends_with("_raw").name.suffix("_clean"),
)
```