---
name: Track Belharra Therapeutics announcements across all five content collections
description: >-
  Read Belharra Therapeutics' full published record — press releases, third-party coverage and blog
  posts — from the three separate collections they actually live in, with correct pagination and
  incremental sync. Prevents the common failure of querying only /wp/v2/posts and missing most of
  the archive.
api: openapi/belharra-therapeutics-content-openapi.yml
base_url: https://belharratx.com/wp-json
auth: none
operations:
  - listPostTypes
  - listCategories
  - listPosts
  - getPost
  - listPressReleases
  - getPressRelease
  - listCompanyNews
  - getCompanyNewsItem
generated: '2026-08-06'
method: generated
---

# Track Belharra Therapeutics announcements

Belharra Therapeutics is a chemoproteomics drug-discovery company in San Diego. It runs no
developer program. This skill reads the public WordPress REST content API behind
`https://belharratx.com/wp-json`. **Send no credentials** — none are required and none will be
accepted.

## The thing that will trip you up

Belharra's published material is spread across **three separate collections**, not one. An agent
that queries `/wp/v2/posts` alone sees 10 records and misses 25 more:

| Collection | Route | Items at harvest | What it holds |
|---|---|---|---|
| Posts | `/wp/v2/posts` | 10 | The blog, categorised Company News or Press Releases |
| Press releases | `/wp/v2/press-release` | 4 | Belharra's own announcements |
| Company news | `/wp/v2/company-news` | 21 | Third-party press coverage |

The `posts` collection **overlaps** the other two — the same Sanofi announcement appears both as a
post and as a press release. Deduplicate on the rendered `title` or on the `slug`, not on `id`:
the three collections share one wp_posts ID sequence, so IDs never collide, but the same story
carries a different ID in each collection.

## Step 1 — confirm the collections still exist

Types can be added or removed by a plugin update at any time. Do not hardcode.

```
GET /wp/v2/types
```

`listPostTypes` returns the registered types keyed by name. Look for `press-release`,
`company-news` and `multimedia-file`, and read each one's `rest_base` — that is the path segment to
use. At harvest time the type name and the `rest_base` were identical for all three, but confirm
rather than assume.

## Step 2 — read the category vocabulary (posts only)

```
GET /wp/v2/categories?per_page=100
```

`listCategories` returned exactly two terms: **Company News** (id `1`, 7 posts) and **Press
Releases** (id `7`, 3 posts). The `post_tag` taxonomy is registered but empty, so `listTags`
returns `[]` — do not build a tag filter.

Note that **none of the three custom post types declares a taxonomy**. Category filtering works on
`/wp/v2/posts` only. On `press-release` and `company-news` your only filters are date window,
`slug`, `menu_order` and `search`.

## Step 3 — page each collection

```
GET /wp/v2/press-release?per_page=100&orderby=date&order=desc
GET /wp/v2/company-news?per_page=100&orderby=date&order=desc
GET /wp/v2/posts?per_page=100&orderby=date&order=desc
```

- `per_page` maxes at **100**. Asking for more returns HTTP 400 `rest_invalid_param` — the API
  rejects, it does not clamp.
- Read `X-WP-Total` and `X-WP-TotalPages` from the response headers to size the job. Both are
  exposed cross-origin via `Access-Control-Expose-Headers`.
- Follow the RFC 8288 `Link` header's `rel="next"` rather than incrementing `page` blindly.
- Every collection fits in one page of 100 at current volumes, but do not rely on that.

Trim the payload — every record inlines rendered HTML `content` and a full `acf` block:

```
GET /wp/v2/press-release?per_page=100&_fields=id,slug,date,modified,link,title
```

## Step 4 — filter by category on posts only

```
GET /wp/v2/posts?categories=7&per_page=100
```

Use `categories_exclude` for the inverse, and `tax_relation=AND|OR` when combining. Again: this
parameter does not exist on the custom post types.

## Step 5 — incremental sync

Re-run with a modification window rather than re-reading everything:

```
GET /wp/v2/press-release?modified_after=2026-01-01T00:00:00&orderby=modified&order=desc
```

`after` / `before` filter on publish date; `modified_after` / `modified_before` filter on last
edit. For change detection use the **modified** pair — Belharra back-edits records long after
publication (the January 2023 debut press release carries a December 2024 `modified` timestamp).

**Calibrate your expectations for freshness.** At harvest the newest press release was
2025-01-10 and the newest post was 2024-06-18, but corporate pages were being modified as recently
as 2026-06-26. The site is maintained; the announcement stream is simply quiet. An empty
incremental poll is the normal result, not a failure.

## Step 6 — fetch one record

```
GET /wp/v2/press-release/{id}
GET /wp/v2/company-news/{id}
GET /wp/v2/posts/{id}
```

`getPressRelease`, `getCompanyNewsItem` and `getPost`. An ID from one collection returns **404
`rest_post_invalid_id`** on another collection's route even though it is a valid ID somewhere on
the site — the shared ID sequence makes this a common and confusing bug. Always pair an ID with the
collection you read it from.

## What is in a record

Core posts carry `excerpt`, `author`, `categories`, `tags`, `comment_status` and `sticky`. The
three custom post types carry **none** of those — they are flat, and add `menu_order` instead.
All of them carry:

- `title.rendered` and `content.rendered` — HTML-escaped markup, not plain text. Unescape before use.
- `featured_media` — a media ID, or `0` when unset. Resolve via `/wp/v2/media/{id}`, or add
  `_embed` to inline it under `_embedded['wp:featuredmedia']`.
- `acf` — an Advanced Custom Fields object present on every record. On `company-news` this is where
  the **outbound link to the third-party article** lives. Its field definitions are behind the
  401-gated admin, so inspect the object rather than assuming a shape.

## Errors

Errors are the WordPress envelope, **not** RFC 9457 problem+json:

```json
{"code":"rest_post_invalid_id","message":"Invalid post ID.","data":{"status":404}}
```

Branch on `code`, never on `message`. See
[`errors/belharra-therapeutics-problem-types.yml`](../errors/belharra-therapeutics-problem-types.yml).

## Etiquette

No `RateLimit-*` headers exist, so there is no signal to back off against. `robots.txt` advertises
`Crawl-delay: 10` — honour it. Responses are edge-cached for 600 seconds, so polling faster than
that returns the same bytes. This is a small company's corporate site, not a metered API; a full
re-read is under 40 requests.
