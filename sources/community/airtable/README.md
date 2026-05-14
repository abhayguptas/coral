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
SELECT key, kind, required, is_set, default_value
FROM coral.inputs
WHERE schema_name = 'airtable'
ORDER BY key;
```

This is especially useful for Airtable because the source is intentionally
JSON-first for dynamic schema content.

## Rate Limits

Airtable API enforces rate limits (typically 30 requests/second). This source
uses cursor-based pagination with a default page size of 100, which helps
respect these limits. If you encounter rate limit errors, consider reducing
query frequency or adding delays between requests when performing batch
operations.

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

## Validation

If you are developing this source in the Coral repo, run:

```sh
cargo run --locked -p coral-cli -- source lint ./sources/community/airtable/manifest.yaml
make lint-sources
make docs-generate
make docs-check
```

Then add and validate the source:

```sh
coral source add airtable
coral source test airtable
```

Inspect the installed shape:

```sh
coral sql "SELECT table_name FROM coral.tables WHERE schema_name = 'airtable' ORDER BY table_name"
coral sql "SELECT table_name, column_name, data_type FROM coral.columns WHERE schema_name = 'airtable' ORDER BY table_name, ordinal_position"
coral sql "SELECT key, kind, required, is_set, default_value FROM coral.inputs WHERE schema_name = 'airtable' ORDER BY key"
```

Then verify the data flow with a few real queries. Start by listing bases:

```sh
coral sql "SELECT id, name, permission_level FROM airtable.bases ORDER BY name"
```

Use one `base_id`, then:

```sh
coral sql "SELECT id, name, primary_field_id FROM airtable.base_tables WHERE base_id = 'YOUR_BASE_ID' ORDER BY name"
```

Use one `table_id`, then:

```sh
coral sql "SELECT id, name, fields, views FROM airtable.base_tables WHERE base_id = 'YOUR_BASE_ID' AND id = 'YOUR_TABLE_ID'"
coral sql "SELECT record_id, created_time, fields FROM airtable.records WHERE base_id = 'YOUR_BASE_ID' AND table_id = 'YOUR_TABLE_ID' LIMIT 10"
```

Optional filtered record checks:

```sh
coral sql "SELECT record_id, fields FROM airtable.records WHERE base_id = 'YOUR_BASE_ID' AND table_id = 'YOUR_TABLE_ID' AND view = 'YOUR_VIEW_ID' LIMIT 10"
coral sql "SELECT record_id, fields FROM airtable.records WHERE base_id = 'YOUR_BASE_ID' AND table_id = 'YOUR_TABLE_ID' AND formula = '{Status} = ''Open''' LIMIT 10"
```
