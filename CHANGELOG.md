# Changelog

## 1.0.0 — 2026-05-24

Initial production release of the Ibis skill targeting Ibis v12.x.

### SKILL.md (main body)
- Mental model, pandas-to-ibis translation table, quick reference for all core operations
- Chaining style guide, deferred expression `_` usage, column name constants pattern
- Static type checker guidance
- Reference table with "when to load" trigger hints for all 7 reference files

### Reference files
- **core-ops.md** — Selectors API (s.cols, s.across, s.numeric, s.of_type), extended select/mutate/filter patterns, dynamic column names, conditional logic (null-safe cases, coalesce, least/greatest), null vs NaN vs inf distinction
- **joins.md** — Generic .join() full signature, all 6 predicate forms, 9 shorthand methods, `_right` suffix problem (when/why/fix), self-joins via .view(), production patterns (iterative loop, anti-join exclusion, priority-tiered union)
- **aggregations.md** — Full aggregate function reference with `where` parameter, window functions (ranking, offset, cumulative), dedup-via-rank pattern, struct construction (ibis.struct with dt.Struct types), ARRAY\<STRUCT\> via .collect(), ibis.array()/ibis.map() API
- **spatial.md** — Full GeoSpatialValue API (measurements, relationships, operations, coordinates), WKT/WKB conversion, spatial join patterns (centroid, proximity, buffer corridor), composite patterns (overlap analysis, zone membership, nearest-neighbor)
- **types.md** — Primitive/complex/geospatial types, nullable variants, type introspection (isinstance + is_* methods), type conversion, extended casting (.try_cast, Table.cast, literal(None) type requirement), schemas, type annotations
- **udfs.md** — Builtin scalar/aggregate UDF pattern, name aliasing, parameterised return types, UDF vs raw SQL comparison, other flavors (python/pandas/pyarrow), backend compatibility notes
- **backends.md** — DuckDB (do_connect, extensions, read/write formats, table management), BigQuery (do_connect, auth, table ops with partition_by/cluster_by, region-aware memtables), extended memtable patterns, export formats, SQL interop (discouraged)

### Design decisions
- Gotchas distributed inline across relevant reference files rather than a separate gotchas.md
- Types and UDFs split into separate files for progressive disclosure
- All examples use generic names (no production-specific identifiers)
- All struct type annotations use dt.Struct() syntax, not string format
- Knowledge corpus (ibis-temp-docs/) used for synthesis, removed after skill completion
