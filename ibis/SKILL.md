---
name: ibis
description: >-
  Ibis dataframe library reference for Python. Use when writing, reviewing, or debugging
  ibis code, ibis expressions, or ibis-framework pipelines. Also use when the user works
  with DuckDB or BigQuery via a Python dataframe API, references pandas-like lazy
  expressions, or needs backend-agnostic data transformations — even if they don't
  explicitly say "ibis". Covers table operations, joins, aggregations, geospatial queries,
  type system, UDFs, and data handling patterns.
metadata:
  version: "1.0"
  ibis-version: "12.x"
---

# Ibis — Living Reference Skill

Targets **Ibis v12.x**. Most patterns are forward-compatible with future versions.

## Mental Model

Ibis is SQL-with-Python-syntax. You build lazy query expressions using a DataFrame-style API; nothing executes until you call `.execute()` or a terminal method (`to_pandas`, `to_parquet`, etc.).

- Expressions are **lazy** and composable — they compile to SQL (or equivalent) at execution time
- `ibis.Table` is **immutable** — every method returns a new table expression
- **Backend-agnostic** — the same code runs on DuckDB, BigQuery, and 20+ other backends
- **No row index** — unlike pandas, rows have no implicit positional index

## Pandas-to-Ibis Translation

Ibis is lazy (builds a query); pandas is eager (executes immediately). Call `.execute()` to materialize an Ibis expression into a pandas DataFrame. There is no row index — use `.order_by()` for deterministic ordering.

| pandas | Ibis |
|---|---|
| `df[["a", "b"]]` | `t.select("a", "b")` |
| `df["new"] = expr` | `t.mutate(new=expr)` |
| `df[df.x > 0]` | `t.filter(_.x > 0)` |
| `df.drop(columns=["a"])` | `t.drop("a")` |
| `df.rename(columns={...})` | `t.rename(new="old")` |
| `df.groupby("k").agg(...)` | `t.group_by("k").aggregate(...)` |
| `df.merge(other, on="k")` | `t.join(other, "k")` or `t.left_join(other, t.k == other.k)` |
| `df.head(10)` | `t.head(10)` |
| `df.sort_values("x")` | `t.order_by("x")` |
| `df.drop_duplicates()` | `t.distinct()` |
| `df.fillna(0)` | `t.fill_null(0)` — NOT `.fillna()` |
| `df.dropna()` | `t.drop_null()` — NOT `.dropna()` |
| `df.value_counts()` | `t.value_counts()` |
| `~df.col` | `~_.col` — NOT `.negate()` |
| `df.col.tolist()` | `_.col.collect()` — aggregation, not a row-level op |
| `pd.concat([a, b])` | `ibis.union(a, b)` |
| `len(df)` | `t.count().execute()` |
| `df.dtypes` | `t.schema()` |

## Quick Reference

### Imports

```python
import ibis
from ibis import _                    # deferred column expression
import ibis.selectors as s            # column selectors
import ibis.expr.datatypes as dt      # type system
```

### Connection

```python
con = ibis.duckdb.connect()           # in-memory DuckDB
con = ibis.connect("duckdb://")       # same, URL form
con = ibis.bigquery.connect(project_id="proj", dataset_id="ds") # note: do_connect offers greater flexibility for bigquery, see extended reference for more.
```

### Table Construction

```python
t = ibis.memtable({"col": [1, 2, 3]})                    # from dict
t = ibis.memtable(pd.DataFrame(...))                      # from pandas
t = ibis.memtable([], schema={"col": "string"})           # empty with schema
t = con.table("my_table")                                 # from backend
```

### Select & Project

```python
t.select("a", "b")                    # by name
t.select(new_name=_.old_name)         # rename in select
t.select(s.cols("a", "b"))            # via selector (preferred when selecting columns)
t.select(~s.cols("drop_me"))          # exclude columns (preferred when selecting columns)
t.drop("a", "b")                      # drop columns - discouraged; use the negated selector ~s.cols() unless using in the same expression chain in which the column to be dropped was created.
```

### Mutate

```python
t.mutate(area=_.x * _.y)             # add computed column
```

See [core-ops.md](references/core-ops.md) for dynamic column names, chained mutations, and struct column creation.

### Filter

```python
t.filter(_.x > 0)                                # simple
t.filter((_.x > 0) & (_.y < 100))                # compound (& | ~)
t.filter(_.status.isin(["A", "B"]))               # membership
t.filter(_.name.contains("foo"))                   # substring
t.filter(ibis.literal(False))                      # empty result, same schema
```

### Joins

The generic `.join()` method supports all join types via the `how` parameter:

```python
t.join(right, "key")                               # inner join, string key
t.join(right, t.key == right.key)                   # inner join, explicit predicate
t.join(right, ["col_a", "col_b"])                   # multi-column join
t.join(right, t.key == right.key, how="left", lname='', rname='key_right')       # left join - full api example with join renaming (defaults)
```

Shorthand methods (preferred for readability):

```python
t.left_join(right, t.key == right.key)              # left join
t.inner_join(right, t.id == right.id)               # inner join
```

Also available: `outer_join`, `right_join`, `semi_join`, `anti_join`, `cross_join`, `any_inner_join`, `any_left_join`. See [joins.md](references/joins.md) for full details.

After joins, beware of `_right` suffixed duplicate columns — see [joins.md](references/joins.md) for when they appear and how to handle them.

### Aggregation

```python
t.group_by("key").aggregate(total=_.val.sum(), n=_.count())
```

Common aggregates: `.count()`, `.sum()`, `.mean()`, `.max()`, `.min()`, `.first()`, `.median()`, `.nunique()`

See [aggregations.md](references/aggregations.md) for window functions, struct/array aggregation, and multi-level patterns.

### Order, Limit, Distinct

```python
t.order_by("x")                       # ascending
t.order_by(ibis.desc("x"))            # descending
t.head(10)                            # first 10 rows
t.limit(100, offset=50)               # pagination
t.distinct()                          # deduplicate
t.distinct(on=["key"], keep="first")  # deduplicate on subset
```

### Conditional Logic

```python
ibis.ifelse(_.x > 0, _.x, ibis.literal(0))        # two-branch

ibis.cases(                                         # multi-branch (SQL CASE)
    (_.x < 0, ibis.literal("neg")),
    (_.x == 0, ibis.literal("zero")),
    else_=ibis.literal("pos"),
)
```

### Type Casting & Literals

```python
_.col.cast("float64")                 # string shorthand 
_.col.cast(dt.GeoSpatial())           # dt type instance (preferred for type safety)
ibis.literal(42)                      # scalar literal
ibis.literal(None, type="float64")    # typed null
```

### Set Operations

```python
ibis.union(a, b)                      # union (schemas must match)
ibis.union(*tables)                   # union many
a.difference(b)                       # rows in a not in b
a.intersect(b)                        # rows in both
```

### Execution

```python
df = t.execute()                      # -> pandas DataFrame
t.to_parquet("out.parquet")           # write to file
t.to_pandas()                         # explicit pandas
con.create_table("name", t, overwrite=True)  # persist to backend
n = t.count().execute()               # scalar row count
```

### Schema Introspection

```python
t.columns                             # list of column names
t.schema()                            # Schema {name -> DataType}
"col" in t.columns                    # check column exists
```

## Chaining Style

Prefer multi-line chaining with one operation per line, wrapped in parentheses:

```python
result = (
    table
    .filter(_.area_m2 > 1000)
    .mutate(center=_.geometry.centroid())
    .join(zones, _.center.intersects(zones.zone_geometry))
    .group_by("zone_id")
    .aggregate(count=_.count())
    .order_by(ibis.desc("count"))
)
```

For dependent computations, chain multiple `.mutate()` calls — required to reference columns created in the previous step:

```python
result = (
    table
    .mutate(hull=ST_ConvexHull(_.geometry), area=_.geometry.area())
    .mutate(hull_area=_.hull.area())
    .mutate(convexity=_.area / _.hull_area)
    .drop("hull", "hull_area")
)
```

## The Deferred Expression `_`

`from ibis import _` creates a deferred reference to the current table row. Use it in filter, mutate, aggregate, and select expressions instead of naming the table variable. Prefer explicit table names in deeply nested queries; `_` may not resolve as intended due to the lazy nature of Ibis.

```python
_.column_name                          # access column
_["column_name"]                       # dynamic access (variable name)
_.col.method()                         # chain methods
```

`ibis._` and the imported `_` are identical.

## Column Name Constants

Prefer string constants over literal strings for column names. This prevents typos and enables IDE navigation.

```python
PARCEL_ID = "parcel_id"
GEOMETRY = "geometry"
AREA_M2 = "area_m2"

result = table.select(s.cols(PARCEL_ID, GEOMETRY, AREA_M2))
filtered = table.filter(_[PARCEL_ID] == "P-001")
```

This applies to all column access patterns: `.select()`, `.drop()`, `.filter()`, `.group_by()`, `_[COL]`, `table[COL]`.

## Static Type Checkers

Ibis's lazy evaluation model is largely incompatible with static type checkers (mypy, pyright). Column access returns dynamic expression types that checkers cannot verify at analysis time. Expect `# type: ignore` annotations on Ibis expression chains. See [types.md](references/types.md) for workarounds.

## Reference Files

Detailed API coverage and patterns live in the reference files. This quick reference is NOT exhaustive, real-world ibis has more granular usage. Use the reference files when tasks require complex implementation rather than just simple one-liner usage.

| File | When to load |
|---|---|
| [core-ops.md](references/core-ops.md) | Complex select/filter/mutate patterns, selectors (s.across, s.numeric, s.of_type), dynamic column names, chained mutations, conditional logic beyond basic ifelse/cases |
| [joins.md](references/joins.md) | Multi-table joins, _right suffix issues, self-joins, predicate forms beyond basic string/equality, iterative join patterns |
| [aggregations.md](references/aggregations.md) | Window functions, struct/array aggregation (.collect()), multi-level group_by, conditional aggregation with `where`, ibis.array()/ibis.map() |
| [spatial.md](references/spatial.md) | Any geospatial operation: spatial joins, WKT conversion, geometry methods (area, buffer, intersects), spatial analysis patterns |
| [types.md](references/types.md) | Working with dt.* types, schema creation/comparison, type introspection, casting edge cases (.try_cast, Table.cast), type annotations |
| [udfs.md](references/udfs.md) | Wrapping backend SQL functions as UDFs, @ibis.udf.scalar.builtin or @ibis.udf.agg.builtin, custom UDFs (python/pandas/pyarrow) |
| [backends.md](references/backends.md) | DuckDB/BigQuery connection params (do_connect), read/write formats, memtable patterns, SQL interop, backend-specific quirks |
