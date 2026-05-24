# Type System, Schemas & Casting

The type system lives in `ibis.expr.datatypes` (imported as `dt`). Types are immutable instances used for casting, UDF annotations, schema definitions, and type introspection.

## Primitive Types

```python
import ibis.expr.datatypes as dt

dt.string       # String
dt.int64        # 64-bit integer (default for int)
dt.float64      # 64-bit float (default for float)
dt.boolean      # Boolean
dt.date         # Date
dt.timestamp    # Timestamp
dt.int8         # also: int16, int32
dt.uint8        # also: uint16, uint32, uint64
dt.float32      # also: float16
dt.unknown      # wildcard — matches any type
```

## Complex Types

### Array

```python
dt.Array(dt.string)                                    # Array<String>
dt.Array(dt.float64)                                   # Array<Float64>
dt.Array(dt.Struct({"x": dt.string, "y": dt.int64}))   # Array<Struct> (common for .collect())
```

### Struct

```python
dt.Struct({"name": dt.string, "value": dt.float64})    # dict fields (preferred)
dt.Struct.from_tuples([("name", dt.string), ("value", dt.float64)])  # from tuples 
```

### Map

```python
dt.Map(dt.string, dt.int64)                            # Map<String, Int64>
```

## Geospatial Types

```python
dt.geometry                                            # shorthand for GeoSpatial(geotype="geometry")
dt.GeoSpatial()                                        # backend agonistic generic geospatial
dt.GeoSpatial(geotype="geometry")                      # DuckDB geometry (projected coordinates)
dt.GeoSpatial(geotype="geography", srid=4326)          # BigQuery geography (WGS84) - shorthand: dt.geography
```

Specific geometry subtypes exist but are rarely needed in practice, prefer using the parent dt.GeoSpatial type:

```python
dt.Point, dt.LineString, dt.Polygon
dt.MultiPoint, dt.MultiLineString, dt.MultiPolygon
```

## Nullable Variants

All types are nullable by default. Use `nullable=False` for non-nullable:

```python
dt.String(nullable=True)                               # default
dt.String(nullable=False)                              # non-nullable
```

Nullability must be stripped for comparison (if comparing the underlying type only). E.g. a nullable string is not equal to the same datatype as a non nullable string.:

```python
declared.copy(nullable=True) == actual.copy(nullable=True)
```

## Type Introspection

### isinstance checks

```python
isinstance(dtype, dt.String)
isinstance(dtype, dt.GeoSpatial)
isinstance(dtype, dt.Array)
isinstance(dtype, dt.Struct)
isinstance(dtype, dt.Null)
isinstance(dtype, dt.Unknown)
```

### is_* methods

```python
dtype.is_string()
dtype.is_integer()        # any integer width
dtype.is_floating()       # any float width
dtype.is_numeric()        # integer or float
dtype.is_temporal()       # date, time, or timestamp
dtype.is_geospatial()
dtype.is_nested()         # Array, Map, or Struct
```

### Container type access

```python
array_type.value_type          # element type, e.g. dt.string
struct_type.names               # field names, e.g. {"x", "y"}
struct_type.types               # field types
struct_type["field_name"]       # type of a specific field
str(dtype)                      # string repr, e.g. "float64"
```

## Type Conversion

Convert between Ibis types and other ecosystems:

```python
# From external type systems
dt.DataType.from_pandas(pandas_dtype)
dt.DataType.from_pyarrow(arrow_type)
dt.DataType.from_numpy(numpy_type)

# To external type systems
ibis_type.to_pandas()
ibis_type.to_pyarrow()
ibis_type.to_numpy()
```

## Extended Casting Patterns

Basic `.cast()` is covered in the main skill body. These are the extended patterns:

### ibis.literal(None) — always specify type

`ibis.literal(None)` without `type=` creates an untyped null which can cause backend errors. Always specify the type:

```python
ibis.literal(None, type=dt.float64)        # correct
ibis.literal(None)                          # avoid — untyped null
```

### .try_cast() — null on failure

Returns NULL instead of raising on invalid cast:

```python
_.col.try_cast("int64")       # "abc" -> NULL instead of error
```

### Table.cast() — cast multiple columns at once

```python
t = t.cast({"area": dt.float, "count": dt.int}) 
```

### WKT string to geometry

```python
# Column cast
t = t.mutate(geometry=_.geom_wkt.cast(dt.GeoSpatial()))

# Scalar literal cast
zone = ibis.literal(wkt_string).cast(dt.GeoSpatial())

# BigQuery geography — use UDF instead of cast - implicit casting to geometry from WKT does not work.
zone = ST_GEOGFROMTEXT(ibis.literal(wkt_string))
```

## Schemas

### Creating schemas

```python
sc = ibis.schema({"name": "string", "area": "float64", "active": "boolean"})
sc = ibis.schema([("name", "string"), ("area", "float64")])
sc = ibis.schema(names=["name", "area"], types=["string", "float64"])
```

Non-nullable fields use `!` prefix, though the explicit `dt` version is preferred (`dt.String(nullable=False)`):

```python
sc = ibis.schema({"id": "!string", "value": "float64"})
```

### Schema from table

```python
sc = table.schema()                    # Schema object
dict(table.schema())                   # plain dict {name: DataType}
table.columns                          # list of column names
```

### Schema methods

```python
sc.equals(other)                       # equality including field order
sc.name_at_position(0)                 # column name at index
sc.names                               # field names
sc.types                               # field types
```

### Schema conversion

```python
Schema.from_pandas(pandas_schema)
Schema.from_pyarrow(pyarrow_schema)
sc.to_pandas()
sc.to_pyarrow()
```

### Schema in memtable

```python
t = ibis.memtable([], schema={"id": "string", "value": "float64"})
```

## Type Annotations for Ibis Code

```python
import ibis

def process(parcels: ibis.Table) -> ibis.Table: ...

def __init__(self, con: ibis.BaseBackend) -> None: ...  # BaseBackend is backend agonistic

def run() -> tuple[ibis.Column, str]: ...
```

## Static Type Checker Workarounds

Ibis's lazy evaluation model is incompatible with mypy/pyright — column access returns dynamic expression types that checkers cannot verify. Common workarounds:

- `# type: ignore` on expression chains (most common) or mypy ignore `["attr-defined"]`
- Wrapper functions with explicit return type annotations
- `typing.cast()` for specific expressions
- Separate typed interfaces from Ibis implementation