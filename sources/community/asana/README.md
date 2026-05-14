# Asana source

Query workspaces, projects, sections, tasks, task stories, and users from Asana using a Personal Access Token.

## Setup

Create a Personal Access Token in the [Asana developer console](https://app.asana.com/-/developer_console), then add it to Coral:

```sh
export ASANA_ACCESS_TOKEN="your_asana_pat"
coral source add asana
```

See the manifest for auth details and optional configuration.

## Rate limits

Asana enforces per-token rate limits based on domain type:
- Free domains: 150 requests per minute
- Paid domains: 1500 requests per minute

Asana also enforces concurrent request limits (50 for GET, 15 for mutations) and
cost-based limits for expensive graph traversals. See the
[Asana rate limits documentation](https://developers.asana.com/docs/rate-limits)
for details.

## Tables

| Table | Notes |
|---|---|
| `workspaces` | Discover `workspace_gid` values visible to the token |
| `projects` | List projects in one workspace; requires `workspace_gid` |
| `sections` | List sections in one project; requires `project_gid` |
| `project_tasks` | List tasks in one project; requires `project_gid` |
| `task_details` | Full details for one task; requires `task_gid` |
| `task_stories` | Comments and activity for one task; requires `task_gid` |
| `users` | List users in a workspace; requires `workspace_gid` |

## Table notes

- `task_details` and `task_stories` are lookup-style tables for a specific `task_gid`.
- `project_tasks` is the main task table. Use it to discover task IDs, then query
  `task_details` or `task_stories` for more data on one task.
- JSON columns (`custom_fields`, `memberships`, `followers`, `projects`, `tags`,
  `workspaces`, `raw`) preserve account-specific schemas and prevent breaking
  changes when the API evolves.
- Timestamps are ISO 8601 strings. Date-only fields (`due_on`, `start_on`) are
  YYYY-MM-DD strings.

## Example queries

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

## Development

To lint and test the source in the Coral repo:

```sh
cargo run --locked -p coral-cli -- source lint ./sources/community/asana/manifest.yaml
cargo run --locked -p coral-cli -- source test asana
```
