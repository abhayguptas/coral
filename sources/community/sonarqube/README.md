# SonarQube source

This community source lets Coral query projects, issues, quality gates, security hotspots,
and code metrics from SonarQube Server or SonarQube Cloud.

The v1 source is focused on engineering intelligence: projects, code quality metrics,
security analysis, and release-readiness evaluation.

## Authentication

Create a bearer token in SonarQube under:

- **SonarQube Server**: User > My Account > Security > Tokens
- **SonarQube Cloud**: Organization > Members > Tokens

Token must have:
- 'Scan' scope for analysis-related data
- 'Read on projects' scope for project metadata

Then export:

```sh
export SONARQUBE_BASE_URL="https://sonarqube.company.com"  # or https://sonarcloud.io
export SONARQUBE_TOKEN="your_bearer_token"
```

For self-hosted SonarQube, use your instance URL. For SonarCloud, keep `https://sonarcloud.io`.

## Quick start

```sh
coral source add sonarqube
coral source test sonarqube
coral sql "SELECT table_name FROM coral.tables WHERE schema_name = 'sonarqube' ORDER BY table_name"
```

If you update the token later, run `coral source add sonarqube` again to refresh credentials.

## Rate Limits

SonarQube API rate limits vary based on plan and instance configuration:

- **SonarQube Server**: Typically configurable; check your instance settings
- **SonarQube Cloud**:
  - Community: 10 requests/second
  - Professional: 20 requests/second
  - Enterprise: Contact SonarSource

This source uses explicit pagination to respect limits. If you encounter rate limit errors,
consider reducing query frequency or filtering by project scope.

## Tables

| Table | Notes |
|---|---|
| `current_user` | Validates token and returns authenticated user info |
| `projects` | SonarQube projects accessible to the token |
| `issues` | Code issues (requires project_key filter) |
| `security_hotspots` | Security hotspots requiring review (requires project_key filter) |
| `measures` | Project quality metrics (requires project_key and metric_keys filters) |
| `quality_gates` | Configured quality gate definitions |
| `project_quality_gate_status` | Quality gate evaluation results for projects (requires project_key filter) |

## How to query it

List all accessible projects:

```sql
SELECT project_key, project_name, last_analysis_date
FROM sonarqube.projects
ORDER BY last_analysis_date DESC
LIMIT 20;
```

Find high-severity issues in a project:

```sql
SELECT issue_key, severity, type, message, component, line
FROM sonarqube.issues
WHERE project_key = 'my-backend-api'
  AND severity IN ('BLOCKER', 'CRITICAL')
ORDER BY severity DESC, creation_date DESC
LIMIT 20;
```

List unreviewed security hotspots:

```sql
SELECT hotspot_key, component, vulnerability_probability, message
FROM sonarqube.security_hotspots
WHERE project_key = 'my-backend-api'
  AND status = 'TO_REVIEW'
ORDER BY creation_date DESC
LIMIT 20;
```

Check project quality metrics:

```sql
SELECT metric_key, value
FROM sonarqube.measures
WHERE project_key = 'my-backend-api'
  AND metric_keys = 'coverage,bugs,vulnerabilities,sqale_index';
```

Monitor quality gate status for release gating:

```sql
SELECT project_key, status, last_analysis_date
FROM sonarqube.project_quality_gate_status
WHERE project_key = 'my-backend-api'
  AND status = 'ERROR';
```

## Common metrics

Common metric keys for the `measures` table:

- **Coverage**: `coverage` (overall), `line_coverage`, `branch_coverage`
- **Quality**: `bugs`, `code_smells`, `vulnerabilities`, `reliability_issues`
- **Ratings**: `reliability_rating`, `security_rating`, `maintainability_rating`
- **Duplication**: `duplicated_lines_density`
- **Debt**: `sqale_index` (technical debt), `security_hotspots`

Comma-separate multiple metrics: `coverage,bugs,vulnerabilities`.

See [SonarQube metrics documentation](https://docs.sonarsource.com/sonarqube/latest/user-guide/reports/metrics/)
for the complete list and definitions.

## Issue and hotspot statuses

**Issue statuses**: OPEN, CONFIRMED, RESOLVED, REOPENED, CLOSED

**Hotspot statuses**: TO_REVIEW, ACKNOWLEDGED, FIXED, SAFE

**Hotspot probability**: HIGH, MEDIUM, LOW

## Validation

If you are developing this source in the Coral repo, run:

```sh
cargo run --locked -p coral-cli -- source lint ./sources/community/sonarqube/manifest.yaml
```

Then add and validate the source:

```sh
coral source add sonarqube
coral source test sonarqube
```

Inspect the installed shape:

```sh
coral sql "SELECT table_name FROM coral.tables WHERE schema_name = 'sonarqube' ORDER BY table_name"
coral sql "SELECT table_name, column_name, data_type FROM coral.columns WHERE schema_name = 'sonarqube' ORDER BY table_name, ordinal_position"
coral sql "SELECT key, kind, required, is_set FROM coral.inputs WHERE schema_name = 'sonarqube' ORDER BY key"
```

Then verify data flow with representative queries:

```sh
coral sql "SELECT * FROM sonarqube.current_user LIMIT 1"
coral sql "SELECT * FROM sonarqube.projects LIMIT 1"
coral sql "SELECT project_key FROM sonarqube.projects LIMIT 1" | xargs -I {} \
  coral sql "SELECT * FROM sonarqube.issues WHERE project_key = '{}' LIMIT 1"
```
