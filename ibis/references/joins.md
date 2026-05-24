# Joins

Extended join reference covering predicate forms, column overlap handling, self-joins, and production patterns. Basic join syntax is in the main skill body.

## The Generic .join() Method

```python
Table.join(right, predicates, how="inner", lname="", rname="{name}_right")
```

| Parameter | Purpose |
|---|---|
| `right` | Right table expression |
| `predicates` | Join condition(s) — string, column equality, list, 2-tuple, or boolean expression |
| `how` | Join type: `"inner"`, `"left"`, `"outer"`, `"right"`, `"semi"`, `"anti"`, `"cross"`, `"any_inner"`, `"any_left"` |
| `lname` | Format string for overlapping left columns (default: `""` — no prefix) |
| `rname` | Format string for overlapping right columns (default: `"{name}_right"`) |

## Predicate Forms

### String key — shared column name

```python
t.join(right, "key")                   # single shared column
t.join(right, ["col_a", "col_b"])      # multi-column
```

Always produces `_right` suffixed duplicates for all overlapping columns — see [The _right Suffix Problem](#the-_right-suffix-problem).

### Explicit equality

```python
t.join(right, t.key == right.key)
```

On inner join, the join column appears once (no `_right` suffix). On left/outer joins, both columns are kept.

### Compound predicates

```python
t.join(
    right,
    (t.region == right.region) & (t.category == right.category),
)
```

### 2-tuple form

Each element can be a Column, Deferred (`_`), or lambda `(Table) -> Column`:

```python
t.join(right, [
    (_.id, _.id),
    (_.name.lower(), lambda r: r.name.lower()),
])
```

### Spatial predicates

Geometry boolean expressions work as join predicates. See [spatial.md](spatial.md) for full spatial join patterns.

```python
t.join(right, t.geometry.intersects(right.zone_geometry))
```

### Avoid `_` in join predicates

The deferred `_` can be ambiguous in join predicates — it may resolve to the wrong table. Always use explicit table references:

```python
# Avoid — _ may resolve unexpectedly
t.join(right, _.id == right.id)

# Prefer — explicit and unambiguous
t.join(right, t.id == right.id)
```

## Shorthand Methods

Preferred for readability. Same signature as `.join()` minus `how`:

```python
t.left_join(right, t.key == right.key)
t.inner_join(right, t.id == right.id)
t.outer_join(right, t.id == right.id)
t.right_join(right, t.id == right.id)
t.semi_join(right, t.id == right.id)      # keep left rows that match
t.anti_join(right, t.id == right.id)      # keep left rows that don't match
t.any_inner_join(right, t.id == right.id) # inner join, keep one match per left row
t.any_left_join(right, t.id == right.id)  # left join, keep one match per left row
```

### cross_join — no predicates

```python
t.cross_join(right)                        # cartesian product of all rows
```

### asof_join — nearest-key match

Matches on the nearest key rather than exact equality. Useful for time-series alignment:

```python
t.asof_join(right, on=t.timestamp >= right.timestamp, tolerance=ibis.interval(hours=1))
```

## The `_right` Suffix Problem

After a join, overlapping column names are disambiguated. The behavior depends on the predicate form:

### When `_right` appears

**String predicates** always produce `_right` on ALL overlapping columns, including the join key:

```python
left.join(right, "id")
# Result columns: id, name, value, id_right, name_right, ...
```

**Left/outer joins with explicit equality** keep both join columns:

```python
left.left_join(right, left.id == right.id)
# Result columns: id, name, value, id_right, other_col
```

### When `_right` does NOT appear

**Inner join with explicit equality** — the join column appears once:

```python
left.inner_join(right, left.id == right.id)
# Result columns: id, name, value, other_col
```

### Controlling the suffix

Use `lname` and `rname` format strings:

```python
left.join(right, "id", lname="l_{name}", rname="r_{name}")
# Result columns: l_id, l_name, r_id, r_name, ...
```

### Removing `_right` columns

Use negative selectors immediately after the join:

```python
result = left.left_join(
    right,
    left["id"] == right["id"],
).select(~s.cols("id_right"))
```

For multiple `_right` columns:

```python
result = joined.select(~s.cols("id_right", "name_right"))
```

## Self-Joins

To self-join a table, call `.view()` on one side to create a distinct table reference:

```python
view = t.view()
t.join(view, t.parent_id == view.id)
```

Without `.view()`, Ibis cannot distinguish the two references and the join will fail.

## Production Patterns

### Post-join column selection

Explicitly pick columns from each side to control the output schema:

```python
result = left.left_join(
    right,
    left.id == right.id,
).select(
    left.id,
    left.name,
    right.value,
    right.category,
)
```

### Sequential multi-join with column selection

Chain joins with explicit column selection at each step:

```python
step1 = base.left_join(enrichment_a, "parcel_id").select(
    base.parcel_id,
    base.geometry,
    enrichment_a.score,
)

result = step1.left_join(enrichment_b, "parcel_id").select(
    step1.parcel_id,
    step1.geometry,
    step1.score,
    enrichment_b.category,
)
```

### Iterative join loop

Join N tables onto a base table, removing `_right` columns after each step:

```python
result = base_table
for name, merge_table in merge_tables.items():
    result = result.left_join(
        merge_table,
        result[join_key] == merge_table[join_key],
    ).select(~s.cols(f"{join_key}_right"))
```

### Anti-join for exclusion

Keep rows from one table that don't appear in another:

```python
existing_ids = processed.select("id").distinct()

new_only = incoming.anti_join(
    existing_ids,
    incoming["id"] == existing_ids["id"],
)
```

### Priority-tiered union with dedup

Combine sources in priority order — anti-join to exclude already-seen IDs, then union:

```python
tier1 = primary_source.select(*common_cols)

tier1_ids = tier1.select("id").distinct()
tier2 = secondary_source.anti_join(
    tier1_ids,
    secondary_source["id"] == tier1_ids["id"],
).select(*common_cols)

combined = ibis.union(tier1, tier2)
```
