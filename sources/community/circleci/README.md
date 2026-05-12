# CircleCI source

This bundled source lets Coral query core CircleCI API v2 data with a token.

The first version is intentionally pipeline-centric and read-only. It focuses
on four useful surfaces:

- the current user
- recent pipelines for one organization
- workflows inside one pipeline
- jobs inside one workflow

## Authentication

Create a CircleCI API token, then export it:

```sh
export CIRCLECI_TOKEN="your_circleci_token"
```

You can override the API base if needed, but the default is correct for normal
CircleCI cloud usage:

```sh
export CIRCLECI_API_BASE="https://circleci.com/api/v2"
```

CircleCI uses the token in the `Circle-Token` request header.

## Quick start

```sh
coral source add circleci
coral source test circleci
coral sql "SELECT table_name FROM coral.tables WHERE schema_name = 'circleci' ORDER BY table_name"
```

If you update the token later, run `coral source add circleci` again so Coral
refreshes the stored credentials.

## Inspect the installed shape

After adding the source, inspect what Coral sees:

```sql
SELECT table_name
FROM coral.tables
WHERE schema_name = 'circleci'
ORDER BY table_name;
```

```sql
SELECT table_name, column_name, data_type, is_nullable
FROM coral.columns
WHERE schema_name = 'circleci'
ORDER BY table_name, ordinal_position;
```

```sql
SELECT key, kind, required, is_set, default_value
FROM coral.inputs
WHERE schema_name = 'circleci'
ORDER BY key;
```

This is useful for confirming required filters and seeing which nested CircleCI
payloads stay as JSON.

## Tables

| Table | Notes |
|---|---|
| `me` | Current user for the configured token |
| `pipelines` | Recent pipelines for one organization; requires `org_slug` |
| `pipeline_workflows` | Workflows inside one pipeline; requires `pipeline_id` |
| `workflow_jobs` | Jobs inside one workflow; requires `workflow_id` |

## About `org_slug`

`pipelines` requires an `org_slug` because CircleCI's list-pipelines endpoint is
organization-scoped.

For GitHub OAuth organizations, the slug typically looks like:

- `gh/my-org`

For GitHub App or GitLab-backed projects, CircleCI documents alternate slug
formats using `circleci` plus provider IDs. Use the slug format shown in your
CircleCI organization settings.

This source does **not** expose full project inventory in v1. It is built
around recent pipeline discovery.

## How to query it

Check the current user:

```sql
SELECT id, login, name, avatar_url
FROM circleci.me;
```

List recent pipelines for one organization:

```sql
SELECT id, number, state, created_at
FROM circleci.pipelines
WHERE org_slug = 'gh/my-org'
ORDER BY created_at DESC
LIMIT 20;
```

Inspect workflows for one pipeline:

```sql
SELECT id, name, status, started_at, stopped_at
FROM circleci.pipeline_workflows
WHERE pipeline_id = 'YOUR_PIPELINE_ID'
ORDER BY started_at DESC
LIMIT 20;
```

Inspect jobs for one workflow:

```sql
SELECT job_number, name, status, type, started_at, stopped_at
FROM circleci.workflow_jobs
WHERE workflow_id = 'YOUR_WORKFLOW_ID'
ORDER BY started_at DESC
LIMIT 20;
```

## Table behavior notes

- `pipeline_workflows` is a lookup-style table. It requires one `pipeline_id`.
- `workflow_jobs` is a lookup-style table. It requires one `workflow_id`.
- `trigger`, `vcs`, `errors`, `dependencies`, `requires`, and `raw` remain
  JSON so the source stays stable across different CircleCI accounts.
- This source intentionally does not expose reruns, cancellations, artifacts,
  tests, contexts, or insights in v1.

## Validation

If you are developing this source in the Coral repo, run:

```sh
cargo run --locked -p coral-cli -- source lint ./sources/circleci/manifest.yaml
make lint-sources
make docs-generate
make docs-check
```

Then add and validate the source:

```sh
cargo run --locked -p coral-cli -- source add circleci
cargo run --locked -p coral-cli -- source test circleci
```

Inspect the installed shape:

```sh
cargo run --locked -p coral-cli -- sql "SELECT table_name FROM coral.tables WHERE schema_name = 'circleci' ORDER BY table_name"
cargo run --locked -p coral-cli -- sql "SELECT table_name, column_name, data_type FROM coral.columns WHERE schema_name = 'circleci' ORDER BY table_name, ordinal_position"
cargo run --locked -p coral-cli -- sql "SELECT key, kind, required, is_set, default_value FROM coral.inputs WHERE schema_name = 'circleci' ORDER BY key"
```

Then verify the data flow with a few real queries:

```sh
cargo run --locked -p coral-cli -- sql "SELECT id, login, name FROM circleci.me"
```

Use one `org_slug`, then:

```sh
cargo run --locked -p coral-cli -- sql "SELECT id, number, state, created_at FROM circleci.pipelines WHERE org_slug = 'YOUR_ORG_SLUG' ORDER BY created_at DESC LIMIT 20"
```

Use one `pipeline_id`, then:

```sh
cargo run --locked -p coral-cli -- sql "SELECT id, name, status, started_at, stopped_at FROM circleci.pipeline_workflows WHERE pipeline_id = 'YOUR_PIPELINE_ID' ORDER BY started_at DESC LIMIT 20"
```

Use one `workflow_id`, then:

```sh
cargo run --locked -p coral-cli -- sql "SELECT job_number, name, status, type, started_at, stopped_at FROM circleci.workflow_jobs WHERE workflow_id = 'YOUR_WORKFLOW_ID' ORDER BY started_at DESC LIMIT 20"
```
