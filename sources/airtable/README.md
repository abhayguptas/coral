# Airtable source

This bundled source lets Coral query Airtable metadata and records with a
Personal Access Token.

The design is intentionally metadata-first:

- `airtable.bases` discovers visible base IDs
- `airtable.base_tables` exposes table schema metadata for one base
- `airtable.records` reads records generically from one table

## Why `fields` is JSON

Airtable tables are user-defined and their columns vary from base to base. A
bundled Coral source cannot safely pretend to know every custom field in
advance, so `airtable.records.fields` is exposed as JSON by design.

The same applies to table schema details:

- `airtable.base_tables.fields` is JSON
- `airtable.base_tables.views` is JSON

That keeps the source accurate across different Airtable schemas without adding
runtime code generation.

## Authentication

Create a Personal Access Token with:

- record read access for the bases you want to query
- `schema.bases:read` scope for metadata access

Then export it:

```sh
export AIRTABLE_TOKEN="your_airtable_pat"
```

You can override the API base if needed, but the default is correct for normal
Airtable cloud usage:

```sh
export AIRTABLE_API_BASE="https://api.airtable.com"
```

## Quick start

```sh
coral source add airtable
coral source test airtable
coral sql "SELECT table_name FROM coral.tables WHERE schema_name = 'airtable' ORDER BY table_name"
```

If you update the token later, run `coral source add airtable` again so Coral
refreshes the stored credentials.

## Inspect the installed shape

After adding the source, it is useful to inspect the shape Coral sees:

```sql
SELECT table_name
FROM coral.tables
WHERE schema_name = 'airtable'
ORDER BY table_name;
```

```sql
SELECT table_name, column_name, data_type, is_nullable
FROM coral.columns
WHERE schema_name = 'airtable'
ORDER BY table_name, ordinal_position;
```

```sql
SELECT input_name, kind, has_default
FROM coral.inputs
WHERE source_name = 'airtable'
ORDER BY input_name;
```

This is especially useful for Airtable because the source is intentionally
JSON-first for dynamic schema content.

## Tables

| Table | Notes |
|---|---|
| `bases` | Discover visible `base_id` values |
| `base_tables` | Get schema metadata for one base; requires `base_id` |
| `records` | Get generic records for one table; requires `base_id` and `table_id` |

## How to query it

Start by listing bases:

```sql
SELECT id, name, permission_level
FROM airtable.bases
ORDER BY name;
```

Use one `base_id` to inspect table schema:

```sql
SELECT id, name, primary_field_id, description
FROM airtable.base_tables
WHERE base_id = 'appXXXXXXXXXXXXXX'
ORDER BY name;
```

Inspect full field and view metadata for one table:

```sql
SELECT id, name, fields, views
FROM airtable.base_tables
WHERE base_id = 'appXXXXXXXXXXXXXX'
  AND id = 'tblXXXXXXXXXXXXXX';
```

List records from one table:

```sql
SELECT record_id, created_time, fields
FROM airtable.records
WHERE base_id = 'appXXXXXXXXXXXXXX'
  AND table_id = 'tblXXXXXXXXXXXXXX'
LIMIT 10;
```

Filter records through a view:

```sql
SELECT record_id, fields
FROM airtable.records
WHERE base_id = 'appXXXXXXXXXXXXXX'
  AND table_id = 'tblXXXXXXXXXXXXXX'
  AND view = 'viwXXXXXXXXXXXXXX'
LIMIT 10;
```

Filter records with an Airtable formula:

```sql
SELECT record_id, fields
FROM airtable.records
WHERE base_id = 'appXXXXXXXXXXXXXX'
  AND table_id = 'tblXXXXXXXXXXXXXX'
  AND formula = "{Status} = 'Open'"
LIMIT 10;
```

## Table behavior notes

- `base_tables` returns one row per table in a base.
- `fields` and `views` stay as JSON because Airtable returns them as nested
  arrays under each table schema object.
- `records.fields` is JSON because Airtable record fields are dynamic and
  empty-valued fields may be omitted by the API.
- `records` supports optional `view`, `formula`, and `max_records` filters in
  addition to Coral's normal SQL `LIMIT`.

## Testing instructions

Run these commands from the repo root after building the CLI:

```sh
cargo run --locked -p coral-cli -- source lint ./sources/airtable/manifest.yaml
make lint-sources
make docs-generate
make docs-check
```

For a clean live test, use a temporary Coral config directory:

```sh
export CORAL_CONFIG_DIR="$PWD/.tmp/airtable-demo-config"
mkdir -p "$CORAL_CONFIG_DIR"
target/debug/coral source add airtable
target/debug/coral source test airtable
```

Then verify the main flow manually:

```sh
target/debug/coral sql "SELECT id, name, permission_level FROM airtable.bases ORDER BY name"
target/debug/coral sql "SELECT table_name FROM coral.tables WHERE schema_name = 'airtable' ORDER BY table_name"
target/debug/coral sql "SELECT table_name, column_name, data_type FROM coral.columns WHERE schema_name = 'airtable' ORDER BY table_name, ordinal_position"
target/debug/coral sql "SELECT input_name, kind, has_default FROM coral.inputs WHERE source_name = 'airtable' ORDER BY input_name"
```

Pick one `base_id`, then:

```sh
target/debug/coral sql "SELECT id, name, primary_field_id FROM airtable.base_tables WHERE base_id = 'YOUR_BASE_ID' ORDER BY name"
```

Pick one `table_id`, then:

```sh
target/debug/coral sql "SELECT id, name, fields, views FROM airtable.base_tables WHERE base_id = 'YOUR_BASE_ID' AND id = 'YOUR_TABLE_ID'"
target/debug/coral sql "SELECT record_id, created_time, fields FROM airtable.records WHERE base_id = 'YOUR_BASE_ID' AND table_id = 'YOUR_TABLE_ID' LIMIT 10"
```

Optional filtered record checks:

```sh
target/debug/coral sql "SELECT record_id, fields FROM airtable.records WHERE base_id = 'YOUR_BASE_ID' AND table_id = 'YOUR_TABLE_ID' AND view = 'YOUR_VIEW_ID' LIMIT 10"
target/debug/coral sql "SELECT record_id, fields FROM airtable.records WHERE base_id = 'YOUR_BASE_ID' AND table_id = 'YOUR_TABLE_ID' AND formula = \"{Status} = 'Open'\" LIMIT 10"
```

## Demo recording flow

For a simple demo, use this order:

1. `coral source add airtable`
2. `coral source test airtable`
3. query `airtable.bases`
4. query `airtable.base_tables` with one base ID
5. show `fields` and `views` JSON for one table row
6. query `airtable.records` with that base ID and table ID
7. rerun `airtable.records` with a `view` or `formula` filter

That sequence shows setup, metadata discovery, and generic record access
without pretending that Airtable's custom fields are fixed SQL columns.
