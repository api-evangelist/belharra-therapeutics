---
name: Search Belharra Therapeutics content and resolve the hits
description: >-
  Use cross-content search to find material across posts, pages and the three custom post types
  without knowing where it lives, then map each hit's subtype to a rest_base and resolve it to the
  full object.
api: openapi/belharra-therapeutics-content-openapi.yml
base_url: https://belharratx.com/wp-json
auth: none
operations:
  - search
  - listPostTypes
  - listTaxonomies
  - getPost
  - getPage
  - getPressRelease
  - getCompanyNewsItem
  - getMultimediaFile
generated: '2026-08-06'
method: generated
---

# Search Belharra Therapeutics content and resolve the hits

When you do not know which of Belharra's five content collections holds what you want, search
across all of them at once. This is the right entry point for questions like "what has Belharra
said about Sanofi" or "where is the Searchlight platform described" — the answer may be a page, a
press release, a blog post or a coverage item, and searching one collection at a time will miss it.

**Send no credentials.** Base URL is `https://belharratx.com/wp-json`.

## Step 1 — search

```
GET /wp/v2/search?search=Sanofi&per_page=100
```

`search` returns **lightweight** records only:

```json
{"id": 1234, "title": "…", "url": "https://belharratx.com/…", "type": "post", "subtype": "press-release"}
```

There is no content, no date and no excerpt. Search is a router, not a reader — it tells you where
a thing lives so you can go fetch it.

74 objects were searchable at harvest time. Read `X-WP-Total` for the live count.

## Step 2 — narrow with subtype

```
GET /wp/v2/search?search=chemoproteomics&subtype=press-release,company-news
```

`subtype` accepts `post`, `page`, `company-news`, `multimedia-file`, `press-release`, or `any`
(the default). `type` accepts `post`, `term` or `post-format` and defaults to `post` — the object
class, not the post type. Leave `type` alone unless you specifically want taxonomy terms.

## Step 3 — resolve subtype to a route

Do not hardcode the mapping. Ask:

```
GET /wp/v2/types
```

`listPostTypes` returns each registered type with its `rest_base`. Build `subtype → rest_base`
from that response, then fetch:

```
GET /wp/v2/{rest_base}/{id}
```

On this deployment all five subtypes happen to have a `rest_base` identical to the subtype string,
so `press-release` resolves to `/wp/v2/press-release/{id}`. **That is a property of this
installation, not a rule** — WordPress lets `rest_base` differ from the type name, and a plugin
update can change it. Resolving through `/wp/v2/types` costs one cached request and makes the skill
survive that.

The resolvers are `getPost`, `getPage`, `getPressRelease`, `getCompanyNewsItem` and
`getMultimediaFile`.

## Step 4 — or just use the url

Every search hit carries a `url` — the public permalink. If you only need the human page, use it
directly and skip the resolve step entirely. Use the API resolve when you need structured fields:
`date`, `modified`, `featured_media`, `categories` or the `acf` block.

## Shortcut — skip search when you know the collection

Every content collection accepts the same `search` parameter directly, and returns **full**
objects rather than search stubs:

```
GET /wp/v2/press-release?search=Genentech&per_page=100
GET /wp/v2/pages?search=Searchlight&_fields=id,slug,link,title
```

This is one request instead of two and gives you the whole record. Use `/wp/v2/search` only when
the collection is genuinely unknown. On `/wp/v2/posts` and `/wp/v2/pages` you can also scope the
match with `search_columns`.

## Taxonomy context

```
GET /wp/v2/taxonomies
```

`listTaxonomies` returns `category`, `post_tag`, `nav_menu` and `wp_pattern_category`. Only
`category` carries terms — two of them, Company News and Press Releases. **None of Belharra's three
custom post types declares a taxonomy at all**, so there is no category-based way to slice the
press releases or the coverage archive. Search and date windows are the only filters that work
across the whole surface.

## Practical notes

- Search is unpaginated in spirit but paginated in fact — pass `per_page` up to 100 and read
  `X-WP-Total`. Over 100 returns HTTP 400 `rest_invalid_param`.
- `title` on a search hit is a **plain string**; on a resolved object it is
  `{"rendered": "..."}` with HTML entities escaped. Two different shapes for the same concept —
  handle both.
- Errors are `{code, message, data:{status}}`, not RFC 9457. Branch on `code`.
- Responses are edge-cached 600 seconds and carry `x-robots-tag: noindex`.
