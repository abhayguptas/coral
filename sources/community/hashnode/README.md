# Hashnode source

Query public Hashnode blog data through Coral.

This source uses Hashnode's public GraphQL API at `https://gql.hashnode.com`.
It is read-only and does not require a Hashnode token.

## What this source covers

The source exposes five tables:

- `hashnode.users`: public profile details for one username
- `hashnode.publications`: publication metadata for one host
- `hashnode.publication_posts`: posts from one publication, with cursor pagination
- `hashnode.publication_post`: full content for one post by host and slug
- `hashnode.static_pages`: static page content by host and slug

Private authenticated surfaces such as `me`, drafts, mutations, and publication
administration are intentionally out of scope for this community source.

## Add the source

Because this is a community source, add it from the manifest file:

```bash
cargo run --locked -p coral-cli -- source add --file ./sources/community/hashnode/manifest.yaml
```

To update an existing local install after editing the manifest, run the same
command again.

## Validate the source

From the repo root:

```bash
cargo run --locked -p coral-cli -- source lint ./sources/community/hashnode/manifest.yaml
make lint-sources
make docs-check
```

Then install and run the source tests:

```bash
cargo run --locked -p coral-cli -- source add --file ./sources/community/hashnode/manifest.yaml
cargo run --locked -p coral-cli -- source test hashnode
```

## Inspect the installed shape

List the tables:

```bash
cargo run --locked -p coral-cli -- sql \
  "SELECT table_name FROM coral.tables WHERE schema_name = 'hashnode' ORDER BY table_name"
```

Inspect columns and required filters:

```bash
cargo run --locked -p coral-cli -- sql \
  "SELECT table_name, column_name, type_name, is_required_filter \
   FROM coral.columns \
   WHERE schema_name = 'hashnode' \
   ORDER BY table_name, ordinal_position"
```

## Table behavior

### `hashnode.users`

Fetches one public user profile.

Required filter:

- `username`

**Flattened columns:**
- `bio__text`, `bio__markdown`, `bio__html` - User bio in various formats
- `social_media_links__*` - Individual social platform links (website, github, twitter, instagram, facebook, stackoverflow, linkedin, youtube)

### `hashnode.publications`

Fetches one publication by host.

Required filter:

- `host`

Example host values are custom domains or Hashnode subdomains, such as
`blog.developerdao.com`.

**Flattened columns:**
- `about__text`, `about__markdown`, `about__html` - Publication about section in various formats
- `author__*` - Publication author details (id, username, name, profile_picture)
- `preferences__*` - Publication preferences (logo, dark_mode_enabled, navbar_items)

### `hashnode.publication_posts`

Lists posts from one publication.

Required filter:

- `host`

This table uses Hashnode's GraphQL `posts(first, after)` connection and Coral
`cursor_body` pagination.

**Flattened columns:**
- `cover_image__url` - Post cover image URL
- `author__*` - Post author details (id, username, name, profile_picture)
- `tags__*` - Tag details flattened from array (id, name, slug)

### `hashnode.publication_post`

Fetches full content for one post.

Required filters:

- `host`
- `slug`

**Flattened columns:**
- `content__markdown`, `content__html`, `content__text` - Post content in various formats
- `cover_image__url` - Post cover image URL
- `author__*` - Post author details (id, username, name, profile_picture)
- `tags__*` - Tag details flattened from array (id, name, slug)

**Note:** The `slug_response` column contains the current post slug from the API response, distinct from the `slug` filter used for the lookup.

### `hashnode.static_pages`

Fetches one static page from a publication.

Required filters:

- `host`
- `slug`

**Flattened columns:**
- `content__markdown`, `content__html`, `content__text` - Page content in various formats

**Note:** The `slug_response` column contains the current page slug from the API response, distinct from the `slug` filter used for the lookup.

## Example queries

Fetch publication metadata:

```sql
SELECT id, title, display_title, url, author__name
FROM hashnode.publications
WHERE host = 'blog.developerdao.com';
```

List recent posts with author details:

```sql
SELECT title, slug, url, published_at, read_time_minutes, author__name
FROM hashnode.publication_posts
WHERE host = 'blog.developerdao.com'
ORDER BY published_at DESC
LIMIT 10;
```

Fetch full post content with markdown:

```sql
SELECT title, content__markdown, author__name
FROM hashnode.publication_post
WHERE host = 'blog.developerdao.com'
  AND slug = 'the-developers-guide-to-chainlink-vrf-foundry-edition';
```

Fetch a public user profile with social links:

```sql
SELECT username, name, tagline, followers_count,
       social_media_links__github, social_media_links__twitter
FROM hashnode.users
WHERE username = 'Favourite';
```

Fetch a static page:

```sql
SELECT title, content__markdown
FROM hashnode.static_pages
WHERE host = 'blog.greenroots.info'
  AND slug = 'about';
```

## Notes

### Column naming and flattening

All nested objects in the Hashnode API response are flattened using double-underscore (`__`) notation for consistency and query simplicity. For example:
- `author.id` → `author__id`
- `bio.markdown` → `bio__markdown`
- `preferences.darkMode.enabled` → `preferences__dark_mode_enabled`

This standardization is applied across all tables, ensuring a uniform query experience.

### API details

Hashnode's API is GraphQL-only. Coral does not need a native GraphQL backend
for this source because the current HTTP backend supports POST requests with
JSON body templates.

Hashnode recommends requesting `id` fields to avoid stale cached GraphQL data,
so every query in this source requests IDs for top-level objects and nested
objects where practical.

### Endpoint validation

Hashnode endpoint validation may vary by network or region. Source endpoint
follows Hashnode's documented public GraphQL API at the time of implementation.
