# Core Table Operations

Extended reference for select, mutate, filter, rename, conditionals, and selectors. Basic syntax for all of these is in the main skill body — this file covers the patterns that go beyond the quick reference.

## Selectors

`import ibis.selectors as s`

### Column selection

```python
t.select(s.cols("a", "b", "c"))            # select by name
t.select(~s.cols("drop_me", "drop_this"))   # exclude by name (preferred over .drop())
```

### Type-based selection

```python
t.select(s.numeric())                       # all numeric columns
t.select(s.of_type(dt.string))               # all string columns
t.select(s.of_type(dt.Array))               # all array columns (matches any Array)
t.select(s.of_type(dt.Array(dt.string)))    # specific: Array<String> only
```

### Name-based selection

```python
t.select(s.startswith("geo_"))              # columns starting with prefix
t.select(s.endswith("_id"))                 # columns ending with suffix
t.select(s.contains("price"))              # columns containing substring
```

### Combining selectors

```python
t.select(s.numeric() & ~s.cols("id"))       # numeric columns except "id"
```

### s.across() — bulk transforms

Apply transformations across multiple columns:

```python
t.select(s.across(s.numeric(), _ - _.mean(), names="{col}_centered"))
t.select(s.across(s.numeric(), {"centered": _ - _.mean(), "std": _ / _.std()}, names="{fn}_{col}"))
```

### Selectors for column discovery

Use selectors to find columns dynamically, then access `.columns`:

```python
non_key_cols = t.select(~s.cols("id", "geometry")).columns
numeric_cols = t.select(s.numeric()).columns
```

## Select & Project (Extended)

### Column disambiguation after joins

After a join, prefix with the source table reference:

```python
result = left.left_join(right, left.id == right.id).select(
    left.id,
    left.name,
    right.value,
)
```

### Computed expressions in select

```python
result = t.select(
    "id",
    area=_.geometry.area(),
    label=ibis.ifelse(_.active, ibis.literal("yes"), ibis.literal("no")),
)
```

## Rename

### Keyword mapping

```python
t.rename(new_name="old_name", another="original")
```

### Callable — bulk rename
Accepts any function that manipulates strings

```python
t.rename(str.lower)                        # lowercase all column names
t.rename(str.upper)                        # uppercase all column names
```

### Chained renames

```python
t = t.rename(geo="geometry").rename(area="area_m2")
```

### .name() — inline aliasing in select

Renames a single column expression within `.select()`:

```python
result = t.select(
    t["raw_name"].name("clean_name"),
    t["old_col"].name("new_col"),
)
```

## Mutate (Extended)

### Dynamic column names via dict unpacking

When the column name is in a variable:

```python
col_name = "geometry"
result = t.mutate(**{col_name: t.geom_wkt.cast(dt.GeoSpatial())})
```

### Chained mutations for dependent computations

Each `.mutate()` can reference columns created in the previous step:

```python
result = (
    table
    .mutate(
        hull=ST_ConvexHull(_.geometry),
        area=_.geometry.area(),
    )
    .mutate(hull_area=_.hull.area())
    .mutate(convexity=_.area / _.hull_area)
    .drop("hull", "hull_area")
)
```

## Filter (Extended)

### Dynamic column reference

Access a column by variable name:

```python
col = "status"
result = t.filter(_[col] == "active")
result = t.filter(ibis._[col].isin(target_values))
```

### Membership testing

```python
t.filter(_.status.isin(["active", "pending"]))
t.filter(_.status.notin(["deleted", "archived"]))
```

### Null checking

```python
t.filter(_.value.notnull())
t.filter(_.value.isnull())
```

Never use `== None` for null comparison — it won't work. Always use `.isnull()` / `.notnull()`.

### Null vs NaN vs Inf

These are distinct values — each has its own check:

```python
_.col.isnull()              # NULL only — not NaN, not Inf
_.col.isnan()               # NaN only (FloatingValue only) — not NULL
_.col.isinf()               # Inf only (FloatingValue only) — not NULL
_.col.fill_null(0)          # replaces NULL only — not NaN
```

### Substring matching

```python
t.filter(_.name.contains("foo"))
```

### Empty result with same schema

```python
empty = t.filter(ibis.literal(False))
```

### Complex nested conditions

```python
filtered = table.filter(
    (
        (_.area >= ibis.literal(max_area))
        | (
            (_.ratio_a > ibis.literal(threshold_a))
            & (_.ratio_b > ibis.literal(threshold_b))
        )
    )
    & (_.area > ibis.literal(min_area))
)
```

### Negated membership with OR

```python
filtered = table.filter(
    (~_.category.isin(excluded_categories))
    | (_.type != ibis.literal("special"))
)
```

## Distinct (Extended)

### Subset deduplication

```python
t.distinct(on=["key_col"], keep="first")    # keep first per key
t.distinct(on=["key_col"], keep="last")     # keep last per key
```

## Conditional Logic (Extended)

### Null-safe patterns

Guard against null geometry before computing:

```python
intersection_area = ibis.cases(
    (_.other_geometry.isnull(), ibis.literal(0.0)),
    else_=_.geometry.intersection(_.other_geometry).area(),
)
```

### Nested conditions in ibis.cases

```python
coverage = ibis.cases(
    ((_.area <= 0) | _.intersection.isnull(), ibis.literal(0.0)),
    else_=(_.intersection.area() / _.area) * 100,
)
```

### ibis.ifelse with typed null

When one branch should be null, specify the type:

```python
distance = ibis.ifelse(
    _.target_geometry.notnull(),
    _.center.distance(_.target_geometry) / 1000,
    ibis.literal(None, type=dt.float64),
)
```

### ibis.coalesce — first non-null value

```python
ibis.coalesce(_.preferred_name, _.fallback_name, ibis.literal("unknown"))
```

## Literals & Scalars (Extended)

### ibis.least / ibis.greatest

Min/max across multiple values per row:

```python
min_value = ibis.least(_.col_a, _.col_b, _.col_c)
max_value = ibis.greatest(_.col_a, _.col_b, _.col_c)
```

### Capping values

```python
capped_pct = ibis.least(_.raw_percent, ibis.literal(100.0))
floored_val = ibis.greatest(_.value, ibis.literal(0.0))
```

### ibis.literal with config values

Use `ibis.literal()` to wrap Python variables in filter expressions:

```python
result = t.filter(_.area > ibis.literal(min_area_threshold))
```

## Schema Introspection (Extended)

```python
t.columns                                  # list of column names
t.schema()                                 # Schema {name -> DataType}
dict(t.schema())                           # plain dict
"col" in t.columns                         # check existence
```

### Extract column to Python list

```python
ids = t.select("id").distinct().execute()["id"].tolist()
```

### Scalar extraction

```python
row_count = t.count().execute()
single_value = t.filter(_.id == "X").execute().iloc[0]["col"]
```

### Row ordering

Ibis has no implicit row ordering. Without `.order_by()`, row order is non-deterministic. Always add `.order_by()` when order matters.
