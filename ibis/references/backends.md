# Backends — DuckDB & BigQuery

Extended backend reference covering connection parameters, read/write formats, table management, and SQL interop. Basic connection and execution syntax is in the main skill body.

## DuckDB

### do_connect parameters

```python
con = ibis.duckdb.connect(
    database=":memory:",         # or path to .duckdb file
    read_only=False,
    extensions=["spatial"],      # auto-install and load
    threads=4,
    memory_limit="1GB",
)
```

### From existing connection

```python
import duckdb
native_con = duckdb.connect("mydb.duckdb")
con = ibis.duckdb.from_connection(native_con)
```

### Extensions

```python
con = ibis.duckdb.connect(extensions=["spatial"])   # at connect time (preferred)
con.load_extension("spatial")                        # after connect
```

### Reading data

```python
con.read_csv("data.csv")                            # CSV
con.read_parquet("data.parquet")                     # Parquet
con.read_json("data.jsonl")                          # newline-delimited JSON
con.read_delta("delta_table/")                       # Delta Lake
con.read_xlsx("data.xlsx")                           # Excel (duckdb >= 1.2.0)
con.read_geo("data.geojson")                         # GeoJSON, Shapefile, etc.
```

Cross-database reads:

```python
con.read_sqlite("data.db")
con.read_postgres("postgres://user:pass@host:5432")
con.read_mysql("mysql://user:pass@host:3306/db")
```

All `read_*` methods accept `table_name=` to register the result as a named table.

### Writing data

```python
con.to_csv(expr, "out.csv")
con.to_parquet(expr, "out.parquet")
con.to_json(expr, "out.json")
con.to_delta(expr, "delta_out/")
con.to_xlsx(expr, "out.xlsx")                        # duckdb >= 1.2.0
con.to_geo(expr, "out.geojson", format="GeoJSON")
```

Hive-partitioned Parquet:

```python
con.to_parquet(expr, "output_dir/", partition_by="year")
con.to_parquet(expr, "output_dir/", partition_by=("year", "region"))
```

### Table management

```python
con.table("my_table")                                # reference existing table
con.list_tables()                                    # list all tables
con.list_tables(like="prefix_*")                     # pattern match
con.create_table("name", expr, overwrite=True)       # persist expression
con.drop_table("name")
con.attach("other.duckdb")                           # attach another database
con.attach_sqlite("data.db")                         # attach SQLite database
```

## BigQuery

### do_connect parameters

For full control, instantiate the backend directly:

```python
from ibis.backends.bigquery import Backend

con = Backend()
con.do_connect(
    project_id="my-project",
    dataset_id="my_dataset",
    credentials=credentials,             # google.auth.credentials.Credentials
    location="US",                       # default location for BQ objects
    partition_column="PARTITIONTIME",    # default partition column
)
```

### Authentication

```bash
gcloud auth login --update-adc
gcloud config set core/project <project_id>
```

For programmatic auth, pass a `credentials` object to `do_connect()` or `ibis.bigquery.connect()`.

### From existing client

```python
from google.cloud import bigquery
client = bigquery.Client(project="my-project")
con = ibis.bigquery.from_connection(client)
```

### Table operations

```python
con.create_table(
    "my_table",
    expr,
    overwrite=True,
    partition_by="date_col",             # partition expression
    cluster_by=["region", "category"],   # clustering columns
)
con.insert("my_table", expr)
con.upsert("my_table", expr, on="id")
con.create_view("my_view", expr)
con.set_database("other_dataset")        # switch active dataset
```

### Reading data

```python
con.read_csv("gs://bucket/data.csv")                 # from GCS
con.read_parquet("gs://bucket/data.parquet")
con.read_json("gs://bucket/data.jsonl")
con.read_delta("gs://bucket/delta_table/")
```

### Writing data

```python
con.to_csv(expr, "gs://bucket/out.csv")
con.to_parquet(expr, "gs://bucket/out.parquet")
con.to_json(expr, "gs://bucket/out.json")
con.to_delta(expr, "gs://bucket/delta_out/")
```

### BigQuery-specific

- `query_job_config` parameter on `execute()` and `raw_sql()` for job-level configuration
- Memtable support is **fallback, not native** — large memtables may be slow on BigQuery
- Memtables are automatically created in the location defined in the do_connect function. If the BigQuery data tables are in a non US location (such as australia-southeast2), the location parameter must be passed into do_connect so issues do not occur when trying to join tables that are not in the same region.

### DuckDB quirks

- Random number generation participates in common subexpression elimination (CSE). If the same random expression appears multiple times, DuckDB may reuse the result. Pass arguments to random functions to defeat CSE.

## Memtable Patterns

Basic memtable creation is in the main skill body. These are extended patterns:

### Typed empty tables via pandas

Plain dict with empty lists loses type information. Use a typed pandas DataFrame:

```python
import pandas as pd

empty = ibis.memtable(
    pd.DataFrame({
        "id": pd.Series([], dtype="str"),
        "value": pd.Series([], dtype="float64"),
    })
)
```

### Datetime columns

```python
import datetime

t = ibis.memtable({
    "id": ["A", "B"],
    "event_date": pd.array(
        [datetime.date(2020, 1, 1), datetime.date(2021, 6, 15)],
        dtype="datetime64[ns]",
    ),
})
```

### Nested struct arrays

Pass `list[list[dict] | None]` for `ARRAY<STRUCT>` columns:

```python
t = ibis.memtable({
    "id": ["A", "B"],
    "items": [
        [{"name": "Widget", "price": 5.0}],
        None,
    ],
})
```

### Dynamic construction from rows

```python
def rows_to_memtable(rows: list[dict]) -> ibis.Table:
    data = {col: [r[col] for r in rows] for col in rows[0]}
    return ibis.memtable(data)
```

### With defaults for missing keys

```python
defaults = {"name": "", "value": 0.0, "active": True}
data = {
    col: [r.get(col, default) for r in rows]
    for col, default in defaults.items()
}
t = ibis.memtable(data)
```

## Export Formats

Both backends support:

```python
expr.execute()                           # -> pandas DataFrame
expr.to_pandas()                         # explicit pandas
expr.to_pandas_batches(chunk_size=10000) # pandas in chunks
expr.to_polars()                         # -> polars DataFrame
expr.to_pyarrow()                        # -> PyArrow Table
expr.to_pyarrow_batches(chunk_size=10000)# PyArrow in chunks
expr.to_torch()                          # -> dict of torch Tensors
```

## SQL Interop

Raw SQL is discouraged — prefer Ibis expressions for type safety and composability. Use SQL interop only when an operation has no Ibis equivalent.

```python
# Table expression from SQL (returns ibis.Table — composable)
t = con.sql("SELECT * FROM my_table WHERE x > 0")

# Execute raw SQL directly (returns cursor — not composable)
con.raw_sql("CREATE INDEX idx ON my_table (id)")

# Compile expression to SQL string (for debugging / inspection)
sql_str = ibis.to_sql(expr)
sql_str = con.compile(expr, pretty=True)
```

### The .alias() escape hatch

When you must mix SQL with Ibis expressions, use `.alias()` to expose the table name:

```python
result = (
    table
    .alias("t")
    .sql("SELECT *, my_func(col) AS new_col FROM t")
)
```

This breaks the expression chain and loses type safety — prefer UDFs instead (see [udfs.md](udfs.md)).
