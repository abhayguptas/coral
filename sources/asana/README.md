# Asana source

This bundled source lets Coral query read-only Asana data with a Personal
Access Token. It is designed for common work-management questions such as:

- what workspaces can this token see
- what projects are active in a workspace
- what tasks are in a project
- what comments and activity exist on a task

## Authentication

Create a Personal Access Token in the Asana developer console, then export it:

```sh
export ASANA_ACCESS_TOKEN="your_asana_pat"
```

You can override the API base URL if needed, but the default is correct for
normal Asana cloud usage:

```sh
export ASANA_API_BASE="https://app.asana.com/api/1.0"
```

## Quick start

```sh
coral source add asana
coral source test asana
coral sql "SELECT table_name FROM coral.tables WHERE schema_name = 'asana' ORDER BY table_name"
```

If you update the token later, run `coral source add asana` again so Coral
refreshes the stored credentials.

## Inspect the installed shape

After adding the source, it is useful to inspect the shape Coral sees:

```sql
SELECT table_name
FROM coral.tables
WHERE schema_name = 'asana'
ORDER BY table_name;
```

```sql
SELECT table_name, column_name, data_type, is_nullable
FROM coral.columns
WHERE schema_name = 'asana'
ORDER BY table_name, ordinal_position;
```

```sql
SELECT input_name, kind, has_default
FROM coral.inputs
WHERE source_name = 'asana'
ORDER BY input_name;
```

This is especially useful when validating required filters and nested JSON
columns such as `custom_fields`, `memberships`, and `projects`.

## Tables

| Table | Notes |
|---|---|
| `workspaces` | Discover `workspace_gid` values visible to the token |
| `projects` | List projects in one workspace; requires `workspace_gid` |
| `sections` | List sections in one project; requires `project_gid` |
| `project_tasks` | List tasks in one project; requires `project_gid` |
| `task_details` | Full details for one task; requires `task_gid` |
| `task_stories` | Comments and activity for one task; requires `task_gid` |
| `users` | List users in one workspace; requires `workspace_gid` |

## How to query it

Start by discovering a workspace:

```sql
SELECT gid, name, is_organization
FROM asana.workspaces
ORDER BY name;
```

List projects in that workspace:

```sql
SELECT gid, name, archived, owner_name, team_name
FROM asana.projects
WHERE workspace_gid = '1200000000000001'
ORDER BY name;
```

List sections in one project:

```sql
SELECT gid, name, created_at
FROM asana.sections
WHERE project_gid = '1200000000001001'
ORDER BY created_at;
```

List recent project tasks:

```sql
SELECT gid, name, assignee_name, completed, due_on, modified_at
FROM asana.project_tasks
WHERE project_gid = '1200000000001001'
ORDER BY modified_at DESC
LIMIT 25;
```

Fetch one task in detail:

```sql
SELECT gid, name, notes, assignee_name, due_on, custom_fields
FROM asana.task_details
WHERE task_gid = '1200000000002001';
```

Inspect comments and activity for one task:

```sql
SELECT created_at, created_by_name, resource_subtype, text
FROM asana.task_stories
WHERE task_gid = '1200000000002001'
ORDER BY created_at DESC;
```

Map workspace users for assignee lookups:

```sql
SELECT gid, name, email
FROM asana.users
WHERE workspace_gid = '1200000000000001'
ORDER BY name;
```

## Table behavior notes

- `task_details` and `task_stories` are lookup-style tables. They are meant for
  one known `task_gid` at a time.
- `project_tasks` is the main task listing table in v1. Use it first to
  discover task IDs, then drill into `task_details` or `task_stories`.
- `memberships`, `custom_fields`, `followers`, `projects`, `tags`, `workspaces`,
  and `raw` stay as JSON so the source stays stable even when account-specific
  schemas differ.
- Asana returns timestamps as ISO 8601 values. Date-only fields such as
  `due_on` and `start_on` stay as strings in `YYYY-MM-DD` format.

## Testing instructions

Run these commands from the repo root after building the CLI:

```sh
cargo run --locked -p coral-cli -- source lint ./sources/asana/manifest.yaml
make lint-sources
make docs-generate
make docs-check
```

For a clean live test, use a temporary Coral config directory:

```sh
export CORAL_CONFIG_DIR="$PWD/.tmp/asana-demo-config"
mkdir -p "$CORAL_CONFIG_DIR"
target/debug/coral source add asana
target/debug/coral source test asana
```

Then verify the main flow manually:

```sh
target/debug/coral sql "SELECT gid, name FROM asana.workspaces ORDER BY name"
target/debug/coral sql "SELECT table_name FROM coral.tables WHERE schema_name = 'asana' ORDER BY table_name"
target/debug/coral sql "SELECT table_name, column_name, data_type FROM coral.columns WHERE schema_name = 'asana' ORDER BY table_name, ordinal_position"
target/debug/coral sql "SELECT input_name, kind, has_default FROM coral.inputs WHERE source_name = 'asana' ORDER BY input_name"
```

Use the `workspace_gid` returned by `asana.workspaces`. Do not assume an older
ID from another token or account will work.

Pick one `workspace_gid`, then:

```sh
target/debug/coral sql "SELECT gid, name, archived FROM asana.projects WHERE workspace_gid = 'YOUR_WORKSPACE_GID' ORDER BY name LIMIT 20"
```

Pick one `project_gid`, then:

```sh
target/debug/coral sql "SELECT gid, name, assignee_name, completed, modified_at FROM asana.project_tasks WHERE project_gid = 'YOUR_PROJECT_GID' ORDER BY modified_at DESC LIMIT 20"
target/debug/coral sql "SELECT gid, name, created_at FROM asana.sections WHERE project_gid = 'YOUR_PROJECT_GID' ORDER BY created_at"
```

Pick one `task_gid`, then:

```sh
target/debug/coral sql "SELECT gid, name, notes, due_on, custom_fields FROM asana.task_details WHERE task_gid = 'YOUR_TASK_GID'"
target/debug/coral sql "SELECT created_at, created_by_name, resource_subtype, text FROM asana.task_stories WHERE task_gid = 'YOUR_TASK_GID' ORDER BY created_at DESC"
```

## Demo recording flow

For a simple demo, use this order:

1. `coral source add asana`
2. `coral source test asana`
3. query `asana.workspaces`
4. query `asana.projects` with one workspace ID
5. query `asana.sections` with one project ID
6. query `asana.project_tasks` with the same project ID
7. query `asana.task_details` with one task ID
8. query `asana.task_stories` with the same task ID

That sequence shows setup, validation, discovery, and drill-down without
requiring any unsupported join-driven filter behavior.
