# User-Defined Functions (UDFs)

UDFs wrap backend-native SQL functions as typed Python callables. Prefer UDFs over raw SQL — they are type-safe, composable in expression chains, and avoid the `.sql()` / `.alias()` escape hatch. Always prefer using the Ibis built-in functions over defining a UDF. E.g `col.geometry.centroid()` over defining an `ST_CENTROID`

## Why UDFs over raw SQL

```python
# Raw SQL — requires alias escape, loses type safety, breaks expression chains
result = (
    table
    .alias("t")
    .sql("SELECT *, ST_MakeValid(geometry::GEOMETRY) AS valid_geometry FROM t")
)

# UDF — type-safe, composable, chainable
result = table.mutate(valid_geometry=ST_MakeValid(_.geometry))
```

## Builtin Scalar UDFs

The primary pattern. Wraps an existing backend SQL function as an Ibis expression. The function body is always `...` (ellipsis) — Python never executes it.

### Basic pattern

```python
import ibis
import ibis.expr.datatypes as dt

@ibis.udf.scalar.builtin
def ST_MakeValid(geometry: dt.geometry) -> dt.geometry: ...

@ibis.udf.scalar.builtin
def ST_ConvexHull(geometry: dt.geometry) -> dt.geometry: ...

@ibis.udf.scalar.builtin
def ST_PointOnSurface(geometry: dt.geometry) -> dt.geometry: ...

@ibis.udf.scalar.builtin
def ST_Perimeter(geometry: dt.geometry) -> float: ...

@ibis.udf.scalar.builtin
def ST_Transform(geometry: dt.geometry, source_srid, target_srid, always_xy=False) -> dt.geometry: ...

@ibis.udf.scalar.builtin
def ST_XMax(geometry: dt.geometry) -> float: ...

@ibis.udf.scalar.builtin
def ST_XMin(geometry: dt.geometry) -> float: ...

# valid - uses ibis data types (preferred)
@ibis.udf.scalar.builtin
def ST_YMax(geometry: dt.geometry) -> dt.float: ...

# valid - no need for parameter type
@ibis.udf.scalar.builtin
def ST_YMin(geometry) -> float: ...   
```

### Name aliasing

When the Python function name differs from the SQL function name, use `name`:

```python
@ibis.udf.scalar.builtin(name="TIMESTAMP_MILLIS")
def to_timestamp_ms(int64_expr) -> dt.timestamp: ...
```

### Parameterised return types

For BigQuery geography output, specify `geotype` and `srid` on the return type:

```python
@ibis.udf.scalar.builtin
def ST_GEOGFROMTEXT(wkt: str) -> dt.GeoSpatial(geotype="geography", srid=4326): ...
```

### Using builtin UDFs in expressions

Call like regular functions on column expressions:

```python
result = (
    table
    .mutate(
        valid_geom=ST_MakeValid(_.geometry),
        hull=ST_ConvexHull(_.geometry),
        perimeter=ST_Perimeter(_.geometry),
    )
    .mutate(hull_area=_.hull.area())
    .mutate(convexity=_.area_m2 / _.hull_area)
    .drop("hull", "hull_area")
)
```

## Builtin Aggregate UDFs

Same decorator pattern, but `@ibis.udf.agg.builtin`:

```python
@ibis.udf.agg.builtin
def ST_Union_Agg(geometry: dt.geometry) -> dt.geometry: ...
```

Used in group_by aggregation:

```python
merged = parcels.group_by("parcel_id").aggregate(
    geometry=ST_Union_Agg(_.geometry),
)
```

## Shared decorator parameters

All UDF decorators accept:

| Parameter | Purpose |
|---|---|
| `name` | Backend function name override (when you want Python name to differ from SQL name) |
| `database` | Database location in backend catalog |
| `catalog` | Catalog location in backend catalog |
| `signature` | Tuple `((arg0type, arg1type, ...), returntype)` — alternative to type annotations |

## Other Scalar UDF Flavors

For custom logic not available as a backend function. Default to `builtin` when wrapping an existing backend function.

### @ibis.udf.scalar.python

Row-by-row execution in python. **Slow** — avoid for large datasets:

```python
@ibis.udf.scalar.python
def str_magic(x: str) -> str:
    return f"{x}_magic"
```

### @ibis.udf.scalar.pandas

Vectorized, accepts pandas Series. Faster than `python` for large data:

```python
@ibis.udf.scalar.pandas
def str_cap(x: str) -> str:
    return x.str.capitalize()
```

### @ibis.udf.scalar.pyarrow

Vectorized, accepts PyArrow Arrays. Best performance for custom logic:

```python
import pyarrow.compute as pc
from datetime import date

@ibis.udf.scalar.pyarrow
def weeks_between(start: date, end: date) -> int:
    return pc.weeks_between(start, end)
```

### Complex type annotations

All flavors support `dt.Struct` and `dt.Map` as parameter/return types:

```python
@ibis.udf.scalar.python
def add_one(x: dt.Struct({"a": "int"})) -> int:
    return x["a"] + 1

@ibis.udf.scalar.python
def map_lookup(x: dt.Map(dt.string, dt.int64)) -> int:
    return x["key"] + 1
```

## Backend compatibility

Not all backends support all UDFs. A UDF wrapping a DuckDB spatial function could fail on BigQuery and vice versa. Define backend-specific UDFs separately:
- DuckDB spatial: uses `dt.geometry`, requires `extensions=["spatial"]` at connect time
- BigQuery geography: uses `dt.GeoSpatial(geotype="geography", srid=4326)`, wraps `ST_GEOG*` functions

## Gotchas

- Builtin UDFs do not support static type checking due to their empty-body definition. Static type checkers like mypy or Pyright will intentionally fail
- A return type of the UDF is required, the parameter types are not
- Builtin UDF bodies must be `...` (ellipsis) — the function is never called in Python
- UDFs wrapping nonexistent backend functions fail at **execution time**, not definition time — errors surface only when `.execute()` is called
- DuckDB spatial UDFs require `extensions=["spatial"]` at connect time
- BigQuery geography UDFs use `dt.GeoSpatial(geotype="geography", srid=4326)`, not `dt.geometry`
- The `name` parameter maps Python name → SQL name; without it, the Python function name is used as-is