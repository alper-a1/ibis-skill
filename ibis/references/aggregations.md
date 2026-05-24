# Aggregations, Window Functions & Collections

Extended reference for group_by/aggregate, all aggregate functions, window functions, and struct/array/map patterns used in aggregation context. Basic aggregation syntax is in the main skill body.

## group_by().aggregate()

Full signature:

```python
t.group_by(*by, **key_exprs).aggregate(*metrics, having=(), **kwargs)
```

| Parameter | Purpose |
|---|---|
| `by` | Grouping columns — strings, column expressions, or list of either |
| `key_exprs` | Named grouping expressions via kwargs |
| `metrics` | Positional aggregate expressions |
| `having` | Post-aggregation boolean filter(s) |
| `kwargs` | Named aggregate expressions |

### Multi-column group_by

```python
summary = (
    table.group_by(["region", "category"])
    .aggregate(
        total=_.amount.sum(),
        avg_price=_.price.mean(),
        n_unique=_.customer_id.nunique(),
    )
)
```

## Aggregate Functions

All column-level aggregates accept a `where` parameter for conditional aggregation.

### General

| Function | Description |
|---|---|
| `.count(where=)` | Non-null count |
| `_.count()` | Row count (on deferred table expression) |
| `.first(where=)` | First non-null value |
| `.last(where=)` | Last non-null value |
| `.arbitrary(where=)` | Any value (non-deterministic) |
| `.nunique(where=)` | Count of distinct values |
| `.approx_nunique(where=)` | Approximate distinct count |
| `.collect(where=, order_by=, include_null=, distinct=)` | Gather values into array |
| `.group_concat(sep=",", where=, order_by=)` | Concatenate values into string |

### Numeric

| Function | Description |
|---|---|
| `.sum(where=)` | Sum |
| `.mean(where=)` | Mean |
| `.median(where=)` | Median |
| `.approx_median(where=)` | Approximate median |
| `.max(where=)` | Maximum |
| `.min(where=)` | Minimum |
| `.std(where=, how="sample")` | Standard deviation (`how="pop"` for population) |
| `.var(where=, how="sample")` | Variance |
| `.quantile(q, where=)` | Exact quantile |
| `.approx_quantile(q, where=)` | Approximate quantile |
| `.corr(right, where=, how="sample")` | Correlation |
| `.cov(right, where=, how="sample")` | Covariance |

### Boolean

| Function | Description |
|---|---|
| `.any(where=)` | True if any value is true |
| `.all(where=)` | True if all values are true |

### Conditional aggregation with `where`

```python
result = table.group_by("region").aggregate(
    total_sales=_.amount.sum(),
    premium_sales=_.amount.sum(where=_.tier == "premium"),
    n_active=_.id.count(where=_.status == "active"),
)
```

## Aggregation Patterns

### Boolean flag from aggregate comparison

Derive a boolean directly from an aggregate:

```python
result = joined.group_by("parcel_id").aggregate(
    in_zone=(_.overlap_area.max() > 0),
)
```

### Dedup via dict comprehension + .first()

Keep first value per group for all non-key columns:

```python
deduplicated = table.group_by("id").aggregate(
    **{col: _[col].first() for col in table.columns if col != "id"}
)
```

### Selector-discovered columns for dynamic aggregation

Use selectors to find non-key columns dynamically:

```python
non_key_cols = table.select(~s.cols("id", "geometry")).columns

merged = table.group_by("id").aggregate(
    geometry=ST_Union_Agg(_.geometry),
    **{col: _[col].first() for col in non_key_cols},
)
```

### Duplicate detection

```python
duplicates = (
    table.group_by("id")
    .aggregate(row_count=_.count())
    .filter(_.row_count > 1)
)
dup_count = duplicates.count().execute()
```

### Multi-level aggregation

Aggregate within sub-groups, then aggregate sub-groups:

```python
# Level 1: sum per category within each group
per_category = joined.group_by("parcel_id", "category").aggregate(
    total_coverage=_.coverage_percent.sum(),
    start_date=_.start_date.first(),
)

# Level 2: collect categories into struct array per group
result = per_category.group_by("parcel_id").aggregate(
    categories=ibis.struct({
        "category": _.category,
        "coverage_percent": _.total_coverage,
    }, type=dt.Struct({"category": dt.string, "coverage_percent": dt.float64})).collect(),
)
```

## Window Functions

### Window specification

```python
window = ibis.window(
    group_by="parcel_id",            # partition by (string, list, or column expr)
    order_by=ibis.asc("distance_km"),  # sort within partition
)
```

Sort direction helpers:

```python
ibis.asc("col")                       # ascending (default)
ibis.desc("col")                      # descending
ibis.asc("col", nulls_first=True)     # nulls first
```

### Applying window functions

Two forms — `.over(window)` or inline `.over(...)`:

```python
# Named window
window = ibis.window(group_by="id", order_by=ibis.asc("date"))
ranked = table.mutate(rank=ibis.row_number().over(window))

# Inline window
ranked = table.mutate(
    rank=ibis.row_number().over(
        ibis.window(group_by="id", order_by=ibis.asc("date"))
    )
)
```

### Ranking functions

| Function | Description |
|---|---|
| `ibis.row_number()` | 0-based row number within partition |
| `ibis.rank()` | Rank with gaps (ties get same rank) |
| `ibis.dense_rank()` | Rank without gaps |
| `ibis.percent_rank()` | Relative rank as fraction (0-1) |
| `ibis.cume_dist()` | Cumulative distribution (0-1) |
| `ibis.ntile(buckets)` | Divide into N equal-sized buckets |

### Offset functions

```python
_.col.lag(offset=1, default=None)      # value N rows before
_.col.lead(offset=1, default=None)     # value N rows after
_.col.nth(n, where=None)              # nth value (0-indexed) in window
```

### Cumulative functions

```python
_.col.cumsum(where=, group_by=, order_by=)
_.col.cummax(where=, group_by=, order_by=)
_.col.cummin(where=, group_by=, order_by=)
_.col.cummean(where=, group_by=, order_by=)
```

### Value-finding functions

```python
_.col.argmax(key=None, where=None)     # value that maximizes key
_.col.argmin(key=None, where=None)     # value that minimizes key
```

## Window Patterns

### Dedup-via-rank

Add a rank column, filter to rank == 0 (first per group), drop the rank column:

```python
deduped = (
    table.mutate(
        _rank=ibis.row_number().over(
            ibis.window(
                group_by="id",
                order_by=ibis.asc("priority"),
            )
        )
    )
    .filter(_._rank == 0)
    .drop("_rank")
)
```

### Running difference

```python
result = table.mutate(
    prev_value=_.value.lag(1).over(
        ibis.window(group_by="id", order_by=ibis.asc("date"))
    ),
).mutate(
    delta=_.value - _.prev_value,
)
```

## Struct Construction

### ibis.struct() — create struct column

```python
result = table.mutate(
    details=ibis.struct({
        "name": _.name,
        "score": _.score,
    })
)
```

### With explicit type annotation

The `type` parameter accepts a `dt.Struct` type:

```python
ibis.struct(
    {"category": _.category, "coverage": _.total_coverage},
    type=dt.Struct({"category": dt.string, "coverage": dt.float64}),
)
```

### Nested structs

Create intermediate struct columns with `.mutate()`, then reference them in an outer struct:

```python
with_structs = table.mutate(
    _forecast=ibis.struct({
        "rating": _.rating_value,
        "projected": _.projected_value,
    }),
    _metrics=ibis.struct({
        "utilization_pct": _.utilization_pct,
    }),
)

result = with_structs.group_by("id").aggregate(
    details=ibis.struct({
        "name": _.name,
        "forecast": _._forecast,
        "metrics": _._metrics,
    }).collect(),
)
```

### Scalar struct in aggregation (without .collect())

Without `.collect()`, produces a single struct value using `.first()` semantics:

```python
result = table.group_by("id").aggregate(
    dates=ibis.struct({
        "start": _.start_date.first(),
        "end": _.end_date.first(),
    }, type=dt.Struct({"start": dt.date, "end": dt.date})),
)
```

## Array Aggregation with .collect()

`.collect()` gathers row-level values into an array during group_by. Combined with `ibis.struct()`, it creates `ARRAY<STRUCT>` columns.

### Basic .collect()

```python
result = table.group_by("id").aggregate(
    tags=_.tag.collect(),
)
```

### Struct .collect() — ARRAY\<STRUCT\>

```python
result = table.group_by("id").aggregate(
    items=ibis.struct(
        {
            "code": _.code.cast("int64"),
            "description": _.description,
            "coverage_percent": _.coverage_percent,
        },
        type=dt.Struct({"code": dt.int64, "description": dt.string, "coverage_percent": dt.float64}),
    ).collect()
)
```

Type annotation can be omitted when types are inferrable, though explicit is preferred:

```python
result = table.group_by("id").aggregate(
    history=ibis.struct({
        "event": _.event_name,
        "date": _.event_date,
    }).collect(),
)
```

### Boolean flag from array presence

```python
result = result.mutate(has_items=_.items.notnull())
```

## ibis.array()

```python
arr = ibis.array([1, 2, 3])                    # literal array
arr = ibis.array([_.col_a, _.col_b])            # array from column expressions
```

### ArrayValue methods

| Method | Description |
|---|---|
| `.length()` | Number of elements |
| `.contains(value)` | Whether array contains value |
| `.unnest()` | Flatten into rows (NULLs and empties dropped) |
| `.filter(predicate)` | Filter elements (callable or Deferred) |
| `.map(func)` | Transform each element |
| `.flatten()` | Remove one level of nesting |
| `.sort()` | Sort elements |
| `.unique()` | Unique values |
| `.concat(other)` | Concatenate arrays |
| `.intersect(other)` | Array intersection |
| `.union(other)` | Array union |
| `.join(sep)` | Join elements into string |
| `.zip(other)` | Zip into array of structs |

## ibis.map()

```python
m = ibis.map({"a": 1, "b": 2})                 # from dict
m = ibis.map(_.key_col, _.value_col)            # from column expressions
```

### MapValue methods

| Method | Description |
|---|---|
| `.get(key, default=None)` | Value for key |
| `.keys()` | Extract keys as array |
| `.values()` | Extract values as array |
| `.contains(key)` | Whether map contains key |
| `.length()` | Number of key-value pairs |

## StructValue access

```python
_.struct_col["field_name"]              # access field by name
_.struct_col.fields                     # dict of field name -> type
_.struct_col.names                      # field names
_.struct_col.types                      # field types
_.struct_col.lift()                     # project struct fields into table columns
```
