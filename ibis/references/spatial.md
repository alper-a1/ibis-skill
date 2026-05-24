# Geospatial Operations

Full reference for geospatial methods, spatial joins, and multi-step spatial analysis patterns. For geospatial types (`dt.geometry`, `dt.GeoSpatial`), see [types.md](types.md). For ST_* UDF definitions, see [udfs.md](udfs.md). Note that not every spatial function may work with all backends - if errors occur look into creating a UDF native to that backend.

## Setup

DuckDB requires the spatial extension:

```python
con = ibis.duckdb.connect(extensions=["spatial"])
```

BigQuery uses geography type natively (SRID 4326, WGS84). No extension needed.

## WKT/WKB Conversion

### WKT to geometry

```python
# Column cast (DuckDB)
t = t.mutate(geometry=_.geom_wkt.cast(dt.GeoSpatial()))

# Scalar literal cast (DuckDB)
zone = ibis.literal(wkt_string).cast(dt.GeoSpatial())

# BigQuery geography — implicit cast does not work, use UDF
zone = ST_GEOGFROMTEXT(ibis.literal(wkt_string))
```

### Full WKT-to-geometry table builder

```python
t = ibis.memtable({"id": ["A"], "geom_wkt": ["POINT(0 0)"]})
t = t.mutate(geometry=_.geom_wkt.cast(dt.GeoSpatial())).drop("geom_wkt")
t = t.mutate(area_m2=_.geometry.area())
```

### Geometry to WKT/WKB (serialization)

```python
_.geometry.as_text()           # WKT without SRID
_.geometry.as_ewkt()           # WKT with SRID
_.geometry.as_binary()         # WKB without SRID
_.geometry.as_ewkb()           # WKB with SRID
```

## Measurement Methods

```python
_.geometry.area()              # area of geometry
_.geometry.length()            # length (zero for polygons)
_.geometry.perimeter()         # perimeter
_.geometry.distance(other)     # distance between two geometries
_.geometry.max_distance(other) # 2D maximum distance in projected units
```

DuckDB returns area/distance in the units of the projected coordinate system (typically meters). BigQuery geography returns geodesic meters.

### Production examples

```python
result = table.mutate(area_m2=_.geometry.area())

joined = joined.mutate(
    distance_km=_.center_point.distance(_.project_geometry) / 1000,
)
```

## Spatial Relationships (Boolean Predicates)

### Core predicates

```python
_.geometry.intersects(other)           # share any points (most common join predicate)
_.geometry.contains(other)             # geometry fully contains other
_.geometry.within(other)               # geometry fully within other (inverse of contains)
_.geometry.d_within(other, distance=)  # any part within distance threshold
_.geometry.d_fully_within(other, distance=)  # entirely within distance
_.geometry.disjoint(other)             # no points in common
```

### Additional predicates

```python
_.geometry.touches(other)              # share boundary but not interior
_.geometry.crosses(other)              # interior intersection, not fully contained
_.geometry.overlaps(other)             # share space, same dimension, not fully contained
_.geometry.covers(other)               # no point of other is outside self
_.geometry.covered_by(other)           # no point of self is outside other
_.geometry.contains_properly(other)    # contains, excluding common border points
_.geometry.geo_equals(other)           # geometries are equal
_.geometry.ordering_equals(other)      # equal with matching point ordering
```

## Geometry Operations

```python
_.geometry.intersection(other)         # intersection geometry
_.geometry.union(other)                # merge two geometries
_.geometry.difference(other)           # geometry difference
_.geometry.buffer(radius)              # all points within radius distance
_.geometry.centroid()                   # centroid point
_.geometry.envelope()                  # bounding box polygon
_.geometry.simplify(tolerance, preserve_collapsed)  # simplify geometry
_.geometry.flip_coordinates()          # swap x/y coordinates
_.geometry.line_merge()                # merge MultiLineString into LineString
_.geometry.line_substring(start, end)  # clip linestring using fractional positions (0-1)
```

### Production examples

```python
parcels = parcels.mutate(center_point=_.geometry.centroid())

buffered = lines.mutate(_buffer=_.geometry.buffer(buffer_distance_m))

joined = joined.mutate(
    intersection_geom=_.geometry.intersection(_.zone_geometry),
)
```

## Point & Line Access

```python
_.geometry.x()                 # X coordinate (point only)
_.geometry.y()                 # Y coordinate (point only)
_.geometry.start_point()       # first point of LINESTRING
_.geometry.end_point()         # last point of LINESTRING
_.geometry.point_n(n)          # nth point (negative indices count from end)
_.geometry.geometry_n(n)       # nth geometry of multi-geometry (1-based)
_.geometry.line_locate_point(other)  # distance along line as fraction (0-1)
```

## Bounds

```python
_.geometry.x_min()             # bounding box X minimum
_.geometry.x_max()             # bounding box X maximum
_.geometry.y_min()             # bounding box Y minimum
_.geometry.y_max()             # bounding box Y maximum
```

For functions not natively in Ibis (e.g. ST_XMax), use builtin UDFs — see [udfs.md](udfs.md).

## Properties

```python
_.geometry.geometry_type()     # type string: "POLYGON", "MULTIPOLYGON", etc.
_.geometry.is_valid()          # whether geometry is valid
_.geometry.n_points()          # number of points
_.geometry.n_rings()           # number of rings (including outer)
_.geometry.srid()              # spatial reference identifier
_.geometry.azimuth(other)      # angle in radians from horizontal
```

## Coordinate Systems

```python
_.geometry.srid()              # get SRID
_.geometry.set_srid(srid)      # set SRID (does not transform coordinates)
_.geometry.transform(srid)     # transform to new SRID (reprojects coordinates)
_.geometry.convert(source, target)  # CRS conversion
```

DuckDB uses projected coordinate systems. BigQuery uses WGS84 (SRID 4326) exclusively — all geography values are geodesic.

## Column-Level Aggregate

```python
_.geometry.unary_union()       # aggregate union of all geometries in column
_.geometry.unary_union(where=_.area_m2 > 100)  # conditional aggregate
```

For group_by aggregation of geometries, use the `ST_Union_Agg` UDF — see [udfs.md](udfs.md):

```python
merged = parcels.group_by("zone_id").aggregate(
    geometry=ST_Union_Agg(_.geometry),
)
```

## Constructing Points

```python
x_value.point(y_value)         # construct POINT from numeric coordinates
```

## Spatial Join Patterns

Spatial joins use geometry predicates in place of equality predicates. Rename geometry columns before joining to avoid column name clashes.

### Point-in-polygon via centroid

Faster than full geometry intersection — use centroid for approximate containment:

```python
parcels = parcels.mutate(center_point=_.geometry.centroid())
zones = zones.rename(zone_geometry="geometry")

joined = parcels.join(
    zones,
    parcels.center_point.intersects(zones.zone_geometry),
)
```

### Spatial left join

```python
zones = zones.rename(zone_geometry="geometry")

joined = parcels.left_join(
    zones,
    parcels.geometry.intersects(zones.zone_geometry),
)
```

### Proximity join

```python
joined = parcels.left_join(
    projects,
    parcels.center_point.d_within(projects.geometry, distance=radius_m),
)
```

### Buffer corridor join

Buffer a geometry first, then join on the buffer intersecting another geometry:

```python
buffered = lines.mutate(_buffer=_.geometry.buffer(distance_m))

mapped = buffered.join(
    parcels.select(s.cols("parcel_id", "parcel_geometry")),
    buffered._buffer.intersects(parcels.parcel_geometry),
).select(~s.cols("parcel_geometry", "_buffer"))
```

## Composite Spatial Patterns

### Overlap analysis

Spatial join -> intersection -> area -> coverage percent:

```python
zones = zones.rename(zone_geometry="geometry")

joined = parcels.left_join(
    zones,
    parcels.geometry.intersects(zones.zone_geometry),
)

joined = (
    joined.mutate(
        intersection_geom=_.geometry.intersection(_.zone_geometry),
    )
    .mutate(
        intersection_area=ibis.cases(
            (_.intersection_geom.isnull(), ibis.literal(0.0)),
            else_=_.intersection_geom.area(),
        ),
    )
    .mutate(
        coverage_percent=ibis.cases(
            ((_.area_m2 <= 0) | (_.intersection_area <= 0), ibis.literal(0.0)),
            else_=(_.intersection_area / _.area_m2) * 100,
        ),
    )
)
```

### Boolean zone membership

Simplified overlap — produces a boolean flag per parcel:

```python
zones = zones.rename(zone_geometry="geometry")

joined = parcels.left_join(
    zones,
    parcels.geometry.intersects(zones.zone_geometry),
)

joined = joined.mutate(
    overlap_area=ibis.ifelse(
        _.zone_geometry.notnull(),
        _.geometry.intersection(_.zone_geometry).area(),
        ibis.literal(0.0),
    )
)

result = joined.group_by("parcel_id").aggregate(
    in_zone=(_.overlap_area.max() > 0),
)
```

### Centroid join with deduplication

Use centroid for faster point-in-polygon, then deduplicate many-to-one matches:

```python
parcels = parcels.mutate(center_point=_.geometry.centroid())
zones = zones.rename(zone_geometry="geometry")

joined = parcels.join(
    zones,
    parcels.center_point.intersects(zones.zone_geometry),
)

deduplicated = joined.group_by("parcel_id").aggregate(
    **{col: _[col].first() for col in joined.columns if col != "parcel_id"}
)
```

### Nearest-neighbor via cross join + window rank

All-pairs distance -> rank by distance -> take closest:

```python
parcels = parcels.mutate(center_point=_.geometry.centroid())

crossed = parcels.cross_join(stations)
crossed = crossed.mutate(
    distance_km=crossed.center_point.distance(crossed.station_geometry) / 1000
)

ranked = crossed.mutate(
    rank=ibis.row_number().over(
        ibis.window(
            group_by="parcel_id",
            order_by=ibis.asc("distance_km"),
        )
    )
)
nearest = ranked.filter(_.rank == 0)

result = nearest.select(
    "parcel_id",
    dist_to_station=_.distance_km,
    nearest_station=_.station_name,
)
```